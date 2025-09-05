# AQS（AbstractQueuedSynchronizer）

AQS 是一个抽象类，为 Java 一系列锁以及同步器或者同步对象的底层提供了实现的框架。

AQS 的核心思想是利用一个双向队列来保存等待锁的线程，同时利用一个 state 变量来表示锁的状态。AQS 同步器可以分为独占模式和共享模式两种。

- 独占模式是指同一时刻只允许一个线程获取锁，常见的实现类有 ReentrantLock；
- 共享模式是指同一时刻允许多个线程同时获取锁，常见的实现类有 Semaphore、CountDownLatch、CyclicBarrier 等；