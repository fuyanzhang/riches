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