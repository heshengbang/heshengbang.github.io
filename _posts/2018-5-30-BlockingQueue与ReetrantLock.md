---

layout: post

title: BlockingQueue与ReetrantLock

date: 2018-5-30

tags: Java基础

---

### 阻塞队列
- 阻塞队列与我们平常接触的普通队列(LinkedList或ArrayList等)的最大不同点，在于阻塞队列支持阻塞添加和阻塞删除方法。
	- 阻塞添加：所谓的阻塞添加是指当阻塞队列元素已满时，队列会阻塞加入元素的线程，直到队列元素不满时才重新唤醒线程执行元素加入操作
	- 阻塞删除：阻塞删除是指在队列元素为空时，删除队列元素的线程将被阻塞，直到队列不为空再执行删除操作(一般都会返回被删除的元素)

- Java中的阻塞队列都实现了BlockingQueue接口，继承关系是：
Iterable <- Collection <- Queue <- BlockingQueue

- 该接口源码如下：
```java
public interface BlockingQueue<E> extends Queue<E> {
	//将指定的元素插入到此队列的尾部（如果立即可行且不会超过该队列的容量）
	//在成功时返回 true，如果此队列已满，则抛IllegalStateException。
    boolean add(E e);
    boolean offer(E e);
	//将指定的元素插入此队列的尾部，如果该队列已满，则一直等到（阻塞）
    void put(E e) throws InterruptedException;
	//将指定的元素插入到此队列的尾部（如果立即可行且不会超过该队列的容量）
	// 将指定的元素插入此队列的尾部，如果该队列已满，
	//则在到达指定的等待时间之前等待可用的空间,该方法可中断
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
	//获取并移除此队列的头部，如果没有元素则等待（阻塞），
	//直到有元素将唤醒等待线程执行该操作
    E take() throws InterruptedException;
	//获取并移除此队列的头部，在指定的等待时间前一直等待获取元素
	//超过时间方法将结束
    E poll(long timeout, TimeUnit unit) throws InterruptedException;
    int remainingCapacity();
	//从此队列中移除指定元素的单个实例（如果存在）
    boolean remove(Object o);
    public boolean contains(Object o);
    int drainTo(Collection<? super E> c);
    int drainTo(Collection<? super E> c, int maxElements);
}
```
  将源码中的方法分为三类：
	- 插入方法
		- `add(E e)`：添加成功返回true，失败抛IllegalStateException异常
		- `offer(E e)`：成功返回 true，如果此队列已满，则返回 false
		- `put(E e)`：将元素插入此队列的尾部，如果该队列已满，则一直阻塞
	- 删除方法
		- `remove(Object o)`：移除指定元素,成功返回true，失败返回false
		- `poll()`：获取并移除此队列的头元素，若队列为空，则返回 null
		- `take()`：获取并移除此队列头元素，若没有元素则一直阻塞
	- 检查方法
		- `element()`：获取但不移除此队列的头元素，没有元素则抛异常
		- `peek()`：获取但不移除此队列的头；若队列为空，则返回 null

  阻塞队列的对元素的增删查操作主要就是上述的三类方法，通常情况下我们都是通过这3类方法操作阻塞队列

### LinkedBlockingQueue的基本概要
- LinkedBlockingQueue是一个由链表实现的有界队列阻塞队列，但大小默认值为Integer.MAX_VALUE，所以我们在使用LinkedBlockingQueue时建议手动传值，为其提供我们所需的大小，避免队列过大造成机器负载或者内存爆满等情况。
- LinkedBlockingQueue的三种构造函数：
	- `public LinkedBlockingQueue()`
	- `public LinkedBlockingQueue(int capacity)`
	- `public LinkedBlockingQueue(Collection<? extends E> c)`

  源码如下：
  ```java
	public LinkedBlockingQueue() {
		this(Integer.MAX_VALUE);
	}
	public LinkedBlockingQueue(int capacity) {
		if (capacity <= 0) throw new IllegalArgumentException();
		this.capacity = capacity;
		last = head = new Node<E>(null);
	}
	public LinkedBlockingQueue(Collection<? extends E> c) {
		this(Integer.MAX_VALUE);
		final ReentrantLock putLock = this.putLock;
		putLock.lock(); // Never contended, but necessary for visibility
		try {
			int n = 0;
			for (E e : c) {
				if (e == null)
					throw new NullPointerException();
				if (n == capacity)
					throw new IllegalStateException("Queue full");
				enqueue(new Node<E>(e));
				++n;
			}
			count.set(n);
		} finally {
			putLock.unlock();
		}
	}
  ```
	- 有三种方式可以构造LinkedBlockingQueue，通常情况下，我们建议创建指定大小的LinkedBlockingQueue阻塞队列
	- LinkedBlockingQueue队列也是按 FIFO（先进先出）排序元素
	- 队列的头部是在队列中时间最长的元素，队列的尾部 是在队列中时间最短的元素，新元素插入到队列的尾部，而队列执行获取操作会获得位于队列头部的元素
	- 在正常情况下，链接队列的吞吐量要高于基于数组的队列（ArrayBlockingQueue）
		- 因为LinkedBlockingQueue内部实现添加和删除操作使用的两个ReenterLock来控制并发执行，而ArrayBlockingQueue内部只是使用一个ReenterLock控制并发，因此LinkedBlockingQueue的吞吐量要高于ArrayBlockingQueue

### LinkedBlockingQueue的实现原理
- 原理概论
	- LinkedBlockingQueue是一个基于链表的阻塞队列，其内部维持一个基于链表的数据队列，实际上我们对LinkedBlockingQueue的API操作都是间接操作该数据队列，这里我们先看看LinkedBlockingQueue的内部成员变量
	```java
        public class LinkedBlockingQueue<E> extends AbstractQueue<E>
            implements BlockingQueue<E>, java.io.Serializable {
            // 节点类，用于存储数据
            static class Node<E> {
                E item;
                /**
                 * One of:
                 * - the real successor Node
                 * - this Node, meaning the successor is head.next
                 * - null, meaning there is no successor (this is the last node)
                 */
                Node<E> next;
                Node(E x) { item = x; }
            }
            // 阻塞队列的大小，默认为Integer.MAX_VALUE
            private final int capacity;
            // 当前阻塞队列中的元素个数
            private final AtomicInteger count = new AtomicInteger();
            // 阻塞队列的头结点
            transient Node<E> head;
            // 阻塞队列的尾节点
            private transient Node<E> last;
            // 获取并移除元素时使用的锁，如take, poll, etc
            private final ReentrantLock takeLock = new ReentrantLock();
            // notEmpty条件对象，当队列没有数据时用于挂起执行删除的线程
            private final Condition notEmpty = takeLock.newCondition();
            // 添加元素时使用的锁如 put, offer, etc
            private final ReentrantLock putLock = new ReentrantLock();
            // notFull条件对象，当队列数据已满时用于挂起执行添加的线程
            private final Condition notFull = putLock.newCondition();
        }
    ```
    - 从源码可以看出每个添加到LinkedBlockingQueue队列中的数据都将被封装成Node节点，添加的链表队列中，其中head和last分别指向队列的头结点和尾结点。
	- LinkedBlockingQueue内部分别使用了takeLock 和 putLock 对并发进行控制，也就是说，添加和删除操作并不是互斥操作，可以同时进行，这样也就可以大大提高吞吐量
	- 这里再次强调如果没有给LinkedBlockingQueue指定容量大小，其默认值将是Integer.MAX_VALUE，如果存在添加速度大于删除速度时候，有可能会内存溢出，这点在使用前希望慎重考虑
	- 至于LinkedBlockingQueue的实现原理图如下：
	![LinkedBlockingQueue原理图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/LinkedBlockingQueue原理图.jpg)

- 添加方法的实现原理
	- 添加方法
		- `add()`方法由AbstractQueue抽象类代为默认实现，源码如下：
            ```java
            public boolean add(E e) {
                if (offer(e))
                    return true;
                else
                    throw new IllegalStateException("Queue full");
            }
        ```
        该方法实际上的实现是调用`offer()`方法来实现的

        - `offer()`的源码如下：
        ```java
            public boolean offer(E e) {
                //添加的元素如果为null 则直接抛出错误
                if (e == null) throw new NullPointerException();
                //获取队列中当前元素个数
                final AtomicInteger count = this.count;
                //判断队列中的个数是否已满
                if (count.get() == capacity)
                    //如果队列已满就添加失败
                    return false;
                int c = -1;
                //创建新节点
                Node<E> node = new Node<E>(e);
                //获取添加锁
                final ReentrantLock putLock = this.putLock;
                //加锁
                putLock.lock();
                try {
                    //再次判断队列是否已满，考虑并发情况
                    if (count.get() < capacity) {
                        //添加元素
                        enqueue(node);
                        //拿到当前未添加新元素时的队列长度
                        c = count.getAndIncrement();
                        if (c + 1 < capacity)
                            //唤醒下一个添加线程，执行添加操作
                            notFull.signal();
                    }
                } finally {
                    putLock.unlock();
                }
                // 由于存在添加锁和消费锁，而消费锁和添加锁都会持续唤醒等待线程，因此count肯定会变化
                //这里的if条件表示如果队列中还有1条数据
                if (c == 0)
                    //如果还存在数据那么就唤醒消费锁
                    signalNotEmpty();
                return c >= 0;
            }
            //入队操作
            private void enqueue(Node<E> node) {
                 //队列尾节点指向新的node节点
                 last = last.next = node;
            }
            //signalNotEmpty方法
            private void signalNotEmpty() {
                final ReentrantLock takeLock = this.takeLock;
                //消费锁加锁
                takeLock.lock();
                try {
                    //唤醒在不为空条件上等待的消费线程
                    notEmpty.signal();
                } finally {
                    takeLock.unlock();
                }
            }
		```
        Offer()源码中首先判断了队列是否已经满了，如果满了直接返回。此时，如果没满就获取锁，在获取锁并加锁以后，再次判断队列是否已满：
        	- 如果满了就直接添加锁解锁，并判断此时队列容量大小，如果等于0就唤醒在不为空条件上面等待的线程
        	- 如果此时仍然没满，就直接入队列，然后判断此时队列的个数是否已满：
        		- 如果满了，就解锁，并判断此时队列容量大小，如果等于0就唤醒在不为空条件上面等待的线程
        		- 如果没满，就唤醒在未满条件上等待的线程，而后解锁，并判断此时队列容量大小，如果等于0就唤醒在不为空条件上面等待的线程

		- `put()`源码如下：
		```java
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
        ```
	- 移除方法
		- `remove()`同样是在AbstractQueue中默认实现，也和`add()`类似，是调用poll()实现，源码如下：
		```java
            public boolean remove(Object o) {
                if (o == null) return false;
                fullyLock();
                try {
                    for (Node<E> trail = head, p = trail.next;
                         p != null;
                         trail = p, p = p.next) {
                        if (o.equals(p.item)) {
                            unlink(p, trail);
                            return true;
                        }
                    }
                    return false;
                } finally {
                    fullyUnlock();
                }
            }
            void fullyLock() {
                putLock.lock();
                takeLock.lock();
            }
            void unlink(Node<E> p, Node<E> trail) {
                p.item = null;
                trail.next = p.next;
                if (last == p)
                    last = trail;
                if (count.getAndDecrement() == capacity)
                    notFull.signal();
            }
            void fullyUnlock() {
                takeLock.unlock();
                putLock.unlock();
            }
        ```
        remove方法删除指定的对象同时对putLock和takeLock加锁，因为remove方法删除的数据的位置不确定，为了避免造成并发安全问题，所以需要对2个锁同时加锁
        - `poll()`源码如下：
        ```java
            public E poll() {
                final AtomicInteger count = this.count;
                if (count.get() == 0)
                    return null;
                E x = null;
                int c = -1;
                final ReentrantLock takeLock = this.takeLock;
                takeLock.lock();
                try {
                    if (count.get() > 0) {
                        x = dequeue();
                        c = count.getAndDecrement();
                        if (c > 1)
                            notEmpty.signal();
                    }
                } finally {
                    takeLock.unlock();
                }
                if (c == capacity)
                    signalNotFull();
                return x;
            }
            private E dequeue() {
                Node<E> h = head;
                Node<E> first = h.next;
                h.next = h; // help GC
                head = first;
                E x = first.item;
                first.item = null;
                return x;
            }
            private void signalNotFull() {
                final ReentrantLock putLock = this.putLock;
                putLock.lock();
                try {
                    notFull.signal();
                } finally {
                    putLock.unlock();
                }
            }
        ```
        - `take()`源码如下：
        ```java
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

### LinkedBlockingQueue和ArrayBlockingQueue迥异
- 队列大小有所不同：ArrayBlockingQueue是有界的初始化必须指定大小，而LinkedBlockingQueue可以是有界的也可以是无界的(Integer.MAX_VALUE)，对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。
- 数据存储容器不同：ArrayBlockingQueue采用的是数组作为数据存储容器，而LinkedBlockingQueue采用的则是以Node节点作为连接对象的链表。
- 由于ArrayBlockingQueue采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而LinkedBlockingQueue则会生成一个额外的Node对象。这可能在长时间内需要高效并发地处理大批量数据时，对于GC可能存在较大影响。
- 两者的实现队列添加或移除的锁不一样，ArrayBlockingQueue实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个ReenterLock锁，而LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。