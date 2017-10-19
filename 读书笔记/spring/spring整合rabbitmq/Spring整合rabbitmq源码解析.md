### rabbitmq相关标签 ###

spring-rabbit中标签有：

```
1. rabbit:connection-factory:创建connection-factory。
2. rabbit:template：创建org.springframework.amqp.rabbit.core.RabbitTemplate实例，用来方便的访问broker。
3. rabbit:admin：创建org.springframework.amqp.rabbit.core.RabbitAdmin实例，用来管理exchanges，queues和bindings。
4. rabbit:queue：rabbit队列，用于消费者接受消息。若broker中有相同名称的queue，则使用该queue，若无，则新建。
5. rabbit:queue-arguments：queue的附加信息。
6. rabbit:direct-exchange：创建direct类型的exchange。
7. rabbit:topic-exchange：创建topic类型的exchange。
8. rabbit:fanout-exchange：创建fanout类型的exchange。
9. rabbit:headers-exchange：创建handers类型的exchange。
10.rabbit:exchange-arguments：exchange附加参数。
11.rabbit:binding-arguments：exchange的绑定参数。
12.rabbit:annotation-driven：启动通过注解@RabbitListener 方式声明Listener
13.rabbit:listener-container：创建listenercontainer实例。默认为SimpleMessageListenerContainer。
```

标签定义在spring-rabbit-xxx.jar包中。handler为`org.springframework.amqp.rabbit.config.RabbitNamespaceHandler`


### queue概述 ###


### exchange概述 ###

### 消息发送流程 ###

### 消息消费流程 ###
