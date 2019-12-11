---
description: The challenge of concurrency programming.
---

# 并发编程的挑战

并发编程的目的是为了让程序运行得更快，但并不是启动更多的线程就能让程序最大限度的并发执行。下面介绍几种并发编程的挑战以及解决方案。

## 上下文切换

即使是单核处理器也支持多线程执行代码，CPU通过给每个线程分配CPU时间片来实现这个机制。**时间片**是CPU分配给各个线程的时间。因为时间片非常短，所以CPU通过不断的切换线程执行，让我们感觉多个线程是同时执行的，时间片一般是几十毫秒。

CPU通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。在切换前会保存上一个任务的状态，以便下次切换会这个任务，再加载到这个任务的状态。所以任务从保存到再加载的过程就是一次**上下文切换**。

### 多线程一定快吗？

```java
/*串行和并行对比*/
public class ConcurrencyTest {

    private static final long count = 1000000001;

    public static void main(String[] args) throws InterruptedException {
        concurrency();
        serial();
    }

    private static void serial() {
        long start = System.currentTimeMillis();
        int a = 0;
        for (long i = 0; i < count; i++) {
            a += 5;
        }
        int b = 0;
        for (long i = 0; i < count; i++) {
            b--;
        }
        long time = System.currentTimeMillis() - start;
        System.out.println("serial:" + time + "ms,b=" + b + ",a=" + a);
    }

    private static void concurrency() throws InterruptedException {
        long start = System.currentTimeMillis();
        Thread thread = new Thread(() -> {
            int a = 0;
            for (long i = 0; i < count; i++) {
                a += 5;
            }
        });
        thread.start();
        int b = 0;
        for (long i = 0; i < count; i++){
            b--;
        }
        thread.join();
        long time = System.currentTimeMillis() - start;
        System.out.println("concurrency:" + time + "ms,b=" + b);
    }
}
```

| 循环次数 | 串行执行耗时/ms | 并发执行耗时/ms | 并发比串行快多少 |
| :--- | :--- | :--- | :--- |
| 1百万 | 4 | 52 | 慢 |
| 1千万 | 10 | 53 | 慢 |
| 1亿 | 68 | 84 | 慢 |
| 10亿 | 656 | 354 | 快 |

测试结果显示，当并发执行累计不超过1亿次的时候，速度会比串行执行要慢，因为**线程有创建和上下文切换的开销**。

### 如何减少上下文切换

减少上下文切换的方法有**无锁并发编程**、**CAS算法**、使用**最少线程**和使用**协程**。

* **无锁并发编程**。多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些方法来避免使用锁。
* **CAS算法**。Java的Atomic包使用CAS算法来更新数据，而不需要加锁。
* **使用最少线程**。避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
* **协程**。在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。

## 死锁

**锁**是个非常有用的工具，运用场景非常多，因为它使用起来非常简单，而且易于理解。但同时它也会带来一些困扰，那就是可能会引起死锁，一旦产生死锁，就会造成系统功能不可用。

```java
/*死锁演示*/
public class DeadLockDemo {

    private static final String A = "A";
    private static final String B = "B";

    public static void main(String[] args) {

        new DeadLockDemo().deadLock();
    }

    private void deadLock() {
        Thread t1 = new Thread(() -> {
            synchronized (A){
                try{
                    Thread.currentThread().wait(2000);
                }catch (InterruptedException e){
                    e.printStackTrace();
                }
                synchronized (B) {
                    System.out.println("1");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (B) {
                synchronized (A) {
                    System.out.println("2");
                }
            }
        });
        t1.start();
        t2.start();
    }
}
```

避免死锁的几个常见方法。

* 避免一个线程同时获取多个锁。
* 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源。
* 尝试使用定时锁，使用lock.tryLock\(timeout\)来替代使用内部锁机制。
* 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况。

## 资源限制的挑战

### 什么是资源限制

资源限制是指在进行并发编程时，程序的执行速度受限于计算机硬件资源或软件资源。

### 资源限制引发的问题

在并发编程中，将代码执行速度加快的原则是将代码中串行的部分变成并发执行，但是如果将某段串行的代码并发执行，因为受限于资源，仍然在串行执行，这时候程序不仅不会加快执行，反而会更慢，因为增加了上下文切换是资源调度的时间。

### 如何解决资源限制的问题

对于硬件资源限制，可以考虑使用集群并行执行程序。对于软件资源限制，可以考虑使用资源池将资源复用。

### 在资源限制情况下进行并发编程

如何在资源限制的情况下，让程序执行的更快呢？方法就是，根据不同的资源限制调整程序的并发度，比如下载文件程序依赖于两个资源——带宽和硬盘读写速度。

