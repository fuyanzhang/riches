## Quartz 调度流程 ##
![quartz任务调度流程](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/task%E5%AD%A6%E4%B9%A0/quartz/quartz%E8%B0%83%E5%BA%A6.png)

## Quartz 恢复机制 ##
调度器在start的时候，做了两件事情。
1、启动集群的健康性检查线程。
2、启动集群的misfired处理线程。
具体代码如下：
```
public void schedulerStarted() throws SchedulerException {

        if (isClustered()) {
            clusterManagementThread = new ClusterManager();
            if(initializersLoader != null)
                clusterManagementThread.setContextClassLoader(initializersLoader);
            clusterManagementThread.initialize();
        } else {
            try {
                recoverJobs();
            } catch (SchedulerException se) {
                throw new SchedulerConfigException(
                        "Failure occured during job recovery.", se);
            }
        }

        misfireHandler = new MisfireHandler();
        if(initializersLoader != null)
            misfireHandler.setContextClassLoader(initializersLoader);
        misfireHandler.initialize();
        schedulerRunning = true;
        
        getLog().debug("JobStore background threads started (as scheduler was started).");
    }
```



## Quartz misfired机制 ##

每种trigger都有自己独特的misfired处理机制。

quartz的trigger有如下几种：
DailyTimeInterValTriggerImpl,CronTriggerImpl,SimpleTriggerImpl,CalendarIntervalTriggerImpl.其中最常用的是CronTriggerImpl,SimpleTriggerImpl两种。下面就最常见的两种trigger的misfired处理机制做个说明。
misfired 主要是修改下次执行时间。不同的策略有不同的nextfireTime计算方式。

1. CronTriggerImpl

	上代码：
	```
		    public void updateAfterMisfire(org.quartz.Calendar cal) {
        int instr = getMisfireInstruction();

        if(instr == Trigger.MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY)
            return;

        if (instr == MISFIRE_INSTRUCTION_SMART_POLICY) {
            instr = MISFIRE_INSTRUCTION_FIRE_ONCE_NOW;
        }

        if (instr == MISFIRE_INSTRUCTION_DO_NOTHING) {
            Date newFireTime = getFireTimeAfter(new Date());
            while (newFireTime != null && cal != null
                    && !cal.isTimeIncluded(newFireTime.getTime())) {
                newFireTime = getFireTimeAfter(newFireTime);
            }
            setNextFireTime(newFireTime);
        } else if (instr == MISFIRE_INSTRUCTION_FIRE_ONCE_NOW) {
            setNextFireTime(new Date());
        }
    }
	```

当前策略只有四种，分别为：
*MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY=-1*

*MISFIRE_INSTRUCTION_SMART_POLICY=0*

*MISFIRE_INSTRUCTION_FIRE_ONCE_NOW=1*

*MISFIRE_INSTRUCTION_DO_NOTHING=2*

```
- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY 
	当策略为-1时，会将之前misfired的所有周期的job trigger一遍，直到当前任务执行周期。
	“select * from triggers where schedulername="" and state="?"and netx_fire_time <="?" and (misfire_instr=-1 or( misfire_instr != -1 and next_fire_time >=?))”这个sql中【misfire_instr=-1】是关键代码。
- MISFIRE_INSTRUCTION_SMART_POLICY MISFIRE_INSTRUCTION_FIRE_ONCE_NOW
	CronTrigger的默认策略是0，与1策略一样，资源准备完成后执行一次，即将nextfiretime设置成当前时间。等待有资源后调用。若一直没资源，那么会一直被misfired，直到有资源，立即被执行一次，然后就是正常的调用了。
- MISFIRE_INSTRUCTION_DO_NOTHING
	一坨复杂的计算，主要功能就是misfired之后，啥也不干，然后等待正常的调用。
```


2. SimpleTriggerImpl
上代码：
```
  public void updateAfterMisfire(Calendar cal) {
        int instr = getMisfireInstruction();
        
        if(instr == Trigger.MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY)
            return;
        
        if (instr == Trigger.MISFIRE_INSTRUCTION_SMART_POLICY) {
            if (getRepeatCount() == 0) {
                instr = MISFIRE_INSTRUCTION_FIRE_NOW;
            } else if (getRepeatCount() == REPEAT_INDEFINITELY) {
                instr = MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT;
            } else {
                // if (getRepeatCount() > 0)
                instr = MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT;
            }
        } else if (instr == MISFIRE_INSTRUCTION_FIRE_NOW && getRepeatCount() != 0) {
            instr = MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT;
        }

        if (instr == MISFIRE_INSTRUCTION_FIRE_NOW) {
            setNextFireTime(new Date());
        } else if (instr == MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT) {
            Date newFireTime = getFireTimeAfter(new Date());
            while (newFireTime != null && cal != null
                    && !cal.isTimeIncluded(newFireTime.getTime())) {
                newFireTime = getFireTimeAfter(newFireTime);

                if(newFireTime == null)
                    break;
                
                //avoid infinite loop
                java.util.Calendar c = java.util.Calendar.getInstance();
                c.setTime(newFireTime);
                if (c.get(java.util.Calendar.YEAR) > YEAR_TO_GIVEUP_SCHEDULING_AT) {
                    newFireTime = null;
                }
            }
            setNextFireTime(newFireTime);
        } else if (instr == MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT) {
            Date newFireTime = getFireTimeAfter(new Date());
            while (newFireTime != null && cal != null
                    && !cal.isTimeIncluded(newFireTime.getTime())) {
                newFireTime = getFireTimeAfter(newFireTime);

                if(newFireTime == null)
                    break;
                
                //avoid infinite loop
                java.util.Calendar c = java.util.Calendar.getInstance();
                c.setTime(newFireTime);
                if (c.get(java.util.Calendar.YEAR) > YEAR_TO_GIVEUP_SCHEDULING_AT) {
                    newFireTime = null;
                }
            }
            if (newFireTime != null) {
                int timesMissed = computeNumTimesFiredBetween(nextFireTime,
                        newFireTime);
                setTimesTriggered(getTimesTriggered() + timesMissed);
            }

            setNextFireTime(newFireTime);
        } else if (instr == MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT) {
            Date newFireTime = new Date();
            if (repeatCount != 0 && repeatCount != REPEAT_INDEFINITELY) {
                setRepeatCount(getRepeatCount() - getTimesTriggered());
                setTimesTriggered(0);
            }
            
            if (getEndTime() != null && getEndTime().before(newFireTime)) {
                setNextFireTime(null); // We are past the end time
            } else {
                setStartTime(newFireTime);
                setNextFireTime(newFireTime);
            } 
        } else if (instr == MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT) {
            Date newFireTime = new Date();

            int timesMissed = computeNumTimesFiredBetween(nextFireTime,
                    newFireTime);

            if (repeatCount != 0 && repeatCount != REPEAT_INDEFINITELY) {
                int remainingCount = getRepeatCount()
                        - (getTimesTriggered() + timesMissed);
                if (remainingCount <= 0) { 
                    remainingCount = 0;
                }
                setRepeatCount(remainingCount);
                setTimesTriggered(0);
            }

            if (getEndTime() != null && getEndTime().before(newFireTime)) {
                setNextFireTime(null); // We are past the end time
            } else {
                setStartTime(newFireTime);
                setNextFireTime(newFireTime);
            } 
        }

    }
```

simpleTrigger有7种策略。

*MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY=-1*

*MISFIRE_INSTRUCTION_SMART_POLICY=0*

*MISFIRE_INSTRUCTION_FIRE_NOW = 1*

*MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT = 2*

*MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT = 3*

*MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT = 4*

*MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT = 5*

```
- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY 
	 同cornTrigger的策略[获取到资源后，立即执行，将错过的任务执行。然后按照正常频率执行]
- MISFIRE_INSTRUCTION_SMART_POLICY
	若当前任务只执行一次，那么其策略选取MISFIRE_INSTRUCTION_FIRE_NOW
	若当前任务无限次执行，则其策略为MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
	其他场景任务策略为：MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
- MISFIRE_INSTRUCTION_FIRE_NOW
	立刻执行。eg：若任务定义为18:00执行，每10min执行一次，执行5次，那么正常情况下执行的时间点为[18:00,18:10,18:20,18:30,18:40],若在18:09分宕机，直到18:25才恢复，那么下次执行的点就是18:25，最终执行的时间点为：[18:00,18:25,18:35,18:45,18:55,19:05].此处执行了6次。从代码里也能看出，fire now策略只改变了触发时间，而没有改变重复次数。

- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
	当前时间点立即执行一次，然后以该时间点开始，IntervalInSeconds为时间间隔，依次执行。执行次数不变。eg：例子中的执行时间点为[18:00,18:25,18:35,18:45,18:55]

- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
	当前时间点立刻执行，以后以该时间点开始，依次执行。如果有错过的任务，则就丢弃，不再执行。

- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
	下一个周期点执行，之前的错过的任务就错过了，不再执行。eg：若任务定义为18:00执行，每10min执行一次，执行5次，那么正常情况下执行的时间点为[18:00,18:10,18:20,18:30,18:40],若在18:09分宕机，直到18:27才恢复，那么下次执行的点就是18:30，其中18:10和18:20两次任务就不再执行了，最终执行的时间点为：[18:00,18:30,18:40]

- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT
	保证任务执行次数不丢。计算下一个执行点开始执行。eg： 若任务定义为18:00执行，每10min执行一次，执行5次，那么正常情况下执行的时间点为[18:00,18:10,18:20,18:30,18:40],若在18:09分宕机，直到18:27才恢复，那么下次执行的点就是18:30。任务最终的执行时间点为：[18:00,18:30,18:40,18:50,19:00]

```






####
quartz不满足我们的场景：

1、没有运维平台。
2、我们有大量的一次性任务。quartz的simpleTrigger执行完之后，会删除已有的任务。
3、我们的任务需要根据机器的负载进行调度任务的执行机器。


