---
description: The underlying principle of Java concurrency mechanism.
---

# Java并发机制的底层实现原理

Java代码在编译后会编程Java字节码，字节码被类加载器加载到JVM里，JVM执行字节码，最终需要转化为汇编指令在CPU上执行，Java中所使用的并发机制以来与JVM的实现和CPU的指令。

## volatile的应用

在多线程并发编程中synchronized和volatile都扮演着重要的角色，volatile是轻量级的synchronized，它在多处理开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另一个线程能读到这个修改的值。如果volatitle变量修饰符使用恰当的话，它比synchronized的使用和执行成本更低，因为他不会引起线程上下文的切换和调度。

### volatile的定义与实现原理

**volatile定义**：Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

Java语言提供了volatile，在某些情况下比锁更加方便。如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。

**CPU的术语定义**

|                             术语                             |        英文单词        | 术语描述                                                     |
| :----------------------------------------------------------: | :--------------------: | ------------------------------------------------------------ |
| 内存屏障 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |    menmory barrires    | 一组处理器指令，用于实现对内存操作的顺序限制                 |
|                            缓冲行                            |       cache line       | CPU高速缓存中可以分配的最小存储单位。处理器填写缓存行时会加载整个缓存行，现代CPU需要执行几百次CPU指令 |
|                           原子操作                           |   atomic operations    | 不可中断的一个或一系列操作                                   |
|                          缓存行填充                          |    cacha line fill     | 当处理器识别到从内存中读取操作数是可缓存的，处理器读取整个高速缓存行到适当的缓存(L1，L2，L3的或所有) |
|                           缓存命中                           |       cache hit        | 如果进行高速缓存行填充操作的内存位置仍然是下次处理器访问的地址时，处理从缓存中读取操作数，而不是从内存读取 |
|                            写命中                            |       write hit        | 当处理器将操作数写回到一个内存缓存的区域时，它首先会检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数写回到缓存，而不是写回到内存 |
|                            写缺失                            | write misses the cache | 一个有效的缓存行被写入到不存在的内存区域                     |

**volatile是如何来保证可见性的呢？**

Java代码如下：

```java
instance = new Singleton();
```

转换为汇编代码，如下：

```assembly
0x01a3deld: movb $0×0,0×1104800(%esi);0x01a3de24: lock add1 $0×0,(%esp);
```

有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码，Lock前缀的指令在多核处理器下会引发两件事情。

- 将当前处理器缓存行的数据写回到系统内存。
- 这个写回内存的操作会使在其他CPU里缓存了改内存地址的数据无效。

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存(L1，L2或其他)后再进行操作，但操作完不知道何时会写到内存。如果对声明了volatile的变量进行写操作，JVM会向处理器发送一条Lock前缀的指令，将这个变量所在的缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以在多处理器下，为了保证各个处理器的缓存是一致的，就回实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理的缓存行设置成无效状态，当处理器对这个数据进行修改操作时，会重新从系统内存中吧数据读到处理缓存里。

**volatile的两条实现原则**

- Lock前缀指令会引起处理器缓存会写到内存。
- 一个处理器的缓存回写到内存会导致其他处理器的缓存无效。

### volatile的使用优化

著名的Java并发编程大师Doug lea在JDK 7的并发包里新增一个队列集合类LinkedTransferQueue，它在使用volatile变量时，用一种追加字节的方式来优化队列出队和入队的性能。

LinkedTransferQueue的代码如下：

```java
/*队列中的头部节点*/
private transient final PaddedAtomicReference<QNode> head;
/*队列中的尾部节点*/
private transient final PaddedAtomicReference<QNode> tail;
static final class PaddedAtomicReference <T> extends AtomicReference <T> {
    //使用很多4个字节的引用追加到64字节
    Object P0, p1, p2, p3, p4, p5, p6, p7, p8, p9, pa, pb, pc, pd, pe;
    PaddedAtomicReference(T r) {
        super();
    }
}
public class AtomicReference <V> implements java.io.Serializable {
    private volatile V value;
    //省略其他代码
}
```

**追加字节能优化性能**。`LinkedTransferQueue`这个类，它使用一个内部类类型来定义队列的头节点和尾节点，而这个内部类`PaddedAtomicReference`相对于父类`AtomicReference`只做了一件事情，就是将共享变量追加到64字节。有个对象的引用占4个字节，追加了15个占60个字节，再加上父类的value变量共64个字节。

**为什么追加64字节能够提高并发编程的效率呢**？对于目前主流的处理器的L1，L2或L3缓存的高速缓存行是64个字节宽，不支持部分填充缓存行，这意味着，如果队列的头节点和尾节点都不足64字节的话，处理器会将他们都读到同一个高速缓存行中，在多处理器下每个处理器都缓存同样的头、尾节点，当一个处理器试图修改头节点时，会将整个缓存行锁定，那么在缓存一致性机制的作用下，会导致其他处理器不能访问自己高速缓存中的尾节点，而队列的入队和出队操作则需要不停修改头节点和尾节点，所以在多处理器的情况下将会严重影响到队列的效率。使用追加到64字节的方式来填满高速缓冲区的缓存行，避免头、尾节点加载到同一个缓存行，使头、尾节点在修改时不会互相锁定。

**那么是不是在使用volatile变量时都应该追加到64字节呢？**

以下两种场景不应该使用这种方式：

- **缓存行非64字节宽的处理器。**
- **共享变量不会被频繁的写。**

## synchronized的实现原理与应用

在多线程并发编程中`synchronized`一直是元老级的角色，很多人会称呼它为重量级锁。

利用`synchronized`实现同步的基础：Java中的每一个对象都可以作为锁。表现为以下3种形式：

- 对于普通同步方法，锁是当前实例对象。
- 对于静态同步方法，锁是当前类的Class对象。
- 对于同步方法块，锁是`Synchonized`括号里配置的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。从JVM规范中可以看到`Synchonized`在JVM里的实现原理，JVM基于进入和退出`Monitor`对象来实现方法同步和代码块同步，但两者的实现细节不一样。具体可以详细查阅JVM规范。

### Java对象头

`synchronized`用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽存储对象头，如果对象是非数组类型，则用2个字款存储对象头。在32位虚拟机中，1字款等于4字节，即32bit。

### 锁的升级与对比

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，目的是为了提高获得锁和释放锁的效率。

#### 偏向锁

大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获得锁时，会在对象头和栈帧中的锁记录里存储偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需要简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置为1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

#### 偏向锁的撤销

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他的线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

#### 关闭偏向锁

偏向锁在Java 6和Java 7里默认启用的，但是它在应用程序启动几秒之后才激活，如有必要可以使用JVM参数来关闭延迟：`-XX:BiasedLockingStartupDelay=0`。如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：`-XX:-UseBiasedLocking=false`，那么程序默认会进入轻量级锁状态。

### 轻量级锁

#### 轻量级锁加锁

线程在执行同步块之间，JVM会先在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头重的Mark Word复制到锁记录中，官方称为DIsplaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获得锁。

#### 轻量级锁解锁

轻量级锁解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

### 锁的优缺点对比

|                           锁                           |                             优点                             |                      缺点                      |              适用场景              |
| :----------------------------------------------------: | :----------------------------------------------------------: | :--------------------------------------------: | :--------------------------------: |
| 偏向锁&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 加锁和解锁不需要额外的消耗，和执行非同步方法相比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 |  适用于只有一个线程访问同步块场景  |
|                        轻量级锁                        |           竞争的线程不会阻塞，提高了程序的响应速度           | 如果始终得不到锁竞争的线程，适用自旋会消耗CPU  | 追求响应时间，同步块执行速度非常快 |
|                        重量级锁                        |               线程竞争不适用自旋，不会消耗CPU                |             线程阻塞，响应时间缓慢             |   追求吞吐量，同步块执行速度较快   |

## 原子操作的实现原理

原子（atomic）本意是“不能被进一步分割的最小粒子”，而原子操作意为“不可被中断的一个或一系列操作”。

### 术语定义

**CPU术语定义**

| 术语名称                                                     | 英文                   | 解锁                                                         |
| ------------------------------------------------------------ | ---------------------- | ------------------------------------------------------------ |
| 缓存行&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Cache line             | 缓存的最小操作单位                                           |
| 比较并交换                                                   | Compare and Swap       | CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换 |
| CPU流水线                                                    | CPU pipeline           | CPU流水线的工作方式就像工业生产上的装配流水线，在CPU中由5-6个不同功能的电路单元组成一条指令处理流水线，然后将一条X86指令分成5-6步后再由这些电路单元分别执行，这样就能实现在一个CPU时钟周期完成一条指令，因此提高CPU的运算速度 |
| 内存顺序冲突                                                 | Memory order violation | 内存顺序冲突一般是由假共享引起的，假共享是指多个CPU同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效，当出现这个内存顺序 |

### 处理器如何实现原子操作

32位IA-32处理器使用基于对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作。Pentium 6 和最新的处理器能自动保证单处理器对同一个缓存里进行16/32/64位的操作是原子的，但是复杂的内存操作处理器是不能自动保证其原子性的，比如跨总线宽度、跨多个缓存行和跨页表的访问。

#### 使用总线锁保证原子性

**第一个机制是通过总线锁保证原子性。**如果多个处理器同时对共享变量进行读写改操作（i++就是经典的读写改操作），那么共享变量就回被多个处理器同时进行操作，这样读写改操作就不是原子性的，操作完之后共享变量的值会和期望的不一致。举个例子，如果i=1，进行两次i++操作，我们期望的结果是3，但是有可能结果是2。因为多个处理器同时从各自的缓存中读取变量i，分别进行加1操作，然后分别写入系统内存中。

那么，想要保证度读写改共享变量的操作是原子性的，就必须保证CPU1读写改共享内存的时候，CPU2不能操作缓存了该共享变量内存地址的缓存。

处理器使用总线锁就可以解决这个问题。所谓总线锁就锁使用处理器提供的一个LOCK #信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。

#### 使用缓存锁保证原子性

**第二个机制是通过缓存锁来保证原子性。**在同一时刻，我们只需保证对某个内存地址的操作是原子性的，但总线锁把CPU和内存之间的通信锁定了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁的开销比较大，目前处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

频繁使用的内存会缓存在处理器的L1，L2，L3高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁。所谓“**缓存锁定**”是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声明LOCK #信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

**但是有两种情况下处理器不会使用缓存锁定。**

- 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行时，则处理器会调用总线锁定。
- 有些处理器不支持缓存锁定。

### Java如何实现原子操作

在Java中可以通过**锁**和**循环CAS**的方式来实现原子操作。

#### 使用循环CAS实现原子操作

JVM中的CAS操作正是利用了处理器提供的CMPXCHG指令实现的。自旋CAS实现的基本思路就是循环进行CAS操作直至成功为止，以下代码实现了一个基于CAS线程安全的计算器方法safeCount和一个非线程安全的计数器count。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    private AtomicInteger atomicI = new AtomicInteger(0);
    private int i = 0;

    public static void main(String[] args) {
        final Counter cas = new Counter();
        List<Thread> ts = new ArrayList<Thread>(600);
        long start = System.currentTimeMillis();
        for (int j = 0; j < 100; j++){
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++){
                        cas.count();
                        cas.safeCount();
                    }
                }
            });
            ts.add(t);
        }
        for (Thread t : ts) {
            t.start();
        }
        /*等待所有线程执行完成*/
        for (Thread t : ts) {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(cas.i);
        System.out.println(cas.atomicI.get());
        System.out.println(System.currentTimeMillis() - start);
    }
    /*使用CAS实现线程安全计数器*/
    private void safeCount() {
        for (;;) {
            int i = atomicI.get();
            boolean suc = atomicI.compareAndSet(i, ++i);
            if (suc) {
                break;
            }
        }
    }
    /*非线程安全计数器*/
    private void count() {
        i++;
    }
}
```

从Java 1.5开始，JDK的并发包里提供了一些类来支持原子操作，如`AtomicBoolean`（用原子的方式更新boolean值）、`AtomicInteger`（用原子方式更新的int值）和`AtomicLong`（用原子方式更新long值）。这些原子包装类还提供了有用的工具方法，比如用原子的方式将当前值自增1和自减1。

#### CAS实现原子操作的三大问题

在Java并发包中有一些并发框架也使用了自旋CAS的方式来实现原子操作，比如`LinkedTransferQueue`类的Xfer方法。CAS虽然很高效地解决了原子操作，但是CAS仍然存在三大问题。ABA问题，循环时间长开销大，以及只能保证一个共享变量的原子操作。

##### ABA问题

因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A->B->A就会变成1A->2B->3A。从Java 1.5开始，JDK的Atomic包里提供了一个类`AtomicStampedReference`来解决ABA问题。这个类的`compareAndSet`方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

```java
public boolean compareAndSet(
						V expectedReference,	//预期引用
						V newReference,			  //更新后的引用
						int expectedStamp,		//预期标志
						int newStamp,					//更新后的标志
)
```

##### 循环时间长开销大

自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升。pause指令有两个作用：

- 可以延迟流水线执行指令，使CPU不会消耗过多的执行资源。
- 可以避免在退出循环的时候因内存顺序冲突而引起的CPU流水线被清空，从而提高CPU的执行效率。

##### 只能保证一个共享变量的原子操作

对多个共享内存操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的方法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i=2，j=a，合并一下ij=2a，然后用CAS来操作ij，JDK提供了`AtomicReference`类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

#### 使用锁机制实现原子操作

锁机制保证了只有获得锁的线程才能操作锁定的内存区域，JVM内部实现了很多中锁机制，有偏向锁、轻量级锁和互斥锁。除了偏向锁，JVM实现锁的方式都用了循环CAS，即当一个线程想进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块的时候使用循环CAS释放锁。