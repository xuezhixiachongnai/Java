# RocketMQ

#### 核心组件

##### Producer

生产者，发送消息到 Broker。支持同步/异步/单向，三种发送模式

并且内置失败重试，保证消息尽可能送达

##### Consumer

消息消费者，从 Broker 拉取消息并处理

支持两种消费模式，由 Broker Push（主动推送）或者是 消费者 Pull（拉取），实际内部统一为长轮询拉去

##### Broker

消息存储与转发中心。存储消息，维护消费队列，接收 Producer 发来的消息推送给 Consumer

Broker 支持主从架构，可以实现高可用和主从分离。Broker 的 Master 主要负责读写数据，Slave 节点负责拷贝 Master 节点的数据做保存，防止主节点挂掉导致数据丢失

##### NameServer

路由中心，保存 Topic 到 Broker 的映射关系

它是轻量级、无状态的。可以水平扩展。NameServer 可以形成集群，但每一个 NameServer 都保存着同样的数据

Broker 启动时会主动向其注册相关信息。Producer/Consumer 会向 NameServer 查询路由信息

这样做实现了路由和储存的解耦，避免 Broker 既存消息又做路由导致单点问题。

NameServer 不参与消息传输，避免性能瓶颈。

它类似于 DNS 的作用，让客户端能快速找到 Broker。

##### Topic

主题，消息的逻辑分类，Producer 按 Topic 发送，Consumer 按 Topic 订阅

每一个主题下会有多个队列，不同队列分布在不同的 Borker中。Producer 会把消息路由到某个队列

##### Message Queue

消息队列，Topic 下的分区，Producer 写入时选择队列，Consumer 消费时分配队列

这样设计便于负载均衡和扩展

##### Tag

附属于 Topic 中的二级分类，一个 Topic 可以包含多个 Tag，例如

```sh
Topic: OrderTopic
 ├─ TagA：订单创建
 ├─ TagB：订单支付
 ├─ TagC：订单取消
```

Tag 用在 Consumer 在订阅消息时对消息过滤。它可以指定 Tag，只接收某些 Tag 消息

Tag 可以实现逻辑分类，在同一个 Topic 中可以对消息做进一步分类，方便管理消息

