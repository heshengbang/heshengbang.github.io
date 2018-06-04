---

layout: post

title: Map框架分析（四）HashMap的内部类

date: 2018-6-3

tags: Java基础

---

本文基于JDK 1.8

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
	- [HashMap中的方法和属性](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法和属性/)
		- [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
		- [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
		- [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### 简述
- HashMap中一共有15个内部类，其中13个是自己独有的，另外2个是继承自AbstractMap。它们的关系如下图所示（红线）：
![HashMap中的内部类](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/HashMap源码分析/HashMap_InnerClass.png)
	- 继承自Abstract的内部类在HashMap中并未直接使用到，个人感觉是作为一种继承AbstractMap的默认方法实现时的一种附属产品，也可能是作为一种潜在的拓展。
		- [SimpleEntry，前文有介绍，点击查看](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
		- [SimpleImmutableEntry，前文有介绍，点击查看](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

	- HashMap中自己定义了十三个内部类，分别是：
		- Node：基于hash的节点，被大多数Map.Entry的实现类所使用。如下面的TreeNode、LinkedHashMap的Entry
			- TreeNode
		- HashIterator：抽象迭代器
			- KeyIterator：HashMap中的Key集合的迭代器
			- ValueIterator：HashMap中的Value集合的迭代器
			- EntryIterator：HashMap中的Entry集合的迭代器
		- HashMapSpliterator：抽象的并行迭代器
			- KeySpliterator：HashMap中的Key集合的并行迭代器
			- ValueSpliterator：HashMap中的Value集合的并行迭代器
			- EntrySpliterator：HashMap中的Entry集合的并行迭代器
		- Values：HashMap中的value的集合
		- KeySet：HashMap中的key的集合
		- EntrySet：HashMap中的Entry的集合

- 通过HashMap的内部类的结构就非常清晰了。分别是继承自AbstractMap的SimpleEntry和SimpleImmutableEntry，HashMap自己定义的13个内部类大致分成四部分，分别是用作链表节点的Node及其子类用作红黑树节点的TreeNode，集合迭代器HashIterator及其子类KeyIterator、ValueIterator、EntryIterator，并行迭代器HashMapSpliterator及其子类KeySpliterator、ValueSpliterator、EntrySpliterator，Key的集合、value的集合、Entry的集合。

- 如上面的结构示意图所示，HashMap中定义了抽象的迭代器类和并行迭代器，并分别针对Key/Value/Entry三种集合去实现了迭代器和并行迭代器，而EntrySet和KeySet都继承自抽象的AbstractSet，保证了该集合里面元素的唯一性，Values则没有继承自AbstractSet而是AbstractCollection。众所周知，HashMap中的key值是唯一的，而value却可以不是，而Entry包含了key-value，因此也势必是唯一的。联系HashMap的继承关系和特性，依次从四个方面回忆HashMap的内部类，相对就很好记忆了。

### 内部类解析
- [SimpleEntry，前文有介绍，点击查看](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

- [SimpleImmutableEntry，前文有介绍，点击查看](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

- Node源码及注释如下：
```java
	//基于hash的节点，被大多数Map.Entry的实现类所使用。如下面的TreeNode、LinkedHashMap的Entry
    static class Node<K,V> implements Map.Entry<K,V> {
    	// key值对应的hash值
        final int hash;
        // 该节点的key
        final K key;
        // 该节点的value
        V value;
        // 该节点的下一个节点。HashMap对应的hash桶后面链接的链表结构，单向的
        Node<K,V> next;
        // 构造函数
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        // 获取key
        public final K getKey()        { return key; }
        // 获取value
        public final V getValue()      { return value; }
        // 重写的toString方法
        public final String toString() { return key + "=" + value; }
        // 获取hashCode的方法。通过key和value的hashCode进行异或运算得到Node的hash值
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
        // 设置Node中心的value值
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        // 判断指定对象与Node是否相同
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```
	- 如上源码所示，HashMap.Node中值得关注的只有一个点，即Node求hashcode的方法。调用Objects的方法获取key和value的hash值，然后进行异或运算。事实上，Objects类中的工具类，hashCode()其实也是调用对象本身的Native hashCode()方法去获取hashCode。

- [TreeNode，后文有介绍，点击查看](http://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)













