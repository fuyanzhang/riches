## elastic-job 调度系统源码阅读##

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
        listenerManager.startAllListeners();
        leaderService.electLeader();
        serverService.persistOnline(enabled);
        instanceService.persistOnline();
        shardingService.setReshardingFlag();
        monitorService.listen();
        if (!reconcileService.isRunning()) {
            reconcileService.startAsync();
        }
    }
```

