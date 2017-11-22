## 背景 ##
最近在写一个任务调度系统，里边用到了spring的task的功能。为了更深入的了解其中的原理，看了下里边的实现。作为笔记记录一下。
## 源码走读 ##

spring里有两种方式实现任务，一种是基于注解的方式，通过`@Scheduled`来配置任务，还有一种方式是通过spring的配置文件进行`task:scheduled-tasks`。

spring处理的入口如下`org.springframework.scheduling.config.TaskNamespaceHandler`:

```
 public void init() {
        this.registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
        this.registerBeanDefinitionParser("executor", new ExecutorBeanDefinitionParser());
        this.registerBeanDefinitionParser("scheduled-tasks", new ScheduledTasksBeanDefinitionParser());
        this.registerBeanDefinitionParser("scheduler", new SchedulerBeanDefinitionParser());
    }
```
依次处理4个标签，`task:annotation-driven`，`task:executor`，`task:scheduled-tasks`，`task:scheduler`。
分别从4个标签说开去:

