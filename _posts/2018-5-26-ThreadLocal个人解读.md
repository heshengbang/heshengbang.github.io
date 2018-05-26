---
layout: post
title: ThreadLocal个人解读
date: 2018-5-26
tags: Java基础
---

### ThreadLocal的原理
- 对ThreadLocal的理解
	- 一个例子：
	```java
    class ConnectionManager {
    private static Connection connect = null;
    public static Connection openConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
    public static void closeConnection() {
        if(connect!=null)
            connect.close();
    }
    ```
    ①假设有这样一个数据库链接管理类，这段代码在单线程中使用是没有任何问题的，但是如果在多线程中使用就会出现线程安全问题。
    	- 这里面的2个方法都没有进行同步，很可能在openConnection方法中会多次创建connect
    	- 由于connect是共享变量，那么必然在调用connect的地方需要使用到同步来保障线程安全，因为很可能一个线程在使用connect进行数据库操作，而另外一个线程调用closeConnection关闭链接

    综上，必须将这段代码的两个方法进行同步处理，并且在调用connect的地方需要进行同步处理。
    ②加上同步处理以后，这样将会大大影响程序执行效率，因为一个线程在使用connect进行数据库操作的时候，其他线程只有等待。
    ③事实上，假如每个线程中都有一个connect变量，各个线程之间对connect变量的访问实际上是没有依赖关系的，即一个线程不需要关心其他线程是否对这个connect进行了修改的。
    	- 事实上，之所以没选择懒汉模式，是因为创建创建链接非常消耗性能，给服务器带来非常巨大的压力。因此选择饿汉模式，并且一次创建以后一直使用。

    ④这种情况下使用ThreadLocal是再适合不过的了，因为ThreadLocal在每个线程中对该变量会创建一个副本，即每个线程内部都会有一个该变量，且在线程内部任何地方都可以使用，线程之间互不影响，这样一来就不存在线程安全问题，也不会严重影响程序执行性能。

	- jdk中关于这个类的注释说明
>This class provides thread-local variables.  These variables differ from their normal counterparts in that each thread that accesses one (via its {@code get} or {@code set} method) has its own, independently initialized copy of the variable.  {@code ThreadLocal} instances are typically private static fields in classes that wish to associate state with a thread (e.g.,a user ID or Transaction ID).

	- <b>中文说明</b>：ThreadLocal类用来提供线程内部的局部变量。这种变量在多线程环境下访问(通过`get()`或`set()`方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量。ThreadLocal实例通常来说都是private static类型的，用于线程关联某一个状态，例如与（用户id和事务id）相关联。

	- 总的来说，ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。

- ThreadLocal最常见的操作就是set()、get()、remove()三个动作
	- set()，将当前线程的线程局部变量的副本设置为指定的值。大多数子类都不需要重写此方法，仅依靠{@link #initialValue}方法设置线程本地变量值。
	```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ```
	- get()，返回在当前线程的线程本地变量中的一个值拷贝。如果在当前线程中变量没有值，则首先调用{@link #initialValue}方法将其初始化后返回。
    ```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ```
    - remove()，删除当前线程的线程本地变量中的某个值。如果线程本地变量的值后面由当前线程通过调用{@linkplain #get}读取，则通过调用其{@link #initialValue}方法重新初始化其值，除非其值在过渡期间通过{@linkplain #set}重新设置。这可能会导致在当前线程中多次调用initialValue方法。
    ```
    public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
    ```
### ThreadLocalMap的源码
```java
static class ThreadLocalMap {
		// Entry:代表 ThreadLocalMap 中的项，作用和HashMap的一致
        //继承WeakReference，使用 ThreadLocal作为ref field引用
        //如果entry.get() == null 可以认定ThreadLocal 没有引用了
        //也意味着 entry可以从table中 擦出了。
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** 与此ThreadLocal相关联的值 */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //初始容量 - 必须是二的幂
        private static final int INITIAL_CAPACITY = 16;
        //该表根据需要调整大小，但table.length必须始终为2的幂。
        private Entry[] table;
        //table中的数据条目数量
        private int size = 0;
        //下一个扩容的临界点，默认是0
        private int threshold;
        //设置调整阈值的大小，以保持最坏的2/3负载系数
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
        //增加i的模数长度
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
        //减少i的模数长度
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        //构建一个包含初始（firstKey，firstValue）的新Map
        //ThreadLocalMaps是懒加载。当至少有一个条目放入时，只创建一个。
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
        //构造一个包含给定父映射的所有可继承ThreadLocals的新映射。
        //仅由createInheritedMap调用。
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
        //获取key 对应的Entry值，也是ThreadLocal.get() 方法的实现
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
        //用来找未能直接定位的值
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
        //对key 设置 value 值
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                //如果找到了，直接设置 value 值即可
                if (k == key) {
                    e.value = value;
                    return;
                }
                //可以认定 k 已经没有引用了
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            //最后的选择就是新增Entry项
            tab[i] = new Entry(key, value);
            int sz = ++size;
            //如果清理过期的slot失败了
            //并且sz>= threadhold，进行rehash
            //注意：是新增后检测，如果没有多余的了，那么就先rehash
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        //根据key删除一项Entry
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                 //如果找到
                if (e.get() == key) {
                	//清理
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        //替换一个过期的entry项
        //在对特定key操作期间 ，value 会保存在entry中，对key 而言，无论entry 是否存在
        //副作用，是会擦出所以过期的entry项
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;
            //待过期的slot位置
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                 // 向前找，知道找到e.get() == null的为止
                if (e.get() == null)
                    slotToExpunge = i;
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == key) {
                    e.value = value;
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
        //擦除一个过期的entry
        //通过rehash在staleSlot 和next slot之间任何可能冲突的entrys
        //同样也会擦出任何其他 在找到下一个null entry知道遇到的 过期的entry
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            //擦除在staleSlot位置的value
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            //数量减一
            size--;
            // rehash直到遇到下一个null的值
            Entry e;
            int i;
            //找到staleSlot 对应的（可能是）同hash值的 下一个位置
            for (i = nextIndex(staleSlot, len);
            	//不为null就继续，知道遇到null终止
                 (e = tab[i]) != null;
                 //从i开始下一个元素
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                //如果k = null，说明当前 e 对应的 ThreadLocal 已经没有引用
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                	//找到位置k.threadLocalHashCode 理论应该在的位置
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                    	//如果h ！=i ，那么需要把当前元素放到h应该在的位置上去！
                        //如果h == i，那么可以认定不用动了，e在它应该在的位置了！
                        tab[i] = null;
                        //从h开始一直向下走，直到寻找到null
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        //把e放在table[h]的位置或者h散列后的第一个null位置
                        tab[h] = e;
                    }
                }
            }
            //第一个不为null的位置
            return i;
        }
        //清空一些过期的数据。考虑到速度和垃圾回收的效率，做>>>1的一个折中方案
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
            	//下一个位置
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                	//数组的长度
                    n = len;
                    removed = true;
                    //清除在i位置过期的entry项
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
        //rehash：先扫描过期的进行清除，清除后还不够就resize() 数据
        private void rehash() {
        	 //先清除过期的值
            expungeStaleEntries();
            // 使用较低的阈值加倍以避免滞后
            if (size >= threshold - threshold / 4)
                resize();
        }
        //容量加倍
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;
            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }
            setThreshold(newLen);
            size = count;
            table = newTab;
        }
        //清除table中的所有陈旧条目
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
    }
```

### ThreadLocal的使用
- 就如开篇举例的那样，ThreadLocal的常见应用场景是解决 数据库连接、Session管理等
- 在`mysql-connector-java`的SessionAssociationInterceptor类中，有一个ThreadLocal的应用：
```java
public class SessionAssociationInterceptor implements StatementInterceptor {
    ...
    protected final static ThreadLocal<String> sessionLocal = new ThreadLocal<String>();
    public static final void setSessionKey(String key) {
        sessionLocal.set(key);
    }
    public static final void resetSessionKey() {
        sessionLocal.set(null);
    }
    public static final String getSessionKey() {
        return sessionLocal.get();
    }
	...
}
```