---

layout: post

title: Map框架分析（三）AbstractMap抽象类分析

date: 2018-6-3

tags: Java基础

---

本文基于JDK 1.8。

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

### AbstractMap的内部类
- AbstractMap一共有两个内部类，分别是SimpleEntry和SimpleImmutableEntry。尽管SimpleEntry和SimpleImmutableEntry共享了一些代码，但是它们两个是不相关的无关类，由于子类中无法增加或者修改final性质的字段，所以它们无法共享表示，并且由于重复的代码量太少而无法提取抽象类。
	- SimpleEntry，一个Entry支持key-value，Entry的值可以使用setValue方法进行修改。这个类可以帮助构建Map的实现类，实现一些个性化的功能
	- SimpleImmutableEntry，一个维护了不可变key和value的Entry。这个类不支持方法setValue。这个类可以很方便的返回含key-value映射的线程安全的快照。

- SimpleEntry源码及注释说明如下：
```java
public static class SimpleEntry<K, V> implements Map.Entry<K, V>, java.io.Serializable {
        private static final long serialVersionUID = -8499721149061103585L;
        // 注意，key是final的
        private final K key;
        private V value;
        //  提供了两种类型的构造函数
        public SimpleEntry(K key, V value) {
            this.key = key;
            this.value = value;
        }
        public SimpleEntry(Map.Entry<? extends K, ? extends V> entry) {
            this.key = entry.getKey();
            this.value = entry.getValue();
        }
        // 获取key
        public K getKey() {
            return key;
        }
        // 获取value
        public V getValue() {
            return value;
        }
        // 设置value，并返回之前的值
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }
        // 判断与另一个对象是否相等
        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
            return eq(key, e.getKey()) && eq(value, e.getValue());
        }
        // 获取hashcode
        public int hashCode() {
            return (key == null ? 0 : key.hashCode()) ^
                    (value == null ? 0 : value.hashCode());
        }
        // 重写了toString()方法
        public String toString() {
            return key + "=" + value;
        }
    }
```

- SimpleImmutableEntry源码及注释说明如下：
```java
public static class SimpleImmutableEntry<K,V> implements Map.Entry<K,V>, java.io.Serializable {
        private static final long serialVersionUID = 7138329143949025153L;
        // 与SimpleEntry不同的是，SimpleImmutableEntry的value也是不可变的
        private final K key;
        private final V value;
        // 同样提供了两种构造函数
        public SimpleImmutableEntry(K key, V value) {
            this.key   = key;
            this.value = value;
        }
        public SimpleImmutableEntry(Map.Entry<? extends K, ? extends V> entry) {
            this.key   = entry.getKey();
            this.value = entry.getValue();
        }
        // 返回这个Entry的key
        public K getKey() {
            return key;
        }
        // 返回这个Entry的value
        public V getValue() {
            return value;
        }
        // 当调用这个类的setValue时，会抛出UnsupportedOperationException异常
        public V setValue(V value) {
            throw new UnsupportedOperationException();
        }
        // 判断该类与另一个对象是否相等
        // 这里面调用了AbstractMap中的eq静态工具方法
        // return o1 == null ? o2 == null : o1.equals(o2);
        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            return eq(key, e.getKey()) && eq(value, e.getValue());
        }
        // 返回Entry对应的hashcode
        public int hashCode() {
            return (key   == null ? 0 :   key.hashCode()) ^
                    (value == null ? 0 : value.hashCode());
        }
        // 重写了toString()方法
        public String toString() {
            return key + "=" + value;
        }
    }
```

### 方法和属性
- AbstractMap的源码和注释如下：
```java
    public abstract class AbstractMap<K, V> implements Map<K, V> {
        // 唯一的构造方法， 子类调用父类的构造方法通常是隐式的
        protected AbstractMap() {
        }

        // 查询操作

        // 返回此Map有多少个Entry
        public int size() {
            return entrySet().size();
        }
        // 判断该Map是否为空
        public boolean isEmpty() {
            return size() == 0;
        }
        // 判断此类是否包含指定的value
        public boolean containsValue(Object value) {
            Iterator<Entry<K, V>> i = entrySet().iterator();
            if (value == null) {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (e.getValue() == null)
                        return true;
                }
            } else {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (value.equals(e.getValue()))
                        return true;
                }
            }
            return false;
        }
        // 判断此类是否包含指定的key
        public boolean containsKey(Object key) {
            Iterator<Map.Entry<K, V>> i = entrySet().iterator();
            if (key == null) {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (e.getKey() == null)
                        return true;
                }
            } else {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (key.equals(e.getKey()))
                        return true;
                }
            }
            return false;
        }
        // 获取指定key对应的value，如果没有返回null
        public V get(Object key) {
            Iterator<Entry<K, V>> i = entrySet().iterator();
            if (key == null) {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (e.getKey() == null)
                        return e.getValue();
                }
            } else {
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (key.equals(e.getKey()))
                        return e.getValue();
                }
            }
            return null;
        }

        // 修改操作

        // 设置key-value键值对，抛出UnsupportedOperationException
        public V put(K key, V value) {
            throw new UnsupportedOperationException();
        }
        // 移出指定key
        public V remove(Object key) {
            Iterator<Entry<K, V>> i = entrySet().iterator();
            Entry<K, V> correctEntry = null;
            if (key == null) {
                while (correctEntry == null && i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (e.getKey() == null)
                        correctEntry = e;
                }
            } else {
                while (correctEntry == null && i.hasNext()) {
                    Entry<K, V> e = i.next();
                    if (key.equals(e.getKey()))
                        correctEntry = e;
                }
            }
            V oldValue = null;
            if (correctEntry != null) {
                oldValue = correctEntry.getValue();
                i.remove();
            }
            return oldValue;
        }

        // 批量操作

        // 将指定Map中所有元素放入到此Map中
        public void putAll(Map<? extends K, ? extends V> m) {
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
                put(e.getKey(), e.getValue());
        }
        // 清空当前Map中的所有Entry
        public void clear() {
            entrySet().clear();
        }
        // key的集合，注意，该集合是volatile的
        transient volatile Set<K> keySet;
        // value的集合，注意，该集合是volatile的
        transient volatile Collection<V> values;
        // 获取key的集合
        public Set<K> keySet() {
            if (keySet == null) {
                keySet = new AbstractSet<K>() {
                    public Iterator<K> iterator() {
                        return new Iterator<K>() {
                            private Iterator<Entry<K, V>> i = entrySet().iterator();
                            public boolean hasNext() {
                                return i.hasNext();
                            }
                            public K next() {
                                return i.next().getKey();
                            }
                            public void remove() {
                                i.remove();
                            }
                        };
                    }
                    public int size() {
                        return AbstractMap.this.size();
                    }
                    public boolean isEmpty() {
                        return AbstractMap.this.isEmpty();
                    }
                    public void clear() {
                        AbstractMap.this.clear();
                    }
                    public boolean contains(Object k) {
                        return AbstractMap.this.containsKey(k);
                    }
                };
            }
            return keySet;
        }
        // 返回所有值的集合
        public Collection<V> values() {
            if (values == null) {
                values = new AbstractCollection<V>() {
                    public Iterator<V> iterator() {
                        return new Iterator<V>() {
                            private Iterator<Entry<K, V>> i = entrySet().iterator();

                            public boolean hasNext() {
                                return i.hasNext();
                            }

                            public V next() {
                                return i.next().getValue();
                            }

                            public void remove() {
                                i.remove();
                            }
                        };
                    }

                    public int size() {
                        return AbstractMap.this.size();
                    }

                    public boolean isEmpty() {
                        return AbstractMap.this.isEmpty();
                    }

                    public void clear() {
                        AbstractMap.this.clear();
                    }

                    public boolean contains(Object v) {
                        return AbstractMap.this.containsValue(v);
                    }
                };
            }
            return values;
        }
        // 获取Entry的集合
        public abstract Set<Entry<K, V>> entrySet();

        // 比较和hashcode

        // 判断是否相等
        public boolean equals(Object o) {
            if (o == this)
                return true;
            if (!(o instanceof Map))
                return false;
            Map<?, ?> m = (Map<?, ?>) o;
            if (m.size() != size())
                return false;
            try {
                Iterator<Entry<K, V>> i = entrySet().iterator();
                while (i.hasNext()) {
                    Entry<K, V> e = i.next();
                    K key = e.getKey();
                    V value = e.getValue();
                    if (value == null) {
                        if (!(m.get(key) == null && m.containsKey(key)))
                            return false;
                    } else {
                        if (!value.equals(m.get(key)))
                            return false;
                    }
                }
            } catch (ClassCastException unused) {
                return false;
            } catch (NullPointerException unused) {
                return false;
            }
            return true;
        }
        // 获取hashcode
        public int hashCode() {
            int h = 0;
            Iterator<Entry<K, V>> i = entrySet().iterator();
            while (i.hasNext())
                h += i.next().hashCode();
            return h;
        }
        // 重写了toString方法
        public String toString() {
            Iterator<Entry<K, V>> i = entrySet().iterator();
            if (!i.hasNext())
                return "{}";
            StringBuilder sb = new StringBuilder();
            sb.append('{');
            for (; ; ) {
                Entry<K, V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                sb.append(key == this ? "(this Map)" : key);
                sb.append('=');
                sb.append(value == this ? "(this Map)" : value);
                if (!i.hasNext())
                    return sb.append('}').toString();
                sb.append(',').append(' ');
            }
        }
        // 重写了clone方法，实际上还是用的底层native方法去完成，clone出来的Map它的key集合和value集合都是空
        protected Object clone() throws CloneNotSupportedException {
            AbstractMap<?, ?> result = (AbstractMap<?, ?>) super.clone();
            result.keySet = null;
            result.values = null;
            return result;
        }
        // 一个私有的工具类方法，判断两个对象是否相等，提供给内部类使用
        private static boolean eq(Object o1, Object o2) {
            return o1 == null ? o2 == null : o1.equals(o2);
        }
    }
```

- AbstractMap中的方法大部分和Map接口一样，只是没有Map接口中的默认实现方法，因此被分为四类
	- 查询操作
		- `public int size()` 返回此Map有多少个Entry
        - `public boolean isEmpty()` 判断该Map是否为空
        - `public boolean containsValue(Object value)` 判断此类是否包含指定的value
        - `public boolean containsKey(Object key)` 判断此类是否包含指定的key
        - `public V get(Object key)` 获取指定key对应的value，如果没有返回null
	- 修改操作
        - `public V put(K key, V value)` 设置key-value键值对，抛出UnsupportedOperationException
        - `public V remove(Object key)` 移除指定key
	- 批量操作
        - `public void putAll(Map<? extends K, ? extends V> m)` 将指定Map中所有元素放入到此Map中
        - `public void clear()` 清空当前Map中的所有Entry
        - `public Set<K> keySet()` 获取key的集合，该集合中包含了自定义的迭代器
        - ` public Collection<V> values()` 返回所有值的集合，该集合中包含了自定义的迭代器
        - `public abstract Set<Entry<K, V>> entrySet()` 获取Entry的集合

	- 重写Object对象的方法
		- `public boolean equals(Object o) ` 判断是否相等
		- `public int hashCode()` 获取hashcode
		- `public String toString()` 重写了toString方法
		- `protected Object clone()` 重写了clone方法，实际上还是用的底层native方法去完成，clone出来的Map它的key集合和value集合都是空
		- `private static boolean eq(Object o1, Object o2)` 一个私有的工具类方法，判断两个对象是否相等，提供给内部类使用

- AbstractMap中几乎所有类都来自Map接口，AbstractMap给大部分方法提供了一种默认的实现方式，但也有少部分类没有给实现，而是被声明为abstract，需要实现类自己去定制
