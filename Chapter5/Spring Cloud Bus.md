# Spring Cloud Bus(消息总线)

## RabbitMQ基础概念

- Broker: 可以理解为消息队列服务器的实体， 它是一个中间件应用， 负责接收消息生产者的消息， 然后将消息发送至消息接收者或者其他的Broker。
- Exchange: 消息交换机，是消息第一个到达的地方，消息通过它指定的路由规则，分发到不同的消息队列中去。
- Queue: 消息队列， 消息通过发送和路由之后最终到达的地方， 到达Queue的消息即进入逻辑上等待消费的状态。 每个消息都会被发送到一个或多个队列 。
- Binding: 绑定，它的作用就是把Exchange和Queue按照路由规则绑定起来， 也就是Exchange和Queue之间的虚拟连接 。
- Routing Key: 路由关键字，Exchange根据这个关键字进行消息投递。
- Virtual host: 虚拟主机， 它是对Broker的虚拟划分， 将消费者、 生产者和它们依赖的AMQP相关结构进行隔离，一般都是为了安全考虚。比如，我们可以在一个Broker中设置多个虚拟主机， 对不同用户进行权限的分离 。
- Connection: 连接， 代表生产者、 消费者、 Broker之间进行通信的物理网络。
- Channel: 消息通道，用千连接生产者和消费者的逻辑结构 。在客户端的每个连接里，可建立多个Channel, 每个Channel代表一个会话任务， 通过Channel可以隔离同一连接中的不同交互内容。
- Producer: 消息生产者， 制造消息并发送消息的程序。
- Consumer: 消息消费者， 接收消息并处理消息的程序

![01](.\images\bus-01.png)

1. 在ConfigServer中也引入SpringCloud Bus, 将配置服务端也加入到消息总线中来。
2. /actuator/bus-refresh请求不再发送到具体服务实例上， 而是发送给Config Server, 并通过destination参数来指定需要更新配置的服务或实例。通过上面的改动，我们的服务实例不需要再承担触发配置更新的职责。 同时，对于Git的触发等配置都只需要针对ConfigServer即可， 从而简化了集群上的 一 些维护工作。

