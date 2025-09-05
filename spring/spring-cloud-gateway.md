# SpringCloudGateWay



## 概念

网关是介于客户端和服务器端之间的中间层，所有的外部请求都会先经过网关这一层。也就是说，API 的实现方更多地只需要考虑业务逻辑，而安全、性能、监控可以交由网关来完成，这样既提高业务灵活性又不缺安全性。

网关的核心功能特性： 请求路由、权限控制、限流。

- 权限控制：网关作为微服务入口，需要校验用户是否有请求资格，如果没有则进行拦截。     

- 路由和负载均衡：一切请求都必须先经过gateway，但网关不处理业务，而是根据某种规则，把请求转发到某个微服务，这个过程叫做路由。当然路由的目标服务有多个时，还需要做负载均衡。     

- 限流：当请求流量过高时，在网关中按照下流的微服务能够接受的速度来放行请求，避免服务压力过大。     

在SpringCloud中网关的实现包括两种： gateway、zuul。Zuul是基于Servlet的实现，属于阻塞式编程。而SpringCloudGateway则是基于Spring5中提供的WebFlux，属于响应式编程的实现，具备更好的性能。

## 搭建gateway网关服务

引入依赖

```xml
<!--nacos服务注册发现依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
 
<!--网关gateway依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

路由配置包括：

1. 路由id：路由的唯一标示
2. 路由目标（uri）：路由的目标地址，http代表固定地址，lb代表根据服务名负载均衡
3. 路由断言（predicates）：判断路由的规则
4. 路由过滤器（filters）：对请求或响应做处理

```yaml
server:
  port: 10010
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos地址
    gateway:
      routes:
        - id: user-service 　　# 路由标示，必须唯一
          # uri: http://127.0.0.1:8081 # 路由的目标地址。也可以通过http的写成固定地址，但不推荐使用
          uri: lb://userservice # 路由的目标地址。lb就是负载均衡的意思，后面跟服务名称
          predicates: # 路由断言，判断请求是否符合规则
            - Path=/user/** # 路径断言，判断路径是否是以/user开头，如果是则符合
        - id: order-service
          uri: lb://orderservice
          predicates:
            - Path=/order/**
```

然后外部服务即可直接通过访问网关来访问微服务，而无需直接访问微服务的地址，如下通过直接访问 http://localhost:10010/order/101 即可访问到 order-service 的服务，网关会自动将匹配到的路由转发至对应的服务中，并且会自动实现负载均衡。

## 路由断言工厂

我们在配置文件中写的断言规则只是字符串，这些字符串会被 Predicate Factory（断言工厂） 读取并处理，转变为路由判断的条件。例如 Path=/user/** 是按照路径匹配，这个规则是由rg.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory 类来处理的，像这样的断言工厂在SpringCloudGateway还有十几个。

## 路由过滤工厂

GatewayFilter 是网关中提供的一种过滤器，可以对进入网关的请求和微服务返回的响应做处理

<img src="https://img2022.cnblogs.com/blog/1424359/202205/1424359-20220512215445412-868575425.png">

## 全局过滤器（GlobalFilter）

全局过滤器（GlobalFilter）与 GatewayFilter 的 default-filters 的作用类似，也是处理一切进入网关的请求和微服务响应，对所有路由都生效。区别在于 GatewayFilter 通过配置定义，只能处理一些比较简单的逻辑，而 GlobalFilter 的逻辑通过代码实现，可以自定义复杂逻辑。

要想实现全局过滤器需要实现 GlobalFilter 接口：

```java
public interface GlobalFilter {
   /**
    *  处理当前请求，有必要的话通过{@link GatewayFilterChain}将请求交给下一个过滤器处理
    *
    * @param exchange 请求上下文，里面可以获取Request、Response等信息 
    * @param chain 用来把请求委托给下一个过滤器 
    * @return {@code Mono<Void>} 返回标示当前过滤器业务结束   
   */
   Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

## 网关解决跨域问题

```yaml
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos地址
    gateway:
      # gateway的全局跨域请求配置
      globalcors:
        add-to-simple-url-handler-mapping: true # 解决options请求被拦截问题
        corsConfigurations:
          '[/**]':  # 匹配的路由
            allowedHeaders: "*"  # 允许在请求中携带的头信息
            allowedOrigins: "*"  # 允许哪些网站的跨域请求。也可以通过数组的方式来只指定一些特定的网站
            allowCredentials: true  # 是否允许携带cookie
            allowedMethods: "*"  # 允许的跨域ajax的请求方式。也可以通过数组的方式来只指定一些特定的请求方式
            maxAge: 360000 # 跨域检测的有效期
```

