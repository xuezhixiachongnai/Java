# NIO

在 1.4 版本之前，Java IO 类是阻塞式的；从 1.4 版本开始，引进了新的异步 IO 库，被称为 Java New IO 类库，简称为 Java NIO

Java NIO 类库包含以下三个核心组件：

- Channel（通道）
- Buffer（缓冲区）
- Selector（选择器）

Java NIO 属于 IO 多路复用模型

NIO 和传统的 IO（OIO）相比，OIO是面向流的，read() 操作总是以流的方式顺序的从一个流中读取一个或多个字符，它是顺序的，不能随意的改变读取指针的位置，也不能前后移动流中的数据。

而 NIO 引入了 Channel（通道）和 Buffer（缓冲区）的概念，读取和写入都是对缓冲区进行交互。用户程序只需要从通道中读取数据到缓冲区中，或将数据从缓冲区中写入到通道中。NIO不像OIO那样是顺序操作，可以随意地读取Buffer中任意位置的数据，可以随意修改Buffer中任意位置的数据。

OIO 调用 read()、write() 时，该线程被阻塞，只有数据被读取或者写入完成后，该线程才能干其他事，否则在此期间将一直等待。

而 NIO 在调用 read() 时系统底层已经把数据准备到通道了，应用程序只需要将数据从通道复制到缓冲区即可，即使没有准备好，当前线程也可以执行其他操作，无需等待。

### Buffer

缓冲区主要与通道之间进行数据的读写操作。通道提供文件、网络读取数据的渠道，但是读写的数据都会经过缓存。

读操作是指将数据从通道读取到缓存区中；写操作是将数据从缓存区写入通道。它本质是上就是一块内存。

Buffer 类是一个抽象类，对应 Java 的主要数据类型在 NIO 有 8 种 缓冲区类，分别是：ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer、LongBuffer、ShortBuffer、MappedByteBuffer。MappedByteBuffer是专门用于内存映射的一种ByteBuffer类型。不同的Buffer子类，其能操作的数据类型能够通过名称进行判断，比如IntBuffer只能操作Integer类型的对象。

Buffer 基类并没有定义缓存内存区的字段，这个字段被具体定义到了子类种。如ByteBuf子类就拥有一个byte[]类型的数组成员final byte[] hb，作为自己的读写缓冲区，数组的元素类型与Buffer子类的操作类型相互对应。

Buffer 类额外提供了一些重要属性来记录读写的状态和位置：

- capacity（容量）：表示缓存的大小，一旦写入的对象超过了 capacity，缓冲区就满了，不能再写入了。Buffer 类的对象在初始化时会按照 capacity 分配内部数组的内存，数组一旦确定大小，之后就不能被改变。

- position（读写位置）：表示当前位置，它的属性值与缓冲区的读写模式有关

  > 当新建一个缓冲区实例时，缓冲区处于写入模式，这时时可以写入数据的。在写入完数据之后，需要调用 **flip()** 切换到读模式才能读取缓冲区中的数据。
  >
  > 在刚进入写模式的时候，position 的值为 0，表示当前的写入位置从头开始；每当一个数据写到缓冲区之后，position 会向后移动到下一个可写的位置；当 position 的值达到 limit 时，表示缓存空间满了。
  >
  > 在转变到读模式后。limit 的属性值被设置成写模式的 position 值，然后 position 被设置成 0，每读取后 position 会向后移动，直到达到 limit。

- limit（读写的限制）：表示可以写入或者读取的最大上限，其属性值的具体含义，也与缓冲区的读写模式有关，在不同的模式下，limit的值的含义是不同的

  - 在写入模式下，limit属性值的含义为可以写入的数据最大上限。在刚进入到写入模式时，limit的值会被设置成缓冲区的capacity容量值，表示可以一直将缓冲区的容量写满
  - 在读取模式下，limit的值含义为最多能从缓冲区中读取到多少数据

- mark（标记）：缓冲区操作过程当中，可以将当前的position的值临时存入mark属性中；需要的时候，可以再从mark中取出暂存的标记值，恢复到position属性中，重新从position位置开始处理

**Buffer 类的重要方法**

通过`allocate()`方法来获取一个 Buffer 实例对象

```java
public class BufferTest {
    
    static IntBuffer buffer = null;

    static void initBuffer(int capacity) {
        buffer = IntBuffer.allocate(capacity);
    }

    static void getLog() {
        System.out.println("capacity: " + buffer.capacity());
        System.out.println("position: " + buffer.position());
        System.out.println("limit: " + buffer.limit());
        System.out.println("----------------");
    }
    public static void main(String[] args) {
        initBuffer(20);
        getLog();
    }
}
```

打印的结果是：

```sh
capacity: 20
position: 0
limit: 20
```

通过`put()`方法将数据写入缓冲区

调用`allocate()`方法分配内存，返回实例对象后，缓冲区处于写模式，这时调用`put()`方法，就可以向其写入数据

```java
    static void initBuffer(int capacity) {
        buffer = IntBuffer.allocate(capacity);
        for (int i = 0; i < 5; i++) {
            buffer.put(i);
            getLog();
        }
    }
```

```sh
capacity: 20
position: 1
limit: 20
----------------
capacity: 20
position: 2
limit: 20
----------------
capacity: 20
position: 3
limit: 20
----------------
capacity: 20
position: 4
limit: 20
----------------
capacity: 20
position: 5
limit: 20
----------------
```

从结果也能够看出 position 的变化

如果不使用`flip()`改变写入模式直接调用`get()`的化是会直接返回当前 position 的值，并执行`position++`

```java
public static void main(String[] args) {
    initBuffer(20);
    for (int i = 0; i < 5; i++) {
        buffer.put(i);
    }
    getLog();
    buffer.flip();
    getLog();
}
```

```sh
capacity: 20
position: 5
limit: 20
----------------
capacity: 20
position: 0
limit: 5
----------------
```

可以看到执行`flip()`方法之后的 position 变化

想要获取数据要调用`get()` 

```java
public static void main(String[] args) {
    initBuffer(20);
    for (int i = 0; i < 5; i++) {
        buffer.put(i);
    }
    buffer.flip();
    for (int i = 0; i < 5; i++) {
        get();
        getLog();
    }
}

static void get() {
    System.out.println(buffer.position() + "_num " + buffer.get());
}
```

```sh
0_num 0
capacity: 20
position: 1
limit: 5
----------------
1_num 1
capacity: 20
position: 2
limit: 5
----------------
2_num 2
capacity: 20
position: 3
limit: 5
----------------
3_num 3
capacity: 20
position: 4
limit: 5
----------------
4_num 4
capacity: 20
position: 5
limit: 5
----------------
```

Buffer 也提供了一个`get(int index)`的方法，可以直接获取指定位置的数据

```java
public static void main(String[] args) {
    initBuffer(20);
    for (int i = 0; i < 5; i++) {
        buffer.put(i);
    }
    for (int i = 0; i < 5; i++) {
        get(i);
        getLog();
    }
}

static void get(int index) {
    System.out.println("index_num: " + buffer.get(index));
}
```

```sh
index_num: 0
capacity: 20
position: 5
limit: 20
----------------
index_num: 1
capacity: 20
position: 5
limit: 20
----------------
index_num: 2
capacity: 20
position: 5
limit: 20
----------------
index_num: 3
capacity: 20
position: 5
limit: 20
----------------
index_num: 4
capacity: 20
position: 5
limit: 20
----------------
```

读完数据之后，可以调用`Buffer.clear()`或`Buffer.compact()`方法清空或压缩缓冲区后再写数据

如果要重读数据可以执行`rewind()`方法，这会让 position 属性回到 0

也可以使用`mark()`和`reset()`方法。

前者是记录当前的 position 值；后者是让 position 回到 mark 标记的位置

### Channel

Channel 担任的角色和 OIO 中流的角色差不多。在 OIO 中，一个网络连接需要两个流：一个输入流；一个输出流。Java 程序通过这两个单向流不断的进行输入输出操作。

而在 NIO 中，一个网络连接使用一个通道表示，所有的 IO 操作都是通过这个连接通道完成的。这个通道是双向的，既可以读数据，也可以写数据，

NIO 中的 Channel 主要的实现类有：

- FileChannel：文件通道，用于文件的数据读写
- SocketChannel：套接字通道，用于 Socket 套接字 TCP 连接的数据读写
- ServerSocketChannel：服务器套接字通道，用于监听 TCP 连接请求，为每个监听到的请求创建一个 SocketChannel 
- DatagramChannel：数据报通道，用于UDP协议的数据读写

**FileChannel**

是专门用来操作文件的通道，通过FileChannel，既可以从一个文件中读取数据，也可以将数据写入到文件中。

FileChannel 只能为阻塞模式，不能设置成非阻塞模式

可以通过文件的输入输出流来获取 FileChannel 文件通道

```java 
public class FileChannelTest {
    
    public static void main(String[] args) throws IOException {
        try (FileInputStream inputStream = new FileInputStream("../test/test.txt");
            FileOutputStream outputStream = new FileOutputStream("../test/test.txt");) {
            FileChannel inputChannel = inputStream.getChannel(); // 从输入流获取 Channel
            FileChannel outputChannel = outputStream.getChannel(); // 从输出流获取 Channel
            // 写入通道需要是将一个 ByteBuffer 缓存中的数据写入 Channel
            ByteBuffer wriBuffer = ByteBuffer.wrap("xuezhixiaxuenai".getBytes(StandardCharsets.UTF_8));
            while (wriBuffer.hasRemaining()) {
                outputChannel.write(wriBuffer); // 写入 Channel，返回写入的字节数
            }
            outputChannel.force(false); // //强制刷新到磁盘
            outputChannel.close(); // 关闭 Channel
            ByteBuffer readBuffer = ByteBuffer.allocate(100); // 创建一个从 Channel 中读取数据的缓存
            // 调用 read() 进行读取，Channel 中的数据会写入 Buffer
            while (inputChannel.read(readBuffer) != -1) {

            }
            readBuffer.flip();
            while (readBuffer.hasRemaining()) {
                System.out.print((char) readBuffer.get());
            }
            inputChannel.close();
        }
    }
}
```

**SocketChannel**

在 NIO 中，涉及网络连接的通道有两个，一个是  SocketChannel 负责连接的数据传输，对应 OIO 中的 Socket 类；另一个是 ServerSocketChannel 负责连接的监听，对应 OIO 的 ServerSocket 类。

ServerSocketChannel 仅仅应用于服务器端，而 SocketChannel 则同时处于服务器端和客户端，所以，对应于一个连接，两端都有一个负责传输的 SocketChanne l传输通道。

无论是ServerSocketChannel，还是SocketChannel，都支持阻塞和非阻塞两种模式。在阻塞模式下，SocketChannel通道的connect连接、read读、write写操作，都是同步的和阻塞式的，在效率上与Java旧的OIO的面向流的阻塞式读写操作相同。在非阻塞模式下，通道的操作是异步、高效率的，这也是相对于传统的OIO的优势所在。

> SocketChannel 通过调用`configureBlocking()`来设置阻塞还是非阻塞，`ture`为阻塞，`false`为非阻塞。

下面介绍非阻塞模式下通道的打开、读写和关闭操作。

```java
public class SocketClient {

    public static void main(String[] args) throws IOException {
        // 1. 创建非阻塞 SocketChannel
        SocketChannel clientChannel = SocketChannel.open();
        clientChannel.configureBlocking(false);

        // 2. 尝试连接服务器
        clientChannel.connect(new InetSocketAddress("127.0.0.1", 8080));

        // 3. 打开 Selector
        Selector selector = Selector.open();

        // 4. 注册连接事件
        clientChannel.register(selector, SelectionKey.OP_CONNECT);

        // 5. 创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 6. Scanner 用于读取用户输入
        Scanner scanner = new Scanner(System.in);

        while (true) {
            selector.select(); // 阻塞等待事件发生
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectedKeys.iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove(); // 移除已处理事件，避免重复处理

                // 7. 处理连接完成事件
                if (key.isConnectable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    if (channel.finishConnect()) {
                        System.out.println("已连接服务器！");
                        // 注册读写事件
                        channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                    }
                }

                // 8. 处理可读事件
                if (key.isReadable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    buffer.clear();
                    int bytesRead = channel.read(buffer); // read() 会返回读取的字节数
                    if (bytesRead > 0) {
                        buffer.flip();
                        String msg = StandardCharsets.UTF_8.decode(buffer).toString();
                        System.out.println("收到服务器消息：" + msg);
                    } else if (bytesRead == -1) { // read()返回-1表示服务器关闭了当前的 TCP 连接，所以客户端也要关闭
                        // 服务器关闭连接
                        System.out.println("服务器关闭连接");
                        channel.close();
                        return; // 退出程序
                    }
                }

                // 9. 处理可写事件
                if (key.isWritable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    System.out.print("请输入消息：");
                    String input = scanner.nextLine();
                    if (input.equalsIgnoreCase("bye")) {
                        // 用户输入 bye，关闭连接
                        channel.close();
                        System.out.println("客户端已关闭连接");
                        return;
                    }

                    // 写数据到服务器
                    buffer.clear();
                    buffer.put(input.getBytes(StandardCharsets.UTF_8));
                    // 从 buffer 中读取数据，写入 channel 中，所以需要把 buffer 切换成读模式
                    buffer.flip();
                    while (buffer.hasRemaining()) {
                        channel.write(buffer);
                    }
                    // 写完后可以继续监听读事件
                }
            }
        }
    }
}
```

上述代码使用了 IO 多路复用技术，这个技术用于客户端有点脱裤子放屁的感觉。这里只有一个客户端连接，因此可以直接使用阻塞。

着重讲一下`read()`和`write()`操作

Channel 的`read()`方法会从 Channel 中读取数据写入 buffer 中，返回读取的字节数。这时 buffer 中的 position 位置就会变化，因此下次从 buffer 中读数据时需要调用`flip()`方法重置 position。在非阻塞情况下，`read()`可能返回 0。如果返回 -1 表示 TCP 连接已经断开。`read()`操作并不会一次性将 Channel 中的数据全部读入 buffer 中，如果要将 Channel 中的数据全部读完，可以多次调用。

`write()`方法会从 buffer 中读取数据写入 Channel 中，因此在其方法前需要调用`flip()`操作使 buffer 中 position 返回到 0 的位置，以便正确读写。它会返回写入的字节数可能小于 buffer 中的数据长度，需要循环写

**DatagramChannel**

在 Java 中使用 UDP 协议传输数据比 TCP 更加简单。和 Socket 套接字的 TCP 传输协议不同，UDP 协议不是面向连接的协议。使用UDP协议时，只要知道服务器的IP和端口，就可以直接向对方发送数据。

UDP 客户端

```java
public class UDPClient {
    public static void main(String[] args) throws IOException {
        DatagramChannel clientChannel = DatagramChannel.open(); // 获取 DatagramChannel 实例
        Scanner scanner = new Scanner(System.in);

        System.out.print("请输入要发送的消息：");
        String message = scanner.nextLine();

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.put(message.getBytes(StandardCharsets.UTF_8));
        buffer.flip(); // 切换到读模式才能发送

        clientChannel.send(buffer, new InetSocketAddress("127.0.0.1", 20241));
        System.out.println("消息已发送");

        clientChannel.close();
    }
}
```

通过调用 `send()`方法向 Channel 中写入数据

UDP 服务端 

```java
public class UDPServer {
    public static void main(String[] args) throws IOException {
        DatagramChannel serverChannel = DatagramChannel.open();
        serverChannel.bind(new InetSocketAddress(20241)); // 绑定服务端接受端口
        System.out.println("UDP 服务端已启动，端口 20241 等待消息...");

        ByteBuffer buffer = ByteBuffer.allocate(1024);

        while (true) {
            buffer.clear(); // 清空 buffer，准备接收
            InetSocketAddress clientAddress = (InetSocketAddress) serverChannel.receive(buffer);
            if (clientAddress != null) {
                buffer.flip(); // 切换到读模式
                String msg = new String(buffer.array(), 0, buffer.limit());
                System.out.println("收到客户端 " + clientAddress + " 消息：" + msg);
            }
        }
    }
}
```

UDP 客户端和服务端都是使用阻塞模式编写的，UDP 也可以使用非阻塞模式。

UDP 通过调用`receive()`方法将 Channel 中的数据读入 buffer 中。

### Selector

IO 多路复用是指一个线程或者进程可以同时监听多个文件描述符，一旦其中的一个或者多个文件描述符可读或者可写，该监听线程或进程就能够进行 IO 事件的查询。

**IO 事件**：表示通道某种 IO 操作已经准备就绪。

Java NIO 定义了四种 NIO 事件，用来映射操作系统 IO 事件。这四种事件用 SelectionKey 的四个常量来表示：

- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE

在 Java 应用层面是通过 Selector 选择器来进行 IO 事件的监听与查询的。在 NIO 中，一般是一个单线程处理一个选择器，一个选择器监听多个通道的事件。这样只用一个线程就处理多个通道的方法，会很大程度上减少线程之间上下文切换开销。

可以通过调用通道的 `Channel.register(Selector sel, int ops)`方法，将通道实例注册到一个选择器中。`register()`方法中的两个参数，第一个用来指定通道注册到的选择器实例；第二个参数是指定选择器要监控的 IO 事件类型。

> 什么是 IO 事件
>
> 这个概念容易混淆，这里特别说明一下。这里的IO事件不是对通道的IO操作，而是通道处于某个IO操作的就绪状态，表示通道具备执行某个IO操作的条件。比方说某个SocketChannel传输通道，如果完成了和对端的三次握手过程，则会发生“连接就绪”（OP_CONNECT）的事件。再比方说某个ServerSocketChannel服务器连接监听通道，在监听到一个新连接的到来时，则会发生“接收就绪”（OP_ACCEPT）的事件。还比方说，一个SocketChannel通道有数据可读，则会发生“读就绪”（OP_READ）事件；一个等待写入数据的SocketChannel通道，会发生写就绪（OP_WRITE）事件。

**SelectableChannel**

可选择通道，并不是所有的通道都可以被选择器监控或选择。比如 FileChannel 文件通道就不能被选择器复用。判断一个通道是否能被选择器监控或选择，就是看这个类是否继承了抽象类 SelectableChannel。NIO 中所有网络链接 Socket 套接字通道，都继承了SelectableChannel 类，都是可选择的。

```sh
客户端应用         客户端内核          网络         服务端内核           服务端应用
   │ write()         │ send buffer     │             │ recv buffer        │ read()
   │  ─────────────▶ │ ──────────────▶ │ ─────────▶  │ ────────────────▶  │
   │                 │                 │             │                    │

```

如上图所示，服务端创建一个 Channel，客户端会再创建一个 Channel，各自的 Channel 持有各自主机上的文件描述符，两者之间的数据交换是通过 TCP 来进行交流的。

一个 SocketServer 如下

```java 
public class SocketServer {

    public static void main(String[] args) throws IOException {
        // 1. 打开 ServerSocketChannel 并配置非阻塞
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress("localhost", 8080));
        System.out.println("服务器启动，监听端口 8080...");

        // 2. 打开 Selector 用于多路复用
        Selector selector = Selector.open();

        // 3. 注册 ServerSocketChannel 的 accept 事件
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 4. 事件循环
        while (true) {
            selector.select(); // 阻塞等待事件
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectedKeys.iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove(); // 移除已处理事件，避免重复处理

                // 5. 处理新的客户端连接
                if (key.isAcceptable()) {
                    ServerSocketChannel srvChannel = (ServerSocketChannel) key.channel();
                    SocketChannel clientChannel = srvChannel.accept(); // 获取新客户端通道
                    clientChannel.configureBlocking(false);
                    // 注册读事件，监听客户端发来的数据
                    clientChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("accepted: " + clientChannel.getRemoteAddress());
                }

                // 6. 处理客户端可读事件
                if (key.isReadable()) {
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024); // 单独的读缓冲区
                    int bytesRead = -1;

                    try {
                        bytesRead = clientChannel.read(readBuffer); // 返回读取的字节数
                    } catch (IOException e) {
                        // 客户端异常关闭
                        System.out.println("客户端异常断开: " + e.getMessage());
                        key.cancel();
                        clientChannel.close();
                        continue;
                    }

                    if (bytesRead > 0) {
                        readBuffer.flip(); // 切换到读取模式
                        String msg = StandardCharsets.UTF_8.decode(readBuffer).toString();
                        System.out.println("收到消息: " + msg);

                        // 7. 回复客户端
                        String reply = "ok: " + msg;
                        ByteBuffer writeBuffer = ByteBuffer.wrap(reply.getBytes(StandardCharsets.UTF_8));
                        while (writeBuffer.hasRemaining()) {
                            clientChannel.write(writeBuffer); // 循环写，保证写完
                        }

                        // 如果客户端发 "bye"，关闭连接
                        if (msg.trim().equalsIgnoreCase("bye")) {
                            System.out.println("客户端断开连接");
                            key.cancel();
                            clientChannel.close();
                        }
                    } else if (bytesRead == -1) {
                        // 客户端主动关闭
                        System.out.println("客户端主动关闭连接");
                        key.cancel();
                        clientChannel.close();
                    }
                }
            }
        }
    }
}
```

在  Java NIO 中，`SelectionKey` 是 **Selector 注册的通道（Channel）与事件之间的关联对象**，当调用`channel.register(selector, ops)`注册一个通道时Selector 会返回一个 `SelectionKey`，表示 **这个通道和它关注的事件**。

`SelectionKey` 里记录了：

- 该通道 (`channel`)
- 该选择器 (`selector`)
- 关注的事件 (`interestOps`)
- 就绪事件 (`readyOps`)

可以通过它拿到相应的对象。

当**不再关注某些事件**

- 可以调用如下代码取消监听写事件：

  ```
  key.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);
  ```

- 如果完全不再需要通道了，就调用 `cancel()`。取消通道在 Selector 上的注册取消通道在 Selector 上的注册。

但不会关闭通道，关闭需要 调用`channel.close()`

一个 Channel 可以同时就绪多个事件，就拿读写事件来说

- 如果 Channel 中存在数据，就会触发读事件
- 如果 Channel 中存在空闲内存，就会触发写事件

在调用`read()`方法从 Channel 中读取数据的时候，可以阻塞一次性全部将数据读取到 buffer 缓冲区，也可以只读一次，下次会继续触发读事件。