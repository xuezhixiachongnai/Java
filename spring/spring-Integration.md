# spring-Integration

**Spring Integration** 框架基于 Spring 提供了一种实现企业集成模式的方式。用于不同企业应用实现消息传递，简化企业应用集成的复杂性

组成 Spring Integration 的三大组件是 **channel**、**filter** 和 **message**

```sh
		message			message			 message		message
channel--------->handler--------->channel-------->handler--------->

```

- Message：即消息本身，它由 Payload 和 Header 两部分组成，Payload 是对任意 Java 对象的包装而 Header 则包含了消息的元数据信息，同时 header 也常用于 Http、Mail 等其他消息头部的转换。其基接口为 Message<T>。

- Message Channel：它是消息传输的载体，通常可以分为 point-to-point（点对点）和 publish-subscribe（发布订阅）两种行为模式。此外从通道是否保存消息的角度来说，通道还分为 Pollable Channel 和 Subscribable Channel 两种。
  - Pollable Channel：保存消息，消费者需要主动拉取消息，核心接口为 PollableChannel。
  - Subscribable Channel：可订阅型通道，不存储消息，消费者被动通知消息，核心接口为 SubscribableChannel。这种划分方式也是API接口的划分方式，不同的通道类型对消息流程的处理会有不同的表现形式。

- Message Endpoint：即 Handler，它是消息的消费端，通常与外部系统对接。Spring Integration 提供了多种不同的 EndPoint 满足不同的需求。

### 消息通道（Message Channel）

**Spring Integration** 提供了多种通道的实现，包括DirectChannel（点对点）、PublishSubscribeChannel（发布订阅）、QueueChannel（队列）、PriorityChannel（优先级队列）等。消息通道主要用于传递消息。

### 消息端点（Message Endpoints）

消息端点是消息处理的核心组件，包括转换器（Transformer）、过滤器（Filter）、路由器（Router）、分割器（Splitter）和聚合器（Aggregator）

看一个配置 

```java
// 配置通道类
@Configuration
public class ChannelConfig {

    @Bean
    public MessageChannel inputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel middleChannel() {
        return new DirectChannel();
    }
}
```

```java
@EnableIntegration
@MessageEndpoint
public class HandlerConfig {

    // 声明一个数据源，轮询器 poller 以固定时间将数据发送到绑定的 channel 上
    @InboundChannelAdapter(channel = "inputChannel", poller = @Poller(fixedRate = "500"))
    public Message<String> input() {
        return new Message<>() {
            @Override
            public String getPayload() {
                return "我永远喜欢雪之下雪乃";
            }

            @Override
            public MessageHeaders getHeaders() {
                return new MessageHeaders(Map.of());
            }
        };
    }

    @Transformer(inputChannel = "inputChannel", outputChannel = "middleChannel")
    public Message<?> middleTransformer(Message<?> message) {
        return new GenericMessage<>(
                message.getPayload(),
                Map.of("transformer", "middleTransformer")
        );
    }

    // 用于定义最终处理器
    @ServiceActivator(inputChannel = "middleChannel")
    public void handler(Message<String> message) {
        System.out.println(message.getHeaders().get("transformer"));
        System.out.println(message.getPayload());
    }
}
```

配置处理器

消息会通过 Channel 传递，Handler 会不同的规则处理消息

`@InboundChannelAdapter` 注解是用来定义消息源

`@ServiceActivator` 注解是用来定义定义最终消息处理器 

### Spring Integration MQTT

是一种基于发布订阅模式的消息协议，基于 TCP/IP 协议。一般多用于物联网，广泛应用于工业级别的应用场景

配置依赖

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
    <version>6.2.3</version> <!-- 版本和你的 spring-integration 对齐 -->
</dependency>
```

Spring Integration 通过 `MqttPahoClientFactory` 连接 MQTT Broker

```java
@Bean
public MqttPahoClientFactory mqttClientFactory() {
    DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
    MqttConnectOptions options = new MqttConnectOptions();
    options.setServerURIs(new String[]{"tcp://127.0.0.1:1883"});
    options.setUserName("username");
    options.setPassword("password".toCharArray());
    factory.setConnectionOptions(options);
    return factory;
}
```

配置 Channel 

```java
@Bean
public MessageChannel mqttInputChannel() {
    return new DirectChannel();
}

@Bean
public MessageChannel mqttOutboundChannel() {
    return new DirectChannel();
}
```

订阅消息

让 Spring Integration 监听某个 Topic，收到消息后转发到 Channel

```java
@Bean
public MessageProducer inbound(MqttPahoClientFactory mqttClientFactory) {
    MqttPahoMessageDrivenChannelAdapter adapter =
            new MqttPahoMessageDrivenChannelAdapter(
                    "mqttClientSub", mqttClientFactory, "test/topic");
    adapter.setCompletionTimeout(5000);
    adapter.setConverter(new DefaultPahoMessageConverter());
    adapter.setQos(1);
    adapter.setOutputChannel(mqttInputChannel()); // 将从 Broker 订阅的消息发送到 mqttInputChannel 通道
    return adapter;
}

@ServiceActivator(inputChannel = "mqttInputChannel")
public void handleMessage(String payload) {
    System.out.println("收到消息: " + payload);
}
```

> 这里的 `adapter` 就是一个 **入站网关**，把 MQTT broker 的消息转成 Spring Integration Message，然后放进 `mqttInputChannel`

发布消息

定义一个 **出站网关**，把 Spring Integration 的消息发到 MQTT broker

```java
@Bean
@ServiceActivator(inputChannel = "mqttOutboundChannel")
public MessageHandler outbound(MqttPahoClientFactory mqttClientFactory) {
    MqttPahoMessageHandler messageHandler =
            new MqttPahoMessageHandler("mqttClientPub", mqttClientFactory);
    messageHandler.setAsync(true);  // 异步发送
    messageHandler.setDefaultTopic("test/topic");
    return messageHandler;
}
```

发布消息时只需要把消息发送到 `mqttOutboundChannel` 通道即可

[](https://blog.csdn.net/weixin_55344375/article/details/146569230?ops_request_misc=elastic_search_misc&request_id=1ef3fd4b95370adcf4d0710abfce52ed&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-146569230-null-null.142^v102^pc_search_result_base3&utm_term=Spring%20Integration&spm=1018.2226.3001.4187)

[](https://www.tony-bro.com/posts/1578338213/index.html)