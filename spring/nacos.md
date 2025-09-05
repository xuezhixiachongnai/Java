# Nacos

## 概念

nacos主要整合了注册中心和配置中心。

nacos主要提供一下四大功能：

- 服务发现与服务健康检查
- 动态配置管理
- 动态DNS服务
- 服务和元数据管理

使用nacos分为两部分，启动nacos-server，搭建nacos-client

## nacos-server

单例模式

```sh
startup -m standalone
```

集群模式

```sh
startup
```

## nacos-server

### [注册中心](https://www.cnblogs.com/wenxuehai/p/16179629.html)

nacos-client的搭建，只是往普通spring-boot项目中引入以客户端依赖，添加配置即可。注意nacos-server需要保持启动状态，否则客户端会因为连接不上而导致注册失败

```xml
<!--nacos的管理依赖-->
<dependencyManagement>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.5.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependencyManagement>

<!-- nacos客户端依赖包 -->
<!-- 服务发现 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
server:
  port: 8081
spring:
  application:
    name: orderservice #设置当前应用名称，在eureka中Aoolication显示，通过该名称可以获取路径
  cloud:
    nacos:
      server-addr: http://localhost:8848 # nacos服务地址
```

### [配置中心](https://www.cnblogs.com/wenxuehai/p/16188419.html)

通过在配置中心发布配置，修改服务实例的配置

一个服务如果以 nacos 作为配置中心，则会先拉取 nacos 中管理的配置，然后与本地的配置文件比如 application.yml 中的配置合并，最后作为项目的完整配置，启动项目

- 没有nacos管理配置文件的情况下的项目启动流程：

  ```sh
  启动项目 ---> 读取本地aplication.yml ---> 创建spring容器 ---> 加载Bean
  ```

- 使用nacos管理配置时项目的启动流程：

  ```sh
  启动项目 ---> 读取nacos中配置文件 ---> 读取本地aplication.yml ---> 创建spring容器 ---> 加载Bean
  ```

在Spring Cloud 2022.x之前项目启动的时候需要提前知道 nacos 的环境信息，而 application.yml 在读取 nacos 配置后才会读取，所以无法把 nacos 的相关信息配置在 application.yml 中，此时我们可以使用 bootstrap.yml文件。bootstrap.yml是一个引导文件，优先级高于application.yml，它会在application.yml之前被读取

在Spring Cloud 2022.x之后，弃用bootstrap.yml，直接写在application.yml中即可

要想让服务能够拉取nacos配置中心的配置，需要引入nacos配置管理依赖

```xml
<!--nacos配置管理依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: userservice # 服务名称
  profiles:
    active: dev #开发环境，这里是dev 
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos地址
      config:
        file-extension: yaml # 文件后缀名
```

####  配置热更新

默认情况下，修改了 nacos 配置中心的配置，微服务的配置不会随之更新的，需要重启微服务才能读到新配置。

通过 @Value 和通过 @ConfigurationProperties 来读取配置时，实现热更新的方式不同。

- 如果通过@Value来读取配置，此时需要在使用@Value注入的变量所在类上添加@RefreshScope

- 通过@ConfigurationProperties来读取配置时，无需其他操作

  > ```java
  > @Component
  > @ConfigurationProperties(prefix = "app")
  > public class AppProperties {
  >     private String name;
  >     private String version;
  >     private String author;
  > 
  >     // 一定要有 getter/setter，才能完成绑定
  >     // Lombok 的 @Data/@Getter/@Setter 也可以
  > }
  > 
  > ```
  >
  > ```yaml
  > app:
  >   name: engineer-system
  >   version: 1.0.0
  >   author: hj
  > 
  > ```
  >
  > @ConfrgurationProperties注解的作用是将配置文件里的属性批量注入到一个javabean中

#### [集群搭建](https://www.cnblogs.com/linjiqin/p/15363026.html)

> 不同nacos节点如何是实现数据共享：
>
> - 配置管理的数据保存在mysql中，确保多节点共享和持久化
>
> - 服务注册的数据：
>   - 临时实例主要依赖Raft协议+内存复制在内存中保存一致，不会全部写入mysl
>   - 非临时实例会写入mysql从而实现跨节点共享
