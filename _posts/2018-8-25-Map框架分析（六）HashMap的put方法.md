---

layout: post

title: Map框架分析（六）HashMap的put方法

date: 2018-8-24

tags: Java基础

---

本文基于JDK 1.8。在阅读源码的过程中，发现自己很多地方不能自洽，应该是对源码的理解有很大问题，本文自做记录不作参考，切勿以本文作参考！

### 相关知识点
- [Map](https://www.heshengbang.tech/2018/06/Map框架分析-二-Map接口分析/)
	- 内部接口
	- 方法
- [AbstractMap](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
	- 内部类
	- 方法
- HashMap
	- [HashMap中的内部类](https://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)
		- [HashMap的内部类TreeNode](https://www.heshengbang.tech/2018/06/Map框架分析（九）HashMap的内部类TreeNode/)
	- HashMap中的方法和成员变量
		- [HashMap中的成员变量](https://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)
		- [HashMap中的方法](https://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法/)
            - [HashMap的put方法](https://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
            - [HashMap的resize方法](https://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](https://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### 概述
  本文原起自今年6月份写HashMap的合集，是其中一篇，笔者当时写到一半就弃了，挖坑未填心中一直不安，近一个月又遭遇各种事情，心气难平。昨天思索再三，决定重新开始夯实基础，既然夯实基础那过去挖的坑必然要填上。HashMap这个数据结构在Java这门语言中的重要性不言而喻，因此直接写吧，有不明情况的同学，可以参考以上链接。  
  作为数据结构，HashMap自然包含存数据和取数据这两个基本功能。对Map略有了解的同学就应该知道，HashMap中最基本的元素时Entry，HashMap是Entry的集合，其本身就是Java集合框架中的一个异类，但是如果从Entry这个粒度来看，HashMap其实和List/Set等数据结构没区别。如同所有的集合框架类一样，HashMap是一个可变大小的集合，它存储的是Entry，俗称键值对，它允许使用者直接使用键值对的形式存取元素，而不是像一般的集合类那样整存整取。既然是存取两种操作，HashMap就有不同的实现方式，put就是最基本的存操作。HashMap还有其他一些存操作(putAll/putIfAbsent)，但都是通过put中的方法来实现的。下面展示的是put的源码：
```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
  从源码中可以看出，put方法没有执行任何操作，它只是将自己的参数传到另外一个私有方法putVal()中去。这个putVal()在HashMap的源码中出现过四次，分别是put()/putAll()/putIfAbsent()/readObject()中调用了该方法。四次虽然不多，但是对于整个HashMap实现的功能来说却至关重要。下面将展示putVal方法的源码，在阅读源码前请明确以下几点内容：
- 本文基于JDK8，因此HashMap的结构是哈希数组+链表+红黑树
- 了解HashMap的内部类Node/TreeNode的基本成员变量、方法及方法对应要实现的功能
- 链表的树化

以下是HashMap的putVal()方法的源码：
```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
  putVal()方法有五个参数，第一个是key的哈希值，第二第三个分别是key/value不用细说，最后两个boolean型的onlyIfAbsent、evict。onlyIfAbsent如果为true则表示不修改HashMap中已经存在的key对应的value，如果为false，则直接用给的参数value替换已经存在的value。evict如果为false，则表示HashMap当前处于创建模式。

### put方法详细步骤
- 获取key的哈希值，源码如下：
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
代码的基本逻辑是：
	- 判断key是否为null，如果为null，哈希值就为0
	- 调用key自身的hashcode()方法，该方法是继承自Object的native方法，每个对象都有
	- 将key的哈希值无符号右移16位，左边用0补位得到值
	- 将上一步的值与key的哈希值取异或运算，获得一个32位的哈希值
		- 高16位的值在异或的运算下不会改变，因为补位0与(0/1)做异或，值仍是(0/1)
		- 低16的结果是高16位与低16位的异或结果
		- 这种求哈希的方式是jdk8新增的

- 如果哈希桶数组为null或者长度为0，则进行扩容
- 获取到哈希桶数组的大小n
- 通过&操作取模，这个模的结果即key在哈希桶数组上的索引位置。这段代码源码如下：
```java
    if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
```
这段代码的基本逻辑如下：
	- 通过哈希桶数组容量n-1与哈希值取模，模的结果一定是0 ~ n-1。至此，在哈希桶数组上获得一个索引值。
	- 判定索引值对应的哈希桶是否为空
	- 如果哈希桶为null就创建一个新的节点作为该索引的哈希桶
		- 新创建的节点的哈希值、key、value均被指定
		- 新节点的next节点为null，因为它后面暂时还没有链表节点需要去连接
		- 该处创建的节点为Node节点，区别于树节点TreeNode

- 索引对应的哈希桶p不为空，则进入下一个大的代码模块
- 判定如果哈希桶p对应的哈希值与参数哈希值相等，并且，p的key与参数key是同一个对象(==)或者值相等(equals)，将p的引用保留。代码如下：
```java
    if (p.hash == hash &&
    	((k = p.key) == key ||(key != null && key.equals(k))))
        e = p;
```

- p的哈希值与参数哈希值不相等，或者p的key与参数key既不是同一个对象也不是值相等则判定节点p是否为一个树节点，代码如下：
```java
    else if (p instanceof TreeNode)
        e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
这段代码首先判定了p节点是否为树节点，如果为树节点就调用TreeNode的putTreeVal方法将put的key/value传进去，需要说明以下几点：
	- this是指当前这个HashMap实例
	- tab是指HashMap实例的哈希桶数组
	- 完成TreeNode的putTreeVal方法后，会返回一个TreeNode，不过由于TreeNode继承自LinkedHashMap.Entry，而LinkedHashMap.Entry又继承自HashMap.Node，所以e可以接收TreeNode这个返回值
	- TreeNode#putTreeVal方法会在后面详细展开写，这里就不讲里面的具体实现步骤

- 参数key/value在哈希桶数组上的位置，既不在数组上，也不在树节点上，那只能是在哈希桶数组上连接的链表上
- 循环遍历哈希桶连接的链表，将参数哈希值和参数key和链表上的节点的哈希值和参数key进行比较，如果找到就跳出循环，如果找不到就创建节点，如果创建节点后链表大小超过树化临界值就将链表转换为红黑树，代码如下：
```java
	for (int binCount = 0; ; ++binCount) {
		if ((e = p.next) == null) {
			p.next = newNode(hash, key, value, null);
			if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
				treeifyBin(tab, hash);
				break;
			}
		if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
			break;
		p = e;
	}
```
这段代码基本的逻辑如下：
	- 开启一次没有结束条件的遍历（在JDK8之前，这样写会有死循环的风险）
	- 判定哈希桶连接的链表的节点是否为null，如果为null就使用给定参数创建一个新的Node节点，并将新节点的引用赋值给哈希桶的next，使其相互关联起来
        - 判定创建节点后，哈希桶链接的链表大小是否超过了树化临界点，JDK在此处给出了一个难得的注释，“-1 for 1st”，意思是-1是因为遍历是从0开始的
        - 如果创建节点后<b>大于等于</b>树化临界值就将链表转化为红黑树，传入参数为哈希桶数组和哈希值，然后跳出循环
	- 如果哈希桶连接的链表的节点不为null，就比较其和参数的哈希值和key是否相等，判定条件与上面一致，不详述
		- 如果相等就跳出循环
	- 这段循环代码执行结束后，key/value被插入或已存在的位置依然将被记录下来

- key/value根据key找到一个已经存在的位置
- put调用的putVal()onlyOfAbsent永远为false，则节点将肯定被赋值为参数value，这段代码如下：
```java
    if (e != null) { // existing mapping for key
		V oldValue = e.value;
		if (!onlyIfAbsent || oldValue == null)
			e.value = value;
		afterNodeAccess(e);
		return oldValue;
	}
```
关于这段代码有以下几点需要注意：
	- e != null说明key在当前这个HashMap实例中已经存在，即调用containsKey会返回true
	- onLyIfAbsent为false，即无论怎样，新的value都将替换旧value
	- afterNodeAccess是留给LinkedHashMap的接口
	- 旧的value将被返回

- 如果上面没有return并结束putVal方法就证明key/value是通过新建节点插入到HashMap实例的，这通常会意味着：
    - 将HashMap的结构发生了改变，因此结构被修改次数+1
    - 将HashMap当前大小+1，如果大小超过了扩容阈值就扩容
	- return null;结束putVal方法，也结束整个put方法


以上是整个put方法最直接的步骤，与其强相关的还有两个方法，将在接下来详细解释其基本步骤及逻辑。

### TreeNode#putTreeVal
- `TreeNode#putTreeVal`是TreeNode的成员方法，意即插入新的树节点到红黑树上


### treeifyBin
- treeifyBin意即将<b>大于等于</b>树化临界值的链表转换为红黑树
