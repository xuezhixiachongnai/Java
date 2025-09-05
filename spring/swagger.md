# Swagger

## 概述

swagger是一个规范和完整的框架，用于生成、描述、调用和可视化RestFul风格的web服务，总体目标是使客户端和文件系统作为服务器以同样的速度来更新。文件的方法，参数和模型紧密集成到服务器断的代码，允许API来始终保持同步。

## SpringBoot集成Swagger

```xml
// springboot 版本 2.x 完整，2.6+ 需改 pathmatch 或升级 3.0
 <!--swagger依赖-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
<!--swagger ui-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>

// 或者
<!-- Spring Boot 2.x -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>

// 2.6+ & 3.x
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>

```

**springfox-boot-starter**还是基于**springfox-swagger2**和**springfox-swagger-ui**。**springdoc-openapi-starter-webmvc-ui**则是专门为 Spring Boot 3.x 和 OpenAPI 3 定制，自动扫描 Controller，几乎无需写 Docket，支持最新 Swagger UI 和 OpenAPI 3 功能。

```java
// springfox-boot-starter
@Configuration
@EnableSwagger2
public class Swagger2Config {

    /**创建API应用
     apiInfo() 增加API相关信息
     * 通过select()函数返回一个ApiSelectorBuilder实例,用来控制哪些接口暴露给Swagger来展现，
     指定扫描的包路径来定义指定要建立API的目录。*/

    @Bean
    public Docket coreApiConfig(){
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(adminApiInfo())
                .groupName("adminApi")
                .select()
                //只显示admin下面的路径
                .paths(Predicates.and(PathSelectors.regex("/admin/.*")))
                .build();
    }

    private ApiInfo adminApiInfo(){
        return new ApiInfoBuilder()
                .title("尚融宝后台管理系统--api文档")
                .description("尚融宝后台管理系统接口描述")
                .version("1.0")
                .contact(new Contact("ljd","http://baidu.com","123456778@qq.com"))
                .build();
    }
}
```

```java
// springdoc-openapi-starter-webmvc-ui

@Configuration
public class OpenApiConfig {

    // 全局 OpenAPI 信息
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("我的接口文档")
                        .version("v1.0")
                        .description("这是 Springdoc OpenAPI 自动生成的接口文档"));
    }

    // 分组示例：把 admin 的接口单独分组
    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
                .group("admin")
                .pathsToMatch("/admin/**")
                .build();
    }

    // 分组示例：把用户接口单独分组
    @Bean
    public GroupedOpenApi userApi() {
        return GroupedOpenApi.builder()
                .group("user")
                .pathsToMatch("/user/**")
                .build();
    }
}

```

### 注解使用

- @Api：用于标注一个Controller
- @ApiOperation：用于请求方法上，说明方法的用途和作用
- @ApiImplicitParams与@ApiImplicitParam：它们两个都是作用在方法上用来设置参数，不过不同的是@ApiImplicitParams可以包含多个@ApiImplicitParam用来设置多个参数，而@ApiImplicitParam仅仅只是用来设置一个参数。
- @ApiModel：主要用于字段的实体类上，对整个实体类进行描述
- @ApiModelProperty：主要用于实体类的字段上，对其属性进行描述

