# Java 集合

Java 集合用于保存数量不确定的数据，以及保存具有映射关系的数据。它只能保存对象

Java 集合类主要由两个根接口 `Collection` 和 `Map` 类中派生出来的。`Collection` 派生出三个子接口 `List`、`Set`、`Queue`

`List` 常用实现类有 `ArrayList`、`LinkedList` 

`Queue` 的常用实现类有 `LinkedList`、`ArrayDeque`

`Set` 的常用实现类有 `HashSet`、`TreeSet`

`Map` 的常用实现类有 `TreeMap` 

以上说的都是 `java.util` 包下的，而如果是支持多线程集合的类位于 `java.util.concurrent` 包下

`CopyOnWriteArrayList` 线程安全的 List，基于写时复制，适合读多写少的场景

`CopyOnWriteArraySet` 线程安全的 Set，基于 `CopyOnWriteArrayList` 实现。适合读多写少场景适合。

`ConcurrentSkipListSet` 基于跳表。支持排序，性能比 `TreeSet` 更适合并发环境
`ConcurrentHashMap` 最常用的并发 Map

线程安全的 **Queue / Deque**

- **ArrayBlockingQueue** 有界队列，基于数组，内部用锁。
- **LinkedBlockingQueue** 有界/无界队列，基于链表，常用于生产者-消费者





