## Spring事件机制理解 ##

### 一个小栗子 ###

1. 事件的定义

```
public class TestEvent extends ApplicationEvent{
    public TestEvent(Object source) {
        super(source);
        System.out.println("trigger TestEvent !!!");
    }
```
事件定义需要继承spring的ApplicationEvent。

2. 发布者

```
public class TestPublisher implements  ApplicationEventPublisherAware {

    	private ApplicationEventPublisher publisher ;

	    @Override
	    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
	        this.publisher = applicationEventPublisher;
	    }
	
	    public void publish(){
	        TestEvent event = new TestEvent("hello world");
	        publisher.publishEvent(event);
	        System.out.println("publisher thread id is "+Thread.currentThread().getName());
	    }
	}

```

发布者实现ApplicationEventPublisherAware，同时也可以实现ApplicationContextAware，二者效果是一样的，最终都是调用ApplicationEventPublisher的publishEvent方法。
![](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/spring/spring%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6/pic/ApplicationContext%20%E7%B1%BB%E5%9B%BE.png)
从类图上看到，ApplicationContext同样继承自ApplicationEventPublisher。
代码如下：
```
public class TestPublisher1 implements ApplicationContextAware {

    private ApplicationContext context;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.context = applicationContext;
    }

    public void publish(){
        TestEvent event = new TestEvent("hello world");
        context.publishEvent(event);
    }
}
```
3. 监听者

```
public class TestEventHandler implements ApplicationListener<TestEvent> {
    @Override
    public void onApplicationEvent(TestEvent testEvent) {
        System.out.println("receive event , the data is "+testEvent.getSource());
        System.out.println("receiver thread id is "+ Thread.currentThread().getName());
    }
}

```

监听者实现ApplicationListener，泛型加入事件类型。其中如何处理且听下节分享。

### 源码阅读 ###

监听注册阶段
监听器注册是在spring初始化的时候，监听器也必须声明为一个普通的bean。
```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}
```
以上是bean初始化过程代码。

我们从发布事件的地方作为入口来阅读，即ApplicationEventPublisher.publishEvent();
其中ApplicationEventPublisher是一个接口，其实publishEvent方法的实现在AbstractApplicationContext中。
AbstractApplicationContext类图： ![](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/spring/spring%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6/pic/AbstractApplicationContext.png)

