# Cannal

### Canal 是什么

**Canal** 是阿里巴巴开源的 **基于数据库 binlog 的增量订阅 & 消费组件**。

最早是为了解决 **MySQL 和 HBase/ES 的数据同步**问题。

它会监听 MySQL 的 **binlog** 日志，把数据库的增删改操作实时解析出来，推送给其他系统。

### Canal 的工作原理

**模拟 MySQL 从库（Slave）**

>  MySQL 主从复制的原理：主库写 binlog，从库通过 **dump 协议**拉取 binlog。

Canal 伪装成 MySQL 的一个从库，**订阅 binlog**。拿到 binlog 日志后，**解析**出行级别的变更信息（insert/update/delete）。

然后，Canal 把解析后的数据以事件（event）的形式**投递到下游系统**：

- Kafka / RocketMQ
- ElasticSearch
- HBase / ClickHouse
- 业务自定义消费者

### Canal 的应用场景

1. **数据库同步**
   - MySQL 到 MySQL（跨机房、跨库实时同步）
   - MySQL 到 Oracle / PostgreSQL / ClickHouse
2. **数据库与搜索引擎同步**
   - MySQL 到 ElasticSearch（做搜索系统时，数据实时更新到 ES）
3. **数据库与缓存同步**
   - MySQL 到 Redis（缓存预热、更新保持实时一致）
4. **审计与监控**
   - 监听敏感表的变更，实时触发告警。

### Canal 的架构

```markdown
          ┌─────────────┐
          │   MySQL主库  │
          └──────┬──────┘
                 │ binlog
                 ▼
          ┌─────────────┐
          │   Canal     │  (伪装成Slave)
          ├─────────────┤
          │ binlog解析器 │
          │ 数据存储队列 │
          │ 消费者适配器 │
          └──────┬──────┘
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
  ElasticSearch  Kafka     Redis/HBase
```

组件说明：

- **Canal Server**：核心服务，负责连接 MySQL、拉取并解析 binlog。
- **Canal Instance**：一个数据订阅通道（对应一个数据库实例）。
- **Canal Client**：消费 Canal 解析后的数据。

### Canal 的部署和使用

#### 部署

下载 canal，解压后修改配置：

```properties
# conf/example/instance.properties
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
```

需要在 MySQL 中创建一个账号，授予 **REPLICATION SLAVE** 权限：

```sql
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
```

#### 启动

```sh
sh bin/startup.sh
```

#### 消费数据

- 方式1：Java 客户端 API（直接拉取 binlog 事件）。
- 方式2：Kafka/RocketMQ 适配器（自动把数据投递到消息队列）。
- 方式3：Elasticsearch 适配器（直接同步到 ES）。

### 典型实战案例

1. **MySQL → Elasticsearch**

   业务数据库写入/更新商品信息

   Canal 监听 binlog，把变化推送到 ES

   搜索引擎里的商品数据保持实时一致

2. **MySQL → Redis 缓存**

   用户修改资料 → 写入数据库

   Canal 捕获 binlog → 更新 Redis 中的缓存

   避免出现缓存与数据库不一致问题

3. **MySQL → Kafka → Flink → 数据仓库**

   电商订单表的 binlog 通过 Canal 投递到 Kafka

   Flink 做实时计算（GMV、UV）

   汇总写入 HDFS/ClickHouse

> **多实例配置**
>
> Canal 中的 **一个 Instance 就是一个独立的数据订阅通道**，对应一个数据库的 binlog 拉取配置。一个 Canal Server 可以管理多个 Instance。每个 Instance 有独立的配置文件（`instance.properties`），互不干扰。
>
> 可以理解为 **一个数据库 = 一个 instance**。
>
> ##### 多实例的配置方式
>
> 实例目录结构
>
> 在 `canal/conf/` 目录下，默认有一个示例 instance：
>
> ```markdown
> conf/
>  ├── canal.properties        # 全局配置
>  └── example/                # 默认的 instance
>       └── instance.properties
> ```
>
> 如果要添加多个实例，就新增多个目录：
>
> ```markdown
> conf/
>  ├── canal.properties
>  ├── example/
>  │    └── instance.properties
>  ├── userdb/
>  │    └── instance.properties
>  └── orderdb/
>       └── instance.properties
> ```
>
> 以 `userdb/instance.properties` 为例：
>
> ```properties
> # 指定要监听的MySQL
> canal.instance.master.address=127.0.0.1:3306
> canal.instance.dbUsername=canal
> canal.instance.dbPassword=canal
> 
> # 指定要监听的数据库/表
> canal.instance.filter.regex=userdb\\..*
> ```
>
> 再比如 `orderdb/instance.properties`：
>
> ```properties
> canal.instance.master.address=192.168.10.21:3306
> canal.instance.dbUsername=canal
> canal.instance.dbPassword=canal
> 
> # 只监听订单表
> canal.instance.filter.regex=orderdb\\.t_order
> ```
>
> 启动多实例
>
> 在 `canal.properties` 中配置要加载的实例：
>
> ```
> canal.destinations=example,userdb,orderdb
> ```
>
> 启动 Canal 后，它会同时运行这些实例，每个实例独立去拉取对应 MySQL 的 binlog。

### 在 Spring 中集成 Canal

#### 引入依赖

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.7</version> <!-- 与Canal服务端版本匹配 -->
</dependency>
```

>  Canal 服务端和客户端的版本必须兼容
>
> 一定要启动 Canal 服务器

Java 实现

```java
/**
 * Canal 客户端示例
 * 作用：连接 Canal Server，实时拉取 MySQL binlog 并处理数据
 */
@Component
public class CanalClient implements InitializingBean, DisposableBean {

    /** Canal 连接器，用于与 Canal Server 建立连接 */
    private CanalConnector connector;

    /**
     * Bean 初始化时执行，相当于 @PostConstruct
     * 用于初始化 Canal 连接器，并开启监听线程
     */
    @Override
    public void afterPropertiesSet() {
        // 1. 创建连接器，连接到 Canal Server
        connector = CanalConnectors.newSingleConnector(
                new InetSocketAddress("127.0.0.1", 11111), // Canal Server 地址和端口
                "example", // instance 名称，对应 canal/conf/example
                "canal",   // 用户名（默认空）
                "canal");  // 密码（默认空）

        // 2. 启动独立线程持续监听 Binlog 事件
        new Thread(this::listen).start();
    }

    /**
     * 持续监听 MySQL Binlog 数据
     */
    private void listen() {
        connector.connect(); // 建立连接
        connector.subscribe("mydb\\..*"); // 订阅 mydb 库下的所有表（正则匹配）
        connector.rollback(); // 回滚到未确认的地方，避免丢失

        // 无限循环监听
        while (true) {
            // 批量获取数据，每次最多 100 条
            Message message = connector.getWithoutAck(100);
            long batchId = message.getId();

            if (batchId != -1 && !message.getEntries().isEmpty()) {
                // 有数据才处理
                handleMessage(message);
            }

            // 确认消息，告诉 Canal Server 已经成功消费
            connector.ack(batchId);
        }
    }

    /**
     * 处理一批 Binlog 事件
     */
    private void handleMessage(Message message) {
        for (CanalEntry.Entry entry : message.getEntries()) {
            // 只解析 RowData 类型的事件（即真正的数据变更）
            if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
                try {
                    // 解析变更数据
                    CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
                    CanalEntry.EventType eventType = rowChange.getEventType();
                    System.out.println("====> Binlog事件类型: " + eventType);

                    // 遍历每一行数据的变更
                    for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                        if (eventType == CanalEntry.EventType.INSERT) {
                            // 插入：打印 After 值
                            rowData.getAfterColumnsList()
                                   .forEach(c -> System.out.println(c.getName() + "=" + c.getValue()));
                        } else if (eventType == CanalEntry.EventType.UPDATE) {
                            // 更新：打印更新后的值
                            rowData.getAfterColumnsList()
                                   .forEach(c -> System.out.println(c.getName() + "=" + c.getValue()));
                        } else if (eventType == CanalEntry.EventType.DELETE) {
                            // 删除：打印 Before 值
                            rowData.getBeforeColumnsList()
                                   .forEach(c -> System.out.println(c.getName() + "=" + c.getValue()));
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * Bean 销毁时执行，关闭 Canal 连接
     */
    @Override
    public void destroy() {
        if (connector != null) {
            connector.disconnect();
        }
    }
}
```

这段代码实现了 Spring 项目启动时自动连接 Canal Server，订阅数据库 binlog 事件

> 其中使用了两个 Spring 框架中提供的生命周期接口 **`InitializingBean`** 和 **`DisposableBean`**
>
> `InitializingBean`
>
> ```java
> public interface InitializingBean {
>     void afterPropertiesSet() throws Exception;
> }
> ```
>
> 用于在 **Spring Bean 初始化完成后** 执行自定义逻辑。
>
> 使用场景
>
> - 建立连接（数据库、MQ、Canal 这种就是典型用法）。
> - 校验配置参数是否正确。
> - 启动异步任务。
>
> `DisposableBean`
>
> ```java
> public interface DisposableBean {
>     void destroy() throws Exception;
> }
> ```
>
> 用于在 **Spring 容器销毁 Bean 时** 执行收尾逻辑。
>
> 使用场景
>
> - 释放资源（关闭数据库连接、线程池、消息队列连接、文件句柄）。
> - 停止后台线程，避免内存泄漏。
>
> 在现代 Spring 项目中，更多的是使用注解 `@PostConstruct` 和 `PreDestory` 来实现初始化和销毁的作用
>
> ```java
> @Component
> public class MyBean {
> 
>     @PostConstruct
>     public void init() {
>         System.out.println("Bean 初始化完成");
>     }
> 
>     @PreDestroy
>     public void cleanup() {
>         System.out.println("Bean 即将销毁");
>     }
> }
> ```

#### 过滤表

默认情况下，Canal 会订阅 MySQL 实例下所有数据库、所有表的 binlog。有时候这是不可接受的（数据量过大、性能消耗大），所以必须通过 **正则表达式**来指定要监听的表。 

所以可以添加 `canal.instance.filter.regex` 配置，来实现表的过滤

语法规则

- 格式：`库名.表名`
- 支持正则表达式 (`.*`) 和多个表（用逗号分隔）

示例：

```properties
# 监听 userdb 库的所有表
canal.instance.filter.regex=userdb\\..*

# 监听 userdb.t_user 和 orderdb.t_order 两张表
canal.instance.filter.regex=userdb\\.t_user,orderdb\\.t_order

# 监听所有库下以 t_ 开头的表
canal.instance.filter.regex=.*\\.t_.*
```

注意：

- 点号 `.` 在正则里有特殊意义，必须写成 `\\.`
- 通配所有表时必须双反斜杠：`userdb\\..*`

配置排除表

- 使用 `canal.instance.filter.black.regex` 来排除某些表。

示例：

```properties
# 订阅所有表，但排除 binlog 记录表和日志表
canal.instance.filter.regex=.*\\..*
canal.instance.filter.black.regex=.*\\.(binlog_backup|sys_log)
```

#### Canal 与消息中间件结合使用

架构

```markdown
 MySQL (binlog)
       │
       ▼
  Canal Server (解析 binlog → 事件)
       │
       ▼
  Canal Adapter (kafka/rocketmq/rabbitmq)
       │
       ▼
  消息中间件 (Kafka / RocketMQ / RabbitMQ)
       │
 ┌─────┴────────────┐
 ▼                  ▼
ES/Redis          业务微服务
```

在生产环境中，Canal 一般不会让业务系统直接连它，而是 **把数据写入消息队列/ES**，由下游异步消费。

##### Canal → Kafka

使用 Canal Kafka Adapter

配置文件：`conf/application.yml`

```yaml
canal:
  server:
    mode: kafka
  kafka:
    bootstrap.servers: localhost:9092
    topic: canal-topic
    partition: 0
```

启动后，Canal 会自动把解析好的数据推送到 `canal-topic`。

Spring Boot 作为 Kafka 消费者即可：

```java
@Component
public class CanalKafkaConsumer {

    @KafkaListener(topics = "canal-topic", groupId = "test-group")
    public void listen(String message) {
        System.out.println("收到Canal消息：" + message);
    }
}
```

##### Canal → RocketMQ

```yaml
canal:
  server:
    mode: rocketmq
  rocketmq:
    nameServer: 127.0.0.1:9876
    topic: canal-topic
```

Spring Boot 使用 RocketMQ Starter 消费：

```java
@Component
@RocketMQMessageListener(topic = "canal-topic", consumerGroup = "canal-consumer")
public class CanalRocketMqConsumer implements RocketMQListener<String> {
    @Override
    public void onMessage(String message) {
        System.out.println("收到Canal消息：" + message);
    }
}
```

##### Canal → Elasticsearch

使用 Canal 提供的 **ElasticSearch Adapter**

配置 `es7/mytest_user.yml`

```yaml
dataSourceKey: defaultDS
destination: example
esMapping:
  _index: user_index
  _id: _id
  sql: "select id as _id, name, age from user"
```

启动 adapter 后，Canal 会自动把 MySQL 的数据写入 ES 索引。

**Canal + 消息中间件（Kafka / RocketMQ / RabbitMQ）** 的默认模式只是把 MySQL 的 binlog 事件变更丢到 MQ 里。
这些原始消息大多是 **RowChange 结构 / JSON 格式**，下游要真正能用，还需要 **进一步处理、转换和消费**。

Canal Server 解析 binlog 后，会输出类似的 JSON 消息到 Kafka/RocketMQ：

```json
{
  "database": "mydb",
  "table": "user",
  "pkNames": ["id"],
  "isDdl": false,
  "type": "UPDATE",
  "ts": 1695012345678,
  "sql": "",
  "before": {
    "id": "1001",
    "name": "Alice",
    "age": "20"
  },
  "after": {
    "id": "1001",
    "name": "Alice",
    "age": "21"
  }
}
```

字段含义：

- **database/table** → 来自哪个库、哪个表
- **type** → 事件类型（INSERT/UPDATE/DELETE/DDL）
- **before/after** → 修改前/修改后的行数据
- **ts** → 事件时间戳

它本质上就是一个 **通用的数据库行级别变更事件**。

为什么还需要进一步处理？

1. **数据结构差异**
   - MQ 消息是通用 JSON，业务系统需要转换成内部的 DTO/实体。
2. **业务逻辑需求**
   - 例如：
     - 下游 **缓存系统**（Redis）需要 key/value 形式
     - 下游 **搜索引擎**（ES）需要文档结构
     - 下游 **数据仓库**（ClickHouse/Flink）需要事实表/维度表
3. **幂等和顺序**
   - 下游消费时必须保证重复消费/乱序不会导致数据错误。

> 我们要知道，Canal **默认**传输的其实是它内部的 **Entry 对象（Protobuf 二进制序列化）**，而不是 JSON。
>
> **Canal 内部数据结构**是 `CanalEntry.Entry`（Google Protobuf 定义的）。
>
> 如果在 `canal.properties` 或 MQ 配置中 **没有配置** `canal.mq.flatMessage=true`，Canal 会直接把 **Protobuf 二进制消息**推送到 Kafka / RocketMQ。
>
> 下游消费者拿到的就是字节数组，需要自己用 Protobuf 反序列化，代码大概是：
>
> ```java
> CanalEntry.Entry entry = CanalEntry.Entry.parseFrom(messageBytes);
> ```
>
> 为了让下游业务更容易消费（特别是 Kafka、RocketMQ 这种场景），Canal 提供了一个“扁平化消息格式”：
>
> ```properties
> # canal.properties
> canal.mq.flatMessage = true
> ```
>
> 这样，Canal 会把解析出来的 binlog 事件，转换成 **FlatMessage 对象**，再序列化为 JSON 字符串，推送到 MQ。
>
> 下游就能直接消费 JSON，比如：
>
> ```json
> {
>   "database": "mydb",
>   "table": "user",
>   "type": "UPDATE",
>   "data": [{"id":"1001","name":"Alice","age":"22"}],
>   "old": [{"age":"21"}]
> }
> ```

**Canal 如何和 Elasticsearch 对接** 的呢

Canal 自身只做 **binlog 捕获和解析**，真正和 ES 交互，需要依赖 **canal.adapter**。

`canal.adapter` 是 Canal 官方提供的一个 **数据同步插件系统**。

它内置了一些 **转换器（Adapter/Plugin）**，能把 binlog 事件转成特定系统能接受的数据格式。

对于 ES，Canal 提供了 **ES Adapter**，会把数据库的行数据转成 ES 的文档。

Canal 提供的 ES 转换器在 `canal.adapter` 包里，主要有以下两类适配器：

1. **ES Adapter（es7 / es6）**
   - 直接将 MySQL 表的数据映射到 Elasticsearch 索引。
   - 自动处理 **INSERT/UPDATE/DELETE** → ES 的 **index/update/delete**。
   - 支持 SQL 映射配置，可以灵活选择表字段与 ES 文档的对应关系。
2. **通用 RDB → ES 映射器**
   - 使用配置文件 `yml` 定义 MySQL → ES 的映射规则。
   - 支持多表映射、字段重命名、函数计算。

配置示例（MySQL → ES）

在 `canal.adapter/conf/es7/` 下新建配置文件，比如 `user.yml`：

```yaml
dataSourceKey: defaultDS  # 数据源
destination: example      # 对应 canal.instance 名称
groupId: g1

esMapping:
  _index: user_index       # ES 索引名
  _id: _id                 # ES 主键
  sql: "select id as _id, name, age, email from user"  # SQL 定义映射
  etlCondition: "where id > 0"
  commitBatch: 3000
```

- `dataSourceKey`：数据源配置，定义在 `application.yml`
- `destination`：对应 canal.instance 名称
- `sql`：决定同步哪些字段、如何映射到 ES
- `etlCondition`：ETL 初始条件（全量同步时使用）

运行机制

1. **Canal Server** 采集 MySQL binlog → 生成 Entry/FlatMessage。
2. **Canal Adapter (ES)** 订阅 Canal Server 的数据。
3. 按照 `esMapping` 配置，把数据转成 **Elasticsearch JSON 文档**：
   - INSERT → `index`
   - UPDATE → `update`
   - DELETE → `delete`

```markdown
MySQL → Canal Server → Canal Adapter(ES) → Elasticsearch
```

