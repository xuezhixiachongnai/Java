# Spring Boot

SpringFramework 的组件代码是轻量级的，但它的配置却是重量级的。它的痛点也不止于此

| 痛点             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| **繁琐的配置**   | XML 配置文件（`applicationContext.xml`、`springmvc.xml`）非常多，尤其是大型项目，扫描包、数据源、事务管理都需要手动配置。 |
| **依赖管理复杂** | Spring 各模块（Spring MVC、Spring JDBC、Spring Data 等）版本需要手动协调，容易出现依赖冲突。 |
| **启动部署繁琐** | 传统 Web 项目需要打成 WAR 包，部署到外部容器（Tomcat、Jetty）上。 |
| **自动化程度低** | 各模块集成需要手动配置，例如 Spring MVC + JPA + Redis。      |
| **测试困难**     | 单元测试和集成测试需要手动初始化 Spring 容器。               |
| **约定不统一**   | 不同团队可能会有不同的配置方式和包结构，学习成本高。         |

而 Spring Boot 是 **Spring 生态下的一个开箱即用框架**，核心目标是 **简化 Spring 应用的开发和部署**，尤其是企业级 Web 应用和微服务。它主要解决了传统 Spring 项目中繁琐的配置问题

| Spring Boot 特性                                  | 对应痛点解决方案                                             |
| ------------------------------------------------- | ------------------------------------------------------------ |
| **自动配置（Auto-Configuration）**                | 根据 classpath 的依赖自动配置 Bean，省去了大量 XML 配置。例如引入 `spring-boot-starter-web` 自动配置 DispatcherServlet、Tomcat、Spring MVC。 |
| **起步依赖（Starter POM）**                       | 一个依赖包搞定整个模块所需的依赖，解决版本管理和依赖冲突问题。例如 `spring-boot-starter-data-jpa`。 |
| **嵌入式服务器**                                  | 默认内置 Tomcat/Jetty，无需部署 WAR，可以直接运行 JAR 包。   |
| **约定优于配置（Convention over Configuration）** | 约定默认目录结构（`src/main/java`、`resources`），默认端口 8080，默认日志配置等，大大减少配置量。 |
| **简化测试**                                      | 提供 `@SpringBootTest` 注解，可以方便地启动整个容器进行集成测试。 |
| **现代微服务支持**                                | 内置 Actuator、REST API 支持、Spring Cloud 兼容，便于快速开发微服务。 |

核心特点

1. **自动配置（Auto-Configuration）**
    根据 classpath 中的依赖，Spring Boot 自动配置各种组件。例如：

   - 有 `spring-boot-starter-web` 就自动配置 `DispatcherServlet`、`Tomcat`、`Spring MVC`。
   - 有 `spring-boot-starter-data-jpa` 就自动配置 `EntityManager`、数据源等。

2. **起步依赖（Starter POMs）**
    提供了一系列 “starter” 依赖，一行依赖搞定常用功能，例如：

   ```
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   ```

   自动引入 Spring MVC、Tomcat、Jackson 等所需依赖。

3. **嵌入式服务器**
    默认内置 Tomcat/Jetty/Undertow，可以直接运行 `java -jar` 启动，不需要额外部署 WAR。

4. **无 XML 配置**
    全部使用注解和配置类（Java Config） + `application.properties` 或 `application.yml`，简化了传统 Spring 的 `web.xml`、`applicationContext.xml`。

5. **快速构建微服务和 RESTful API**
    提供 `@RestController`、`@SpringBootApplication` 等注解，快速实现 Web 服务。

核心注解

- `@SpringBootApplication`
   包含：
  - `@Configuration`：标记配置类
  - `@EnableAutoConfiguration`：启用自动配置
  - `@ComponentScan`：自动扫描同包及子包下的组件
- `@RestController`、`@Controller`、`@Service`、`@Repository`

启动方式

```
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

- `SpringApplication.run` 会创建 Spring 容器、加载自动配置、启动嵌入式服务器。

spring boot 让开发者能专注于业务逻辑，而不是框架配置。