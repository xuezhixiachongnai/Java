# [Mybatis](https://www.cnblogs.com/diffx/p/10611082.html)

## 概述

**Mybatis**是一个Java持久层框架（ORM框架的半自动化版本），它的主要作用是将Java对象与数据库专用的SQL映射起来，让程序员可以更方便的操作数据库。

它的核心功能有：

> SQL映射、参数映射、结果映射、事务控制、缓存机制和
>
> 插件扩展（可以在 Executor、StatementHandler、ParameterHandler、ResultSetHandler 等核心点插入拦截器）

## MybaitsConfig

`resource`目录下的mybatis-config配置文件，配置了数据源，事务管理器、数据库链接池等。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <properties>
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/smmdemo?useUnicode=true&amp;characterEncoding=utf-8&amp;allowMultiQueries=true"/>
        <property name="username" value="root"/>
        <property name="password" value="Hanjie1012"/>
    </properties>
	<!-- 开启数据库记录和实体类的映射 -->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

    <!-- 环境，可以配置多个，default：指定采用哪个环境 -->
    <environments default="test">
        <!-- id：唯一标识 -->
        <environment id="test">
            <!-- 事务管理器，JDBC类型的事务管理器 -->
            <transactionManager type="JDBC" />
            <!-- 数据源，池类型的数据源 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/ssmdemo" />
                <property name="username" value="root" />
                <property name="password" value="Hanjie1012" />
            </dataSource>
        </environment>
        <environment id="development">
            <!-- 事务管理器，JDBC类型的事务管理器 -->
            <transactionManager type="JDBC" />
            <!-- 数据源，池类型的数据源 -->
            <dataSource type="POOLED">
                <property name="driver" value="${driver}" /> <!-- 配置了properties，所以可以直接引用 -->
                <property name="url" value="${url}" />
                <property name="username" value="${username}" />
                <property name="password" value="${password}" />
            </dataSource>
        </environment>
    </environments>
    <!-- 引入mapper资源 -->
    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>

</configuration>
```

`resource/mapper`目录下配置查询的sql语句

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- mapper:根标签，namespace：命名空间，随便写，一般保证命名空间唯一 -->
<mapper namespace="com.jh.mapper.UserMapper">
    <!-- statement，内容：sql语句。id：唯一标识，随便写，在同一个命名空间下保持唯一
       resultType：sql语句查询结果集的封装类型,tb_user即为数据库中的表
     -->
    <select id="selectUser" parameterType="string" resultType="com.jh.entity.User">
        select * from tb_user where id = #{id}
    </select>
</mapper>
```

mybatis的基本操作是

```java
public class MybatisTest {

    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        // 解析mybatis-config
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        // 开启会话创建jdbc链接
        try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
            // 命名空间对应
            User user = sqlSession.selectOne("com.jh.mapper.UserMapper.selectUser", "1");
            // UserMapper mapper = sqlSession.getMapper(UserMapper.class);
			// User user = mapper.selectUser(1);
            System.out.println(user);
        }
    }
}
```

### [mybatis事务](https://chatgpt.com/c/689ea2ed-8628-8322-a829-8a62bd33ea3b)

## 核心类

- **SqlSessionFactory**：里面包含了一个`Configuration` 对象（包括数据源、事务工厂、Mapper 映射信息、缓存配置等），用来生成**SqlSession**
- **SqlSession**：每个 `SqlSession` 都对应一次数据库会话，里面会创建和管理一个 JDBC `Connection`，用来操作数据库
- **Mapper**：由`mybatis`生成一个操作数据库的代理类

## 拦截器

已映射语句的执行过程

> - Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed) ：是执行sql语句的核心接口，相当于JDBC的执行引擎和缓存管理器，所有的查询、更新、批处理操作都是由它来完成
> - ParameterHandler (getParameterObject, setParameters)：负责将用户传入的参数对象，按照参数映射（ParameterMapping）绑定到 JDBC 的 PreparedStatement 上
> - ResultSetHandler (handleResultSets, handleOutputParameters)： 负责将 JDBC 查询结果集（ResultSet）转换为 Java 对象
> - StatementHandler (prepare, parameterize, batch, update, query)：负责管理 JDBC Statement 对象的创建、参数绑定、SQL 执行 的核心接口

现在一些MyBatis 插件比如PageHelper都是基于这个原理，有时为了监控sql执行效率，也可以使用插件机制原理

## mybatis缓存

## mybatis分页插件

**PageHelper**是一款第三方分页插件

引入依赖

```xml
<!-- Pagehelper分页插件-->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.2.12</version>
</dependency>
```

再**application.yml**的配置

```yml
#mybatis 分页插件
pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

注入实现类

```java
@Configuration
public class PageHelperConfig {
 
    @Bean
    public PageHelper getPageHelper() {
        PageHelper pageHelper = new PageHelper();
        Properties properties = new Properties();
        properties.setProperty("helperDialect", "mysql");
        properties.setProperty("reasonable", "true");
        properties.setProperty("supportMethodsArguments", "true");
        properties.setProperty("params", "count=countSql");
        pageHelper.setProperties(properties);
        return pageHelper;
    }
}
```

分页操作

```java
    // 1. 开始分页（在执行查询方法前调用）
    PageHelper.startPage(2, 5); // 第2页，每页5条

    // 2. 执行查询
    List<User> list = mapper.selectAll();

    // 3. 封装分页信息
    PageInfo<User> pageInfo = new PageInfo<>(list);
```

## mybatis缓存

- 一级缓存
  - MyBatis一级缓存的生命周期和SqlSession一致。
  - MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。
  - MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement。
- 二级缓存
  - MyBatis的二级缓存相对于一级缓存来说，实现了`SqlSession`之间缓存数据的共享，同时粒度更加的细，能够到`namespace`级别，通过Cache接口实现类不同的组合，对Cache的可控性也更强。
  - MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
  - 在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis、Memcached等分布式缓存可能成本更低，安全

## Mybatis-Plus

### 概述

MyBatis-Plus 是基于 MyBatis 框架的一个增强工具，主要目的是简化 MyBatis 的开发过程，提供更加简洁、方便的 CRUD 操作。它是在保留 MyBatis 强大功能的基础上，通过封装和优化一些常见操作来提高开发效率。

MyBatis-Plus 提供了许多开箱即用的功能，包括自动 CRUD 代码生成、分页查询、性能优化、以及支持多种数据库。与 MyBatis 相比，MyBatis-Plus 的 部分 核心特性包括：

- **无侵入设计**：不会改变 MyBatis 原有的 API 和使用方式，你可以自由选择 MyBatis 和 MyBatis-Plus 的功能。

- **自动 CRUD**：通过 `BaseMapper` 和 `ServiceImpl` 接口，MyBatis-Plus 提供了一系列 CRUD 操作的方法，如 `insert`、`delete`、`update` 和 `select`，减少了重复的 SQL 编写工作。

- **条件构造器**：MyBatis-Plus 提供了条件构造器（如 `QueryWrapper`），可以通过链式编程方式轻松构建复杂的查询条件。

引入依赖

```java
 <dependency>
     <groupId>com.baomidou</groupId>
     <artifactId>mybatis-plus-boot-starter</artifactId>
     <version>3.5.7</version>
 </dependency>
     <dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>9.0.0</version>
</dependency>
```

配置application.properties

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/yourDatabaseName?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf-8
spring.datasource.username=YourUserName
spring.datasource.password=YourPassword
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

### BaseMapper

`BaseMapper` 是 MyBatis-Plus 提供的一个基础 Mapper 接口，它简化了数据访问层（Data Access Layer）的开发。`BaseMapper` 提供了一系列通用的数据库操作方法，这样你就不必手动编写常见的 SQL 语句，从而提升了开发效率。

```java
public interface UserMapper extends BaseMapper<User> {
    // 你可以在这里添加自定义方法
}
```

### IService

`IService` 是 MyBatis-Plus 提供的一个通用服务接口。它定义了一些常见的 CRUD（Create, Read, Update, Delete）操作，并将这些操作抽象成方法。这意味着，当你使用 `IService` 接口时，你无需自己手动编写这些常见的数据库操作方法。

```java
public interface UserService extends IService<User> {
    // 可以定义一些自定义的服务方法
}
```

### ServiceImpl

`ServiceImpl` 是 MyBatis-Plus 提供的一个基础实现类，它实现了 `IService` 接口中的方法。`ServiceImpl` 通常是被继承的，它提供了具体的数据库操作方法的实现。开发者只需在自己定义的服务实现类中继承 `ServiceImpl` 类，就可以获得默认的 CRUD 功能。

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    // 你可以重写 ServiceImpl 中的方法，或者定义更多的业务逻辑
}
```

### 字段映射

MyBatis-Plus 的自动映射规则主要涉及如何将数据库表和字段自动映射到 Java 实体类及其属性

> MyBatis 默认的 `自动映射` 是 **数据库列名和 Java 字段名完全一致** 时才会自动匹配。
>
> - 如果你的数据库列名是 `id` → 对应实体类 `id` 自动映射
> - 如果数据库列名是 `user_name` → 对应实体类 `userName`  默认不会自动匹配（除非特殊配置）

在mybatis-plus中，可以使用注解来实现自定义映射

- @TableName：指定表名
- @TableField：指定自定义的字段名，还可以用来指定该字段的存在属性
- @TableId：指定主键

### QueryWrapper条件构造器

### 逻辑删除

配置

```yml
mybatis-plus:  
  global-config:  
    db-config: 
      # 全局逻辑删除配置
      logic-delete-field: valid # 全局逻辑删除的实体字段名
      # 若逻辑已删除和未删除的值和默认值一样，则可以不配置这2项
      logic-delete-value: 0 # 逻辑已删除值(默认为1)  
      logic-not-delete-value: 1 # 逻辑未删除值(默认为0)  

```

针对某个实体类字段

```java
@TableField(value = "valid", select = false)
@TableLogic(value = "1", delval = "0")
private Boolean valid;
```

