# MyBatis

MyBatis 是一个**优秀的持久层框架**，主要用于**简化 JDBC 操作**和 **SQL 映射**。它的定位是**半自动 ORM（对象关系映射，Object Relational Mapping）框架**

#### MyBatis 的核心特点

1. **SQL 映射（SQL Mapping）**
   - 开发者自己写 SQL 语句，MyBatis 负责将 **SQL 与 Java 对象字段映射**。
   - 保证了 SQL 的灵活性，不像 Hibernate 那样完全自动生成 SQL。
2. **避免 JDBC 繁琐代码**
   - 传统 JDBC 需要写很多模板代码（连接、PreparedStatement、ResultSet 映射等）。
   - MyBatis 通过**配置文件（XML）或注解**，自动完成参数设置与结果映射。
3. **灵活的配置方式**
   - 可以在 **XML 文件**里写 SQL，也可以直接在 **Mapper 接口**里用注解写 SQL。
4. **支持动态 SQL**
   - 通过 `<if>`、`<choose>`、`<foreach>` 等标签，可以根据条件拼接 SQL。
5. **集成方便**
   - 常与 Spring / Spring Boot 配合使用，整合简单。

这里，我们先只讲纯 Mybatis 项目

创建一个 maven 项目，引入依赖

```xml
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.19</version>
    </dependency>
```

使用 Mybatis 时，需要一个全局配置文件`mybatis-config.xml`，这里面要配置很多全局变量，Mybatis 也靠它来启动和加载 Mapper

`mybatis-config.xml` 里配置：

- 数据源
- 事务
- 别名
- 插件
- `<mappers>`（告诉 MyBatis 去哪里加载 SQL 映射文件）。

在 resources 目录下配置`mybatis-config.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!-- 根标签 -->
<configuration>
    <properties>
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatis-110?useUnicode=true&amp;characterEncoding=utf-8&amp;allowMultiQueries=true"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </properties>

    <!-- 环境，可以配置多个，default：指定采用哪个环境 -->
    <environments default="test">
        <!-- id：唯一标识 -->
        <environment id="test">
            <!-- 事务管理器，JDBC类型的事务管理器 -->
            <transactionManager type="JDBC" />
            <!-- 数据源，池类型的数据源 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatis-110" />
                <property name="username" value="root" />
                <property name="password" value="123456" />
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
</configuration>
```

配置中可以设置 properties 引用值

`<environments>`中可以配置不同生产环境下的配置，通过`default`用来指定使用哪个。

`<environment>`下是来配置 mybatis 的全局变量的。

接下来创建一个实体类

```java
public class User {

    private int id;

    private String userName;

    private int age;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", userName='" + userName + '\'' +
                ", age=" + age +
                '}';
    }
}
```

接着，创建它的 dao

```java
public interface UserDao {

    List<User> findAll();

    User findById(int id);

    void save(User user);

    void delete(int id);

    void update(User user);
}
```

daoimpl

```java
public class UserDaoImpl implements UserDao {

    private SqlSession sqlSession;

    public UserDaoImpl() {

    }

    public UserDaoImpl(SqlSession sqlSession) {
        this.sqlSession = sqlSession;
    }


    @Override
    public List<User> findAll() {
        return sqlSession.selectList("UserMapper.findAll");
    }

    @Override
    public User findById(int id) {
        return sqlSession.selectOne("UserMapper.findById", id);
    }

    @Override
    public void save(User user) {
        sqlSession.insert("UserMapper.save", user);
    }

    @Override
    public void delete(int id) {
        sqlSession.delete("UserMapper.delete", id);
    }

    @Override
    public void update(User user) {
        sqlSession.update("UserMapper.update", user);
    }
}
```

在 resource/mapper 目录下创建一个`UserMapper.xml`的 sql 映射文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- mapper:根标签，namespace：命名空间，随便写，一般保证命名空间唯一 -->
<mapper namespace="UserMapper">
    <!-- statement，内容：sql语句。id：唯一标识，随便写，在同一个命名空间下保持唯一
       resultType：sql语句查询结果集的封装类型,tb_user即为数据库中的表
     -->
    <select id="findById" resultType="com.jh.entity.User">
        select * from user where id = #{id}
    </select>

    <select id="findAll" resultType="com.jh.entity.User">
        select * from user
    </select>

    <insert id="save">
        insert into user (user_name, age) VALUE (#{userName}, #{age})
    </insert>

    <delete id="delete">
        delete from user where id = #{id}
    </delete>

    <update id="update" parameterType="int">
        update user set user_name = #{UserName}, age = #{age} where id = #{id}
    </update>
</mapper>
```

在 mybatis-config.xml 中添加该文件，使 myabtis 能够加载它

```java
<mappers>
    <mapper resource="mapper/UserMapper.xml"/>
</mappers>
```

做测试

```java
public class Main {

    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
            SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
            try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
                UserDaoImpl userDao = new UserDaoImpl(sqlSession);
                User user1 = new User();
                user1.setUserName("雪之下雪乃");
                user1.setAge(18);
                User user2 = new User();
                user2.setUserName("牧濑红莉栖");
                user2.setAge(18);
                userDao.save(user1);
                userDao.save(user2);
                sqlSession.commit();
                List<User> users = userDao.findAll();
                for (User user : users) {
                    System.out.println(user);
                }
                System.out.println(userDao.findById(1));
            }
        }
    }
}
```

现在现总结一下 mybatis 的使用步骤：
mybatis 通过加载配置文件来获取一个 SqlSessionFactory，再由它获取一个 SqlSession。最后由 SqlSession 操作数据库。

**SqlSessionFactoryBuilder**

- **作用**：用来解析 MyBatis 配置文件（`mybatis-config.xml`），创建 `SqlSessionFactory`。
- **特点**：
  - 一次性使用，不需要保留。
  - 线程不安全。

 **SqlSessionFactory**

- **作用**：是一个“工厂”，用来生产 `SqlSession` 对象。
- **特点**：
  - 全局唯一，一般应用启动时创建一次，整个项目复用。
  - 线程安全，可以在多个线程中共享。

 **SqlSession**

- **作用**：真正的数据库操作对象，包含执行 SQL 的所有方法。
- **常见方法**：
  - `selectOne()`、`selectList()` → 查询
  - `insert()`、`update()`、`delete()` → 写操作
  - `commit()`、`rollback()` → 事务控制
  - `getMapper()` → 获取 Mapper 接口代理对象
- **特点**：
  - 线程不安全。
  - 一般是 **方法内获取，用完立即关闭**，不要长时间保存。

在 mybatis 中，事务默认是手动提交的，不会自动提交

当我们通过 `sqlSessionFactory.openSession()` 获取一个 `SqlSession` 时：

- 默认情况下：**`openSession(false)`** → 事务关闭自动提交，需要手动 `commit()`。

- 如果希望自动提交，可以将 false 改为 true：

  ```
  SqlSession session = sqlSessionFactory.openSession(true);
  ```

  这样每次 `insert`、`update`、`delete` 执行后都会自动提交。

在 mybatis 中有两种方式执行数据库操作，上写的一种是不使用 Mapper 代理的方式，直接调用`selectOne`等方法的方式

```java
try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
    User user = sqlSession.selectOne("UserMapper.findById", 1);
}
```

这种方式的原理是往`selectOne`中传入的是映射文件 mapper 中的 namespace 和 sql 的 id。

mybatis 会根据这个 id 在 **Configuration** 里找到对应的 **MappedStatement**（存放 SQL、参数、返回类型等信息的对象）。

然后执行 JDBC 操作：

- 解析 SQL
- 绑定参数
- 执行 SQL
- 映射结果集到对象

Configuration 和 MappedStatement 类是 MyBatis 的底层对象

**Configuration**

包所在位置：`org.apache.ibatis.session.Configuration`

作用

- 是 MyBatis 的 **全局配置对象**，在加载 `mybatis-config.xml` 和 `mapper.xml` 时创建并填充。
- 里面保存了所有运行所需的信息：
  - 数据源、事务管理器
  - 类型别名、类型处理器（TypeHandler）
  - Mapper 映射关系
  - 所有 SQL 的定义（以 `MappedStatement` 形式存放）

**MappedStatement**

包所在位置：`org.apache.ibatis.mapping.MappedStatement`

作用

- 表示**一个具体的 SQL 配置项**（对应 `<select> / <insert> / <update> / <delete>` 标签）。
- 里面封装了执行 SQL 所需的全部信息：
  - SQL 的 ID（namespace + id）
  - SQL 文本（可以是动态 SQL）
  - 输入参数类型（parameterType）
  - 输出结果映射（resultType / resultMap）
  - SQL 的命令类型（SELECT / UPDATE / INSERT / DELETE）
  - 缓存设置、超时时间等

总结来说

`SqlSessionFactory` 在初始化时，会读取`mybatis-config.xml`生成一个 `Configuration`，其中会加载全局配置和，文件中所涉及的所有 mapper 的映射信息，mapper 信息会映射成一个`MappedStatement`保存。执行 SQL 时就是从 `Configuration` 里取 `MappedStatement` 拿到 SQL 模板和参数映射信息。交给 `Executor` 执行，最终返回结果。

我们在 selectOne 中传入的参数，是如何正确匹配到 sql 中的呢

```sh
SQL: SELECT * FROM user WHERE id = #{id}
            │
            ▼
ParameterMapping("id")   ← 描述占位符
            │
            ▼
ParameterHandler.getParameterValue()
            │
            ▼
Java 值 (123)
            │
            ▼
TypeHandler.setParameter(preparedStatement, index, 123, JDBC_TYPE)
            │
            ▼
PreparedStatement.execute()
```

MyBatis 会解析 SQL 文本，找到所有`#{xxx}`占位符。接着会将每个占位符封装成一个 **`ParameterMapping`** 对象，并将 SQL 中的占位符被替换成 `?`，形成最终可执行的 SQL（`BoundSql.sql`）。

然后`ParameterHandler`会获取 `selectOne` / `update` 等方法的参数，将参数封装成 `ParamMap` 对象，它是一个 map，就是通过其中的 key 和 占位符匹配的。正确匹配获得参数后由`TypeHandler` 根据 Java 类型与 JDBC 类型，将值绑定到 `PreparedStatement` 对应占位符最后调用 `preparedStatement.setXXX(index, value)` 执行 SQL

下面是具体的参数绑定细节

**单参数**（基本类型 / 包装类 / String）

- MyBatis 会自动封装成 `ParamMap`：

  ```
  {"param1": param, "value": param}
  ```

- SQL 中占位符：

  ```
  SELECT * FROM user WHERE id = #{param1}  或 #{value}
  ```

  paramMap 中的 param1 会匹配 sql 中的 param1

- **宽容机制**：

  - 如果你写 `#{任意名字}`，单值参数会直接绑定到占位符上。

**多参数**（Object[] / List / 多个参数）

- MyBatis 自动封装成 `ParamMap`：

  ```
  param1 -> 第一个参数
  param2 -> 第二个参数
  ...
  ```

- 和 SQL 中 `#{param1}`、`#{param2}` 对应

- 不加 `@Param` 的话，不能直接用自定义名字

如果是**对象参数**（JavaBean）

- mybatis 会直接通过反射获取属性值，将属性名和占位符匹配：
  - `#{name}` → `user.getName()`
  - `#{age}` → `user.getAge()`

如果是**Map 参数**

- 直接按 key 查找：
  - `#{name}` → `map.get("name")`
  - `#{age}` → `map.get("age")`

可以看到，可以传入的参数类型是很多的

上面讲了非代理方法，下面说一下常用的代理方法

当我们执行完上面的代码，看到查询结果时会发现

```sh
User{id=1, userName='null', age=18}
```

名字的值为空，但是数据库中的值确实是存在的。这就要说到**结果集映射机制**

整体流程如下

```sh
SQL 执行 → JDBC ResultSet → ResultSetHandler → TypeHandler → Java 对象 → 返回给调用者
```

JDBC PreparedStatement 执行完 SQL后返回 `ResultSet`，**ResultSetHandler** 类会获取结果，读取 ResultSet 的每一行并对其执行**行到对象的映射**

我们在 Mapper XML 中声明了返回对象的类型`<resultType="User">`，MyBatis 会实例化该对象。然后根据 **ResultMapping** 找到对应的 Java 属性，调用对应的 **TypeHandler** 将列值转换为 Java 类型并将其填充到实例化的对象中。

如果有嵌套对象 (`<association>` / `<collection>`)

MyBatis 会递归调用 ResultSetHandler

如果有 Map 返回类型，会把列名当 key，值填充到 Map 中

最后返回结果

- 如果是 `selectOne()`
  - 返回单个对象
- 如果是 `selectList()`
  - 返回 List<JavaObject>
- 如果是 Map
  - 返回 Map<key, value>

执行流程图：

```sh
SQL 执行 → JDBC ResultSet
         │
         ▼
ResultSetHandler.handleResultSets()
         │
         ▼
遍历 ResultSet 每行 → 创建 Java 对象
         │
         ▼
遍历每列 → TypeHandler 转换值
         │
         ▼
列值设置到对象属性 → 完成一行映射
         │
         ▼
集合封装 → 返回给调用者
```

**ResultMapping** 如何处理类和数据库列的匹配

如果 Mapper XML 只写了 `resultType="User"`，MyBatis 会将数据库**列名和属性名做匹配**：

如果是**驼峰映射**

- 数据库列名 `user_name` → Java 属性 `userName`
- 数据库列名 `id` → Java 属性 `id`

但是它默认不开启，需要自己手动配置，这就是为什么我们的 userName 属性拿不到值的原因

```xml
<configuration>
  <settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
  </settings>
</configuration>
```

我们也可以给查询的数据做别名，这样也能正确的匹配

```xml
<select id="findById" resultType="com.jh.entity.User">
    select id, user_name userName, age from user where id = #{id}
</select>
```

当列名和属性名不一致，或者有嵌套对象时，也可以用 `<resultMap>` 明确映射

```xml
<resultMap id="userMap" type="User">
    <id column="user_id" property="id"/>
    <result column="user_name" property="userName"/>
    <result column="age" property="age"/>
</resultMap>

<select id="findUser" resultMap="userMap">
    SELECT user_id, user_name, age FROM user
</select>
```

接下来我们将 demo 修改成动态代理 Mapper 的模式

非动态代理模式，每个 Mapper 都要手动实现

```java
public interface UserMapper {
    User findById(int id);
}

public class UserMapperImpl implements UserMapper {
    private SqlSession sqlSession;
    public UserMapperImpl(SqlSession sqlSession) {
        this.sqlSession = sqlSession;
    }
    @Override
    public User findById(int id) {
        return sqlSession.selectOne("UserMapper.findById", id);
    }
}
```

而动态代理 Mapper 接口，只需要调用`getMapper()`返回**动态生成的代理对象**无需实现类即可直接执行 SQL，并统一处理参数、结果、缓存和事务，极大简化开发和维护成本。

```java
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
User user = mapper.findById(1);
```

如果希望使用 mybatis 通过的动态代理的接口，就需要 namespace 中的值和需要对应的 Mapper 接口的全路径一致。Mapper中namespace 的定义本身是没有限制的，只要不重复即可，但如果使用 mybatis 的动态代理，则 namespace 必须为 mapper 接口的全路径

```xml
<mapper namespace="com.jh.mapper.UserMapper">
</mapper>
```

mapper 类

```java
public interface UserMapper {

    List<User> findAll();

    User findById(int id);

    void save(User user);

    void delete(int id);

    void update(User user);
}
```

使用

```java
public class Main {

    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
            SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
            try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
                UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
                List<User> users = userMapper.findAll();
                for (User user : users) {
                    System.out.println(user);
                }
            }
        }
    }
}
```

生成 Mapper 类的代理过程是怎样的

```sh
Mapper 接口方法调用
          │
          ▼
MapperProxy.invoke()
          │
          ▼
MappedStatement ← Mapper XML / 注解
          │
          ▼
SqlSession 执行 SQL
          │
          ▼
Executor → StatementHandler → JDBC PreparedStatement
          │
          ▼
ParameterHandler 绑定参数 → ? 占位符
          │
          ▼
SQL 执行 → ResultSet
          │
          ▼
ResultSetHandler → TypeHandler → Java对象
          │
          ▼
MapperProxy 返回结果 → 业务层
```

当调用该方法时

```
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
```

`sqlSession.getMapper(Class<T> type)` 内部其实执行的是这样的程序

```
MapperProxyFactory<T> factory = new MapperProxyFactory<>(type);
return factory.newInstance(sqlSession);
```

通过 **MapperProxyFactory** 创建代理对象 `MapperProxy` 接着调用时

```
User user = mapper.findById(123);
```

MapperProxy 的 `invoke()` 方法会被触发。该程序会获取`findById`**方法对应的 MappedStatement**，然后执行和非代理方法`selectOne()`的流程

**`MapperProxyFactory`**

- 负责为 Mapper 接口生成 **动态代理对象**。
- 不需要手写实现类，就可以让接口方法直接执行 SQL。

**`MapperProxy`**

实际创建代理对象使用的是 **JDK 动态代理**：

```
Proxy.newProxyInstance(
    type.getClassLoader(),
    new Class[]{type},
    new MapperProxy<>(sqlSession, type, methodCache)
)
```

- 代理对象类型是 **MapperProxy**，实现了 Mapper 接口
- 拦截所有接口方法调用

 当调用`MapperProxy.invoke()`时，所有 Mapper 方法调用都会被代理对象的 `invoke()` 方法拦截，接着将接口方法调用转化为 MyBatis SQL 执行流程

**我们再看一下`MappedStatement`的生成和使用**

作用

- 核心对象，记录每个 Mapper 方法对应的 SQL 与映射规则
- 包含信息：
  - SQL 语句或 SQL ID（namespace + id）
  - SQL 类型（SELECT / INSERT / UPDATE / DELETE）
  - 返回类型（resultType / resultMap）
  - 参数类型（parameterType）
  - 缓存策略 / KeyGenerator / fetchSize / timeout 等

生成流程

1. **全局配置加载时**：
   - `SqlSessionFactory` 解析 Mapper XML
   - 每个 `<select>` / `<insert>` / `<update>` / `<delete>` → 封装为 `MappedStatement`
   - 存储在 `Configuration.mappedStatements` Map 中
2. **使用阶段**：
   - **MapperProxy.invoke()** 调用方法时：
     - 根据方法 ID 从 Configuration 获取 `MappedStatement`
   - **Executor** 执行 SQL 时：
     - 从 `MappedStatement` 获取 SQL 文本 / 参数映射 / ResultMap
   - **ParameterHandler**：
     - 读取 MappedStatement 的参数信息，绑定方法参数到 SQL 占位符
   - **ResultSetHandler**：
     - 读取 MappedStatement 的返回类型 / ResultMap，将 JDBC ResultSet 转 Java对象

**讲一下 Executor** 执行器

MyBatis 把 SQL 解析、参数绑定、结果映射的职责拆分到不同类，最后由Executor 统一调度。

```sh
Executor
   │
   ├─ StatementHandler.prepare() → 生成 JDBC PreparedStatement
   │
   ├─ ParameterHandler.setParameters() → 将方法参数绑定到 ? 占位符
   │
   ├─ StatementHandler.execute() → 执行 SQL
   │
   └─ ResultSetHandler.handleResultSets() → 将 ResultSet 映射成 Java 对象
```

现在来描述一下 Executor 调用 ParameterHandler、StatementHandler、ResultSetHandler 等组件完成 SQL 执行和结果映射的过程：

**Executor** 是 MyBatis 执行 SQL 的顶层入口，它会接收 MappedStatement、参数、ResultHandler 等信息，调用 StatementHandler 完成 SQL 执行。

**StatementHandler** 在执行时会准备 SQL 语句，即准备 PreparedStatement 或 CallableStatement 执行类，然后 **ParameterHandler**

会将方法参数集 **ParaMap** 映射到 SQL 占位符 `?`，会使用 TypeHandler 处理类型转换，最终的效果会是：

```
preparedStatement.setInt(1, 123);
preparedStatement.setString(2, "abc");
```

执行**PreparedStatement.execute()**，JDBC 执行 SQL → 返回 ResultSet

**ResultSetHandler**，会将 JDBC ResultSet 转换成 Java 对象

最终返回结果

##### 使用 `@Param` 注解优化参数映射

在使用 mapper 动态代理的时候可以使用`@Param("xxx")` 为参数指定名字，这时SQL 中可以直接使用名字，增强可读性

```java
public interface UserMapper {
    User findByIdAndName(@Param("id") int id, @Param("name") String name);
}
```

XML:

```xml
<select id="findByIdAndName" resultType="User">
    SELECT * FROM user WHERE id = #{id} AND name = #{name}
</select>
```

MyBatis 会将方法参数包装成 `ParamMap`：

```
{id=1, name="Tom"}
```

占位符 `#{id}` 和 `#{name}` 就可以正确匹配

> 拦截器：
>
> MyBatis 允许在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：
>
> ```
> Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
> ParameterHandler (getParameterObject, setParameters)
> ResultSetHandler (handleResultSets, handleOutputParameters)
> StatementHandler (prepare, parameterize, batch, update, query)
> ```
>
> 现在一些MyBatis 插件比如PageHelper都是基于这个原理，有时为了监控sql执行效率，也可以使用插件机制

`#{}`是占位符（PreparedStatement 参数绑定），会被替换为 `?`，通过 **PreparedStatement.setXXX()** 绑定参数

特点：

1. **防止 SQL 注入**
2. 与参数名 **可对应**（当有 `@Param` 时使用名字匹配）
3. 单参数方法时，**自动对应参数值**

`${}`本质是**字符串拼接**，不是占位符，它会直接把参数值拼到 SQL 中。常用于 **动态表名、列名**，不做类型转换，也不防 sql 注入

#### MyBatis 支持动态 sql

**if** 条件如果成立，则把 sql 字符串拼接上

```xml
<select id="queryUserList" resultType="com.zpc.mybatis.pojo.User">
    select * from tb_user WHERE sex=1
    <if test="name!=null and name.trim()!=''">
      and name like '%${name}%'
    </if>
</select>
```

**choose when otherwise** 相当于 Java 的 switch

```xml
<select id="queryUserListByNameOrAge" resultType="com.zpc.mybatis.pojo.User">
    select * from tb_user WHERE sex=1
    <!--
    1.一旦有条件成立的when，后续的when则不会执行
    2.当所有的when都不执行时,才会执行otherwise
    -->
    <choose>
        <when test="name!=null and name.trim()!=''">
            and name like '%${name}%'
        </when>
        <when test="age!=null">
            and age = #{age}
        </when>
        <otherwise>
            and name='鹏程'
        </otherwise>
    </choose>
</select>
```

**where** 多条件查询

```xml
<select id="queryUserListByNameAndAge" resultType="com.zpc.mybatis.pojo.User">
    select * from tb_user
    <!--如果多出一个and，会自动去除，如果缺少and或者多出多个and则会报错-->
    <where>
        <if test="name!=null and name.trim()!=''">
            and name like '%${name}%'
        </if>
        <if test="age!=null">
            and age = #{age}
        </if>
    </where>
</select>
```

**foreach** 遍历

```xml
<!-- 将传入的 String[] 遍历，然后拼接到了 sql 上 -->
<select id="queryUserListByIds" resultType="com.zpc.mybatis.pojo.User">
    select * from tb_user where id in
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</select>
```

**如果存在对象嵌套**，结果集的映射该如何处理

```java
class User {
    private String name;
    private Account account;
}
```

使用`<resultMap>`来映射字段

#### MyBatis 的缓存

MyBatis 有 **两级缓存**：

| 缓存级别 | 存储范围               | 生命周期                             | 使用对象           |
| -------- | ---------------------- | ------------------------------------ | ------------------ |
| 一级缓存 | SqlSession（会话级）   | 与 SqlSession 同生命周期             | 默认开启，无需配置 |
| 二级缓存 | Mapper（namespace 级） | Mapper 对应的 Configuration 生命周期 | 默认关闭，需要配置 |

- **一级缓存**：同一个 SqlSession 内查询的相同 SQL、相同参数会从缓存取结果
- **二级缓存**：跨 SqlSession，也就是多个 SqlSession 共享缓存

二级缓存概念

- **缓存级别**：Mapper Namespace
- **存储位置**：`Cache` 接口的实现（默认是 PerpetualCache + LRU 策略）
- **数据来源**：
  - 查询操作返回的结果会被放入二级缓存
  - 更新/插入/删除操作会清空相关缓存（默认清空 Mapper 对应 namespace 的缓存）
- **作用**：减少数据库访问，提高性能，跨会话共享查询结果

使用二级缓存

配置缓存 在Mapper XML 中开启

```
<mapper namespace="com.example.UserMapper">
    <cache
        eviction="LRU"      <!-- 缓存回收策略（LRU、FIFO、Soft、Weak） -->
        flushInterval="60000" <!-- 自动刷新间隔，单位毫秒 -->
        size="512"           <!-- 缓存对象数量上限 -->
        readOnly="false"/>   <!-- 是否只读 -->
</mapper>
```

配置说明

| 属性          | 说明                                         |
| ------------- | -------------------------------------------- |
| eviction      | 缓存回收策略                                 |
| flushInterval | 自动刷新间隔                                 |
| size          | 缓存条目上限                                 |
| readOnly      | 是否只读（只读可直接返回对象，否则需要拷贝） |

Mapper 查询方法自动使用二级缓存，更新方法默认会清空当前 namespace 的缓存

```
List<User> users = userMapper.findAll(); // 查询结果放入二级缓存
```

**mybatis 缓存**只**适合**：

- 读多写少的表
- 查询结果复用率高

**不适合**：

- 高并发频繁更新表
- 对实时性要求非常高的场景

接下来，我们使用 Spring 来简化使用 MyBatis 时需要的配置

## 在纯 Spring 中配置 MyBatis

Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.15</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>2.2.2</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.1.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.3.30</version>
    </dependency>
</dependencies>
```

`mybatis-spring`是 spring 集成 myabtis 需要的依赖

`mysql-connector-j`是 MySQL JDBC 驱动，提供数据库连接

`spring-jdbc` 在这里的作用主要是 **提供 Spring 对 JDBC 的支持和封装**，可以更方便地管理数据库连接、事务，以及避免手动编写大量 JDBC 样板代码。

`spring-jdbc` 提供了：

1. **DataSource 管理**
   - Spring 可以创建和管理数据库连接池（如 HikariCP、DBCP）
2. **JdbcTemplate**
   - 封装了 JDBC 的查询、更新、批量操作
   - 自动管理连接、PreparedStatement、ResultSet 的关闭
   - 异常统一转换为 Spring DataAccessException 系列
3. **事务支持**
   - Spring 的声明式事务（`@Transactional`）依赖于 JDBC 事务管理器
   - `spring-jdbc` 提供了 `DataSourceTransactionManager`事务管理器
4. **异常处理**
   - JDBC 异常被转换成 Spring 的统一异常体系，不用手动 try-catch SQLExceptions

虽然 MyBatis 本身可以直接操作 JDBC，但在 **Spring 集成 MyBatis** 时，`spring-jdbc` 主要作用是：

1. **提供数据源和事务管理**

```java
@Bean
public DataSourceTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

- `DataSourceTransactionManager` 来自 `spring-jdbc`
- 支持 Spring 的声明式事务 (`@Transactional`)

1. **封装 JDBC 资源管理**

- MyBatis 的 `SqlSessionFactoryBean` 内部会依赖 Spring 的 `DataSource`
- Spring 自动管理连接释放、事务提交/回滚

使用 spring 注解的方式配置 MyBatis

使用 **`@Configuration` 注解类**

```java
@Configuration
@MapperScan("com.example.mapper") // 扫描 Mapper 接口
@EnableTransactionManagement // 开启注解事务
public class MyBatisConfig {

    @Bean
    public DataSource dataSource() {
        // 使用 HikariCP 也可以
        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/testdb?useSSL=false&serverTimezone=UTC");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        return dataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        return factoryBean.getObject();
    }

    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

基础的 mybatis 是在`myabtis-config.xml`配置数据源和事务。在 spring 项目中，不但把这些配置使用注解配置，而且把`SqlSession`都交由 spring 容器管理，极大简化了配置方式和代码耦合度

Mapper 接口使用注解

```java
@Mapper
public interface UserMapper {

    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(@Param("id") int id);

    @Insert("INSERT INTO user(name, age) VALUES(#{name}, #{age})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(User user);

    @Update("UPDATE user SET name=#{name}, age=#{age} WHERE id=#{id}")
    int update(User user);

    @Delete("DELETE FROM user WHERE id=#{id}")
    int delete(@Param("id") int id);
}
```

`@Mapper` 是 **MyBatis 提供的注解**，用于标记一个**接口是 Mapper 接口**

Spring 扫描到带有 `@Mapper` 的接口后，会**为接口生成代理对象**（MapperProxy），并将其注册到 Spring 容器中

它的作用等价于原来 MyBatis XML 配置中的 `<mapper resource="mapper/UserMapper.xml"/>`

spring 中，无需显示的调用管理 SqlSessin，执行 SQL 时由 MyBatis-Spring 自动获取、关闭 SqlSession

使用 `@Mapper` 后，可以配合 `@MapperScan("包路径")` 自动扫描整个包的 Mapper 接口

> 这两者可以单独使用，一起使用效果更好

`@MapperScan` 是 MyBatis-Spring 提供的扫描器，它的作用是扫描指定包下的接口，**为每个接口生成 MapperProxy 代理对象**并注册到 Spring 容器中，相当于给接口打上了 `@Component` 效果。

`@Select()`是使用注解的方式来实现简单的 sql 查询，之前的`mapper.xml`方式仍保留，且不做变化，适用于复杂 sql

Service 层调用

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Transactional
    public User getUser(int id) {
        return userMapper.findById(id);
    }
}
```

`@Transactional` 注解

**声明式事务**

- 概念：通过**配置或注解**来声明哪些方法需要事务管理，由 Spring 框架在运行时自动控制事务的开启、提交和回滚。

- **特点**：
  1. **无需手动编写 commit/rollback**
  2. **业务代码和事务逻辑分离**
  3. 可针对类、方法级别控制事务

原理

Spring 的声明式事务底层依赖 **AOP（代理）机制**：

1. 给目标方法使用事务注解
2. Spring 为目标对象生成代理（JDK 动态代理或 CGLIB）
3. 调用方法时：
   - 代理先开启事务（TransactionManager）
   - 执行业务方法
   - 方法正常返回 → 提交事务
   - 方法抛异常 → 回滚事务

## Spring Boot 项目中配置 MyBatis

Maven 依赖

Spring Boot 提供了 **starter**，大大简化依赖配置：

```xml
<dependencies>
    <!-- Spring Boot MyBatis starter -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>3.0.2</version>
    </dependency>

    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.1.0</version>
    </dependency>
</dependencies>
```

`mybatis-spring-boot-starter` 是 **MyBatis 官方提供的 Spring Boot Starter**，核心作用是：

1. **自动配置 MyBatis**
   - 自动创建 `SqlSessionFactory`
   - 自动创建 `SqlSessionTemplate`（线程安全的 SqlSession）
   - 自动扫描 Mapper 接口并生成 MapperProxy
2. **整合 Spring Boot 特性**
   - 可以直接使用 Spring Boot 的 **数据源配置**（`spring.datasource.*`）
   - 支持 Spring 的 **声明式事务**（`@Transactional`），并且也是自动创建配置

Spring Boot 推荐使用 **application.properties** 或 **application.yml**：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    # 可选：连接池配置（HikariCP 默认）
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 30000
      pool-name: HikariCP

mybatis:
  type-aliases-package: com.example.entity   # 实体类包
  mapper-locations: classpath*:mapper/*.xml  # XML Mapper 文件位置
  configuration:
    map-underscore-to-camel-case: true       # 下划线转驼峰
    cache-enabled: true                       # 启用二级缓存
    lazy-loading-enabled: false               # 延迟加载
    aggressive-lazy-loading: false
    multiple-result-sets-enabled: true
    use-generated-keys: true
    default-executor-type: SIMPLE

# 可选：日志配置
logging:
  level:
    com.example.mapper: debug
```

Mapper 接口仍然使用 @Mapper 注解方式

Spring Boot 与纯 Spring 项目相比，简化点主要有：

| 功能              | 纯 Spring 配置                        | Spring Boot 配置                     |
| ----------------- | ------------------------------------- | ------------------------------------ |
| SqlSessionFactory | 手动配置 Bean                         | 自动配置，无需手动定义               |
| SqlSession        | 手动获取                              | 自动注入 Mapper 接口，代理对象已管理 |
| 数据源            | 手动配置 DataSource Bean              | 配置 application.yml 即可            |
| 事务管理          | 手动配置 DataSourceTransactionManager | 自动配置，可用 `@Transactional`      |

启动一个 Spring Boot 项目

```java
@SpringBootApplication
@MapperScan("com.example.mapper")
public class AppBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(AppBootApplication.class, args);
    }
}
```

Service 层：

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Transactional
    public User getUser(int id) {
        return userMapper.findById(id);
    }
}
```

## MyBatis-Plus
