---

layout: post

title: Map框架分析（四）HashMap的内部类

date: 2018-6-3

tags: Java基础

---

本文基于JDK 1.8。

### 相关知识点
- [Map](https://www.heshengbang.tech/2018/06/Map框架分析-二-Map接口分析/)
	- 内部接口
	- 方法
- [AbstractMap](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
	- 内部类
	- 方法
- HashMap
	- [HashMap中的内部类](https://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)
		- [HashMap的内部类TreeNode](https://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
	- HashMap中的方法和成员变量
		- [HashMap中的成员变量](https://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)
		- [HashMap中的方法](https://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap中的方法/)
            - [HashMap的put方法](https://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
            - [HashMap的resize方法](https://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](https://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### 简述
- HashMap中一共有15个内部类，其中13个是自己独有的，另外2个是继承自AbstractMap。它们的关系如下图所示（红线）
![HashMap中的内部类](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/HashMap源码分析/HashMap_InnerClass.png)
	- 继承自Abstract的内部类在HashMap中并未直接使用到，个人感觉是作为一种继承AbstractMap的默认方法实现时的一种附属产品，也可能是作为一种潜在的拓展。
		- [SimpleEntry，前文有介绍，点击查看](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
		- [SimpleImmutableEntry，前文有介绍，点击查看](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

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
- [SimpleEntry，前文有介绍，点击查看](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

- [SimpleImmutableEntry，前文有介绍，点击查看](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)

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
        public final K getKey() {
        	return key;
        }
        // 获取value
        public final V getValue() {
        	return value;
        }
        // 重写的toString方法
        public final String toString() {
        	return key + "=" + value;
        }
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

- [TreeNode，后文有介绍，点击查看](https://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)

- HashIterator的源码及注释如下：
```java
abstract class HashIterator {
		// next entry to return
        Node<K,V> next;
        // current entry
        Node<K,V> current;
        // for fast-fail
        // fail-fast 机制是java集合(Collection)中的一种错误机制
        // 当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件
        int expectedModCount;
        // current slot
        int index;
        // 构造器方法
        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            // 哈希桶数组不为空时，一路遍历将next变成桶的最后一个元素
            if (t != null && size > 0) {
            	// advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
        // 是否还有下一个元素
        public final boolean hasNext() {
            return next != null;
        }
        // 下一个节点
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            // 前面进行了同值操作，这里不相等，说明有多个线程访问抛错
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            // 下一个节点为null，抛错
            if (e == null)
                throw new NoSuchElementException();
            // 下一个节点为空同时，哈希桶不为空
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
        // 移除节点
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```
	- HashIterator是一个抽象类，主要用来给KeyIterator、ValueIterator、EntryIterator提供一个抽象的方法和成员变量。它的三个实现类中完成了对应的个性化操作。

- KeyIterator的源码及注释如下：
```java
final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() {
        	return nextNode().key;
        }
}
```

- ValueIterator的源码及注释如下：
```java
final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() {
        	return nextNode().value;
        }
}
```

- ValueIterator的源码及注释如下：
```java
final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() {
        	return nextNode();
        }
}
```

- HashMapSpliterator的源码及注释如下：
```java
    static class HashMapSpliterator<K,V> {
        final HashMap<K,V> map;
        // current node
        Node<K,V> current;
        // current index, modified on advance/split
        int index;
        // one past last index
        int fence;
        // size estimate
        int est;
        // for comodification checks
        int expectedModCount;
        HashMapSpliterator(HashMap<K,V> m, int origin,
                           int fence, int est,
                           int expectedModCount) {
            this.map = m;
            this.index = origin;
            this.fence = fence;
            this.est = est;
            this.expectedModCount = expectedModCount;
        }
        // initialize fence and size on first use
        final int getFence() {
            int hi;
            if ((hi = fence) < 0) {
                HashMap<K,V> m = map;
                est = m.size;
                expectedModCount = m.modCount;
                Node<K,V>[] tab = m.table;
                hi = fence = (tab == null) ? 0 : tab.length;
            }
            return hi;
        }
        public final long estimateSize() {
            getFence(); // force init
            return (long) est;
        }
    }
```

- KeySpliterator的源码及注释如下：
```java
    static final class KeySpliterator<K,V>
        extends HashMapSpliterator<K,V>
        implements Spliterator<K> {
        KeySpliterator(HashMap<K,V> m, int origin, int fence, int est,
                       int expectedModCount) {
            super(m, origin, fence, est, expectedModCount);
        }

        public KeySpliterator<K,V> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid || current != null) ? null :
                new KeySpliterator<>(map, lo, index = mid, est >>>= 1,
                                        expectedModCount);
        }

        public void forEachRemaining(Consumer<? super K> action) {
            int i, hi, mc;
            if (action == null)
                throw new NullPointerException();
            HashMap<K,V> m = map;
            Node<K,V>[] tab = m.table;
            if ((hi = fence) < 0) {
                mc = expectedModCount = m.modCount;
                hi = fence = (tab == null) ? 0 : tab.length;
            }
            else
                mc = expectedModCount;
            if (tab != null && tab.length >= hi &&
                (i = index) >= 0 && (i < (index = hi) || current != null)) {
                Node<K,V> p = current;
                current = null;
                do {
                    if (p == null)
                        p = tab[i++];
                    else {
                        action.accept(p.key);
                        p = p.next;
                    }
                } while (p != null || i < hi);
                if (m.modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }

        public boolean tryAdvance(Consumer<? super K> action) {
            int hi;
            if (action == null)
                throw new NullPointerException();
            Node<K,V>[] tab = map.table;
            if (tab != null && tab.length >= (hi = getFence()) && index >= 0) {
                while (current != null || index < hi) {
                    if (current == null)
                        current = tab[index++];
                    else {
                        K k = current.key;
                        current = current.next;
                        action.accept(k);
                        if (map.modCount != expectedModCount)
                            throw new ConcurrentModificationException();
                        return true;
                    }
                }
            }
            return false;
        }

        public int characteristics() {
            return (fence < 0 || est == map.size ? Spliterator.SIZED : 0) |
                Spliterator.DISTINCT;
        }
    }
```

- ValueSpliterator的源码及注释如下：
```java
static final class ValueSpliterator<K,V>
        extends HashMapSpliterator<K,V>
        implements Spliterator<V> {
        ValueSpliterator(HashMap<K,V> m, int origin, int fence, int est,
                         int expectedModCount) {
            super(m, origin, fence, est, expectedModCount);
        }
        public ValueSpliterator<K,V> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid || current != null) ? null :
                new ValueSpliterator<>(map, lo, index = mid, est >>>= 1,
                                          expectedModCount);
        }
        public void forEachRemaining(Consumer<? super V> action) {
            int i, hi, mc;
            if (action == null)
                throw new NullPointerException();
            HashMap<K,V> m = map;
            Node<K,V>[] tab = m.table;
            if ((hi = fence) < 0) {
                mc = expectedModCount = m.modCount;
                hi = fence = (tab == null) ? 0 : tab.length;
            }
            else
                mc = expectedModCount;
            if (tab != null && tab.length >= hi &&
                (i = index) >= 0 && (i < (index = hi) || current != null)) {
                Node<K,V> p = current;
                current = null;
                do {
                    if (p == null)
                        p = tab[i++];
                    else {
                        action.accept(p.value);
                        p = p.next;
                    }
                } while (p != null || i < hi);
                if (m.modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
        public boolean tryAdvance(Consumer<? super V> action) {
            int hi;
            if (action == null)
                throw new NullPointerException();
            Node<K,V>[] tab = map.table;
            if (tab != null && tab.length >= (hi = getFence()) && index >= 0) {
                while (current != null || index < hi) {
                    if (current == null)
                        current = tab[index++];
                    else {
                        V v = current.value;
                        current = current.next;
                        action.accept(v);
                        if (map.modCount != expectedModCount)
                            throw new ConcurrentModificationException();
                        return true;
                    }
                }
            }
            return false;
        }
        public int characteristics() {
            return (fence < 0 || est == map.size ? Spliterator.SIZED : 0);
        }
    }
```



- EntrySpliterator的源码及注释如下：
```java
static final class EntrySpliterator<K,V>
        extends HashMapSpliterator<K,V>
        implements Spliterator<Map.Entry<K,V>> {
        EntrySpliterator(HashMap<K,V> m, int origin, int fence, int est,
                         int expectedModCount) {
            super(m, origin, fence, est, expectedModCount);
        }
        public EntrySpliterator<K,V> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid || current != null) ? null :
                new EntrySpliterator<>(map, lo, index = mid, est >>>= 1,
                                          expectedModCount);
        }
        public void forEachRemaining(Consumer<? super Map.Entry<K,V>> action) {
            int i, hi, mc;
            if (action == null)
                throw new NullPointerException();
            HashMap<K,V> m = map;
            Node<K,V>[] tab = m.table;
            if ((hi = fence) < 0) {
                mc = expectedModCount = m.modCount;
                hi = fence = (tab == null) ? 0 : tab.length;
            }
            else
                mc = expectedModCount;
            if (tab != null && tab.length >= hi &&
                (i = index) >= 0 && (i < (index = hi) || current != null)) {
                Node<K,V> p = current;
                current = null;
                do {
                    if (p == null)
                        p = tab[i++];
                    else {
                        action.accept(p);
                        p = p.next;
                    }
                } while (p != null || i < hi);
                if (m.modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }

        public boolean tryAdvance(Consumer<? super Map.Entry<K,V>> action) {
            int hi;
            if (action == null)
                throw new NullPointerException();
            Node<K,V>[] tab = map.table;
            if (tab != null && tab.length >= (hi = getFence()) && index >= 0) {
                while (current != null || index < hi) {
                    if (current == null)
                        current = tab[index++];
                    else {
                        Node<K,V> e = current;
                        current = current.next;
                        action.accept(e);
                        if (map.modCount != expectedModCount)
                            throw new ConcurrentModificationException();
                        return true;
                    }
                }
            }
            return false;
        }
        public int characteristics() {
            return (fence < 0 || est == map.size ? Spliterator.SIZED : 0) |
                Spliterator.DISTINCT;
        }
    }
```

- KeySet源码及注释如下：
```java
final class KeySet extends AbstractSet<K> {
        public final int size() {
        	return size;
        }
        public final void clear() {
        	HashMap.this.clear();
        }
        public final Iterator<K> iterator() {
        	return new KeyIterator();
        }
        public final boolean contains(Object o) {
        	return containsKey(o);
        }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
```

- Values的源码及注释如下：
```java
    final class Values extends AbstractCollection<V> {
        public final int size() {
        	return size;
        }
        public final void clear() {
        	HashMap.this.clear();
        }
        public final Iterator<V> iterator() {
        	return new ValueIterator();
        }
        public final boolean contains(Object o) {
        	return containsValue(o);
        }
        public final Spliterator<V> spliterator() {
            return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super V> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.value);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
```

- EntrySet的源码及注释如下：
```java
    final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public final int size() {
        	return size;
        }
        public final void clear() {
        	HashMap.this.clear();
        }
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        public final boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Node<K,V> candidate = getNode(hash(key), key);
            return candidate != null && candidate.equals(e);
        }
        public final boolean remove(Object o) {
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>) o;
                Object key = e.getKey();
                Object value = e.getValue();
                return removeNode(hash(key), key, value, true, true) != null;
            }
            return false;
        }
        public final Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
```