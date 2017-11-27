## Quartz 调度流程 ##
![quartz任务调度流程](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/task%E5%AD%A6%E4%B9%A0/quartz/quartz%E8%B0%83%E5%BA%A6.png)


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
MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY=-1
MISFIRE_INSTRUCTION_SMART_POLICY=0
MISFIRE_INSTRUCTION_FIRE_ONCE_NOW=1
MISFIRE_INSTRUCTION_DO_NOTHING=2

```
- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY 
	当策略为-1时，会将之前misfired的所有job trigger一遍，直到当前任务执行周期。
- MISFIRE_INSTRUCTION_SMART_POLICY MISFIRE_INSTRUCTION_FIRE_ONCE_NOW
	CronTrigger的默认策略是0，与1策略一样，资源准备完成后执行一次
- MISFIRE_INSTRUCTION_DO_NOTHING
	比较复杂一点
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