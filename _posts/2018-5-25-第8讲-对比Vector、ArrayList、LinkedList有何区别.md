---
layout: post
title: 对比Vector、ArrayList、LinkedList有何区别
date: 2018-5-25
tags: Java核心技术36讲笔记
---

## 对比Vector、ArrayList、LinkedList有何区别

### 对比Vector、ArrayList、LinkedList有何区别
- 这三者都是实现集合框架中的 List，也就是所谓的有序集合，因此具体功能也比较近似，比如都提供按照位置进行定位、添加或者删除的操作，都提供迭代器以遍历其内容等。但因为具体的设计区别，在行为、性能、线程安全等方面，表现又有很大不同;
- Verctor 是 Java 早期提供的线程安全的动态数组，如果不需要线程安全，并不建议选择，毕竟同步是有额外开销的。Vector 内部是使用对象数组来保存数据，可以根据需要自动的增加容量，当数组已满时，会创建新的数组，并拷贝原有数组数据;
- ArrayList 是应用更加广泛的动态数组实现，它本身不是线程安全的，所以性能要好很多。与 Vector 近似，ArrayList 也是可以根据需要调整容量，不过两者的调整逻辑有所区别，Vector 在扩容时会提高 1 倍，而 ArrayList 则是增加 50%;
- LinkedList 顾名思义是 Java 提供的双向链表，所以它不需要像上面两种那样调整容量，它也不是线程安全的;
- Vector 和 ArrayList 作为动态数组，其内部元素以数组形式顺序存储的，所以非常适合随机访问的场合。除了尾部插入和删除元素，往往性能会相对较差，比如要在中间位置插入一个元素，需要移动后续所有元素;
- LinkedList 进行节点插入、删除却要高效得多，但是随机访问性能则要比动态数组慢;
```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{...}
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{...}
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{...}
```
![list1继承谱系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/list1继承谱系图.png)

### 关键点
- Java 集合框架的设计结构
- Java 提供的主要容器（集合和 Map）类型，了解或掌握对应的数据结构、算法，思考具体技术选择
- 将问题扩展到性能、并发等领域
- 集合框架的演进与发展
- 内部排序，至少掌握基础算法如归并排序、交换排序（冒泡、快排）、选择排序、插入排序等
- 外部排序，掌握利用内存和外部存储处理超大数据集
- 排序的稳定性，为什么稳定，稳定性意味着什么
- 对不同数据集，各种排序的最好或最差情况
- 从某个角度如何进一步优化（比如空间占用，假设业务场景需要最小辅助空间，这个角度堆排序就比归并优异）等

### Java 的集合框架
- Collection 接口是所有集合的根，集合的基础谱系图可见上图。由Collection扩展开提供了三大类集合，分别是：
	- List，最常用的有序集合，它提供了方便的访问、插入、删除等操作
		- 实现List接口，并继承AbstractCollection，则是AbstractList
			- ArrayList就继承自AbstractList
	- Set，Set 是不允许重复元素的，这是和 List 最明显的区别，也就是不存在两个对象 equals 返回 true。日常开发中有很多需要保证元素唯一性的场合
		- 实现Set接口，并继承Abstractcollection，则是AbstractSet
			- HashSet就继承自AbstractSet
	- Queue/Deque，则是 Java 提供的标准队列结构的实现，除了集合的基本功能，它还支持类似先入先出（FIFO， First-in-First-Out）或者后入先出（LIFO，Last-In-First-Out）等特定行为。这里不包括 BlockingQueue，因为通常是并发编程场合，所以被放置在并发包里
		- 实现Queue接口，并继承Abstractcollection，则是AbstractQueue
			- Priority就继承自AbstractQueue

- 各种具体集合实现，至少了解基本特征和典型使用场景，以 Set 的几个实现为例：
	- TreeSet 支持自然顺序访问，但是添加、删除、包含等操作要相对低效（log(n) 时间）
	- TreeSet 支持自然顺序访问，但是添加、删除、包含等操作要相对低效（log(n) 时间）
	- LinkedHashSet，内部构建了一个记录插入顺序的双向链表，因此提供了按照插入顺序遍历的能力，与此同时，也保证了常数时间的添加、删除、包含等操作，这些操作性能略低于 HashSet，因为需要维护链表的开销
	- 在遍历元素时，HashSet 性能受自身容量影响，所以初始化时，除非有必要，不然不要将其背后的 HashMap 容量设置过大。而对于 LinkedHashSet，由于其内部链表提供的方便，遍历性能只和元素多少有关系

- 上面提到的集合框架都不是线程安全的，如果想要提供线程安全的支持，可以使用Collections 工具类中，提供了一系列的 synchronized 方法，比如：
```java
static <T> List<T> synchronizedList(List<T> list)
//类似方法来实现基本的线程安全集合
List list = Collections.synchronizedList(new ArrayList());
```
其实现原理，基本就是将每个基本方法，比如 get、set、add 之类，都通过 synchronizd 添加基本的同步支持，非常简单粗暴，但也非常实用。注意这些方法创建的线程安全集合，都符合迭代时 fail-fast 行为，当发生意外的并发修改时，尽早抛出 ConcurrentModificationException 异常，以避免不可预计的行为

### Java 提供的默认排序算法
- 需要区分是 `Arrays.sort()` 还是 `Collections.sort()` （底层是调用 Arrays.sort()）；什么数据类型；多大的数据集（太小的数据集，复杂排序是没必要的，Java 会直接进行二分插入排序）等。
	- 对于原始数据类型，目前使用的是所谓双轴快速排序（Dual-Pivot QuickSort），是一种改进的快速排序算法，早期版本是相对传统的快速排序，可以阅读[源码](http://hg.openjdk.java.net/jdk/jdk/file/26ac622a4cab/src/java.base/share/classes/java/util/DualPivotQuicksort.java)。
	- 对于对象数据类型，目前则是使用[TimSort](http://hg.openjdk.java.net/jdk/jdk/file/26ac622a4cab/src/java.base/share/classes/java/util/TimSort.java)，思想上也是一种归并和二分插入排序（binarySort）结合的优化排序算法。TimSort 并不是 Java 的独创，简单说它的思路是查找数据集中已经排好序的分区（这里叫 run），然后合并这些分区来达到排序的目的
	- Java 8 引入了并行排序算法（直接使用 parallelSort 方法），这是为了充分利用现代多核处理器的计算能力，底层实现基于 fork-join 框架，当处理的数据集比较小的时候，差距不明显，甚至还表现差一点；但是，当数据集增长到数万或百万以上时，提高就非常大了，具体还是取决于处理器和系统环境

- Java 8 之中，Java 平台支持了 Lambda 和 Stream，相应的 Java 集合框架也进行了大范围的增强，以支持类似为集合创建相应 stream 或者 parallelStream 的方法实现，可以非常方便的实现函数式代码
-  阅读 Java 源代码会发现，这些 API 的设计和实现比较独特，它们并不是实现在抽象类里面，而是以默认方法的形式实现在 Collection 这样的接口里！这是 Java 8 在语言层面的新特性，允许接口实现默认方法，理论上来说，原来实现在类似 Collections 这种工具类中的方法，大多可以转换到相应的接口上。
- 在 Java 9 中，Java 标准类库提供了一系列的静态工厂方法，比如，List.of()、Set.of()，大大简化了构建小的容器实例的代码量。根据业界实践经验发现相当一部分集合实例都是容量非常有限的，而且在生命周期中并不会进行修改。但是，在原有的 Java 类库中，我们可能不得不写成：
```java
ArrayList<String>  list = new ArrayList<>();
list.add("Hello");
list.add("World");
```
而利用新的容器静态工厂方法，一句代码就够了，并且保证了不可变性
`List<String> simpleList = List.of("Hello","world");`
更进一步，通过各种 of 静态工厂方法创建的实例，还应用了一些所谓的最佳实践，比如，它是不可变的，符合线程安全的需求；它因为不需要考虑扩容，所以空间上更加紧凑等。

- 如果查看 of 方法的源码，会发现一个特别有意思的地方： Java 已经支持所谓的可变参数（varargs），但是官方类库还是提供了一系列特定参数长度的方法，看起来似乎非常不优雅，为什么呢？这其实是为了最优的性能，JVM 在处理变长参数的时候会有明显的额外开销，如果需要实现性能敏感的 API，也可以进行参考。

### 实现一个云计算任务调度系统，希望可以保证 VIP 客户的任务被优先处理，有哪些可以利用的数据结构或者标准的集合类型呢？更进一步讲，类似场景大多是基于什么数据结构呢？