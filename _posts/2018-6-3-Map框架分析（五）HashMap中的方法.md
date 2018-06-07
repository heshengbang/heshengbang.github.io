---



layout: post



title: Map框架分析（五）HashMap中的方法



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

		- [HashMap的内部类TreeNode](http://www.heshengbang.tech/2018/06/Map框架分析（九）HashMap的内部类TreeNode/)

	- HashMap中的方法和成员变量

		- [HashMap中的成员变量](http://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)

		- [HashMap中的方法](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap中的方法/)

            - [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)

            - [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)

            - [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)



### HashMap的构造方法
- `public HashMap(int initialCapacity, float loadFactor)`
	- 给HashMap传入初始化大小，以及负载因子，其他的采用默认值即可。通常来讲这个构造方法不常用，如果打算指定初始化哈希桶大小，完全可以使用HashMap的一参构造函数，而如果打算修改loadFactor则是官方不建议执行的操作，默认的0.75其实是综合考虑的最优解。
	- 源码如下：
	```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
        // MAXIMUM_CAPACITY 在HashMap的成员变量部分有提到，是2的30次方
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        this.loadFactor = loadFactor;
        // 根据容量计算扩容的阈值
        this.threshold = tableSizeFor(initialCapacity);
    }
    ```

- `public HashMap(int initialCapacity)`
	- 指定一个初始容量来创建HashMap
	- 源码如下：
	```java
    public HashMap(int initialCapacity) {
    	// DEFAULT_LOAD_FACTOR在HashMap的成员变量部分有提到，默认值是0.75
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    ```

- `public HashMap()`
	- 使用默认的负载因子
	- 源码如下：
	```java
    public HashMap() {
    	// 所有成员变量介使用默认值
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
    ```

- `public HashMap(Map<? extends K, ? extends V> m)`
	- 将传入的一个map集合作为初始集合，创建一个新的HashMap
	- 源码如下：
	```java
    public HashMap(Map<? extends K, ? extends V> m) {
    	// 使用默认负载因子
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 
        putMapEntries(m, false);
    }
    ```

### HashMap的静态方法
- `static final int tableSizeFor(int cap)`
	- 获取给定容量向上的最近的2的幂次方。例如，如果cap为3，则返回4，如果给定cap为7则返回8，如果给定cap为21则返回32。如果传入的cap小于2，返回值都会是1。
	- 源码如下：
	```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        // >>> 是无符号向左移
        // | 是或操作符，a|=b即a=a|b
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```
- `static final int hash(Object key)`
	- 获取传入对象的哈希值
	- 源码如下：
	```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    ```
    - 说明：
    	- 如果传入的对象为null，则哈希值为0，这解释了为什么HashMap中只能有一个null作为键值
    	- 如果传入的对象不为null，就获取到其哈希值，因为哈希值是int所以是4字节*8位，一共32位。哈希值向右移16位，用高16位和低16位做异或运算，得到最终的哈希值。

- `static final int hash(Object key)`
	- 获取传入对象的哈希值。

### HashMap的实例方法
- `final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict)`
	- 在HashMap的clone()、HashMap()、putAll()被调用，用来将已有的map作为数据放入HashMap中。
	- 源码如下：
	```java
    // evict 是一个标记，用来确定调用这个方法是初始化时调用的还是在插入节点时调用的
    // 如果初始化时调用，则为false，如果是插入时(putAll)调用则为true
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        // 当传入的Map没值，就不用继续操作了
        if (s > 0) {
        	// 如果哈希桶为空，则需要进行必要的初始化工作
            if (table == null) {
            	// 根据传入的map大小以及负载因子获取HashMap的容量
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
                // 如果容量超过了扩容临界值，则根据传入容量向上取2的幂次方作为扩容临界值
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            // 如果传入的map大小超过了扩容临界值，则必然需要扩容
            else if (s > threshold)
                resize();
            // 遍历传入的map，将其挨个放入HashMap的数组中
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
    ```

### HashMap的私有方法
- `final Node<K,V>[] resize()`
	- 扩容，当插入元素后，发现元素总个数大于threshold时，进行扩容
	- 源码如下：
	```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        // 获取扩容前的容量和阈值
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        // 定义并计算新的容量和阈值
        int newCap, newThr = 0;
        if (oldCap > 0) {
        	// 如果扩容前的容量已经超过最大了，就直接返回旧的哈希桶并结束，不再进行扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 如果旧的容量大于默认初始化的容量，则新容量直接取旧容量的二倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 阈值也直接取旧阈值的两倍
                newThr = oldThr << 1;
        }
        // 如果旧容量小于等于0，则新容量就等于旧的阈值
        else if (oldThr > 0)
        	// initial capacity was placed in threshold
            newCap = oldThr;
        // 如果旧容量和旧阈值都小于等于0，则直接使用默认的容量，和默认的阈值
        else {
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        // 将新的阈值赋给阈值成员变量
        threshold = newThr;
        // 禁止显示一些无关紧要的警告
        @SuppressWarnings({"rawtypes","unchecked"})
        // 根据新的容量创建一个新的哈希桶
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 将新的哈希桶传递给哈希桶成员变量
        table = newTab;
        // 以下代码的目的：如果旧的哈希桶不为空就开始将旧的哈希桶中的数据通过rehash放到新的哈希桶中
        if (oldTab != null) {
        	// 遍历哈希桶中的所有元素
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 如果当前元素为空则进入下一次循环
                if ((e = oldTab[j]) != null) {
                	// 首先去掉旧哈希桶中的引用，这样去掉所有的旧哈希桶的引用过后可以被gc
                    oldTab[j] = null;
                    // 如果当前元素后面没有链表结构
                    // 则直接通过其哈希值和新的哈希桶容量进行位与操作获取其在新哈希桶中的位置
                    // 找到在新的哈希桶中的位置后，直接赋值replacementNode
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果当前元素是一个红黑树节点实例，则证明元素背后有红黑树结构
                    else if (e instanceof TreeNode)
                    	// splite方法定义在HashMap.TreeMap中，用于将整个树形结构从旧的哈希桶中移到新的哈希桶中
                        // this是旧哈希桶所在的HashMap实例，newTab是新的哈希桶，j是元素在旧哈希桶中的索引，oldCap是旧的哈希桶的容量
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 当前元素不为空，也不为树形结构，那么只能为链表结构
                    else {
                    	// 扩容以后，链表的节点将被分为两部分
                        // 以4/11在8的哈希桶中的的索引一致都是(4 mod 7)=4为例
                        // 在16的哈希桶中，4的索引还是(4 mod 15)=4，但是11的索引却变成了(11 mod 15) = 7+4
                        // 就像上面的例子一样，扩容后，原本在一个链表的节点会被分为两部分
                        // 一部分的索引不变，另一部分的索引变成了之前的位与结果+索引
                        // 这里定义两个子链表的头结点和尾节点，是为了分别保持其顺序，以防JDK1.8之前，resize之后链表中元素顺序倒置的问题
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 循环遍历链表结构
                        do {
                            next = e.next;
                            // 在put元素的时候，e.hash & (oldCap-1)
                            // 此处用e.hash & oldCap，可以将桶中的元素一分为二。这里的具体原理见底部【扩展】部分的代码
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 将元素所在的链表分成两两个子链表以后
                        // 其中一个子链表在哈希桶中的索引将没变化，所以直接将子链表的头节点放入哈希桶中
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 另一个子链表的索引是旧索引+旧容量，将链表头放入新哈希桶对应的位置即可
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        // 返回新的哈希桶
        return newTab;
    }
    ```
    - 步骤：
    	- 根据旧哈希桶数组的大小和扩容阈值加倍获取到新的哈希桶数组大小和阈值。如果旧容量等于0，则使用旧阈值为新的哈希桶数组容量，如果旧阈值也小于等于0，则使用默认的哈希桶数组大小和阈值。
    	- 根据新的哈希桶数组容量创建新的哈希桶数组
    	- 循环遍历旧的哈希桶数组，判断每一个元素是否为null
    		- 如果为null，进入下一次循环
    		- 如果不为null，判断其是否有下一个节点
                - 如果没有下一个节点，通过hash&(capacity-1)获取索引，然后放到新哈希桶对应的位置
                - 如果有下一个节点，则判断其是否为树节点
                	- 如果为树节点，则调用HashMap.TreeNode中的split()方法将以该节点为根节点拆分并分别放到新的哈希桶数组的对应位置
                	- 如果不为树节点，则为链表结构，将以该节点为头结点的链表拆分为两个子链表分别放入新的哈希桶数组的对应位置
- `final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)`
- `Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next)`，将TreeNode树节点转换为普通的Node节点。该方法仅在TreeNode的untreeify()方法中被调用。
	- 源码如下：
	```java
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        return new Node<>(p.hash, p.key, p.value, next);
    }
    ```










### 扩展
1. 如何将哈希桶中的元素一分为二？
	- 元素通过hash & (capacity-1)寻找到它在哈希桶数组中的位置，所以在同一个哈希桶中的元素通过这种方式计算的值恒定只有一个
	- 在resize的过程中，需要将哈希桶中的元素分为两部分，此时执行的操作是hash & capacity
	- 测试代码如下：
	```java
    for (int i = 0; i < 1000; i++) {
            // 模拟hash值的随机性
            int hash = new Random().nextInt(10000);
            // 该数字表示该hash在哈希桶数组对应的索引
            int index = hash & 15;
            // 模拟哈希桶数组中，索引为3的哈希桶
            if (index == 3) {
                // resize的时候，相同索引的元素会被区分为两部分
                int diff = hash & 16;
                // 从输出结果来看，resize中对链表或者红黑树的区分都是有用的
                System.out.println(hash + " -> " + index + "    " + diff);
            }
        }
    ```
    - 具体的原因在于capacity总是2的幂次方，例如8，16，32之类的，它们-1之后，的二进制表示分别是111,1111,11111，任意一个数字同这些数字进行&操作，都是求模运算。对于任意数字，比capacity多的位，不管是什么，capacity都填充为0，因此&操作总能得出0，capacity具备的位数，不管你是什么，&操作之后，你该是什么还是什么，以此达到求模运算的目的。在上一步骤求出一堆模数相同的数字，8,16,32的二进制是1000,10000,10000，它有且仅有一个数字为1，其他位数皆为0。对于任意数字，对上那些为0的位数的时候，不管你是什么，&总会返回0，而对上那个为1的位数，不管你是什么，&总会得出你本来的数字。那些模数相同的数字，它们因此会被区分为两部分。