# Spring

### Spring 是什么

Spring 是一个开源的轻量级 Java 开发框架，其目标是简化 Java EE 的开发

它的核心思想是：

- **IOC** 控制反转：把对象的创建和依赖关系交给 Spring 容器管理
- **AOP** 面向切面编程：把日志、事务、安全等通用功能从业务中剥离出来，降低耦合

### Spring 的核心模块

Spring 实际上是一整个生态系统，组基础的就是 **Spring Framework**，主要包含几个核心模块：

- **Core Container** 核心容器：
  - 包含 IOC 容器，负责对象的创建、管理、依赖注入
  - 核心接口：`ApplicationContext`
- **AOP** 面向切面编程：用来处理日志、事务、安全校验等横切关注点
- **Data Access** 数据访问：
  - 提供了简化的 JDBC 封装，方便操作数据库
  - 集成 ORM 框架，如：MyBatis、JPA
- **Spring MVC**
  - 基于 Servlet 的 Web 框架，常用于构建 Web 应用
  - 使用**前端控制器模式（DispatcherServlet）**，分发请求给 Controller
- **Integration** 集成：与消息队列，邮件服务、任务调度、远程调用等集成

### Spring 的优势

- 通过 IOC 和 DI（依赖注入），降低类与类之间的耦合度
- AOP 让日志、事务管理更容易扩展
- 封装 JDBC，减少重复代码
- 拥有强大的生态

### Spring 生态

- **Spring Framework**：基础框架，提供 IOC、AOP、MVC 等核心功能。

- **Spring Boot**：快速开发框架，约定大于配置，能快速启动 Spring 应用。

- **Spring Cloud**：微服务框架，构建分布式系统（服务发现、负载均衡、配置中心）。

- **Spring Data**：简化数据库和 NoSQL 操作。

- **Spring Security**：安全框架，处理认证和授权。