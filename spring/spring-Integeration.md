# spring-Integration

## 引言

**Spring Integration**框架基于Spring提供了一种实现企业集成模式的方式。用于不同企业应用实现消息传递，简化企业应用集成的复杂性

## 一、核心概念

 **Spring Integration**的设计基于消息传递架构，其核心是消息（Message）、通道（Channel）和处理器（Handler）。消息作为数据传输的基本单位。包含有效载荷（payload）和消息头（header）。通道负责连接消息生产者和消费者，可以是点对点（direct）或者发布/订阅（publish-subscribe）模式。处理器负责处理消息，执行业务逻辑。

## 二、消息通道（Message Channel）

**Spring Integration**提供了多种通道的实现，包括DirectChannel（点对点）、PublishSubscribeChannel（发布订阅）、QueueChannel（队列）、PriorityChannel（优先级队列）等。消息通道主要用于传递消息。

## 三、消息端点（Message Endpoints）

消息端点是消息处理的核心组件，包括转换器（Transformer）、过滤器（Filter）、路由器（Router）、分割器（Splitter）和聚合器（Aggregator）

`https://blog.csdn.net/weixin_55344375/article/details/146569230?ops_request_misc=elastic_search_misc&request_id=1ef3fd4b95370adcf4d0710abfce52ed&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-146569230-null-null.142^v102^pc_search_result_base3&utm_term=Spring%20Integration&spm=1018.2226.3001.4187`