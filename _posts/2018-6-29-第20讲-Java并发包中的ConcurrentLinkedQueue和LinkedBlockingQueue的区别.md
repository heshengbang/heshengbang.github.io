---
layout: post
title: Java并发包中的ConcurrentLinkedQueue和LinkedBlockingQueue的区别
date: 2018-6-29
tags: Java核心技术36讲笔记
---

### Java并发包中的ConcurrentLinkedQueue和LinkedBlockingQueue的区别
- 通常情况下，并发包下的所有容器都习惯性的被称为并发容器，但是严格来讲，类似ConcurrentLinkedQueue这种“Concurrent”容器才是真正代表并发。
- Concurrent类型基于lock-free，在常见的多线程访问场景，一般可以提供较高的吞吐量
- LinkedBlockingQueue内部则是基于锁，并提供了BlockingQueue的等待性方法
- java.util.concurrent包提供的容器（Queue/List/Set）、Map，从命名上可以大概区分为Concurrent、CopyOnWrite和BlockingXxx 等三类，同样是线程安全容器，可以简单的认为：
	- Concurrent类型没有类似CopyOnWrite之类的容器相对较重的修改开销，但是Concurrent也因此无法提供较高的遍历一致性（弱一致性）。也就是说，当迭代器进行遍历的时候，如果Concurrent类型被修改了，迭代器仍然可以继续进行遍历。
	- 与弱一致性对应的，同步容器常见的行为是“fast-fail”，也就是检测到容器在遍历过程中发生了修改，则抛出ConcurrentModificationException，而不再继续遍历。
	- 弱一致性的另外一个体现是，size等操作准确性是有限的，未必100%准确。同时，读取的性能具有一定的不确定性。

### 关键点
- 了解并发包中不同容器的设计目的和实现区别
- 队列和Executor框架提供的各种线程池
	- 哪些队列是有界的，哪些是无界的（从底层实现去解释）
	- 根据特定场景需求，适当选择合适的队列去实现
	- 从源码的角度去解释常见的线程安全队列是如何实现的，并进行了哪些改进从而提高性能表现

### 线程安全队列
- 常见的集合中如LinkedList是Deque，只不过不是线程安全的。下图大致罗列了java并发类库提供的各种各样的线程安全队列实现：
![java线程安全的队列实现](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/java线程安全的队列实现.png)

- 从基本的数据结构的角度分析，有两个特别的Deque实现，ConcurrentLinkedDeque 和 LinkedBlockingDeque。Deque的侧重点是支持对队列头尾都进行插入和删除，所以提供了特定的方法如：
	- 尾部插入时需要的addLast(e)、offerLast(e)。虽然在插入失败的时候表现不同，但是它们都是试图将元素插入到队尾。
	- 尾部删除所需要的removeLast()、pollLast。
  从上面这些角度，能够理解 ConcurrentLinkedDeque和 LinkedBlockingQueue 的主要功能区别，也就足够日常开发的需要了。

- 从行为的特征来看，绝大部分Queue都实现了BlockingQueue接口。在常规队列操作基础上，Blocking意味着其提供了特定的等待性操作，获取时（take）等待元素进队，或者插入时（put）等待队列出现空位。
- 另外，从有界和无界的角度进行区分：
	- ArrayBlockingQueue 是最典型的的有界队列，其内部以 final 的数组保存数据，数组的大小就决定了队列的边界，所以我们在创建ArrayBlockingQueue 时，都要指定容量，如：
	```
    	public ArrayBlockingQueue(int capacity, boolean fair)
    ```
    - LinkedBlockingQueue，容易被误解为无边界，但其实其行为和内部代码都是基于有界的逻辑实现的，只不过如果我们没有在创建队列时就指定容量，那么其容量限制就自动被设置为 Integer.MAX_VALUE，成为了无界队列。
    - SynchronousQueue，这是一个非常奇葩的队列实现，每个删除操作都要等待插入操作，反之每个插入操作也都要等待删除动作。那么这个队列的容量是多少呢？是 1 吗？其实不是的，其内部容量是 0。
    - PriorityBlockingQueue 是无边界的优先队列，虽然严格意义上来讲，其大小总归是要受系统资源影响。
    - DelayedQueue 和 LinkedTransferQueue 同样是无边界的队列。对于无边界的队列，有一个自然的结果，就是 put 操作永远也不会发生其他BlockingQueue的那种等待情况。

- 分析不同队列的底层实现，BlockingQueue基本都是基于锁实现。常见的ArrayBlockingQueue中有一个锁，而LinkedBlockingQueue有两个锁。
- 虽然ArrayBlockingQueue和LinkedBlockingQueue都是基于锁实现，但是它们基于不同的条件变量来完成各自的操作。ArrayBlockingQueue的notEmpty和notFull都是同一个再入锁的条件变量，而LinkedBlockingQueue则改进了锁的操作粒度，头/尾操作使用不同的锁，所以在通用场景下，它的吞吐量相对要更好一些。下面分别展示ArrayBlockingQueue和LinkedBlockingQueue的take()方法的源代码：
```
	// ArrayBlockingQueue
    public E take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                while (count == 0)
                    notEmpty.await();
                return dequeue();
            } finally {
                lock.unlock();
            }
        }
    // LinkedBlockingQueue，注意该方法中的锁对象
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

- 类似ConcurrentLinkedQueue等则是基于CAS的无锁技术，不需要在每个操作时使用锁，所以扩展性表现要更加优异。相较于另类的SynchronousQueue，在JDK6中，其实发生了非常大的变化，利用CAS替换了原本基于锁的逻辑，同步开销比较小。它是Executors.newCachedThreadPool()的默认队列


### 队列使用场景与典型用例
- 实际开发中，队列被普遍用于生产者消费者场景。比如，利用BlockingQueue来实现，由于其提供的等待机制，使用者无需关心线程间的协调工作。参考如下代码：
```
    public class ConsumerProducer {
        private static final String EXIT_MSG = "Good bye!";

        public static void main(String[] args) {
            // 使用较小的队列，以更好地在输出中展示其影响
            BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
            Producer producer = new Producer(queue);
            Consumer consumer = new Consumer(queue);
            new Thread(producer).start();
            new Thread(consumer).start();
        }

        static class Producer implements Runnable {
            private BlockingQueue<String> queue;

            Producer(BlockingQueue<String> q) {
                this.queue = q;
            }

            @Override
            public void run() {
                for (int i = 0; i < 20; i++) {
                    try {
                        Thread.sleep(5L);
                        String msg = "Message" + i;
                        System.out.println("Produced new item: " + msg);
                        queue.put(msg);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                try {
                    System.out.println("Time to say good bye!");
                    queue.put(EXIT_MSG);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        static class Consumer implements Runnable {
            private BlockingQueue<String> queue;

            Consumer(BlockingQueue<String> q) {
                this.queue = q;
            }

            @Override
            public void run() {
                try {
                    String msg;
                    while (!EXIT_MSG.equalsIgnoreCase((msg = queue.take()))) {
                        System.out.println("Consumed item: " + msg);
                        Thread.sleep(10L);
                    }
                    System.out.println("Got exit message, bye!");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```
  如果自己尝试过用synchronized或者Lock去实现生产者消费者模型，就应该明白，上面的代码减少了多少为了保持同步而必须的模板化代码。

- 以LinkedBlockingQueue、ArrayBlockingQueue和SynchronousQueue为例，分析开发中如何选择：
	- 考虑应用场景中是否对队列边界有要求。ArrayBlockingQueue是有明确的容量限制的，而LinkedBlockingQueue则取决于我们是否在创建时指定，SynchronousQueue则干脆不能缓存任何元素
	- 从空间利用角度，数组结构的ArrayBlockingQueue要比LinkedBlockingQueue紧凑，因为其不需要创建所谓的节点，但是其初始分配的是一段连续的空间，所以初始内存需求更大。
	- 通用场景中，LinkedBlockingQueue的吞吐量一般优于ArrayBlockingQueue，因为它实现了更加细粒度的锁操作
	- ArrayBlockingQueue的实现更加简单，性能也更好预测，表现稳定，在稳定性要求很高的系统中是首选
	- 如果我们需要实现的是两个线程之间接力性（handoff）的场景，可以用CountDownLatch但是SynchronousQueue也是更加完美符合这种场景的，而线程间协调和数据传输统一起来，代码也更加规范
	- 很多时候，SynchronousQueue的性能表现，往往大大超过其他实现，尤其是在队列元素较小的场景