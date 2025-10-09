# [Sentinel](https://www.cnblogs.com/crazymakercircle/p/14285001.html)

**Sentinel 的使用可以分为两个部分:**

- 控制台（Dashboard）：控制台主要负责管理推送规则、监控、集群限流分配管理、机器发现等。
- 核心库（Java 客户端）：不依赖任何框架/库，能够运行于 Java 7 及以上的版本的运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。

## 概念

提供了流量控制、熔断降级、系统负载保护等多个维度来保障服务之间的稳定性

sentinel的使用可以分为两个部分：

- 控制台（Dashboard）：控制台主要负责管理推送规则、监控、集群限流分配管理、机器发现等

  > 启动控制台：java  -server -Xms64m -Xmx256m  -Dserver.port=8849 -Dcsp.sentinel.dashboard.server=localhost:8849 -Dproject.name=sentinel-dashboard -jar /work/sentinel-dashboard-1.7.xx.jar
  >
  > 将客户端接入控制台：
  >
  > ```xml
  > <dependency>
  >     <groupId>com.alibaba.cloud</groupId>
  >     <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
  > </dependency>
  > ```
  >
  > ```yaml
  > spring:
  >   cloud:
  >     sentinel:
  >       transport:
  >         dashboard: 192.168.180.137:8888   #sentinel控制台的请求地址
  > 
  > ```
  >
  > 

- 核心库（java 客户端）：不依赖任何框架，能够运行于 java 7 及以上版本的运行时环境

> - 响应时间（RT）：响应时间是指系统对请求作出响应的时间
> - 吞吐量（TPS）：吞吐量是指系统在单位时间内处理请求的数量
> - 并发用户数：并发用户数是指系统可以同时承载的正常使用系统功能的用户的数量
> - 每秒查询率（QPS）：每秒的响应请求数，也即是最大吞吐能力

##  使用Sentinel来进行熔断与限流

Sentinel 可以简单的分为 Sentinel 核心库和 Dashboard。核心库不依赖 Dashboard，但是结合Dashboard 可以取得最好的效果。
使用 Sentinel 来进行熔断保护，主要分为几个步骤:

1. 定义资源

   > 资源：可以是任何东西，一个服务，服务里的方法，甚至是一段代码。

2. 定义规则

   > 规则：Sentinel 支持以下几种规则：流量控制规则、熔断降级规则、系统保护规则、来源访问控制规则
   > 和 热点参数规则。

3. 检验规则是否生效

Sentinel 的所有规则都可以在内存态中动态地查询及修改，修改之后立即生效. 先把可能需要保护的资源定义好，之后再配置规则。

也可以理解为，只要有了资源，我们就可以在任何时候灵活地定义各种流量控制规则。在编码的时候，只需要考虑这个代码是否需要保护，如果需要保护，就将之定义为一个资源。

### 定义资源

**资源**是 Sentinel 的关键概念。它可以是 Java 应用程序中的任何内容，例如，由应用程序提供的服务，或由应用程序调用的其它应用提供的服务，RPC接口方法，甚至可以是一段代码。

只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源。

把需要控制流量的代码用 Sentinel的关键代码 SphU.entry("资源名") 和 entry.exit() 包围起来即可。

```java
    Entry entry = null;
    try {
        // 定义一个sentinel保护的资源，名称为test-sentinel-api
        entry = SphU.entry(resourceName);
        // 模拟执行被保护的业务逻辑耗时
        Thread.sleep(100);
        return a;
    } catch (BlockException e) {
        // 如果被保护的资源被限流或者降级了，就会抛出BlockException
        log.warn("资源被限流或降级了", e);
        return "资源被限流或降级了";
    } catch (InterruptedException e) {
        return "发生InterruptedException";
    } finally {
        if (entry != null) {
            entry.exit();
        }
 
        ContextUtil.exit();
    }
}

```

也可以使用资源注解@SentinelResource

```java
@SentinelResource("HelloWorld")
public void helloWorld() {
    // 资源中的逻辑
    System.out.println("hello world");
}

```

### 定义规则

规则主要有流控规则、 熔断降级规则、系统规则、权限规则、热点参数规则等：

一段硬编码的方式定义流量控制规则如下：

```java
private void initSystemRule() {
    List<SystemRule> rules = new ArrayList<>();
    SystemRule rule = new SystemRule();
    rule.setHighestSystemLoad(10);
    rules.add(rule);
    SystemRuleManager.loadRules(rules);
}
```

加载规则：

```java
FlowRuleManager.loadRules(List<FlowRule> rules); // 修改流控规则
DegradeRuleManager.loadRules(List<DegradeRule> rules); // 修改降级规则
SystemRuleManager.loadRules(List<SystemRule> rules); // 修改系统规则
AuthorityRuleManager.loadRules(List<AuthorityRule> rules); // 修改授权规则
```

### 熔断降级

熔断降级对调用链路中不稳定的资源进行熔断降级是保障高可用的重要措施之一。

由于调用关系的复杂性，如果调用链路中的某个资源不稳定，最终会导致请求发生堆积。Sentinel 熔断降级会在调用链路中某个资源出现不稳定状态时（例如调用超时或异常比例升高），对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断（默认行为是抛出 DegradeException）

### 流量控制

流量控制(Flow Control)，原理是监控应用流量的QPS或并发线程数等指标，当达到指定阈值时对流量进行控制，避免系统被瞬时的流量高峰冲垮，保障应用高可用性。

### 系统保护

### 黑白名单规则



