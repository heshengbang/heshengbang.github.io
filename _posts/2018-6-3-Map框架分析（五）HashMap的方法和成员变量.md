---

layout: post

title: Map框架分析（五）HashMap的方法和成员变量

date: 2018-6-3

tags: Java基础

---

本文基于JDK8

### 相关知识点
- [Map](http://www.heshengbang.tech/2018/06/Map框架分析-二-Map接口分析/)
	- 内部接口
	- 方法
- [AbstractMap](http://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
	- 内部类
	- 方法
- HashMap
	- [HashMap中的内部类](http://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)
	- [HashMap中的方法和属性](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法和属性/)
		- [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
		- [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
		- [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### HashMap的成员变量
- `transient Node<K,V>[] table;`
	- Node类型的数组，在第一次被使用的时候初始化，必要时进行扩容。table一旦被分配，总是2的幂次方。在某些情况下，table的大小可以为0，以允许当前某些不需要的启动机制。
	- table数组就是哈希桶，HashMap是由（数组+链表/红黑树）组成的，而table就是这个数组，这个数组也被称作哈希桶。

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















