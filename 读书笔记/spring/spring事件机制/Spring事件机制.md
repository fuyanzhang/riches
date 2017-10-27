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

				// 【注册监听】
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}
```


以上是bean初始化过程代码。
打开registerListeners()代码如下：


```
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}
		//【获取实现了接口ApplicationListener的bean，将其加入到事件广播器里】
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```

到此，事件监听的注册就宣布完成。

接下来我们看如何publish一个事件并处理该事件的。

我们从发布事件的地方作为入口来阅读，即ApplicationEventPublisher.publishEvent();
其中ApplicationEventPublisher是一个接口，其实publishEvent方法的实现在AbstractApplicationContext中。
AbstractApplicationContext类图： ![](https://github.com/fuyanzhang/riches/blob/master/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/spring/spring%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6/pic/AbstractApplicationContext.png)


来详细看下AbstractApplicationContext的publishEvent方法。

```
protected void publishEvent(Object event, ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<Object>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			//【这个地方就是处理事件的地方】
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```
广播器的广播方法如下:

```
@Override
	public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```
上面的代码功能是获取满足条件的监听器，循环的调用invokeListener方法。
上面方法可以看出，事件的处理有同步和异步两种方式。真正处理的地方是invokeListener方法。
getApplicationListeners主要功能是通过event的source和eventtype（可为null，上面例子为null）找到合适的一组监听器。

invokeListener代码如下：
```
protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				listener.onApplicationEvent(event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			try {
				listener.onApplicationEvent(event);
			}
			catch (ClassCastException ex) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				LogFactory.getLog(getClass()).debug("Non-matching event type for listener: " + listener, ex);
			}
		}
	}
```

直接调用我们监听器实现的onApplicationEvent方法。

以上就是我们最常用的监听器的功能，当然还有其他的功能这里没有做描述。一切都在代码里。