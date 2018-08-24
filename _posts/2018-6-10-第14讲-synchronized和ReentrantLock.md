---
layout: post
title: synchronized和ReentrantLock
date: 2018-6-10
tags: Java核心技术36讲笔记
---

### synchronized 和 ReentrantLock 有什么区别？有人说 synchronized 最慢，这话靠谱吗？
- synchronized 是 Java 内建的同步机制，所以也有人称其为 Intrinsic Locking，它提供了互斥的语义和可见性，当一个线程已经获取当前锁时，其他试图获取的线程只能等待或者阻塞在那里，关于synchronized更详细的内容，[点击查看另一篇文章](https://www.heshengbang.tech/2018/05/Java的synchronized关键字/)
- 在 Java 5 以前，synchronized 是仅有的同步手段，在代码中， synchronized 可以用来修饰方法，也可以使用在特定的代码块儿上，本质上 synchronized 方法等同于把方法全部语句用 synchronized 块包起来
- ReentrantLock，通常翻译为再入锁，是 Java 5 提供的锁实现，它的语义和 synchronized 基本相同。再入锁通过代码直接调用 lock() 方法获取，代码书写也更加灵活。
- ReentrantLock 提供了很多实用的方法，能够实现很多 synchronized 无法做到的细节控制，比如可以控制 fairness，也就是公平性，或者利用定义条件等。但是，编码中也需要注意，必须要明确调用 unlock() 方法释放，不然就会一直持有该锁。
- synchronized 和 ReentrantLock 的性能不能一概而论，早期版本 synchronized 在很多场景下性能相差较大，在后续版本进行了较多改进，在低竞争场景中表现可能优于 ReentrantLock。

### 知识点
- 什么是线程安全
- synchronized基本使用，案例
- ReentrantLock基本使用，案例
- synchronized的原理和底层实现
- ReentrantLock的原理和底层实现
- 锁膨胀、降级、偏斜锁、自旋锁、轻量级锁、重量级锁分别代表什么含义
- 并发包中 java.util.concurrent.lock 各种不同实现和案例分析

### 线程安全
- 保证线程安全:
	- 封装：通过封装将对象内部状态隐藏、保护起来
	- 不可变：java中目前没有真正意义上的不可变，即便是final修饰的对象，也只是声明引用不能被重新赋值，而对象的行为却不受影响

- 基本特征:
	- 原子性，简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制实现
	- 可见性，是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上，volatile 就是负责保证可见性的
	- 有序性，是保证线程内串行语义，避免指令重排等

- 线程不安全的实例：
```java
class ThreadUnsafeSample {
    private int sharedState;
    private void unSafeAction() {
        while (sharedState < 100000) {
            int former = sharedState++;
            int latter = sharedState;
            if (former != latter - 1) {
                System.out.println("Observed data race, former is " +
                        former + ", " + "latter is " + latter);
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ThreadUnsafeSample sample = new ThreadUnsafeSample();
        Thread threadA = new Thread(sample::unSafeAction);
        Thread threadB = new Thread(sample::unSafeAction);
        threadA.start();
        threadB.start();
        threadA.join();
        threadB.join();
    }
}
```
是否出现竞争条件，视机器的性能而定。我在自己电脑上测试，控制台打印如下：
```
Observed data race, former is 25907, latter is 25907
```

- 使用synchronized实现线程安全实例:
```java
class ThreadSafeSample {
    private int sharedState;
    private void safeAction() {
        while (sharedState < 100000) {
            // 通过使用内部锁，来解决共享资源的竞争访问
            synchronized (this) {
                int former = sharedState++;
                int latter = sharedState;
                if (former != latter - 1) {
                    System.out.println("Observed data race, former is " +
                            former + ", " + "latter is " + latter);
                }
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ThreadSafeSample sample = new ThreadSafeSample();
        Thread threadA = new Thread(sample::safeAction);
        Thread threadB = new Thread(sample::safeAction);
        threadA.start();
        threadB.start();
        threadA.join();
        threadB.join();
    }
}
```
使用synchronized关键字，用对象内部锁来解决共享资源的竞争访问。也可以通过定义Lock对象实例，来手动加锁解锁解决共享资源访问的问题。

### ReentrantLock
- ReentrantLock就是Java并发编程最常见的类，俗称再入锁。ReentrantLock表示当一个线程试图获取一个它已经获取的锁时，这个获取动作就自动成功，这是对锁获取粒度的一个概念，也就是锁的持有是以线程为单位而不是基于调用次数。Java 锁实现强调再入性是为了和 pthread 的行为进行区分。
- ReentrantLock再入锁可以设置公平性(fairness)，使用者可以在创建锁对象的时候选择是否公平。例如:
```java
ReentrantLock fairLock = new ReentrantLock(true);
```
这里所谓的公平性是指在竞争场景中，当公平性为真时，会倾向于将锁赋予等待时间最久的线程。公平性是减少线程“饥饿”（个别线程长期等待锁，但始终无法获取）情况发生的一个办法。

- ReentrantLock的公平性支持源自自身内部实现的两个内部类，关系如下图所示：
![ReentrantLock继承结构图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/ReentrantLock继承体系图.jpg)

- 使用 synchronized无法进行公平性的选择，其永远是不公平的。只会随机在等待的线程集中选择一个线程唤醒，进而使用共享资源，这也是主流操作系统线程调度的选择。通用场景中，公平性未必有想象中的那么重要，Java 默认的调度策略很少会导致 “饥饿”发生。与此同时，若要保证公平性则会引入额外开销，自然会导致一定的吞吐量下降。所以，只有在程序确实有公平性需要的时候，才有必要指定它。

- 使用ReentrantLock一定要在try{}finally{}中使用，并且一定要在finally{}中去释放锁，即调用unlock()方法。

- Lock可以认为是synchronized的一个替代。在java内置的关键字synchronized的标识下，jvm会使用对象的内置锁对象对共享资源进行加锁，并在同一语句块中解锁。java默认了锁对象的行为和条件遍历。而JDK1.5出现的Lock接口和Condition接口将锁对象和条件变量的控制权交回使用者手中，由使用者自己去定义锁，在锁上创建条件变量，释放锁。Condition条件对象则是将 wait、notify、notifyAll 等操作转化为相应的对象，将复杂而晦涩的同步操作转变为直观可控的对象行为。

- ArrayBlockingQueue和LinkedBlockingQueue是Lock和Condition典型的应用场景，尤其是LinkedBlockingQueue中的putLock和takeLock两个锁对象，以及notEmpty和notFull两个条件对象的使用，对于理解锁和条件对象非常有帮助，建议阅读源码了解。下面摘取部分代码展示：
```java
	/** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();
    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();
    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();
    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
    // 省略部分代码
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
	// 省略部分代码
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```
  上面代码展示了两个锁和两个条件变量在LinkedBlockingQueue中的使用。在阻塞队列中，多有类似的实现，很多时候如果想手写生产者消费者模型，使用阻塞队列是比较好的选择。

- 通过 signal/await 的组合，完成了条件判断和通知等待线程，非常顺畅就完成了状态流转。注意，signal 和 await 成对调用非常重要，不然假设只有 await 动作，线程会一直等待直到被打断（interrupt）。

- 从性能角度，synchronized 早期的实现比较低效，对比 ReentrantLock，大多数场景性能都相差较大。但是在 Java 6 中对其进行了非常多的改进，可以参考性能对比，在高竞争情况下，ReentrantLock 仍然有一定优势。在大多数情况下，无需纠结于性能，还是考虑代码书写结构的便利性、可维护性等。


[参考](https://time.geekbang.org/column/article/8799)