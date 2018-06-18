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
		- [HashMap的内部类TreeNode](http://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
	- HashMap中的方法和成员变量
		- [HashMap中的成员变量](http://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)
		- [HashMap中的方法](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap中的方法/)
            - [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
            - [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)



### HashMap的构造方法
- `public HashMap(int initialCapacity, float loadFactor)` 给HashMap传入初始化大小，以及负载因子，其他的采用默认值即可。通常来讲这个构造方法不常用，如果打算指定初始化哈希桶大小，完全可以使用HashMap的一参构造函数，而如果打算修改loadFactor则是官方不建议执行的操作，默认的0.75其实是综合考虑的最优解。
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

- `public HashMap(int initialCapacity)` 指定一个初始容量来创建HashMap
	- 源码如下：
	```java
    public HashMap(int initialCapacity) {
            // DEFAULT_LOAD_FACTOR在HashMap的成员变量部分有提到，默认值是0.75
            this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    ```

- `public HashMap()` 使用默认的负载因子
	- 源码如下：
	```java
    public HashMap() {
            // 所有成员变量介使用默认值
            this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
    ```

- `public HashMap(Map<? extends K, ? extends V> m)` 将传入的一个map集合作为初始集合，创建一个新的HashMap
	- 源码如下：
	```java
    public HashMap(Map<? extends K, ? extends V> m) {
            // 使用默认负载因子
            this.loadFactor = DEFAULT_LOAD_FACTOR;
            putMapEntries(m, false);
    }
    ```

### HashMap的静态方法
- `static final int tableSizeFor(int cap)` 获取给定容量向上的最近的2的幂次方。例如，如果cap为3，则返回4，如果给定cap为7则返回8，如果给定cap为21则返回32。如果传入的cap小于2，返回值都会是1。该方法仅在HashMap的构造方法`public HashMap(int initialCapacity, float loadFactor)`以及putMapEntries()和readObject()中被调用。
	- 源码如下：
	```java
    static final int tableSizeFor(int cap) {
            int n = cap - 1;
            // >>> 是无符号左移
            // |是或操作符，a|=b即a=a|b
            n |= n >>> 1;
            n |= n >>> 2;
            n |= n >>> 4;
            n |= n >>> 8;
            n |= n >>> 16;
            return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```

- `static final int hash(Object key)` 获取传入对象的哈希值，该方法在HashMap中很多地方都会用到。
	- 源码如下：
	```java
    static final int hash(Object key) {
            int h;
            return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    ```
    - 说明:
		- 如果传入的对象为null，则哈希值为0，这解释了为什么HashMap中只能有一个null作为键值
		- 如果传入的对象不为null，就获取到其哈希值，因为哈希值是int所以是`4(字节)*8(位)=32(位)`。哈希值向左移16位，用高16位和低16位做异或运算，得到最终的哈希值。

- `static Class<?> comparableClassFor(Object x)` 如果参数x实现了Comparable接口，则返回x的类型，否则返回null。该方法主要在HashMap.TreeMap中和compareComparables()配合使用，分别在find()，treeify()，putTreeVal()中被调用。
	- 源码如下：
	```java
    static Class<?> comparableClassFor(Object x) {
            if (x instanceof Comparable) {
                Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
                if ((c = x.getClass()) == String.class) // bypass checks
                    return c;
                if ((ts = c.getGenericInterfaces()) != null) {
                    for (int i = 0; i < ts.length; ++i) {
                        if (((t = ts[i]) instanceof ParameterizedType) &&
                            ((p = (ParameterizedType)t).getRawType() ==
                             Comparable.class) &&
                            (as = p.getActualTypeArguments()) != null &&
                            as.length == 1 && as[0] == c) // type arg is c
                            return c;
                    }
                }
            }
            return null;
    }
    ```
	- 涉及到的基本知识点
		- ParameterizedType：继承自Type，代表一个提供泛型支持的类型，例如:`java.util.Collection<E>`
			- ParameterizedType#getRawType 获取实际声明的类型对象
			- ParameterizedType#getActualTypeArguments 获取实际作为泛型参数的类型，返回值是一个数组对象
		- Class#getGenericInterfaces：获取该类直接实现的接口
	- 代码逻辑
		- 判断参数x是否实现了Comparable接口，如果没有，则直接返回null
        - 在x实现了Comparable接口的情况下，继续判断下一步骤
        - 参数x是否是String类型的，如果是，则直接返回String.class
        - 在参数x的类型p不是String类型的情况下，继续判断下一步骤
        - 如果p直接实现的接口数组ts为空，则直接返回null
        - 在ts不为空的情况下，继续下一步骤
        - 循环遍历ts，从t开始
			- 如果t是一个参数化类型的实例，同时t的实际声明类型是Comparable.class，同时t的泛型参数的个数为1，并且泛型参数就是x的类型c，则返回类型c
		- 在以上方式都没找到的情况下，则返回null
    - 简单点来说，这个方法就是用来判断传入的对象是否为String类型或者实现了Comparable接口的类的实例，如果是就返回实例的类型，如果不是就返回null

- `static int compareComparables(Class<?> kc, Object k, Object x)` 比较两个对象的大小。该方法主要在HashMap.TreeMap中和comparableClassFor()配合使用，分别在find()，treeify()，putTreeVal()中被调用。
	- 源码如下：
	```java
    static int compareComparables(Class<?> kc, Object k, Object x) {
        	return (x == null || x.getClass() != kc ? 0 : ((Comparable)k).compareTo(x));
    }
    ```
    - 上面的代码拆分为三部分进行分析
    ```java
        // 该语句用来确认对象x既不为null，类型也确实为kc
        // 如果这两个条件中有任意一个不满足，就会直接返回0
        boolean condition = (x==null || x.getClass()!=kc;
        // 在condition为false的情况下，返回下面的判断结果
        boolean result = ((Comparable)k).compareTo(x));
        condition ? 0 : result;
    ```

### HashMap的实例方法
- `public int size()` 获取HashMap当前的大小。
	- 源码如下：
	```java
    public int size() {
            return size;
    }
    ```

- `public boolean isEmpty()` 判断该HashMap实例是否为空。
	- 源码如下：
	```java
    public boolean isEmpty() {
            return size == 0;
    }
    ```

- `public V get(Object key)` 根据key获取value。
	- 源码如下：
	```java
    public V get(Object key) {
            Node<K,V> e;
            return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    ```

- `public V getOrDefault(Object key, V defaultValue)` 获取key对应的value，如果value为null就返回defaultValue。
	- 源码如下：
	```java
    @Override
    public V getOrDefault(Object key, V defaultValue) {
            Node<K,V> e;
            return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
    }
    ```

- `public V putIfAbsent(K key, V value)` 如果key在HashMap中没有对应的值，就插入value。
	- 源码如下：
	```java
    @Override
    public V putIfAbsent(K key, V value) {
            return putVal(hash(key), key, value, true, true);
    }
    ```

- `public boolean remove(Object key, Object value)` 移除指定的key-value键值对。
	- 源码如下：
	```java
    @Override
    public boolean remove(Object key, Object value) {
            return removeNode(hash(key), key, value, true, true) != null;
    }
    ```

- `public boolean containsKey(Object key)` 判断HashMap是否包含某个key值。
	- 源码如下：
	```java
    public boolean containsKey(Object key) {
            return getNode(hash(key), key) != null;
    }
    ```

- `public boolean replace(K key, V oldValue, V newValue)` 使用指定的新值去替换旧的key-value键值对中的value。
	- 源码如下：
	```java
    @Override
    public boolean replace(K key, V oldValue, V newValue) {
            Node<K,V> e; V v;
            if ((e = getNode(hash(key), key)) != null &&
                ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
                e.value = newValue;
                afterNodeAccess(e);
                return true;
            }
            return false;
    }
    ```

- `public V replace(K key, V value)` 用指定的key-value键值对替换HashMap中存在的key-oldValue键值对。
	- 源码如下：
	```java
    @Override
    public V replace(K key, V value) {
            Node<K,V> e;
            if ((e = getNode(hash(key), key)) != null) {
                V oldValue = e.value;
                e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
            return null;
    }
    ```

- `public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)` 该方法初始定义在Map接口中，并且提供了默认实现。
	- 源码如下：
	```java
    @Override
    public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {
            if (mappingFunction == null)
                throw new NullPointerException();
            int hash = hash(key);
            Node<K,V>[] tab; Node<K,V> first; int n, i;
            int binCount = 0;
            TreeNode<K,V> t = null;
            Node<K,V> old = null;
            if (size > threshold || (tab = table) == null ||
                (n = tab.length) == 0)
                n = (tab = resize()).length;
            if ((first = tab[i = (n - 1) & hash]) != null) {
                if (first instanceof TreeNode)
                    old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
                else {
                    Node<K,V> e = first; K k;
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k)))) {
                            old = e;
                            break;
                        }
                        ++binCount;
                    } while ((e = e.next) != null);
                }
                V oldValue;
                if (old != null && (oldValue = old.value) != null) {
                    afterNodeAccess(old);
                    return oldValue;
                }
            }
            V v = mappingFunction.apply(key);
            if (v == null) {
                return null;
            } else if (old != null) {
                old.value = v;
                afterNodeAccess(old);
                return v;
            }
            else if (t != null)
                t.putTreeVal(this, tab, hash, key, v);
            else {
                tab[i] = newNode(hash, key, v, first);
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    treeifyBin(tab, hash);
            }
            ++modCount;
            ++size;
            afterNodeInsertion(true);
            return v;
    }
    ```

- `public V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)` 该方法初始定义在Map接口中。
	- 源码如下：
	```java
    public V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
            if (remappingFunction == null)
                throw new NullPointerException();
            Node<K,V> e; V oldValue;
            int hash = hash(key);
            if ((e = getNode(hash, key)) != null &&
                (oldValue = e.value) != null) {
                V v = remappingFunction.apply(key, oldValue);
                if (v != null) {
                    e.value = v;
                    afterNodeAccess(e);
                    return v;
                }
                else
                    removeNode(hash, key, null, false, true);
            }
            return null;
    }
    ```

- `public V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)` 该方法初始定义在Map接口中。
	- 源码如下：
	```java
    @Override
    public V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
            if (remappingFunction == null)
                throw new NullPointerException();
            int hash = hash(key);
            Node<K,V>[] tab; Node<K,V> first; int n, i;
            int binCount = 0;
            TreeNode<K,V> t = null;
            Node<K,V> old = null;
            if (size > threshold || (tab = table) == null ||
                (n = tab.length) == 0)
                n = (tab = resize()).length;
            if ((first = tab[i = (n - 1) & hash]) != null) {
                if (first instanceof TreeNode)
                    old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
                else {
                    Node<K,V> e = first; K k;
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k)))) {
                            old = e;
                            break;
                        }
                        ++binCount;
                    } while ((e = e.next) != null);
                }
            }
            V oldValue = (old == null) ? null : old.value;
            V v = remappingFunction.apply(key, oldValue);
            if (old != null) {
                if (v != null) {
                    old.value = v;
                    afterNodeAccess(old);
                }
                else
                    removeNode(hash, key, null, false, true);
            }
            else if (v != null) {
                if (t != null)
                    t.putTreeVal(this, tab, hash, key, v);
                else {
                    tab[i] = newNode(hash, key, v, first);
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                }
                ++modCount;
                ++size;
                afterNodeInsertion(true);
            }
            return v;
    }
    ```

- `public V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction)` 该方法初始定义在Map接口中。
	- 源码如下：
	```java
    @Override
    public V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
            if (value == null)
                throw new NullPointerException();
            if (remappingFunction == null)
                throw new NullPointerException();
            int hash = hash(key);
            Node<K,V>[] tab; Node<K,V> first; int n, i;
            int binCount = 0;
            TreeNode<K,V> t = null;
            Node<K,V> old = null;
            if (size > threshold || (tab = table) == null ||
                (n = tab.length) == 0)
                n = (tab = resize()).length;
            if ((first = tab[i = (n - 1) & hash]) != null) {
                if (first instanceof TreeNode)
                    old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
                else {
                    Node<K,V> e = first; K k;
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k)))) {
                            old = e;
                            break;
                        }
                        ++binCount;
                    } while ((e = e.next) != null);
                }
            }
            if (old != null) {
                V v;
                if (old.value != null)
                    v = remappingFunction.apply(old.value, value);
                else
                    v = value;
                if (v != null) {
                    old.value = v;
                    afterNodeAccess(old);
                }
                else
                    removeNode(hash, key, null, false, true);
                return v;
            }
            if (value != null) {
                if (t != null)
                    t.putTreeVal(this, tab, hash, key, value);
                else {
                    tab[i] = newNode(hash, key, value, first);
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                }
                ++modCount;
                ++size;
                afterNodeInsertion(true);
            }
            return value;
    }
    ```

- `public void forEach(BiConsumer<? super K, ? super V> action)` 该方法初始定义在Map接口中。
	- 源码如下：
	```java
    @Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key, e.value);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
    }
    ```

- `public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function)` 该方法初始定义在Map接口中。
	- 源码如下：
	```java
    @Override
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
            Node<K,V>[] tab;
            if (function == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                        e.value = function.apply(e.key, e.value);
                    }
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
    }
    ```

- `public Object clone()` 克隆当前HashMap的实例，拥有相同的key-value键值对
	- 源码如下：
	```java
    @Override
    public Object clone() {
            HashMap<K,V> result;
            try {
                result = (HashMap<K,V>)super.clone();
            } catch (CloneNotSupportedException e) {
                // this shouldn't happen, since we are Cloneable
                throw new InternalError(e);
            }
            result.reinitialize();
            result.putMapEntries(this, false);
            return result;
    }
    ```

- `public boolean containsValue(Object value)` 判断HashMap中是否包含指定的value。
	- 源码如下：
	```java
    public boolean containsValue(Object value) {
            Node<K,V>[] tab; V v;
            if ((tab = table) != null && size > 0) {
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                        if ((v = e.value) == value || (value != null && value.equals(v)))
                            return true;
                    }
                }
            }
            return false;
    }
    ```

- `public Set<K> keySet()` 获取HashMap所有键值对中，key值的set集合。
	- 源码如下：
	```java
    public Set<K> keySet() {
            Set<K> ks;
            return (ks = keySet) == null ? (keySet = new KeySet()) : ks;
    }
    ```

- `public Collection<V> values()` 获取HashMap中所有value的集合。
	- 源码如下：
	```java
    public Collection<V> values() {
            Collection<V> vs;
            return (vs = values) == null ? (values = new Values()) : vs;
    }
    ```

- `public Set<Map.Entry<K,V>> entrySet()` 获取HashMap中所有的key-value键值对的set集合。
	- 源码如下：
	```
    public Set<Map.Entry<K,V>> entrySet() {
            Set<Map.Entry<K,V>> es;
            return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }
    ```

- `public V put(K key, V value)` 向HashMap中添加数据。
	- 源码如下：
	```java
    public V put(K key, V value) {
            return putVal(hash(key), key, value, false, true);
    }
    ```

- `public void putAll(Map<? extends K, ? extends V> m)` 将map中所有的键值对放入到HashMap中。
	- 源码如下：
	```java
    public void putAll(Map<? extends K, ? extends V> m) {
            putMapEntries(m, true);
    }
    ```

- `public V remove(Object key)` 从HashMap中移除键值为key的key-value键值对
	- 源码如下：
	```java
    public V remove(Object key) {
            Node<K,V> e;
            return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
    }
    ```

- `public void clear()` 删除HashMap中所有的键值对。
	- 源码如下：
	```java
    public void clear() {
            Node<K,V>[] tab;
            modCount++;
            if ((tab = table) != null && size > 0) {
                size = 0;
                for (int i = 0; i < tab.length; ++i)
                    tab[i] = null;
            }
    }
    ```

### HashMap的私有方法
- `final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict)` 将一个map集合中所有的元素都放入HashMap中，该方法仅在HashMap的clone()、HashMap()、putAll()被调用。
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

- `final Node<K,V>[] resize()` 扩容，当插入元素后，发现元素总个数大于threshold时，进行扩容。该方法在putMapEntry(),putVal(),treeifyBin(),computeIfAbsent(),compute(),merge()等方法中被调用。
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
    - 步骤
		- 根据旧哈希桶数组的大小和扩容阈值加倍获取到新的哈希桶数组大小和阈值。如果旧容量等于0，则使用旧阈值为新的哈希桶数组容量，如果旧阈值也小于等于0，则使用默认的哈希桶数组大小和阈值
		- 根据新的哈希桶数组容量创建新的哈希桶数组
		- 循环遍历旧的哈希桶数组，判断每一个元素是否为null
			- 如果为null，进入下一次循环
			- 如果不为null，判断其是否有下一个节点
				- 如果没有下一个节点，通过hash&(capacity-1)获取索引，然后放到新哈希桶对应的位置
				- 如果有下一个节点，则判断其是否为树节点
					- 如果为树节点，则调用HashMap.TreeNode中的split()方法将以该节点为根节点拆分并分别放到新的哈希桶数组的对应位置
					- 如果不为树节点，则为链表结构，将以该节点为头结点的链表拆分为两个子链表分别放入新的哈希桶数组的对应位置
	- 该方法主要包含两个部分，构建新的哈希桶数组，将旧哈希桶中的数据移动到新的哈希桶数组中
		- 构建新的哈希桶数组：获取新哈希桶的大小，新的扩容阈值
		- 移动数据:
			- 遍历旧哈希桶数组的每个哈希桶
			- 判断其是否没有下一个节点，如果没有下一个节点就根据其hash值和新的哈希桶数组的大小求其新的索引，将其放入
			- 如果哈希桶中的节点有下一个节点，就判断其是否为树节点，如果是树节点就将拆分红黑树，将所有节点放入新的哈希桶数组中
			- 如果哈希桶中的节点不为树节点，就将其对应的链表拆分为两部分，分别放入哈希桶数组中

- `final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)` 将key,value,hash放入HashMap中。该方法在putMapEntries(),put(),putIfAbsent(),readObject()中被调用。onlyIfAbsent表示如果key值已经存在了是否继续插入，evict为true表示不是在创建的情况下调用。
	- 源码如下：
	```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            // 如果哈希桶数组为null，则扩容，并获取扩容后的长度
            if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length;
            // 如果根据哈希找到在哈希桶数组的索引，如果为null，则直接新建node
            if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
            // 如果索引所在节点不为null
            else {
                Node<K,V> e; K k;
                // 如果传入的hash值和哈希桶中的哈希值相同，并且key值也相同，获取当前哈希桶中的节点引用
                if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                // 如果hash和key不相等，但哈希桶中的节点是树节点
                else if (p instanceof TreeNode)
                    // 调用HashMap.TreeNode的putTreeVal方法
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
    - putVal()基本上包含六个步骤
		- 判断哈希桶数组是否为空，如果为空则扩容
		- 根据hash计算在哈希桶数组中的位置，判断索引对应的哈希桶是否为null，如果为null，则直接创建新节点并插入
		- 判断索引对应的哈希桶key和hash是否等于插入节点的值，如果相等则将根据onlyIfAbsent决定是否将新值放入，返回旧的value
		- 如果索引对应的哈希桶中的节点是TreeNode就调用TreeNode的putTreeVal()方法插入节点newTreeNode
		- 如果不是树节点就遍历普通的Node链表，找到合适的位置插入，插入以后判断链表的长度，如果超过了树化阈值，就进行树化
		- 如果插入成功（不包含key值存在而没插入的情况），就判断此时HashMap的大小，如果超过了扩容阈值，就进行扩容

- ` final void treeifyBin(Node<K,V>[] tab, int hash)` 向链表中插入节点后，如果链表的大小超过了树化阈值，就调用此方法将链表转换为红黑树。该方法在putVal()，computeIfAbsent()，compute()，merge()中被调用。
	- 源码如下：
	```java
    final void treeifyBin(Node<K,V>[] tab, int hash) {
            int n, index; Node<K,V> e;
            if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
                resize();
            else if ((e = tab[index = (n - 1) & hash]) != null) {
                TreeNode<K,V> hd = null, tl = null;
                do {
                    TreeNode<K,V> p = replacementTreeNode(e, null);
                    if (tl == null)
                        hd = p;
                    else {
                        p.prev = tl;
                        tl.next = p;
                    }
                    tl = p;
                } while ((e = e.next) != null);
                if ((tab[index] = hd) != null)
                    hd.treeify(tab);
            }
    }
    ```
	- 该方法一共分为两个部分
		- 判断哈希桶数组是否为null，如果为null就扩容，判断hash在对应哈希桶中是否为null，如果为null直接结束
		- 将Node构成的单项链表转换为TreeNode构成的双向链表。如果双向链表的头结点不为null，就用头结点调用TreeNode的树化方法

- `void afterNodeAccess(Node<K,V> p) { }` 为LinkedHashMap准备的回调方法

- `void afterNodeInsertion(boolean evict) { }` 为LinkedHashMap准备的回调方法

- `void afterNodeRemoval(Node<K,V> p) { }` 为LinkedHashMap准备的回调方法

- `Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next)` 将TreeNode树节点转换为普通的Node节点。该方法仅在TreeNode的untreeify()方法中被调用。
	- 源码如下：
	```java
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        	return new Node<>(p.hash, p.key, p.value, next);
    }
    ```

- `Node<K,V> newNode(int hash, K key, V value, Node<K,V> next)` 创建一个非树节点。Node的构造方法见[HashMap中的内部类](http://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)。
	- 源码如下：
	```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        	return new Node<>(hash, key, value, next);
    }
    ```

- `final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)` 移除一个节点，matchValue表示只有在值相等的情况下才移除节点。movable表示移除节点后是否需要移动其他节点。
	- 源码如下：
	```java
    final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
            Node<K,V>[] tab; Node<K,V> p; int n, index;
            if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
                Node<K,V> node = null, e; K k; V v;
                if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                    node = p;
                else if ((e = p.next) != null) {
                    if (p instanceof TreeNode)
                        node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                    else {
                        do {
                            if (e.hash == hash &&
                                ((k = e.key) == key ||
                                 (key != null && key.equals(k)))) {
                                node = e;
                                break;
                            }
                            p = e;
                        } while ((e = e.next) != null);
                    }
                }
                if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
                    if (node instanceof TreeNode)
                        ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                    else if (node == p)
                        tab[index] = node.next;
                    else
                        p.next = node.next;
                    ++modCount;
                    --size;
                    afterNodeRemoval(node);
                    return node;
                }
            }
            return null;
    }
    ```
	- 此方法中分两部分，首先是找到节点，其次是移除节点。
		- 找到节点，需要在哈希桶中找，哈希桶中没找到就判断哈希桶是否有连接的节点，如果有连接的节点就判断是红黑树还是链表，如果是红黑树就用树的方法去找到节点，如果是链表就遍历链表去找节点，直到找到节点。
		- 移除节点，根据参数决定是否需要判断值相等。如果是树节点，就用树的方法去移除节点，如果是哈希桶中的节点就直接将下一个节点放到哈希桶中，如果是链表中的节点，就从链表中去掉它。

- `final Node<K,V> getNode(int hash, Object key)` 根据哈希值和key值去获取一个节点。
	- 源码如下：
	```java
    final Node<K,V> getNode(int hash, Object key) {
            Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (first = tab[(n - 1) & hash]) != null) {
                // always check first node
                if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
                    return first;
                if ((e = first.next) != null) {
                    if (first instanceof TreeNode)
                        return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            return e;
                    } while ((e = e.next) != null);
                }
            }
            return null;
    }
    ```
    - 该方法的思路分为四个部分:
		- 哈希桶是否为空，如果为空直接返回null
		- 哈希桶中的节点是否为需要的节点，如果是，就直接返回哈希桶中的节点
		- 哈希桶中的节点是树节点就用树节点的方式去遍历树，直到找到符合的key和hash
		- 哈希桶中的节点不为树节点，就通过普通Node的方式去遍历链表寻找节点

- `final float loadFactor() { return loadFactor; }` 获取负载因子。

- `final int capacity()`，获取容量
	- 源码如下：
	```java
    final int capacity() {
        	return (table != null) ? table.length : (threshold > 0) ? threshold : DEFAULT_INITIAL_CAPACITY;
    }
    ```

- `private void writeObject(java.io.ObjectOutputStream s)`
	- 源码如下：
	```java
    private void writeObject(java.io.ObjectOutputStream s) throws IOException {
            int buckets = capacity();
            // Write out the threshold, loadfactor, and any hidden stuff
            s.defaultWriteObject();
            s.writeInt(buckets);
            s.writeInt(size);
            internalWriteEntries(s);
    }
    ```

- `private void readObject(java.io.ObjectInputStream s)` 从流里面获取数据重建一个HashMap实例。
	- 源码如下：
	```java
    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
            // Read in the threshold (ignored), loadfactor, and any hidden stuff
            s.defaultReadObject();
            reinitialize();
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new InvalidObjectException("Illegal load factor: " +
                                                 loadFactor);
            s.readInt();                // Read and ignore number of buckets
            int mappings = s.readInt(); // Read number of mappings (size)
            if (mappings < 0)
                throw new InvalidObjectException("Illegal mappings count: " +
                                                 mappings);
            else if (mappings > 0) { // (if zero, use defaults)
                // Size the table using given load factor only if within
                // range of 0.25...4.0
                float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
                float fc = (float)mappings / lf + 1.0f;
                int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                           DEFAULT_INITIAL_CAPACITY :
                           (fc >= MAXIMUM_CAPACITY) ?
                           MAXIMUM_CAPACITY :
                           tableSizeFor((int)fc));
                float ft = (float)cap * lf;
                threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                             (int)ft : Integer.MAX_VALUE);
                @SuppressWarnings({"rawtypes","unchecked"})
                    Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
                table = tab;

                // Read the keys and values, and put the mappings in the HashMap
                for (int i = 0; i < mappings; i++) {
                    @SuppressWarnings("unchecked")
                        K key = (K) s.readObject();
                    @SuppressWarnings("unchecked")
                        V value = (V) s.readObject();
                    putVal(hash(key), key, value, false, false);
                }
            }
    }
    ```

- `TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next)` 新建一个TreeNode节点。
	- 源码如下：
	```java
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        	return new TreeNode<>(hash, key, value, next);
    }
    ```

- `TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next)` 将一个Node转换为TreeNode
	- 源码如下：
	```java
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        	return new TreeNode<>(p.hash, p.key, p.value, next);
    }
    ```

- `void reinitialize()` 重新初始化HashMap，该方法在clone(),readObject()方法中被调用。
	- 源码如下：
	```java
    void reinitialize() {
            table = null;
            entrySet = null;
            keySet = null;
            values = null;
            modCount = 0;
            threshold = 0;
            size = 0;
    }
    ```

- `void internalWriteEntries(java.io.ObjectOutputStream s)`
	- 源码如下：
	```java
    void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
            Node<K,V>[] tab;
            if (size > 0 && (tab = table) != null) {
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                        s.writeObject(e.key);
                        s.writeObject(e.value);
                    }
                }
            }
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
    - 具体的原因在于capacity总是2的幂次方，例如8，16，32之类的，它们-1之后，的二进制表示分别是111,1111,11111，任意一个数字同这些数字进行&操作，都是求模运算。对于任意数字：①比capacity多的位，不管是什么，capacity都填充为0，因此&操作总能得出0 ②capacity具备的位数，不管你是什么，&操作之后，你该是什么还是什么，以此达到求模运算的目的。以1111 & 1010100110为例:
    ```
    	  0000001111
        & 1010100110
          0000000110
        即 0000001111 & 1010100110 = 0000000110 => 678 % 16 = 6
    ```
    - 在上一步骤求出一堆模数相同的数字，8,16,32的二进制是1000,10000,10000，它有且仅有一个数字为1，其他位数皆为0。对于任意数字，对上那些为0的位数的时候，不管你是什么，&总会返回0，而对上那个为1的位数，不管你是什么，&总会得出你本来的数字。那些模数相同的数字，它们因此会被区分为两部分。分别以1111 & 1010100110和1111 & 1010110110为例:
    ```
    	  0000001111
        & 1010100110
          0000000110
        ____________
    	  0000001111
        & 1010110110
          0000000110
    ```
    1010100110(678)和1010110110(694)对(16-1)求模得到同样的余数，但是如果使用16对这两个数字求模，情况就不挑一样了。
    ```
    	  0000010000
        & 1010100110
          0000000000
        ____________
    	  0000010000
        & 1010110110
          0000010000
    ```
    如上所示，在求模得到同样余数的情况下，对2的幂次方进行&操作，依然可以得出不同的结果，这有赖于第二个数(694)在第一个数(678)的基础上增加了16。在JDK8中就是利用这种原理进行扩容时的操作：对同一个哈希桶中的链表或红黑树进行拆分，得出两个子链表(loHead,hiHead)，这两个子链表分别放入原先的哈希桶中(loHead)以及原先的哈希桶索引+旧的哈希桶数组大小的哈希桶(hiHead)中。