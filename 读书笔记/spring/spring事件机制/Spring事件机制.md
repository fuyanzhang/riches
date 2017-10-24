##Spring事件机制理解 ###

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