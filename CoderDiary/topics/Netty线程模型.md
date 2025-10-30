# Netty线程模型

Netty使用的线程模型是Reactor模型，该模型基于NIO实现。

与之相对的还有Proactor模型，该模型基于AIO实现。

## NIO（同步非阻塞IO）

数据读取和写入的主体是应用本身，数据可读可写时由内核通知应用。

## AIO（异步非阻塞IO）

数据读取和写入的主体是内核，读操作完成后由内核通知应用，应用进行数据处理，完成后通知内核进行写入操作。

## Reactor模型

首先，理解 Reactor 的核心思想至关重要：**“不要用你来呼叫我，而是让我来通知你”**。

在传统的网络编程中，我们为每个连接创建一个线程，线程会阻塞在 `read()` 调用上等待数据。这在高并发场景下会消耗大量线程资源，上下文切换成本极高。

Reactor 模式则将这种模式反转。它有一个或多个**事件循环**（Event Loop），不断地轮询一个**事件通知器**（如 Selector），当有连接建立、数据可读、数据可写等事件发生时，事件通知器会返回这些事件，然后事件循环会分发（Dispatch）这些事件到对应的处理器（Handler）中进行处理。

Reactor模型基于NIO实现，有三种形态：
1. 单线程Reactor模型
2. 多线程Reactor模型
3. 主从多线程模型

这三种形态主要区别于 **Reactor 的数量** 和 **处理线程的数量** 之间的组合。

### 形态一：单 Reactor 单线程 {id="reactor_2"}

这是最基本的形态，所有工作都在一个线程内完成。

*   **模型架构：**
    *   **Reactor 线程**：负责监听和分发所有事件，包括 Acceptor 的连接事件和 Handler 的 IO 事件。
    *   **Acceptor**：处理新的客户端连接，并建立对应的 Handler 来处理后续的读写事件。
    *   **Handler**：处理本连接的 IO 事件（读、写、计算等）。

*   **工作流程：**
    1.  Reactor 线程通过 Selector 监听事件。
    2.  如果是连接建立事件，则分发给 Acceptor，Acceptor 处理连接，并创建一个 Handler 来处理该连接后续的所有事件。
    3.  如果是读写事件，则分发给该连接对应的 Handler。
    4.  Handler 负责完成完整的读 -> 业务处理 -> 写 流程。

*   **优点：**
    *   模型简单，没有多线程间的竞争和同步问题。

*   **缺点：**
    *   性能瓶颈明显，只有一个线程，无法利用多核 CPU。
    *   如果某个 Handler 处理过慢或阻塞，会直接影响所有其他连接的响应。
    *   不适用于计算密集型或耗时 IO 操作场景。

*   **适用场景：**
    *   客户端数量有限，或处理速度非常快的场景，如 Redis 的早期版本。

**Java 代码示例：**

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class SingleThreadReactor implements Runnable {
    final Selector selector;
    final ServerSocketChannel serverSocket;

    public SingleThreadReactor(int port) throws IOException {
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        serverSocket.configureBlocking(false);
        // 将 Acceptor 注册到 Selector
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        sk.attach(new Acceptor());
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select(); // 阻塞，等待事件
                Set<SelectionKey> selected = selector.selectedKeys();
                Iterator<SelectionKey> it = selected.iterator();
                while (it.hasNext()) {
                    // 事件分发
                    dispatch(it.next());
                }
                selected.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    void dispatch(SelectionKey k) {
        Runnable r = (Runnable) k.attachment();
        if (r != null) {
            r.run(); // 调用 Acceptor 或 Handler 的 run 方法
        }
    }

    // Acceptor：处理连接事件
    class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel c = serverSocket.accept();
                if (c != null) {
                    new Handler(selector, c); // 创建 Handler 处理后续IO
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // Handler：处理IO事件
    final class Handler implements Runnable {
        final SocketChannel socket;
        final SelectionKey sk;
        ByteBuffer input = ByteBuffer.allocate(1024);
        ByteBuffer output = ByteBuffer.allocate(1024);
        static final int READING = 0, SENDING = 1;
        int state = READING;

        public Handler(Selector sel, SocketChannel c) throws IOException {
            socket = c;
            c.configureBlocking(false);
            sk = socket.register(sel, 0); // 先注册读事件
            sk.attach(this); // 将 Handler 作为附件
            sk.interestOps(SelectionKey.OP_READ); // 关注读事件
            sel.wakeup(); // 唤醒 Selector，因为新注册了事件
        }

        @Override
        public void run() {
            try {
                if (state == READING) read();
                else if (state == SENDING) send();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        void read() throws IOException {
            socket.read(input);
            if (inputIsComplete()) {
                process(); // 业务处理（也在同一个线程！）
                state = SENDING;
                sk.interestOps(SelectionKey.OP_WRITE); // 关注写事件
            }
        }

        void send() throws IOException {
            socket.write(output);
            if (outputIsComplete()) {
                sk.cancel(); // 写完成，关闭连接
            }
        }

        boolean inputIsComplete() { /* ... */ return true; }
        boolean outputIsComplete() { /* ... */ return true; }
        void process() {
            // 模拟业务处理，这里会阻塞 Reactor 线程！
            input.flip();
            byte[] bytes = new byte[input.remaining()];
            input.get(bytes);
            String message = new String(bytes);
            System.out.println("Processing: " + message);
            // 模拟处理结果
            output.clear();
            output.put(("Echo: " + message).getBytes());
            output.flip();
        }
    }

    public static void main(String[] args) throws IOException {
        new Thread(new SingleThreadReactor(8080)).start();
        System.out.println("SingleThreadReactor is running on port 8080");
    }
}
```

---

### 形态二：单 Reactor 多线程 {id="reactor_3"}

这是对形态一最直接的改进，将耗时的业务处理（`process` 方法）从 Reactor 线程中剥离，交给一个线程池来处理，从而解放 Reactor 线程，使其能更快地响应其他 IO 事件。

*   **模型架构：**
    *   **Reactor 线程**：依然只有一个，负责监听和分发所有事件。
    *   **Acceptor**：同上。
    *   **Handler**：只负责数据的读取和发送，**不再负责业务处理**。读取到数据后，将其封装成一个任务（Task）提交给**业务线程池**。
    *   **Worker 线程池**：一个专门用于处理业务逻辑的线程池。

*   **工作流程：**
    1.  同形态一，Reactor 监听和分发事件。
    2.  Handler 收到读事件后，读取数据。
    3.  Handler 将读取的数据作为一个 `Runnable` 对象提交给业务线程池。
    4.  业务线程池中的某个线程会执行该任务，进行业务处理，并生成结果。
    5.  业务处理完成后，Handler（通常通过回调方式）会收到通知，然后向 Reactor 注册写事件。
    6.  Reactor 监听到该连接的 Socket 可写时，分发给 Handler 进行数据发送。

*   **优点：**
    *   充分利用多核 CPU，避免业务处理阻塞 IO 事件循环。
    *   Reactor 线程可以保持高速响应。

*   **缺点：**
    *   Reactor 线程仍然是单线程，在极高并发下，监听和分发本身可能成为瓶颈。
    *   多线程间的数据共享和通信比较复杂。

*   **适用场景：**
    *   绝大多数业务处理耗时的场景。这是最常用的一种形态。

**Java 代码示例（修改 Handler 部分）：**

```java
// ... (Reactor 和 Acceptor 部分与形态一相同)

final class Handler implements Runnable {
    final SocketChannel socket;
    final SelectionKey sk;
    ByteBuffer input = ByteBuffer.allocate(1024);
    static final int READING = 0, SENDING = 1;
    int state = READING;

    // 引入业务线程池
    static ExecutorService pool = Executors.newFixedThreadPool(10);

    public Handler(Selector sel, SocketChannel c) throws IOException { /* ... 与形态一同 ... */ }

    @Override
    public void run() {
        try {
            if (state == READING) read();
            else if (state == SENDING) send();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    void read() throws IOException {
        socket.read(input);
        if (inputIsComplete()) {
            // 关键变化：将 process 任务提交到线程池
            pool.execute(new ProcessTask());
        }
    }

    // 业务处理任务
    class ProcessTask implements Runnable {
        @Override
        public void run() {
            // 在子线程中执行，不阻塞 Reactor
            processAndHandOff();
        }
    }

    synchronized void processAndHandOff() {
        // 真正的业务处理
        process();
        state = SENDING;
        // 业务处理完后，注册写事件。这里需要唤醒 Selector，因为是在子线程中修改的。
        sk.interestOps(SelectionKey.OP_WRITE);
        selector.wakeup();
    }

    void send() throws IOException {
        // ... 发送数据 ...
        socket.write(output);
        if (outputIsComplete()) {
            sk.cancel();
        }
    }

    void process() {
        // 业务处理，现在在子线程中运行
        input.flip();
        byte[] bytes = new byte[input.remaining()];
        input.get(bytes);
        String message = new String(bytes);
        System.out.println("Processing in thread: " + Thread.currentThread().getName());
        // 准备输出
        output.clear();
        output.put(("Echo: " + message).getBytes());
        output.flip();
    }
    // ... inputIsComplete, outputIsComplete ...
}
```

---

### 形态三：主从 Reactor 多线程 {id="reactor_4"}

这是对形态二的进一步扩展，用于解决**单个 Reactor 的瓶颈问题**。它将 Reactor 的功能进行了拆分。

*   **模型架构：**
    *   **Main Reactor（主 Reactor）**：只有一个，负责监听和分发**连接建立事件**。它通过 Acceptor 接收新连接，然后将建立好的 SocketChannel 分配给 Sub Reactor。
    *   **Sub Reactor（从 Reactor）**：可以有多个，每个都是一个独立的线程。Main Reactor 会将新连接注册到某个 Sub Reactor 上。
    *   **Acceptor**：存在于 Main Reactor 中。
    *   **Handler**：存在于 Sub Reactor 中，负责连接的 IO 读写和业务分发（业务仍交给线程池）。

*   **工作流程：**
    1.  Main Reactor 监听 ServerSocketChannel，处理 `OP_ACCEPT` 事件。
    2.  Acceptor 接收到新连接，创建一个 SocketChannel。
    3.  Acceptor 通过某种策略（如轮询）将 SocketChannel 分配给一个 Sub Reactor。
    4.  该 Sub Reactor 将 SocketChannel 注册到自己的 Selector 上，监听 `OP_READ` 等 IO 事件。
    5.  当该连接的 IO 事件就绪时，由它所属的 Sub Reactor 进行分发，调用对应的 Handler。
    6.  Handler 的处理逻辑与形态二完全相同：读 -> 提交线程池 -> 写。

*   **优点：**
    *   职责进一步分离，Main Reactor 可以专注于接收连接，Sub Reactor 专注于处理 IO。
    *   减轻了单个 Reactor 的压力，可以应对极高的并发连接数。Main Reactor 和 Sub Reactor 在不同的线程中运行，可以并行处理。
    *   这是 Netty、Nginx 等高性能网络框架普遍采用的架构。

*   **缺点：**
    *   模型和实现最为复杂。

**Java 代码示例：**

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.*;
import java.util.concurrent.atomic.AtomicInteger;

public class MultiThreadReactor {
    // 主 Reactor
    final Reactor mainReactor;
    // 从 Reactor 组
    final Reactor[] subReactors;
    final int SUB_SIZE;
    final AtomicInteger nextIndex = new AtomicInteger();

    public MultiThreadReactor(int port, int subSize) throws IOException {
        this.SUB_SIZE = Math.max(subSize, 1);
        subReactors = new Reactor[SUB_SIZE];
        for (int i = 0; i < SUB_SIZE; i++) {
            subReactors[i] = new Reactor("SubReactor-" + i);
        }
        mainReactor = new Reactor("MainReactor");
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.socket().bind(new InetSocketAddress(port));
        ssc.configureBlocking(false);
        // 主 Reactor 只关注 Accept 事件，并绑定 Acceptor
        mainReactor.register(ssc, SelectionKey.OP_ACCEPT, new Acceptor(ssc));
    }

    public void start() {
        // 启动所有从 Reactor 线程
        for (Reactor sub : subReactors) {
            new Thread(sub).start();
        }
        // 启动主 Reactor 线程
        new Thread(mainReactor).start();
        System.out.println("MultiThreadReactor is running...");
    }

    // Acceptor：在主 Reactor 中运行
    class Acceptor implements Runnable {
        final ServerSocketChannel serverSocket;

        public Acceptor(ServerSocketChannel ssc) {
            this.serverSocket = ssc;
        }

        @Override
        public void run() {
            try {
                SocketChannel c = serverSocket.accept();
                if (c != null) {
                    // 关键：将新连接均衡地分配给一个 SubReactor
                    Reactor sub = nextSubReactor();
                    // 将新连接和 Handler 的创建任务交给 SubReactor
                    sub.registerChannel(c);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // 简单的轮询策略选择 SubReactor
    private Reactor nextSubReactor() {
        int index = nextIndex.getAndIncrement() % SUB_SIZE;
        return subReactors[index];
    }

    // Reactor 类，代表一个事件循环
    static class Reactor implements Runnable {
        final Selector selector;
        final String name;

        public Reactor(String name) throws IOException {
            this.name = name;
            this.selector = Selector.open();
        }

        // 主 Reactor 用：注册 ServerSocketChannel
        public void register(SelectableChannel ch, int ops, Object att) throws ClosedChannelException {
            ch.register(selector, ops, att);
        }

        // Sub Reactor 用：注册新的 SocketChannel
        public void registerChannel(SocketChannel ch) throws ClosedChannelException {
            // 这个方法可能在主线程中被调用，所以需要 wakeup()
            new Handler(selector, ch);
            selector.wakeup();
        }

        @Override
        public void run() {
            try {
                while (!Thread.interrupted()) {
                    selector.select();
                    var selectedKeys = selector.selectedKeys();
                    var it = selectedKeys.iterator();
                    while (it.hasNext()) {
                        var sk = it.next();
                        dispatch(sk);
                    }
                    selectedKeys.clear();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        void dispatch(SelectionKey k) {
            Runnable r = (Runnable) k.attachment();
            if (r != null) {
                r.run();
            }
        }
    }

    // Handler 类（与形态二的 Handler 几乎一样，运行在 SubReactor 线程中）
    // ... 此处省略，与形态二的 Handler 实现相同，包含线程池 ...
    
    public static void main(String[] args) throws IOException {
        new MultiThreadReactor(8080, 3).start(); // 3个 SubReactor
    }
}
```

### 总结与对比

| 形态                 | Reactor 数量  | 线程数量                | 优点                | 缺点                  | 适用场景                  |
|:-------------------|:------------|:--------------------|:------------------|:--------------------|:----------------------|
| **单 Reactor 单线程**  | 1           | 1                   | 模型简单，无并发问题        | 性能瓶颈，易阻塞            | 连接数少，处理快（如 Redis）     |
| **单 Reactor 多线程**  | 1           | N+1 (Reactor + 线程池) | 充分利用多核，避免业务阻塞IO   | Reactor 单点，高并发下可能瓶颈 | 最常用，业务处理耗时的通用场景       |
| **主从 Reactor 多线程** | M (主1 + 从N) | N+M+1               | 职责分离，性能最高，可应对超高并发 | 实现复杂                | Netty, 高并发服务器（如游戏、IM） |

这三种形态是一个递进和优化的过程，实际应用中（如 Netty）就是基于主从 Reactor 多线程模型构建的，并在此基础上提供了丰富的组件和高度可定制的流水线（Pipeline）。

## Netty

Netty可以参数调整，在以上三种模式之间切换。

### 深入理解“同步”与“异步”

这是理解两者区别的关键。这里的“同步”和“异步”指的是**I/O操作本身（数据在内核缓冲区和用户缓冲区之间的拷贝）由谁完成**。

1.  **NIO 是同步的 (Synchronous)**：
    *   虽然`Selector.select()`是非阻塞的，但当你通过`SelectionKey`知道某个Channel就绪后，你调用`channel.read(buffer)`将数据从内核空间读到你的用户空间缓冲区（Buffer）时，**这个`read`操作仍然是同步的、需要等待的**。
    *   **应用程序是I/O操作的执行者**。

2.  **AIO 是异步的 (Asynchronous)**：
    *   你调用`asynchronousChannel.read(...)`后，这个方法会立刻返回一个`Future`。**操作系统内核会负责将数据从内核空间拷贝到你的缓冲区**，整个过程不需要应用程序参与。
    *   拷贝完成后，操作系统会调用你提供的`CompletionHandler`。
    *   **操作系统内核是I/O操作的执行者**，应用程序只是发起者和接收结果者。