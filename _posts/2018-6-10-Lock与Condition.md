---

layout: post

title: Lock与Condition

date: 2018-6-10

tags: Java基础

---

### Lock
- 概述
	- Lock 实现提供了比使用 synchronized 方法和语句可获得的更广泛的锁定操作。此实现允许更灵活的结构，可以具有差别很大的属性，可以支持多个相关的 Condition 对象。
	- 锁是控制多个线程对共享资源进行访问的工具。通常，锁提供了对共享资源的独占访问。一次只能有一个线程获得锁，对共享资源的所有访问都需要首先获得锁。不过，某些锁可能允许对共享资源并发访问，如ReadWriteLock 的读取锁。
	- synchronized 方法或语句的使用提供了对与每个对象相关的隐式监视器锁的访问，但却强制所有锁获取和释放均要出现在一个块结构中：当获取了多个锁时，它们必须以相反的顺序释放，且必须在与所有锁被获取时相同的词法范围内释放所有锁。
	- 虽然 synchronized 方法和语句的范围机制使得使用监视器锁编程方便了很多，而且还帮助避免了很多涉及到锁的常见编程错误，但有时也需要以更为灵活的方式使用锁。例如，某些遍历并发访问的数据结果的算法要求使用 "hand-over-hand" 或 "chain locking"：获取节点 A 的锁，然后再获取节点 B 的锁，然后释放 A 并获取 C，然后释放 B 并获取 D，依此类推。Lock 接口的实现允许锁在不同的作用范围内获取和释放，并允许以任何顺序获取和释放多个锁，从而支持使用这种技术。
	- 随着灵活性的增加，也带来了更多的责任。不使用块结构锁就失去了使用 synchronized 方法和语句时会出现的锁自动释放功能。在大多数情况下，应该使用以下语句：
	```java
        Lock lock = new SomeImplementation();
        lock.lock();
        try {
            // 接触被这个锁保护的资源
        } finally {
            l.unlock();
        }
    ```
	- 锁定和取消锁定出现在不同作用范围中时，必须谨慎地确保保持锁定时所执行的所有代码用 try-finally 或 try-catch 加以保护，以确保在必要时释放锁。
	- Lock 实现提供了使用 synchronized 方法和语句所没有的其他功能，包括提供了一个非块结构的获取锁尝试 (tryLock())、一个获取可中断锁的尝试 (lockInterruptibly()) 和一个获取超时失效锁的尝试 (tryLock(long, TimeUnit))。
	- Lock 类还可以提供与隐式监视器锁完全不同的行为和语义，如保证排序、非重入用法或死锁检测。如果某个实现提供了这样特殊的语义，则该实现必须对这些语义加以记录。
	- 注意，Lock 实例只是普通的对象，其本身可以在 synchronized 语句中作为目标使用。获取 Lock 实例的监视器锁与调用该实例的任何 lock() 方法没有特别的关系。为了避免混淆，建议除了在其自身的实现中之外，决不要以这种方式使用 Lock 实例。
	- Lock的实现类：ReentrantLock。
	- Lock的有关的接口ReadWriteLock，有关的类ReentrantReadWriteLock

- 源码如下:
```java
public interface Lock {
        void lock();
        void lockInterruptibly() throws InterruptedException;
        boolean tryLock();
        boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
        void unlock();
        Condition newCondition();
}
```

- 方法
	- `void lock();`获取锁。如果锁不可用，出于线程调度目的，将禁用当前线程，并且在获得锁之前，该线程将一直处于休眠状态。
	- `void lockInterruptibly() throws InterruptedException;`如果当前线程未被中断，则获取锁。如果锁可用，则获取锁，并立即返回。如果锁不可用，出于线程调度目的，将禁用当前线程，并且在发生以下两种情况之一以前，该线程将一直处于休眠状态:① 锁由当前线程获得 ② 其他某个线程中断当前线程，并且支持对锁获取的中断。如果当前线程：进入此方法时已经设置了该线程的中断状态 或者 在获取锁时被中断并且支持对锁获取的中断。则将抛出 InterruptedException，并清除当前线程的已中断状态
	- `boolean tryLock();`如果锁可用，则获取锁，并立即返回值 true。如果锁不可用，则此方法将立即返回值 false。此用法可确保如果获取了锁，则会释放锁，如果未获取锁，则不会试图将其释放。此方法的典型使用语句如下:
	```java
        Lock lock = ...;
        if (lock.tryLock()) {
            try {
                // 操作受保护的共享资源
            } finally {
                lock.unlock();
            }
        } else {
            // 如果没获得锁进行的操作
        }
    ```
	- `boolean tryLock(long time, TimeUnit unit) throws InterruptedException;`如果锁在给定的等待时间内处于可获得的状态，并且当前线程未被中断，则获取锁并返回true。如果超过了指定的等待时间也没获得锁，则将返回值 false。如果 time 小于等于 0，该方法将完全不等待。
	- `void unlock();`释放锁
	- `Condition newCondition();`返回一个绑定到此 Lock 实例的新 Condition 实例。在等待条件前，锁必须由当前线程保持。调用 Condition.await() 将在等待前以原子方式释放锁，并在等待返回前重新获取锁。

### Condition
- 概述
	- Condition 将 Object 监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意 Lock 实现组合使用，为每个对象提供多个等待 set（wait-set）。其中，<b>Lock 替代了 synchronized 方法和语句的使用，Condition 替代了 Object 监视器方法的使用</b>。
	- 条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。
	- Condition 实例实质上被绑定到一个锁上。要为特定 Lock 实例获得 Condition 实例，请使用其 newCondition() 方法。
	- 作为一个示例，假定有一个绑定的缓冲区，它支持 put 和 take 方法。如果试图在空的缓冲区上执行 take 操作，则在某一个项变得可用之前，线程将一直阻塞；如果试图在满的缓冲区上执行 put 操作，则在有空间变得可用之前，线程将一直阻塞。我们喜欢在单独的等待 set 中保存 put 线程和 take 线程，这样就可以在缓冲区中的空间变得可用时利用最佳规划，一次只通知一个线程。可以使用两个 Condition 实例来做到这一点。在ArrayBlockingQueue和LinkedBlockingQueue中有类似的实现。
	```java
	class BoundedBuffer {
            final Lock lock = new ReentrantLock();
            final Condition notFull  = lock.newCondition();
            final Condition notEmpty = lock.newCondition();
            final Object[] items = new Object[100];
            int putptr, takeptr, count;
            public void put(Object x) throws InterruptedException {
                lock.lock();
                try {
                    while (count == items.length)
                        notFull.await();
                    items[putptr] = x;
                    if (++putptr == items.length)
                    	putptr = 0;
                    ++count;
                    notEmpty.signal();
                } finally {
                    lock.unlock();
                }
            }
            public Object take() throws InterruptedException {
                lock.lock();
                try {
                    while (count == 0)
                        notEmpty.await();
                    Object x = items[takeptr];
                    if (++takeptr == items.length) takeptr = 0;
                    --count;
                    notFull.signal();
                    return x;
                } finally {
                    lock.unlock();
                }
            }
	}
    ```
	- Condition 实现可以提供不同于 Object 监视器方法的行为和语义，比如受保证的通知排序，或者在执行通知时不需要保持一个锁。如果某个实现提供了这样特殊的语义，则该实现必须记录这些语义。
	- Condition 实例只是一些普通的对象，它们自身可以用作 synchronized 语句中的目标，并且可以调用自己的 wait 和 notification 监视器方法。获取 Condition 实例的监视器锁或者使用其监视器方法，与获取和该 Condition 相关的 Lock 或使用其 waiting 和 signalling 方法没有什么特定的关系。为了避免混淆，建议除了在其自身的实现中之外，切勿以这种方式使用 Condition 实例。

- 源码如下:
```java
public interface Condition {
        void await() throws InterruptedException;
        void awaitUninterruptibly();
        long awaitNanos(long nanosTimeout) throws InterruptedException;
        boolean await(long time, TimeUnit unit) throws InterruptedException;
        boolean awaitUntil(Date deadline) throws InterruptedException;
        void signal();
        void signalAll();
}
```

- 方法
	- `void await() throws InterruptedException;`造成当前线程在接到信号或被中断之前一直处于等待状态。
	- `void awaitUninterruptibly();`造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
	- `long awaitNanos(long nanosTimeout) throws InterruptedException;`造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法的典型用法采用以下形式:
	```java
	synchronized boolean example(long timeout, TimeUnit unit) {
            long nanosTimeout = unit.toNanos(timeout);
            while (!conditionBeingWaitedFor) {
                if (nanosTimeout > 0)
                    nanosTimeout = theCondition.awaitNanos(nanosTimeout);
                else
                    return false;
            }
            // ...
	}
    ```
	- `boolean await(long time, TimeUnit unit) throws InterruptedException;`造成当前线程在接到信号之前一直处于等待状态。
	- `boolean awaitUntil(Date deadline) throws InterruptedException;`造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。
	- `void signal();`如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
	- `void signalAll();`如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。