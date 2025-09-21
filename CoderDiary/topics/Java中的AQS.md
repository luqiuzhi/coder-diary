# Java中的AQS

---

## 一、AQS 是什么？ {id="aqs_1"}

**AQS**（**A**bstract**Q**ueued**S**ynchronizer），即**抽象队列同步器**，是 `java.util.concurrent.locks` 包下的一个核心基础组件。

它提供了一个**通用的、底层的框架**，用于构建各种类型的同步器（如锁、屏障、信号量等）。你可以把它想象成一个“乐高积木”的基础底板，`ReentrantLock`、`Semaphore`、`CountDownLatch` 等这些常用的同步工具都是在这个底板上搭建起来的。

**核心思想**：如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制。AQS 使用一个 **CLH 虚拟队列** 来实现这个机制，将暂时获取不到锁的线程加入到队列中。

---

## 二、核心原理与数据结构

AQS 的核心原理可以概括为三点：**一个状态**、**一个队列**、**一套模板方法**。

### 1. 一个状态：`state`

这是一个用 `volatile` 关键字修饰的 `int` 型变量（`private volatile int state;`），它代表了**共享资源的状态**。

*   **如何解读 `state` 完全由子类决定**，这是 AQS 设计灵活性的关键。
    *   在 `ReentrantLock` 中，`state` 表示**锁的重入次数**（0表示未锁定，>1表示被同一线程重入的次数）。
    *   在 `Semaphore` 中，`state` 表示**剩余的许可证数量**。
    *   在 `CountDownLatch` 中，`state` 表示**计数器还未完成的次数**。
*   所有对共享资源的操作（获取、释放）本质上都是通过 **CAS**（Compare-And-Swap） 操作来原子性地修改这个 `state` 值。

### 2. 一个队列：CLH 变种队列

这是一个**虚拟的双向队列**（CLH 锁的变体），用于存放所有等待获取资源的线程。当线程请求资源失败时，AQS 会将当前线程以及等待状态等信息包装成一个 **Node** 节点，并将其加入队列尾部。

*   **Node 节点**：队列中的每个节点代表一个等待线程。
*   **队列头（Head）和尾（Tail）**：队列有一个虚拟的头节点（dummy node），其后的第一个节点才是真正等待的线程。尾指针指向最新加入的节点。
*   **作用**：这个队列是管理**阻塞线程**和**分配资源**的核心。当资源释放时，AQS 会从队列头部开始，尝试唤醒等待的线程。

### 3. 一套模板方法：设计模式的运用

AQS 使用了**模板方法模式**。它定义了顶级的结构和算法骨架（比如入队、出队、阻塞、唤醒），而将一些关键的操作（如何获取/释放资源）留给子类去具体实现。

*   **AQS 需要子类重写的方法（protected）**：
    *   `tryAcquire(int arg)`：尝试以**独占**模式获取资源。成功返回 true，失败返回 false。
    *   `tryRelease(int arg)`：尝试以**独占**模式释放资源。成功返回 true，失败返回 false。
    *   `tryAcquireShared(int arg)`：尝试以**共享**模式获取资源。返回负数表示失败；0表示成功，但无剩余资源；正数表示成功，且有剩余资源。
    *   `tryReleaseShared(int arg)`：尝试以**共享**模式释放资源。如果释放后允许唤醒后续等待节点返回 true，否则返回 false。
    *   `isHeldExclusively()`：当前同步器是否在独占模式下被线程占用。

*   **AQS 提供给外部使用的方法（public final）**：
    *   `acquire(int arg)`：以独占模式获取资源，忽略中断。如果 `tryAcquire` 失败，线程会进入队列等待，在成功之前一直阻塞。（这是 `lock()` 方法的核心）
    *   `acquireInterruptibly(int arg)`：同上，但响应中断。
    *   `release(int arg)`：以独占模式释放资源。它会调用 `tryRelease`，成功后唤醒队列中的下一个线程。（这是 `unlock()` 方法的核心）
    *   `acquireShared(int arg)`：以共享模式获取资源。（这是 `Semaphore.acquire()` 的核心）
    *   `releaseShared(int arg)`：以共享模式释放资源。（这是 `Semaphore.release()` 的核心）

**工作流程简化版（以 `acquire` 为例）**：
1.  调用子类重写的 `tryAcquire` 方法尝试直接获取资源。
2.  如果成功，直接返回。
3.  如果失败，AQS 将线程包装成 Node 节点，加入到等待队列尾部。
4.  在队列中自旋或检查状态，如果前驱节点是头节点，会再次尝试 `tryAcquire`。
5.  如果成功，将自己设为新的头节点并返回。
6.  如果失败，根据前驱节点的状态判断是否需要阻塞当前线程（通过 `LockSupport.park()`）。
7.  当资源被释放（`release`），队列头部的节点会被唤醒，然后重复步骤4。

---

## 三、两种资源共享模式

1.  **独占模式（Exclusive）**
    *   资源一次只能被一个线程持有。
    *   例如：`ReentrantLock`。
    *   对应方法：`acquire`、`release` 等。

2.  **共享模式（Shared）**
    *   资源可以被多个线程同时持有。
    *   例如：`Semaphore`、`CountDownLatch`。
    *   对应方法：`acquireShared`、`releaseShared` 等。

有些同步器可以同时实现两种模式，如 `ReentrantReadWriteLock`，其读锁是共享的，写锁是独占的。

---

## 四、基于 AQS 构建的同步器举例 {id="aqs_2"}

*   **ReentrantLock**： 使用独占模式。`state` 初始为0，表示未锁定。线程 A 调用 `lock()` 时，会调用 `tryAcquire` 独占该锁并将 `state`+1。此后，其他线程再调用 `tryAcquire` 时就会失败，直到线程 A 调用 `unlock()`（`tryRelease`）将 `state`-1 为0。线程 A 自己可以重复获取此锁（重入），每次 `state`+1，释放时需释放相同的次数。
*   **Semaphore**： 使用共享模式。`state` 初始化为许可证的数量（N）。每次 `acquire()` 调用 `tryAcquireShared`，使 `state`-1（CAS），如果减到负数，线程阻塞。每次 `release()` 调用 `tryReleaseShared`，使 `state`+1（CAS），并唤醒等待的线程。
*   **CountDownLatch**： 使用共享模式。`state` 初始化为计数器的值（N）。`await()` 方法调用 `acquireShared`，只要 `state` 不为0，线程就会阻塞。`countDown()` 方法调用 `tryReleaseShared`，使 `state`-1，当 `state` 减为0时，唤醒所有等待的线程。
*   **ReentrantReadWriteLock**： 组合使用两种模式。读锁使用共享模式，写锁使用独占模式。它通过巧妙的方式，在一个 `state` 变量上同时维护读锁的数量和写锁的重入次数（高16位表示读，低16位表示写）。
*   **ThreadPoolExecutor**： 其中的 Worker 类（工作线程）也利用了 AQS 来实现独占锁，用于控制线程的中断状态。

---

## 五、AQS的精髓

1.  **封装性**：AQS 将复杂的同步器实现细节（如队列管理、线程阻塞/唤醒）封装起来，开发者只需要关注对共享资源 `state` 的获取和释放逻辑即可。
2.  **灵活性**：通过模板方法模式，AQS 允许子类自定义资源的管理方式，从而能够轻松构建出各种功能迥异的同步器。
3.  **高性能**：其内部使用了 CLH 队列和 CAS 操作，避免了重量级锁（如 `synchronized`）的内核态切换，在竞争不激烈的情况下性能很高。
4.  **是 JUC 包的基石**：理解了 AQS，就理解了大部分 JUC 同步工具的工作原理，对于诊断并发问题、进行高性能并发编程有极大帮助。

简单来说，**AQS 就是一个“资源状态管理器”+“线程等待队列”**。你告诉它如何判断资源是否可用（重写 `tryAcquire`），它来负责处理所有排队和阻塞的脏活累活。