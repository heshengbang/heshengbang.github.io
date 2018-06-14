---

layout: post

title: Java的synchronized关键字

date: 2018-5-28

tags: Java基础

---

### 本文结构
- Java 同步关键字（synchronized）
- 实例方法同步
- 静态方法同步
- 实例方法中同步块
- 静态方法中同步块
- synchronized底层语义原理
- synchronized代码块底层原理
- synchronized方法底层原理
- 理解Java对象头与Monitor
- synchronized的可重入性
- 线程中断与等待唤醒

### Java 同步关键字（synchronized）
- 在Java中，synchronized关键字是用来控制线程同步的，就是在多线程的环境下，被标记synchronized的代码块将不会被多个线程同时执行。synchronized既可以加在一段代码上，也可以加在方法上。
- synchronized关键字的作用，简单阐述就是，给某个对象加锁，此时就只有一个线程能够获得这个锁对象，线程执行完毕以后释放锁对象，其他线程才能够获取锁对象进而执行代码。
- synchronized通常可以用于实例方法，静态方法，实例方法中的代码块，静态方法中的代码块，实际情况需要视具体需求而定

### 实例方法同步
- 实例方法同步例子：
```java
public synchronized void add(int value) {
	this.count += value;
}
```

- 实例方法同步是同步在拥有该方法的对象上，意思是同一时刻只有一个线程可以通过这对象实例来访问这个方法

### 静态方法同步
- 静态方法同步例子：
```java
public static synchronized void add(int value) {
	count += value;
}
```

- 静态方法同步是指同步在该方法所在的类对象上。因为在Java虚拟机中一个类只能对应一个类对象，所以同时时刻只有一个线程执行类的这个的静态同步方法。

### 实例方法中同步块
- 实例方法中同步块实例：
```java
public void add(int value) {
	synchronized(this){
    	this.count += value;
	}
}
```

- 当你确实不需要同步整个方法时，可以只对一部分代码加上synchronized关键字。包含的代码量越少，相对来说效率就更高
- 实例方法中同步块，实际上和实例方法同步是一样的，在包含了this之后，也是给当前实例加锁

### 静态方法中同步块
- 静态方法中同步块实例：
```java
public class MyClass {
	public static synchronized void log1(String msg1, String msg2){
		log.writeln(msg1);
        log.writeln(msg2);
	}
	public static void log2(String msg1, String msg2){
		synchronized(MyClass.class){
			log.writeln(msg1);
			log.writeln(msg2);
		}
	}
}
```

- 与静态方法同步类似，这种情况是对类对象加锁，同一时刻只有一个线程能够访问log2方法

### synchronized底层语义原理
- Java虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现，无论是显式同步(有明确的 monitorenter 和 monitorexit 指令,即同步代码块)还是隐式同步(方法同步)都是如此
- 在Java语言中，同步用的最多的地方可能是被synchronized修饰的同步方法，同步方法 并不是由 monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的（下面详述）

### synchronized代码块底层原理
- 同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置
- 当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功
- 如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor，重入时计数器的值也会加1
- 倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor
- 值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束
	- 为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令

### synchronized方法底层原理
- 方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中
- JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法
- 当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置
	- 如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor
- 在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor
- 如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放

- synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用
- 在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因
- 在Java 6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁

### 理解Java对象头与Monitor
- Java对象头，先看对象的组成，如下图所示：
![对象内存组成](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/jvm/对象内存组成.jpg)
  在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充
	- 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐
	- 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可

- Java头对象
	- 它实现synchronized的锁对象的基础
	- 一般而言，synchronized使用的锁对象是存储在Java对象头里的
	- jvm中采用2个字节来存储对象头(如果对象是数组则会分配3个字节，多出来的1个字节记录的是数组长度)
	- 在32位JVM虚拟机中，对象头由mark word，class word，一个32位长度的字节（如果该对象是一个数组），一个32位的填充（如果对齐规则需要的话）以及零个或多个实例组成的字段，数组元素或元数据字段，参见[Object header layout](https://stackoverflow.com/questions/50502663/how-to-calculate-java-arrays-memory-size)
	- 在64位虚拟机中，对象头的组成有所不同，参见

       虚拟机位数 | 头对象结构 | 说明
             - | :-: | -:
       32/64bit | Mark Word | 存储对象的hashCode、锁信息或分代年龄或GC标志等信息
       32/64bit | Class Metadata Address | 类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例

    - 其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

    锁状态 | 25bit | 4bit | 1bit是否是偏向锁 | 2bit锁标志位
    	- | :-: | -:
    无锁状态 | 对象HashCode | 对象分代年龄 | 0 | 01

    - 由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间。32位虚拟机如下图所示：
    ![对象头结构示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/jvm/32_object_header_structure.png)
      64位虚拟机如下图所示:
    ![对象头结构示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/jvm/64_object_header_structure.png)
    - 轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的
    - 重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址
    - 每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态
    - 在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）
    ```
        ObjectMonitor() {
            _header       = NULL;
            _count        = 0; //记录个数
            _waiters      = 0,
            _recursions   = 0;
            _object       = NULL;
            _owner        = NULL;
            _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
            _WaitSetLock  = 0 ;
            _Responsible  = NULL ;
            _succ         = NULL ;
            _cxq          = NULL ;
            FreeNext      = NULL ;
            _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
            _SpinFreq     = 0 ;
            _SpinClock    = 0 ;
            OwnerIsThread = 0 ;
        }
    ```
		- ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)
		- _owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1
		- 若线程调用wait()方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒
		- 若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)
		![ObjectMonitor示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/jvm/ObjectMonitor示意图.jpg)
	- 正是因为ObjectMonitor也即是monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因

### synchronized的可重入性
- 当一个线程试图操作一个由其他线程持有的对象锁的临界资源时，将会处于阻塞状态，但当一个线程再次请求自己持有对象锁的临界资源时，这种情况属于重入锁，请求将会成功
- 在java中synchronized是基于原子性的内部锁机制，是可重入的，因此在一个线程调用synchronized方法的同时在其方法体内部调用该对象另一个synchronized方法，也就是说一个线程得到一个对象锁后再次请求该对象锁，是允许的，这就是synchronized的可重入性
- 当子类继承父类时，子类也是可以通过可重入锁调用父类的同步方法。注意由于synchronized是基于monitor实现的，因此每次重入，monitor中的计数器仍会加1

### 线程中断与等待唤醒
- 线程中断
    - 当线程处于阻塞状态或者试图执行一个阻塞操作时，我们可以使用实例方法interrupt()进行线程中断，行中断操作后将会抛出interruptException异常(该异常必须捕捉无法向外抛出)并将中断状态复位
    - 当线程处于运行状态时，我们也可调用实例方法interrupt()进行线程中断，但同时必须手动判断中断状态，并编写中断线程的代码(其实就是结束run方法体的代码)
    - 线程的中断操作对正在等待获取的锁对象的synchronized方法或者代码块并不起作用。对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它就保存等待，即使调用中断线程的方法，也不会生效

- 等待唤醒
	- 所谓等待唤醒机制主要指的是notify/notifyAll和wait方法
	- 在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常
		- 因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象
		- 之前提到monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor
		- 这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因