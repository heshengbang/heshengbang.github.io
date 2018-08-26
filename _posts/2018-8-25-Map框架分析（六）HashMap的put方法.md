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
  putVal()方法有四个参数，前两个分别是key/value不用细说，后面两个boolean型的onlyIfAbsent、evict。onlyIfAbsent如果为true则表示不修改HashMap中已经存在的key对应的value，如果为false，则直接用给的参数value替换已经存在的value。evict如果为false，则表示HashMap当前处于创建模式。

### 详细步骤
- 
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  

