# Redis 消息队列

## 基于 List 实现消息队列

redis 的数据结构可以实现消息队列的入队和出队操作。生产者将消息队列插到 List 的尾部，消费者从 List 的头部获取消息，实现先进先出的消息处理。

生产者：

```sh
# 把消息 "task1" 放到队列 "queue" 右边
rpush queue "task1"
rpush queue "task2"
rpush queue "task3"
```

消费者

```sh
# 取出队列最左边的消息
lpop queue

# 如果队列为空，阻塞等待。0 表示一直阻塞直到有消息
brpop queue 0 
```

> 使用 List 实现的消息队列没有 ACK 机制，消费者一旦 POP 出来，消息就从队列里移除，如果消费者挂了，消息就丢失了

## 使用 Redis 的发布订阅模式

redis 的发布订阅模式是一种通信模式，发送者发送消息，订阅者接收消息。模式有两种

- 基于频道(Channel)的发布/订阅
- 基于模式(pattern)的发布/订阅