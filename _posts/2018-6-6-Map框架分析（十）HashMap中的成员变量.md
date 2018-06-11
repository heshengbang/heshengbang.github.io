---

layout: post

title: Map框架分析（十）HashMap中的成员变量

date: 2018-6-3

tags: Java基础

---

本文基于JDK 1.8。在阅读源码的过程中，发现自己很多地方不能自洽，应该是对源码的理解有很大问题，本文自做记录不作参考，切勿以本文作参考！

### 相关知识点
- [Map](http://www.heshengbang.tech/2018/06/Map框架分析-二-Map接口分析/)
	- 内部接口
	- 方法
- [AbstractMap](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
	- 内部类
	- 方法
- HashMap
	- [HashMap中的内部类](http://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)
		- [HashMap的内部类TreeNode](http://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
	- HashMap中的方法和成员变量
		- [HashMap中的成员变量](http://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)
		- [HashMap中的方法](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法/)
            - [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
            - [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### HashMap的成员变量
- `transient Node<K,V>[] table;`
	- Node类型的数组，在第一次被使用的时候初始化，必要时进行扩容。table一旦被分配，总是2的幂次方。在某些情况下，table的大小可以为0，以允许当前某些不需要的启动机制。
	- table数组就是哈希桶数组，HashMap是由（数组+链表/红黑树）组成的，而table就是这个数组，这个数组也被称作哈希桶数组。

- `transient Set<Map.Entry<K,V>> entrySet;`
	- 用来缓存entrySet()方法的结果，当entrySet()被调用时优先从这个成员变量获取值，如果它为null，在去调用EntrySet内部类的构造函数，获取所有Entry的set集合，返回EntrySet实例的同时也将实例的引用赋给entrySet。以便下次entrySet()被调用时可以快速获取结果。

- `transient int size;`
	- 当前这个HashMap实例中包含的key-value键值对的数量

- `transient int modCount;`
	- HashMap实例的内部结构被修改的次数。实测了一下，就是指HashMap被插入或移除数据的次数。resize和树化或反树化，通常是插入或移除操作带来的附属操作，因此modCount主要指该HashMap被插入和删除的总次数。

- `int threshold;`
	- 下一次将进行扩容的容量。当某次插入操作之后，size大于threshold的时候，将进行扩容，threshold将变成下一次进行扩容的容量。

- `final float loadFactor;`
	- 负载因子。它的作用：threshold = loadFactor * table.size。该值默认为0.75。

- 一段测试代码：
```java
public class ModCountTest {
    public static void main(String[] args) {
        HashMap<Integer, String> one = new HashMap<>(1);
        //debug System.out.println("modCount="+one.modCount+"   size="+one.size+"   threshold="+one.threshold+"   数组大小="+one.table.length)
        one.put(1, "1");
        one.put(2, "2");
        one.put(3, "3");
        one.put(4, "4");
        one.put(5, "5");
        one.remove(1);
        System.out.println(one.toString());
    }
}
```
上面这段代码的运行结果如下：
```xml
modCount=1   size=1   threshold=1   数组大小=2
modCount=2   size=2   threshold=3   数组大小=4
modCount=3   size=3   threshold=3   数组大小=4
modCount=4   size=4   threshold=6   数组大小=8
modCount=5   size=5   threshold=6   数组大小=8
modCount=6   size=4   threshold=6   数组大小=8
{2=2, 3=3, 4=4, 5=5}
```

### HashMap的静态成员变量
- `static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;`
	- 默认初始化HashMap中能够存放的节点数据的大小，必须为2的幂次方。

- `static final int MAXIMUM_CAPACITY = 1 << 30;`
	- 最大的极限容量。HashMap扩容的极限最大容量。

- `static final float DEFAULT_LOAD_FACTOR = 0.75f;`
	- 默认的负载因子数值。上一节中提到：threshold = loadFactor * table.length，通常使用者不会也不建议指定负载因子的数值，因此0.75在benchmark的测试下，是最优的数值，无更换的必要。

- `static final int TREEIFY_THRESHOLD = 8;`
	- 默认将链表转为树形结构的节点数量的阈值。当向链表中添加一个元素后链表中的节点数量到达这个数字后，HashMap的哈希桶连接的链表将被转换为红黑树。这个值必须比2大，也至少为8，以便节点数量减少的时候，可以从红黑树转换为普通的链表结构。

- `static final int UNTREEIFY_THRESHOLD = 6;`
	- 默认反树化的阈值。在调整大小过程中（移除节点，插入节点导致resize进而rehash），以移除某个节点为例，当某个哈希桶中的节点连接的红黑树中的节点数目小于6时，就不再需要红黑树而将其转换为链表结构。它的值最大不能超过TREEIFY_THRESHOLD。

- `static final int MIN_TREEIFY_CAPACITY = 64;`
	- 默认哈希桶数组中可能存在红黑树的最小容量。（如果哈希桶数组的容量小于这个数值，当插入节点时，如果链表的节点太多，会导致扩容而不是转为红黑树。）这个值至少应该是4 * TREEIFY_THRESHOLD，以避免扩容和转化为红黑树阈值之间的冲突。例如：如果在HashMap的初期，多次插入恰巧插入到同一个哈希桶中，从而形成链表，进而形成红黑树，而哈希桶中的其他位置却没被使用，使HashMap的性能大大降低。

### 注意
- HashMap中需要注意几个值的区别：
	- 首先是哈希桶数组的大小，即table数组的长度
	- 其次是HashMap中存放的key-value键值对的数量，即HashMap存放的总的数据项数量
	- 链表大小，链表是指以HashMap中哈希桶中的元素为头结点构建的链表。即，以table数组的某个元素为头结点的链表结构的大小
	- 红黑树大小，红黑树是指以HashMap中哈希桶的元素为根节点的树形结构。即，以table数组的某个元素为根节点构成的红黑树的大小
	- 当插入元素后，某链表的大小大于8，链表会被转换为红黑树，即所谓的树化
	- 当HashMap的内部结构发生变化时，如果红黑树的大小小于6时，红黑树会转化为链表，即所谓的反树化
	- 当哈希桶数组太小的时候，HashMap内部是不会构建红黑树的。即链表达到一定长度后，会导致扩容，进而结构调整，链表长度会被稀释。只有哈希桶数组的大小达到MIN_TREEIFY_CAPACITY才有可能会产生红黑树。这一块后面HashMap的方法里面应该应该会重点说
	- 有几个问题可以思考一下：
		- 如何构建一个HashMap使其内部可以产生一个红黑树
		- 如何判断一个HashMap内部是否含有树形结构