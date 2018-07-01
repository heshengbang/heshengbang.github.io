---
layout: post
title: AtomicInteger底层实现原理与CAS应用
date: 2018-7-1
tags: Java核心技术36讲笔记
---

### AtomicInteger 底层实现原理是什么？如何在自己的产品代码中应用 CAS 操作？
- AtomicInteger是对int类型的一个封装，提供原子性的访问和更新操作，其原子性操作的实现是基于CAS([compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap))技术。
	- 所谓CAS，表征的是一些列操作的集合，获取当前数值，进行一些运算，利用CAS指令试图进行更新。如果当前数值未变，代表没有其他线程进行并发修改，则成功更新。否则，可能出现不同的选择，要么进行重试，要么就返回一个成功或者失败的结果。
- 从AtomicInteger的内部属性可以看出，它依赖于Unsafe提供的一些底层能力，进行底层操作；同时，以volatile的value字段，记录数值，以保证可见性。见源码如下：
```
    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    private volatile int value;
```
  具体的原子操作细节，可以参考任意一个原子更新方法，比如下面的 getAndIncrement。Unsafe 会利用 value 字段的内存地址偏移，直接完成操作。源码如下：
```
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
```
  因为 getAndIncrement 需要返归数值，所以需要添加失败重试逻辑。
```
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
```
  而类似 compareAndSet 这种返回 boolean 类型的函数，因为其返回值表现的就是成功与否，所以不需要重试。
```
	public final boolean compareAndSet(int expectedValue, int newValue)
```
  CAS 是 Java 并发中所谓 lock-free 机制的基础。应当注意compareAndSet与compareAndSwap的区别，不要混为一谈。

### 关键点
- 虽然在日常开发中未必会涉及到CAS的实现层面，但是理解其中的机制，掌握如何在Java中运用该技术还是十分有必要的，并发编程相关的面试中经常会涉及到。
- CAS更加底层的实现依赖于CPU提供的特定指令，具体根据操作系统结构的不同还存在着明显的区别。比如，x86 CPU提供cmpxchg指令；而在精简指令集的体系架构中，则通常是靠一对指令（如“load and reserve”和“store conditional”）实现的，在大多数处理器上CAS都是非常轻量级的操作，这也是其优势所在。通常，了解到这个程度就已经足够，再底层的指令，对于一般的开发工作来讲，几乎没有帮助。
- 在什么场景下可以采用CAS技术，调用Unsafe毕竟不是大多数场景的最好选择，有没有更加推荐的方式呢
- 对于ReentrantLock、CyclicBarrier等并发结构底层的实现技术的理解

### CAS使用
- 场景：在数据库产品中，为了保证索引的一致性，一个常见的选择是，保证只有一个线程能够排他性的修改一个索引分区，如何在数据库抽象层面去实现这一操作？
- 可以考虑为索引分区对象添加一个逻辑上的锁，例如，以当前独占的线程ID为锁的数值，然后通过原子操作设置lock数值，来实现加锁和释放锁，伪代码如下：
```
    public class AtomicBTreePartition {
        private volatile long lock;
        public void acquireLock(){}
        public void releaseeLock(){}
    }
```
- 在Java代码中，实现锁操作，Unsafe似乎不是最好的选择。例如，类似Cassandra等产品，因为Java 9中移除了Unsafe.monitorEnter()/monitorExit()，导致无法平滑升级到JDK新版本。目前Java 提供了两种公用API来实现CAS操作，比如使用java.util.concurrent.atomic.AtomicLongFieldUpdater，它是基于发射机制创建，我们需要保证类型和字段名称的正确性。
```
    private static final AtomicLongFieldUpdater<AtomicBTreePartition> lockFieldUpdater =
            AtomicLongFieldUpdater.newUpdater(AtomicBTreePartition.class, "lock");
    private void acquireLock(){
        long t = Thread.currentThread().getId();
        while (!lockFieldUpdater.compareAndSet(this, 0L, t)){
            // 等待一会儿，数据库操作可能比较慢
             …
        }
    }
```

- Atomic包提供了最常用的原子性数据类型，甚至是引用、数组等相关原子类型和更新操作工具，是很多线程安全程序的首选。
- 使用原子数据类型和 Atomic*FieldUpdater，创建更加紧凑的计数器实现，以替代 AtomicLong。优化永远是针对特定需求、特定目的，这里的侧重点是介绍可能的思路，具体还是要看需求。如果仅仅创建一两个对象，其实完全没有必要进行前面的优化，但是如果对象成千上万或者更多，就要考虑紧凑性的影响了。而 atomic 包提供的LongAdder，在高度竞争环境下，可能就是比AtomicLong更佳的选择，尽管它的本质是空间换时间。
- 一般来说，进行的类似CAS操作，可以并且推荐使用Variable Handle API去实现，其提供了精细粒度的公共底层API。我在这里强调公共，是因为其API不会像内部API那样，发生不可预测的修改，这一点提供了对于未来产品维护和升级的保障基础，坦白说，很多额外工作量，都是源于我们使用了Hack而非Solution的方式去解决问题。

- CAS也并不是没有副作用，试想，其常用的失败重试机制，隐含着一个假设，即竞争情况是短暂的。在大多数应用场景中，确实大部分重试只会发生一次就获得了成功，但是总是有意外情况，所以在有需要的时候，还是要考虑限制自旋的次数，以免过度消耗CPU。
- 另外一个就是注明的ABA问题，这是通常只在lock-free算法下暴露的问题。前面提到过，CAS是在更新时比较前值，如果对方只是恰好相同，例如期间发生了A -> B -> A的更新，仅仅判断数值是A，可能导致不合理的修改操作。针对这种情况，java提供了AtomicStampedReference工具类，通过引用建立类似版本号（stamp）的方式，来保证CAS的正确性，具体用法请[参考](http://tutorials.jenkov.com/java-util-concurrent/atomicstampedreference.html)
- 大多数情况下，Java开发者并不需要直接利用CAS代码去实现线程安全容器等，更多是通过并发包间接享受到lock-free机制在扩展性上的好处。

### AbstractQueueSynchronizer(AQS)
- AbstractQueuedSynchronizer（AQS）是 Java 并发包中，实现各种同步结构和部分其他组成单元（如线程池中的 Worker）的基础。
- 对于AbstractQueueSynchronizer需要了解四点：
	- 它的正确拼写方法和读法（别笑，确实很有帮助）
	- 为什么需要AbstractQueueSynchronizer
	- 如何使用AbstractQueueSynchronizer
	- 结合JDK源码中的实践，理解AQS的原理和应用

- AQS 的设计初衷：从原理上，一种同步结构往往是可以利用其他的结构实现的，例如使用 Semaphore 实现互斥锁。但是，对某种同步结构的倾向，会导致复杂、晦涩的实现逻辑，所以，AQS选择了将基础的同步相关操作抽象在 自己的实现中，利用 AQS 为我们构建同步结构提供了范本。
- AQS 内部数据和方法，可以简单拆分为：
	- 一个 volatile 的整数成员表征状态，同时提供了 setState 和 getState 方法
	```
    	private volatile int state;
    ```
    - 一个先入先出（FIFO）的等待线程队列，以实现多线程间竞争和等待，这是 AQS 机制的核心之一。
    - 各种基于 CAS 的基础操作方法，以及各种期望具体同步结构去实现的 acquire/release 方法。

- 利用 AQS 实现一个同步结构，至少要实现两个基本类型的方法，分别是 acquire 操作，获取资源的独占权；还有就是 release 操作，释放对某个资源的独占。以 ReentrantLock 为例，它内部通过扩展 AQS 实现了 Sync 类型，以 AQS 的 state 来反映锁的持有情况。
- 下面是 ReentrantLock 对应 acquire 和 release 操作，如果是 CountDownLatch 则可以看作是 await()/countDown()，具体实现也有区别。
```
    // acquire
    public void lock() {
        sync.lock();
    }
    // NonFairSync
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    // FairSync
    final void lock() {
        acquire(1);
    }
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    // release
    public void unlock() {
        sync.release(1);
    }
    // AQS
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
                return true;
        }
        return false;
    }
```
  排除掉一些细节，整体地分析 acquire 方法逻辑，其直接实现是在 AQS 内部，调用了 tryAcquire 和 acquireQueued，这是两个需要搞清楚的基本部分。
	- 首先，我们来看看 tryAcquire。在 ReentrantLock 中，tryAcquire 逻辑实现在 NonfairSync 和 FairSync 中，分别提供了进一步的非公平或公平性方法，而 AQS 内部 tryAcquire 仅仅是个接近未实现的方法（直接抛异常），这是留个实现者自己定义的操作。ReentrantLock在创建时就会确定，是公平锁还是非公平锁。
	- 以非公平的 tryAcquire 为例，其内部实现了如何配合状态与 CAS 获取锁，注意，对比公平版本的 tryAcquire，它在锁无人占有时，并不检查是否有其他等待者，这里体现了非公平的语义。
	- 接下来我再来分析 acquireQueued，如果前面的 tryAcquire 失败，代表着锁争抢失败，进入排队竞争阶段。这里就是我们所说的，利用 FIFO 队列，实现线程间对锁的竞争的部分，算是是 AQS 的核心逻辑。
	- 当前线程会被包装成为一个排他模式的节点（EXCLUSIVE），通过 addWaiter 方法添加到队列中。acquireQueued 的逻辑，简要来说，就是如果当前节点的前面是头节点，则试图获取锁，一切顺利则成为新的头节点；否则，有必要则等待，具体处理逻辑请参考代码中添加的注释。
	```
        final boolean acquireQueued(final Node node, int arg) {
            boolean interrupted = false;
            try {
                // 循环
                for (;;) {
                    // 获取前一个节点
                    final Node p = node.predecessor();\
                    // 如果前一个节点是头结点，表示当前节点合适去 tryAcquire
                    if (p == head && tryAcquire(arg)) {
                        // acquire 成功，则设置新的头节点
                        setHead(node);
                        // 将前面节点对当前节点的引用清空
                        p.next = null;
                        return interrupted;
                    }
                    // 检查是否失败后需要 park
                    if (shouldParkAfterFailedAcquire(p, node))
                        interrupted |= parkAndCheckInterrupt();
                }
            } catch (Throwable t) {
                // 出现异常，取消
                cancelAcquire(node);
                if (interrupted)
                    selfInterrupt();
                throw t;
            }
        }
    ```

- 到这里线程试图获取锁的过程基本展现出来了，tryAcquire 是按照特定场景需要开发者去实现的部分，而线程间竞争则是 AQS 通过 Waiter 队列与 acquireQueued 提供的，在 release 方法中，同样会对队列进行对应操作。


































