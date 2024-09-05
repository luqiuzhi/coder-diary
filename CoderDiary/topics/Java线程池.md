# Java线程池

注：本章节的源码均使用`GraalVM`17版本进行讲解（用JDK8讲的太多了，咱们看看新版本是否有变化）。

Java多线程的原理了解完，接下来就是实操层面——Java线程池的使用。

线程池的好处：

- 降低资源的消耗，提高资源的使用率。线程的创建和销毁都属于是重量级的操作，通过线程池，我们可以实现对线程的复用，减少线程的创建和销毁的动作。
- 提高系统响应速度，从而提高性能和吞吐量。系统接收消息后，不需要等待线程的创建就可以直接进行处理（在线程数未达到最大线程数之前）。
- 更好的管理线程。在没有线程池的情况下，线程的管控较为困难，系统也不能无限制的创建线程，也不能很好的回收线程。

## ThreadPoolExecutor

在Java中主要使用`ThreadPoolExecutor`来创建线程池，该类位于`java.util.concurrent`包路径下。

一般使用`ThreadPoolExecutor`的构造方法创建线程池，该类提供了四个构造方法，***拥有7个参数的方法为核心构造方法***。

```Java
public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                          int maximumPoolSize,  // 线程池最大线程数
                          long keepAliveTime,   // 非核心线程空闲后的存活时间
                          TimeUnit unit,   // 存活时间的单位
                          BlockingQueue<Runnable> workQueue, // 任务等待队列
                          ThreadFactory threadFactory,   // 线程创建工厂
                          RejectedExecutionHandler handler) {  // 任务拒绝策略
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

## Executors

`Executors`可以快速地创建特定类型的线程池，该类位于`java.util.concurrent`包路径下。

但是该类所提供的用于创建不同类型线程池的静态方法，本质上也是调用上面`ThreadPoolExecutor`的7参数构造方法，只是参数不同而已。

在明确当前业务场景的情况下，我认为可以使用`Executors`进行线程池的管理。

但是仍然要注意业务变更后，`Executors`创建的线程池是否会产生性能问题。

## ThreadPoolExecutor源码解析 {id="threadpoolexecutor_1"}

在开始分析`ThreadPoolExecutor`源码之前，我们可以提出几个问题：

- 为什么线程池中的核心线程在执行完`run`方法后不会进入销毁状态？这是不是和线程的7大状态相违背？
- 线程池是如何对线程进行复用的？
- 线程池是如何回收空闲线程的？换言之，是如何判断线程的状态为空闲呢？
- 线程池什么时候会创建一个新线程？

我们从构造方法的参数开始分析：

