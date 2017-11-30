## elastic-job 调度系统源码阅读 ##

### 初始化流程图 ###
![](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/task%E5%AD%A6%E4%B9%A0/elastic-job/pic/job%E5%88%9D%E5%A7%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

### zk目录 ###

### 代码详解（以spring的方式为例） ###

系统在启动时，加载job相关的spring标签 `reg:zookeeper:注册中心相关配置`和`job:simple:任务相关配置`。
spring在解析自定义标签后，初始化bean，这就调到了JobScheduler的init方法。代码如下：
```
  public void init() {
        LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
        JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
        JobScheduleController jobScheduleController = new JobScheduleController(
                createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
        JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
        schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
        jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
    }
```
init主要做了如下几件事情：

1. 持久化job配置。zk上的路径为${root}/jobname/config [${root}]为配置job时指定。
2. 设置任务分片总数。任务分别总数在任务配置时指定，为必传参数。
3. 创建任务调度控制器。主要是创建quartz的调度器和quartz的jobDetail。此处可以看出elastic-job是基于quartz实现的。
 ```
  result.put("org.quartz.threadPool.class", org.quartz.simpl.SimpleThreadPool.class.getName());
        result.put("org.quartz.threadPool.threadCount", "1");
        result.put("org.quartz.scheduler.instanceName", liteJobConfig.getJobName());
        result.put("org.quartz.jobStore.misfireThreshold", "1");
        result.put("org.quartz.plugin.shutdownhook.class", JobShutdownHookPlugin.class.getName());
        result.put("org.quartz.plugin.shutdownhook.cleanShutdown", Boolean.TRUE.toString());
 ```
调度器配置信息如上代码。可以看出elastic-job是用的是内存方式的调度
4. 注册任务。主要是往注册中心类中添加调度器并构造TreeCache缓存。
5. 准备任务启动的相关信息。主要是各种服务的启动【占大头的】。代码如下：
```
public void registerStartUpInfo(final boolean enabled) {
        listenerManager.startAllListeners();   // 1:开启所有监听器
        leaderService.electLeader();    // 2:选取主节点
        serverService.persistOnline(enabled); //3:持久化作业服务器信息 
        instanceService.persistOnline();    //4:持久化作业运行实例
        shardingService.setReshardingFlag();// 5:设置需要重新分片的标记
        monitorService.listen();            //6:初始化作业监听服务
        if (!reconcileService.isRunning()) {
            reconcileService.startAsync();    // 7:调解分布式作业不一致状态服务
        }
    }
```

6. 开始调度任务。从代码上看，elastic-job不支持SimpleTrigger。代码如下：
```
public void scheduleJob(final String cron) {
        try {
            if (!scheduler.checkExists(jobDetail.getKey())) {
                scheduler.scheduleJob(jobDetail, createTrigger(cron));
            }
            scheduler.start();
        } catch (final SchedulerException ex) {
            throw new JobSystemException(ex);
        }
    }
```

#### 详解registerStartUpInfo方法 ####
该方法共干了7件事。一一走读代码。
##### 开启所有监听 #####
代码如下:
```
 /**
     * 开启所有监听器.
     */
    public void startAllListeners() {
        electionListenerManager.start();   //主节点选举监听管理器.
        shardingListenerManager.start();   //分片监听管理器.
        failoverListenerManager.start();   //失效转移监听管理器.
        monitorExecutionListenerManager.start();//幂等性监听管理器.
        shutdownListenerManager.start();   //运行实例关闭监听管理器.
        triggerListenerManager.start();    //作业触发监听管理器.
        rescheduleListenerManager.start(); //重调度监听管理器.
        guaranteeListenerManager.start();  //保证分布式任务全部开始和结束状态监听管理器.
        jobNodeStorage.addConnectionStateListener(regCenterConnectionStateListener); //注册连接状态监听器.
    }
```

1.主节点选举监听
 
	主节点选举监听器添加了2个监听:LeaderElectionJobListener和LeaderAbdicationJobListener
		LeaderElectionJobListener主要监听/jobname/servers/${ip} 节点是否有变化。若有变化且没有主节点，则开始选举主节点[选主节点所用的锁目录为/jobname/leader/election/latch]。主节点在zk上是一个临时节点。节点为/jobname/leader/election/instance，值为任务实例ID。
		LeaderAbdicationJobListener监听/jobname/servers/${ip}。如果服务器状态时disable的，那么就删除主节点，重新选取。

2.分片监听管理器

	该监听器有两个监听;ShardingTotalCountChangedJobListener和ListenServersChangedJobListener
		ShardingTotalCountChangedJobListener 主要监控分片总数是否改变。监控的路径为/jobname/config,若分片总数改变且原分片不为0，则设置需要重新分片。标志位路径为/jobname/leader/sharding/necessary
		ListenServersChangedJobListener 主要监控作业实例和作业服务器是否有变化，若有变化，同样需要重新分片。作业实例和作业服务器的节点分别为/jobname/instances和/jobname/servers/${ip}

3.失效转移监听管理器
	
	该管理器也同样有2个监听：JobCrashedJobListener和FailoverSettingsChangedJobListener
		JobCrashedJobListener主要是做任务失效转移。任务分片数据存放路径/jobname/sharding/${item}/failover下,value值为该分片被分片的任务实例。该任务宕机后，将该任务所属的分片信息放入/jobname/leader/failover/items/${item}路径下，方法是failoverService.setCrashedFailoverFlag(each);。然后选取一个节点来执行该分片【failoverService.failoverIfNecessary()】。failoverService.failoverIfNecessary()主要逻辑是获取分布式锁，获取一个item分片，填充/jobname/sharding/${item}/failover路径数据,value为执行分片的任务实例ID。删除/jobname/leader/failover/items/${item}路径。然后获取调度器进行任务的执行。
		FailoverSettingsChangedJobListener主要是监听/jobname/config中的isFailover是否有变化，如果变为FALSE，即关闭失效转移功能，则删除失效转移信息。即删除/jobname/sharding/${item}/failover节点。

4.幂等性监听管理器

	该管理器的监听器为MonitorExecutionSettingsChangedJobListener，主要是监控/jobname/config中任务信息MonitorExecution是否有更新，若更新为不监控分片执行状态，则删除记录任务执行状态的节点，删除路径为/jobname/sharding/${item}/running路径。

5.运行实例关闭监听管理器

	该监听管理器管理InstanceShutdownStatusJobListener监听。监听/jobname/instances/instanceid是否被remove，若被remove，则将内存中相关的信息删除。

6.作业触发监听管理器

7.重调度监听管理器

	CronSettingAndJobEventChangedJobListener 修改了任务的配置信息时，即【/jobname/config】下的内容时，触发该监听。当发现是任务周期发生变化时，则需要重新调度。其他的修改不重新调度。【在crm系统中，修改了任务的调度周期，可能会出现任务多执行的情况。】

8.保证分布式任务全部开始和结束状态监听管理器
	
9.regCenterConnectionStateListener
	该监听器主要是监听节点与zk的链接状态，若连接异常，则将任务暂停，若连接恢复，则恢复任务。
	```
		 public void stateChanged(final CuratorFramework client, final ConnectionState newState) {
        if (JobRegistry.getInstance().isShutdown(jobName)) {
            return;
        }
        JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
        if (ConnectionState.SUSPENDED == newState || ConnectionState.LOST == newState) {
            jobScheduleController.pauseJob();
        } else if (ConnectionState.RECONNECTED == newState) {
            serverService.persistOnline(serverService.isEnableServer(JobRegistry.getInstance().getJobInstance(jobName).getIp()));
            instanceService.persistOnline();
            executionService.clearRunningInfo(shardingService.getLocalShardingItems());
            jobScheduleController.resumeJob();
        }
    }
	```


##### 选取主节点 #####

选取主节点的逻辑很简单，获取分布式锁/jobname/leader/election/latch,然后创建临时节点/jobname/leader/election/instance，值为任务实例ID。

##### 持久化作业服务器信息 #####

该逻辑也很简单，主要是注册任务的执行机信息。在zk上的路径为/jobname/servers/${ip} ,值为是否被禁用。若未禁用，则是“”，若为disable，则入DISABLED

##### 持久化作业运行实例 #####

该过程主要是注册任务实例。在zk上的路径为/jobname/instances/${instanceid},是一个临时节点。值为“”。

##### 设置需要重新分片的标记 #####

创建分片标志路径，路径为：/jobname/leader/sharding/necessary

##### 初始化作业监听服务 #####

主要用于dump zk上的数据。

##### 调解分布式作业不一致状态服务 #####




### job执行过程（以LiteJob为例） ###

lite模式下，创建quartz的jobDetail使用的job为LiteJob，LiteJob实现了quartz的Job接口。

执行作业过程如下
```
 public final void execute() {
        try {
			//环境检查，注册中心与本地的时间误差是否在容忍范围内
            jobFacade.checkJobExecutionEnvironment();
        } catch (final JobExecutionEnvironmentException cause) {
            jobExceptionHandler.handleException(jobName, cause);
        }
		//获取分片信息
        ShardingContexts shardingContexts = jobFacade.getShardingContexts();
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_STAGING, String.format("Job '%s' execute begin.", jobName));
        }
		
		//设置任务misfired的标志。（判断当前的任务执行前，任务分片是否有running状态，有，则置本次任务为misfired。）
		//判断路径/jobname/sharding/${item}/running节点是否存在，存在，则认为有任务在执行，设本次任务misfired，在
		//zk上添加节点/jobname/sharding/${item}/misfire
        if (jobFacade.misfireIfRunning(shardingContexts.getShardingItemParameters().keySet())) {
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format(
                        "Previous job '%s' - shardingItems '%s' is still running, misfired job will start after previous job completed.", jobName, 
                        shardingContexts.getShardingItemParameters().keySet()));
            }
            return;
        }
        try {
            jobFacade.beforeJobExecuted(shardingContexts);
            //CHECKSTYLE:OFF
        } catch (final Throwable cause) {
            //CHECKSTYLE:ON
            jobExceptionHandler.handleException(jobName, cause);
        }
		//执行任务
        execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
		// 任务执行完之后，查看在任务执行过程中是否有任务没有被执行，若有，则执行之
        while (jobFacade.isExecuteMisfired(shardingContexts.getShardingItemParameters().keySet())) {
            jobFacade.clearMisfire(shardingContexts.getShardingItemParameters().keySet());
            execute(shardingContexts, JobExecutionEvent.ExecutionSource.MISFIRE);
        }
        jobFacade.failoverIfNecessary();
        try {
            jobFacade.afterJobExecuted(shardingContexts);
            //CHECKSTYLE:OFF
        } catch (final Throwable cause) {
            //CHECKSTYLE:ON
            jobExceptionHandler.handleException(jobName, cause);
        }
    }
```


具体的任务执行代码如下：

```
  private void execute(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {
		//如果分片为空，直接返回
        if (shardingContexts.getShardingItemParameters().isEmpty()) {
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format("Sharding item for job '%s' is empty.", jobName));
            }
            return;
        }
		//注册作业启动信息。修改注册中心的任务状态为running，在zk上创建临时节点/jobname/sharding/${item}/running，表示该分片正在执行
        jobFacade.registerJobBegin(shardingContexts);
        String taskId = shardingContexts.getTaskId();
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(taskId, State.TASK_RUNNING, "");
        }
        try {

			//真正的开始执行任务
            process(shardingContexts, executionSource);
        } finally {
            // TODO 考虑增加作业失败的状态，并且考虑如何处理作业失败的整体回路
            jobFacade.registerJobCompleted(shardingContexts);
            if (itemErrorMessages.isEmpty()) {
                if (shardingContexts.isAllowSendJobEvent()) {
                    jobFacade.postJobStatusTraceEvent(taskId, State.TASK_FINISHED, "");
                }
            } else {
                if (shardingContexts.isAllowSendJobEvent()) {
                    jobFacade.postJobStatusTraceEvent(taskId, State.TASK_ERROR, itemErrorMessages.toString());
                }
            }
        }
    }
```

执行任务的代码如下：

```
 private void process(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {
        Collection<Integer> items = shardingContexts.getShardingItemParameters().keySet();
		//若果是一个分片，则同步执行
        if (1 == items.size()) {
            int item = shardingContexts.getShardingItemParameters().keySet().iterator().next();
            JobExecutionEvent jobExecutionEvent =  new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, item);
            process(shardingContexts, item, jobExecutionEvent);
            return;
        }
        final CountDownLatch latch = new CountDownLatch(items.size());
		//若是大于1个分片，则放入线程池中异步执行，直到所有的分片都执行完之后，算该任务在该实例上执行完【=1和>1的场景都一样啊，为啥不同样处理？？？？】
        for (final int each : items) {
            final JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, each);
            if (executorService.isShutdown()) {
                return;
            }
            executorService.submit(new Runnable() {
                
                @Override
                public void run() {
                    try {
                        process(shardingContexts, each, jobExecutionEvent);
                    } finally {
                        latch.countDown();
                    }
                }
            });
        }
        try {
            latch.await();
        } catch (final InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
```

该段代码中的process方法 process(shardingContexts, each, jobExecutionEvent)是一个抽象方法，不同的job有不同的实现，最后都会调到ElasticJob的实现类的execute方法。该方法是真正的业务逻辑实现。






