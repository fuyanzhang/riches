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
		LeaderElectionJobListener主要监听/jobname/servers/${ip} 节点是否有变化。若有变化且没有主节点，则开始选举主节点[选主节点所用的锁目录为/jobname/leader/election/latch]。主节点在zk上是一个临时节点。节点为/jobname/leader/election/instance，值为ip。
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

	该监听管理器管理InstanceShutdownStatusJobListener监听。







