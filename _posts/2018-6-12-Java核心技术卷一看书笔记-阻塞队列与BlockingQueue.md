---
layout: post
title: 阻塞队列与BlockingQueue
date: 2018-6-12
tags: 看书笔记
---

### 本文目录
- 简述
- Queue
- BlockingQueue
- 阻塞队列


### Queue
- Queue译成中文就是队列，用于在处理元素前用于保存元素的集合(collection)。除了基本的 Collection 操作外，队列还提供其他的插入、提取和检查操作。每个方法都存在两种形式：一种抛出异常（操作失败时），另一种返回一个特殊值（null 或 false，具体取决于操作）。

- 队列通常（但并非一定）以 FIFO（先进先出）的方式排序各个元素。不过优先级队列和 LIFO 队列（或堆栈）例外，前者根据提供的比较器或元素的自然顺序对元素进行排序，后者按 LIFO（后进先出）的方式对元素进行排序。无论使用哪种排序方式，队列的头 都是调用 remove() 或 poll() 所移除的元素。在 FIFO 队列中，所有的新元素都插入队列的末尾。其他种类的队列可能使用不同的元素放置规则。每个 Queue 实现必须指定其顺序属性。

- 如果可能，offer 方法可插入一个元素，否则返回 false。这与 Collection.add 方法不同，该方法只能通过抛出未经检查的异常使添加元素失败。offer 方法设计用于正常的失败情况，而不是出现异常的情况，例如在容量固定（有界）的队列中。

- remove() 和 poll() 方法可移除和返回队列的头。哪个元素算队列的头是队列排序策略的功能，而该策略在各种实现中是不同的。remove()和poll()方法仅在队列为空时其行为有所不同：remove()方法抛出一个异常，而 poll() 方法则返回 null。

- element() 和 peek() 都是查询操作，获取队列的头，至于哪个元素算队列的头是队列排序策略的功能，而该策略在Queue的不同实现类中有所不同。这两个方法都是获取队列的头，仅在队列为空获取不到的时候，产生不同的结果，element会抛出异常，而peek则返回null。

- Queue 实现通常不允许插入 null 元素，尽管某些实现（如 LinkedList）并不禁止插入 null。即使在允许 null 的实现中，也不应该将 null 插入到 Queue 中，因为 null 也被用作 poll 方法的一个特殊返回值，表明队列不包含元素(即，队列为空时，使用poll()方法获取队列头时，会返回null，此时无法区分究竟队列是空还是头元素为null)。

- Quue继承自Java中使用最广泛的一个接口Collection接口，是Java Collections Framework的成员，拥有常见集合框架都具备的一些方法，可以像使用多数集合类那样去使用它。Queue的继承体系图如下：
![Queue继承体系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/BlockingQueue继承体系图.jpg)

- Queue的源码如下：
```java
public interface Queue<E> extends Collection<E> {
        boolean add(E e);
        boolean offer(E e);
        E remove();
        E poll();
        E element();
        E peek();
}
```
	- Queue的方法从操作类型分为三类
		- 插入：add()/offer()
		- 移除：remove()/poll()
		- 查找：element()/peek()
	- 从结果处理则分为两类
		- 抛出异常：add()/remove()/element()
		- 返回特殊值：offer()/poll()/peek()

- Quue的方法
	- 抛出异常
		- 插入：`boolean add(E e);` 将指定的元素插入此队列（如果立即可行且不会违反容量限制），在成功时返回 true，如果当前没有可用的空间，则抛出 IllegalStateException。
		- 移除：`E remove();` 获取并移除此队列的头元素。此方法与 poll 唯一的不同在于：此队列为空时将抛出一个异常。
		- 检查：`E element();` 获取，但是不移除此队列的头。此方法与 peek 唯一的不同在于：此队列为空时将抛出一个异常。
	- 返回特殊值
		- 插入：`boolean offer(E e);` 将指定的元素插入此队列（如果立即可行且不会违反容量限制），当使用有容量限制的队列时，此方法通常要优于 add(E)，后者可能无法插入元素，而只是抛出一个异常。
		- 移除：`E poll();` 获取并移除此队列的头，如果此队列为空，则返回 null。
		- 检查：`E peek();` 获取但不移除此队列的头；如果此队列为空，则返回 null。


### BlockingQueue
- BlockingQueue 不接受 null 元素。试图 add、put 或 offer 一个 null 元素时，某些实现会抛出 NullPointerException。null 被用作指示 poll 操作失败的警戒值。

- BlockingQueue 可以是限定容量的。它在任意给定时间都可以有一个 remainingCapacity，超出此容量，便无法无阻塞地 put 附加元素。没有任何内部容量约束的 BlockingQueue 总是报告 Integer.MAX_VALUE 的剩余容量。例如：ArrayBlockingQueue是有界的，创建时必须给予初始容量，而LinkedBlockingQueue则是不限容量的。

- BlockingQueue 实现主要用于生产者-使用者队列，但它另外还支持 Collection 接口。因此，举例来说，使用 remove(x) 从队列中移除任意一个元素是有可能的。然而，这种操作通常不会有效执行，只能有计划地偶尔使用，比如在取消排队信息时。

- BlockingQueue 实现是线程安全的。所有排队方法都可以使用内部锁或其他形式的并发控制来自动达到它们的目的。然而，大量的 Collection 操作（addAll、containsAll、retainAll 和 removeAll）没有必要自动执行，除非在实现中特别说明。因此，举例来说，在只添加了 c 中的一些元素后，addAll(c) 有可能失败（抛出一个异常）。

- BlockingQuue继承自Queue接口，就像上面提到的那样，Quue继承自Java使用最广泛的一个接口Collection接口，因此同样具备常见集合框架都具备的一些方法，可以像使用多数集合类那样去使用它。BlockingQueue的继承体系图如下：![BlockingQueue继承体系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/BlockingQueue继承体系图.jpg)

- BlockingQueue的方法部分继承自Queue，因此Queue中应对不同操作固有的两种处理依然存在，再加上阻塞和超时一共以四种处理形式出现。对于不能立即满足但可能在将来某一时刻可以满足的操作，这四种形式的处理方式不同：
	- 第一种是抛出一个异常（继承自Queue）
		- 插入：`boolean add(E e);` 将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回 true，如果当前没有可用的空间，则抛出 IllegalStateException。
		- 移除：`boolean remove(Object o);` 从此队列中移除指定元素的单个实例（如果存在），如果队列为空，则抛出IllegalStateException。
		- 检查：`E element();` 获取队列头元素，但是不移除此队列的头。此方法与 peek 唯一的不同在于：此队列为空时将抛出一个异常。
	- 第二种是返回一个特殊值（null 或 false，具体取决于操作）
		- 插入：`boolean offer(E e);` 将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回 true，如果当前没有可用的空间，则返回 false。
		- 移除：`E poll();` 获取并移除此队列的头部，在指定的等待时间前等待可用的元素（如果有必要）。
		- 检查：`E peek();` 获取但不移除此队列的头；如果此队列为空，则返回 null。
	- 第三种是在操作可以成功前，无限期地阻塞当前线程
		- 插入：`void put(E e) throws InterruptedException;` 将指定元素插入此队列中，将等待可用的空间（如果有必要）。
		- 移除：`E take() throws InterruptedException;` 获取并移除此队列的头部，在元素变得可用之前一直等待（如果有必要）。
	- 第四种是在放弃前只在给定的最大时间限制内阻塞。四种形式如下：
		- 插入：`boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;` 将指定元素插入此队列中，在到达指定的等待时间前等待可用的空间（如果有必要）。
		- 移除：`E poll(long timeout, TimeUnit unit) throws InterruptedException;` 获取并移除此队列的头部，在指定的等待时间前等待可用的元素（如果有必要）。

- BlockingQueue的源码如下：
```java
public interface BlockingQueue<E> extends Queue<E> {
        boolean add(E e);
        boolean offer(E e);
        void put(E e) throws InterruptedException;
        boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
        E take() throws InterruptedException;
        E poll(long timeout, TimeUnit unit) throws InterruptedException;
        int remainingCapacity();
        boolean remove(Object o);
        public boolean contains(Object o);
        int drainTo(Collection<? super E> c);
        int drainTo(Collection<? super E> c, int maxElements);
}
```

- BlockingQueue的方法
	- `boolean add(E e);`
	  将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回 true，如果当前没有可用的空间，则抛出 IllegalStateException。当使用有容量限制的队列时，通常首选 offer。
	- `boolean offer(E e);`
	  将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回 true，如果当前没有可用的空间，则返回 false。当使用有容量限制的队列时，此方法通常要优于 add(E)，后者可能无法插入元素，而只是抛出一个异常。
	- `void put(E e) throws InterruptedException;`
	  将指定元素插入此队列中，如果没有可用空间就会一直阻塞直至有可用空间。
	- `boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;`
	  将指定元素插入此队列中，在指定时间前未能获得空间，就会返回false。
	- `E take() throws InterruptedException;`
	  获取并移除此队列的头部，在元素变得可用之前一直等待（如果有必要）。
	- `E poll(long timeout, TimeUnit unit) throws InterruptedException;`
	  获取并移除此队列的头部，如果在指定时间内依然无法获取队列的头部，则返回null。
	- `int remainingCapacity();`
	  返回在无阻塞的理想情况下（不存在内存或资源约束）此队列能接受的附加元素数量；如果没有内部限制，则返回 Integer.MAX_VALUE。注意，不能 总是通过检查remainingCapacity 来判断尝试插入元素是否可以成功，因为此时可能存在其他线程在插入或移除元素。
	- `boolean remove(Object o);`
	  从此队列中移除指定元素的单个实例（如果存在）。更确切地讲，如果此队列包含一个或多个满足 o.equals(e) 的元素 e，则移除该元素。如果此队列包含指定元素或者队列由于调用而发生更改，则返回 true。如果指定元素的类与该队列不兼容则抛出ClassCastException，如果指定的元素o为null，则抛出NullPointerException。
	- `public boolean contains(Object o);`
	  如果此队列包含指定元素，则返回 true。更确切地讲，当且仅当此队列至少包含一个满足 o.equals(e) 的元素 e时，返回 true。
	- `int drainTo(Collection<? super E> c);`
	  移除此队列中所有可用的元素，并将它们添加到给定 collection 中。此操作可能比反复轮询此队列更有效。在试图向 collection c 中添加元素没有成功时，可能导致在抛出相关异常时，元素会同时在两个 collection 中出现，或者在其中一个 collection 中出现，也可能在两个 collection 中都不出现。如果试图将一个队列放入自身队列中，则会导致 IllegalArgumentException 异常。此外，如果正在进行此操作时修改指定的 collection，则此操作行为是不确定的。
	- `int drainTo(Collection<? super E> c, int maxElements);`
	  最多从此队列中移除给定数量的可用元素，并将这些元素添加到给定 collection 中。在试图向 collection c 中添加元素没有成功时，可能导致在抛出相关异常时，元素会同时在两个 collection 中出现，或者在其中一个 collection 中出现，也可能在两个 collection 中都不出现。如果试图将一个队列放入自身队列中，则会导致 IllegalArgumentException 异常。此外，如果正在进行此操作时修改指定的 collection，则此操作行为是不确定的。maxElements是允许传输的最大元素个数。例如，blockingQueue中有3个元素["1","2","3"]，调用blockingQueue.drainTo(list,1)后，blockingQueue中元素变为["2","3"];，list中变为["1"]。

- BlockingQueue实现的生产者消费者模型
```java
// 生产者类
class Producer implements Runnable {
    private final BlockingQueue queue;
    Producer(BlockingQueue q) {
        queue = q;
    }
    public void run() {
        try {
            while (true) {
            	// 生产对象到BlockingQueue
                queue.put(produce());
            }
        } catch (InterruptedException ex) {
            System.out.println("Producer broken");
        }
    }
    public Object produce() {
        System.out.println(Thread.currentThread().getName() + " produce a object, remain" + queue.size());
        return new Object();
    }
}
// 消费者类
class Consumer implements Runnable {
    private final BlockingQueue queue;
    Consumer(BlockingQueue q) {
        queue = q;
    }
    public void run() {
        try {
            while (true) {
            	// 消费一个对象
                consume(queue.take());
            }
        } catch (InterruptedException ex) {
            System.out.println("Consumer broken");
        }
    }
    void consume(Object x) {
        System.out.println(Thread.currentThread().getName() + " comsume a object, remain" + queue.size());
        //do nothing
    }
}
public class App {
    public static void main(String[] args) {
        BlockingQueue queue = new ArrayBlockingQueue(10);
        for (int i = 0 ;i < 10; i ++) {
            Thread comsumer = new Thread(new Consumer(queue));
            Thread producer = new Thread(new Producer(queue));
            comsumer.setName("comsumer - " + i);
            producer.setName("producer - " + i);
            comsumer.start();
            producer.start();
        }
    }
}
```


### 阻塞队列
- 狭义的阻塞队列就是指BlockingQueue这个接口，而广义的阻塞队列是指BlockingQueue这个接口及其子接口与实现类。

- 阻塞队列的常见接口和类的继承体系图如下
![阻塞队列继承体系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/阻塞队列继承体系图.jpg)

- BlockingQueue在java.util.concurrent包下面有以下接口或实现类
  接口：
		- BlockingDeque
		- TransferQueue
  实现类：
		- ArrayBlockingQueue
		- DelayQueue
		- LinkedBlockingQueue
		- PriorityBlockingQueue
		- SynchronousQueue
		- ScheduledThreadPoolExecutor.DelayedWorkQueue

- ArrayBlockingQueue是一个有界队列，队列按照FIFO的原则对元素进行排序，初始化时必须指定容量，队列用循环数组实现。队列的头部 是在队列中存在时间最长的元素。队列的尾部 是在队列中存在时间最短的元素。新元素插入到队列的尾部，队列获取操作则是从队列头部开始获得元素。ArrayBlockingQueue是一个典型的“有界缓存区”，固定大小的数组在其中保持生产者插入的元素和使用者提取的元素。一旦创建了这样的缓存区，就不能再增加其容量。试图向已满队列中放入元素会导致操作受阻塞；试图从空队列中提取元素将导致类似阻塞。

- LinkedBlockingQueue是一个基于已链接节点的、无限大(Integer.MAX_VALUE)的阻塞队列。此队列按 FIFO（先进先出）排序元素。队列的头部是在队列中时间最长的元素。队列的尾部是在队列中时间最短的元素。新元素插入到队列的尾部，并且队列获取操作会获得位于队列头部的元素。链接队列的吞吐量通常要高于基于数组的队列，但是在大多数并发应用程序中，其可预知的性能要低。可选的容量范围构造方法参数作为防止队列过度扩展的一种方法。如果未指定容量，则它等于 Integer.MAX_VALUE。除非插入节点会使队列超出容量，否则每次插入后会动态地创建链接节点。

- PriorityBlockingQueue是一个无界阻塞队列，它使用与类 PriorityQueue 相同的顺序规则，并且提供了阻塞获取操作。虽然此队列逻辑上是无界的，但是资源被耗尽时试图执行 add 操作也将失败（导致OutOfMemoryError）。此类不允许使用 null 元素。依赖自然顺序的优先级队列也不允许插入不可比较的对象（这样做会导致抛出 ClassCastException）。此类及其迭代器可以实现 Collection 和 Iterator 接口的所有可选方法。iterator() 方法中提供的迭代器并不 保证以特定的顺序遍历 PriorityBlockingQueue 的元素。如果需要有序地进行遍历，则应考虑使用 Arrays.sort(pq.toArray())。此外，可以使用方法 drainTo 按优先级顺序移除 全部或部分元素，并将它们放在另一个 collection 中。在此类上进行的操作不保证具有同等优先级的元素的顺序。

- DelayQueue是一个无界阻塞队列，只有在延迟期满才能从队列中提取元素。队列的头部是延迟期满后，保存在队列中时间最长的一个元素。如果队列中所有元素都没有期满，则队列没有头部，并且poll也会返回null。当一个元素的 getDelay(TimeUnit.NANOSECONDS)方法返回一个小于等于0的值时，该元素即将期满。可以将这个队列比作一个监狱，元素扔进去以后总要刑满释放过后，才算自由人，而对于BlockingQueue的移除或获取操作，只对这些刑满的元素有效。