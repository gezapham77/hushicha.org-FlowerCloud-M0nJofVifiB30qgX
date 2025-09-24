Java 官方文档明确指出：

> Do not pool virtual threads.
>
> 虚拟线程不是昂贵资源，永远不应该被池化。
>
> 应该为每个任务创建一个新的虚拟线程，它们应该是短暂的、任务级别的。

这是为什么呢？为什么只有虚拟线程 Virtual Thread，却没有虚拟线程池 Virtual Thread Pool 呢？

## 主要原因

之所以只有虚拟线程是因为，**虚拟线程创建成本极低，低到其创建成本远小于线程池的管理成本。**

> 也就是说，线程池的管理成本远远大于虚拟线程的创建成本，所以使用虚拟线程池是一个不划算的操作。

具体来说，传统平台线程的创建涉及分配大量的栈内存（通常~1MB）并与操作系统交互，开销很大。池化是为了复用这些“昂贵”的线程，避免反复申请资源。而虚拟线程由 JVM 在用户态管理，初始栈空间很小（约几百字节），创建和销毁的代价极低，池化带来的收益远小于管理池本身的复杂度。

> **“用完就扔”比“池化复用”更高效、更简单。**一个线程约等于几千个虚拟线程。

## 一任务一虚线程的理念

官方推荐并为每个任务创建一个全新的虚拟线程，例如通过 Executors.newVirtualThreadPerTaskExecutor()，任务完成后虚拟线程即被丢弃。**这种模式代码更清晰，避免了因线程复用可能带来的线程局部变量（ThreadLocal）污染等问题，也无需担心池的大小调优等问题。**

最佳实现代码：

```
// 无需池化 - 直接创建
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
} // 自动关闭（所有虚拟线程完成即销毁）
```

ExecutorService 并不是一个传统意义上的“池”，你可以把它理解为一个**虚拟线程工厂**。每次 submit 一个任务，它都会**立即创建一个新的虚拟线程来执行该任务**，它内部并不维护（一个可复用的）线程队列。

## 如何限制并发？

在**单进程百万虚拟线程的情况下， JVM 内存是完全无压力的**。如果你还是担心太多的虚拟线程会导致程序崩溃，在特定的场景可以使用 Semaphore 等技术来实现局部限流，例如以下代码这样：

```
// 使用信号量而非线程池来限制对某个资源的并发访问
Semaphore semaphore = new Semaphore(100000); // 限制最大并发数为100000

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        executor.submit(() -> {
            semaphore.acquire(); // 获取许可，若已达上限则阻塞等待
            try {
                // 访问受保护的资源或执行需要限流的操作
                callLimitedService();
            } finally {
                semaphore.release(); // 释放许可
            }
        });
    }
}
```

## 小结

虚拟线程 Virtual Thread 因为其创建成本极低（约几百字节），所以不会完全不需要使用池化技术来实现，**因为池化技术的本质是复用那些“昂贵”的线程，避免反复申请资源的**。如果要局部限流虚拟线程可以使用 Semaphore 来实现。

> 本文已收录到我的面试小站 [www.javacn.site](https://github.com)，其中包含的内容有：场景题、SpringAI、SpringAIAlibaba、并发编程、MySQL、Redis、Spring、Spring MVC、Spring Boot、Spring Cloud、MyBatis、JVM、设计模式、消息队列、Dify、Coze、AI常见面试题等。

本博客参考[ssrdog](https://51taomao.com)。转载请注明出处！
