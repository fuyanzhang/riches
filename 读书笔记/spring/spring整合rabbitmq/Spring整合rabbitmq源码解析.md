### rabbitmq相关标签
spring-rabbit中标签有：

```
1. rabbit:connection-factory:创建connection-factory。
2. rabbit:template：创建org.springframework.amqp.rabbit.core.RabbitTemplate实例，用来方便的访问broker。
3. rabbit:admin：创建org.springframework.amqp.rabbit.core.RabbitAdmin实例，用来管理exchanges，queues和bindings。
4. rabbit:queue：
5. rabbit:queue-arguments
6. rabbit:direct-exchange
7. rabbit:topic-exchange
8. rabbit:fanout-exchange
9. rabbit:headers-exchange
10.rabbit:exchange-arguments
11.rabbit:binding-arguments
12.rabbit:annotation-driven
13.rabbit:listener-container
```

标签定义在spring-rabbit-xxx.jar包中。handler为`org.springframework.amqp.rabbit.config.RabbitNamespaceHandler`
