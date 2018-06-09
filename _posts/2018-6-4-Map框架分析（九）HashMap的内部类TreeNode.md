---

layout: post

title: Map框架分析（九）HashMap的内部类TreeNode

date: 2018-6-4

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
		- [HashMap中的方法](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法/)
            - [HashMap的put方法](http://www.heshengbang.tech/2018/06/Map框架分析-六-HashMap的put方法/)
            - [HashMap的resize方法](http://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](http://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)


#### 简述
- TreeNode的继承关系有点曲线救国的味道。因为HashMap.TreeNode继承自LinkedHashMap.Entry，而LinkedHashMap.Entry继承自HashMap.Node，所以在[HashMap中的内部类](http://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)的简述中我直接将TreeNode归类于Node的子类。TreeeNode之所以选择这样绕的继承关系，是为了方便HashMap.TreeNode以后可以用作普通的链表节点或树形结构的节点扩展
- TreeNode是一个将红黑树的概念融合到类设计中的类，虽然它只是作为HashMap的内部类存在，但是它的内容依然充实。

### TreeNode的成员变量
- TreeNode的成员变量分为三大类，分别是继承自Node、继承自LinkedHashMap.Entry、自己独有的。如下所示：
![TreeNode的成员变量](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/HashMap源码分析/TreeNode的fields.jpg)

- 继承自HashMap.Node的成员变量
	- hash，保存当前节点的哈希值
	- key，保存当前节点的key
	- value，保存当前节点的value
	- next，保存当前节点的下一个节点

- 继承自LinkedHashMap的成员变量
	- before，保存双向链表中当前节点的前一个节点
	- after，保存双向链表中当前节点的后一个节点

- TreeNode自有的成员变量
	- parent，保存红黑树的父节点，在作为红黑树的时候需要赋值
	- left，保存红黑树的左子节点
	- right，保存红黑树的右子节点
	- prev，当前节点的前一个节点，在节点被删除时需要解除连接

### TreeNode的方法详解
- TreeNode中频繁使用到了两个HashMap的工具方法`comparableClassFor`和`compareComparables`。如果打算完整透彻的理解TreeNode，跳过这两个方法几乎是不可能的。因此，在分析TreeNode的方法前，先将这两个方法的逻辑捋一遍。
	- `static Class<?> comparableClassFor(Object x)`
		- 作用：如果参数x实现了Comparable接口，则返回x的类型，否则返回null
        - 源码如下：
        ```java
        static Class<?> comparableClassFor(Object x) {
            if (x instanceof Comparable) {
                Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
                // bypass checks
                if ((c = x.getClass()) == String.class)
                    return c;
                if ((ts = c.getGenericInterfaces()) != null) {
                    for (int i = 0; i < ts.length; ++i) {
                        if (((t = ts[i]) instanceof ParameterizedType) &&
                            ((p = (ParameterizedType)t).getRawType() ==
                             Comparable.class) &&
                            (as = p.getActualTypeArguments()) != null &&
                            // type arg is c
                            as.length == 1 && as[0] == c)
                            return c;
                    }
                }
            }
            return null;
        }
        ```
        - 涉及到的基本知识点
        	- ParameterizedType：继承自Type，代表一个提供泛型支持的类型，例如:java.util.Collection<E>
        		- ParameterizedType#getRawType：获取实际声明的类型对象
        		- ParameterizedType#getActualTypeArguments：获取实际作为泛型参数的类型，返回值是一个数组对象
        	- Class#getGenericInterfaces：获取该类直接实现的接口
    	- 基本逻辑：
    		1. 判断参数x是否实现了Comparable接口，如果没有，则直接返回null
    		2. 在x实现了Comparable接口的情况下，继续判断下一步骤
    		3. 参数x是否是String类型的，如果是，则直接返回String.class
    		4. 在参数x的类型p不是String类型的情况下，继续判断下一步骤
    		5. 如果p直接实现的接口数组ts为空，则直接返回null
    		6. 在ts不为空的情况下，继续下一步骤
    		7. 循环遍历ts，从t开始
    			- 如果t是一个参数化类型的实例，同时t的实际声明类型是Comparable.class，同时t的泛型参数的个数为1，并且泛型参数就是x的类型c，则返回类型c
    		8. 在以上方式都没找到的情况下，则返回null
    	- 简单点来说，这个方法就是用来判断传入的对象是否为String类型或者实现了Comparable接口的类的实例

	- `static int compareComparables(Class<?> kc, Object k, Object x)`
		- 作用：比较两个对象的大小
		- 源码如下：
		```java
        static int compareComparables(Class<?> kc, Object k, Object x) {
            return (x == null || x.getClass() != kc ? 0 :
                    ((Comparable)k).compareTo(x));
        }
        ```
        - 基本逻辑：
        	1. 如果参数x为null，则返回0（这并不表示两个对象相等）
        	2. 如果x不为null，但是x的类型不等于传入的参数kc，则返回0（这并不表示两个对象相等）
        	3. 如果x不为null，同时x的类型等于传入的参数kc，则用类型kc中实现的Comparable接口方法比较对象的大小，并返回结果
        	4. 注意，此方法要和comparableClassFor()方法配合使用，或者至少保证参数k的类型实现了Comparable接口，否则会报强转失败

- `final TreeNode<K,V> root()` 返回包含当前节点的树形结构的根节点。基本思路是从当前节点的父节点一直往上拿，直到父节点为空为止。
	- 源码如下：
	```java
    final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
    ```

- `static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root)` 保证给定根节点一定在树节点构成的双向链表结构的最前端。该方法进在HashMap.TreeNode中的treeify(),putTreeVal(),removeTreeNode()中被调用。
	- 源码如下：
	```java
    static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
            int n;
            if (root != null && tab != null && (n = tab.length) > 0) {
                int index = (n - 1) & root.hash;
                TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
                if (root != first) {
                    Node<K,V> rn;
                    tab[index] = root;
                    TreeNode<K,V> rp = root.prev;
                    if ((rn = root.next) != null)
                        ((TreeNode<K,V>)rn).prev = rp;
                    if (rp != null)
                        rp.next = rn;
                    if (first != null)
                        first.prev = root;
                    root.next = first;
                    root.prev = null;
                }
                assert checkInvariants(root);
            }
        }
    ```
    - 基本思路是：
    	- 找到节点所在的哈希桶的位置，判断该哈希桶中的节点和根节点是否为同一个节点，如果相同则直接结束。
    	- 哈希桶中节点和根节点不同时，就将根节点从之前的双向链表中取出（将根节点的前一个节点的下一个节点指向根节点的下一个节点，将根节点的下一个节点的前节点指向根节点的前一个节点）
    	- 将根节点放入哈希桶中，并将根节点的下一个节点指向之前在哈希桶中的节点，并将此节点的前一个节点指向根节点，将根节点的前一个节点指向null。
	- 代码逻辑如下：
		1. 判断给定节点和哈希桶数组均不为空，同时哈系桶数组的大小大于0，否则直接结束
		2. 通过哈系桶数组的大小和给定节点的hash值做位与操作，获取给定节点在哈系桶数组中的位置，然后获取到该位置当前存放的节点first
		3. 判断给定节点和first是否相同，如果相同则直接结束
		4. 将给定节点放到步骤2中确定的哈希桶数组位置中
		5. 获取给定节点的上一个节点rp
		6. 获取给定节点的下一个节点rn
		7. 如果rn不为空，则将rn的上一个节点置为rp
		8. 如果rp不为空，则将rp的下一个节点置为rn
		9. 如果firs不为空，则将first的前一个节点置为给定节点
		10. 将给定节点的前一个节点置为null，后一个节点置为first

- `final TreeNode<K,V> find(int h, Object k, Class<?> kc)` 使用给定的哈希值和key及类型从当前点开始找符合指定哈希值和key以及key的类型的节点。该方法在getTreeNode(),putTreeVal()以及其自身中被调用。
	- 源码如下：
    ```java
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
                TreeNode<K,V> p = this;
                do {
                    int ph, dir; K pk;
                    TreeNode<K,V> pl = p.left, pr = p.right, q;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null || (kc = comparableClassFor(k)) != null) && (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.find(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
                return null;
            }
    ```
	- 步骤如下：
        1. 从当前节点开始遍历，将当前节点赋值给节点p
        2. 获取p节点的左子节点pl和右子节点pr
            - 如果p节点的哈希值ph大于给定哈希值h，则p节点被置为pl
            - 否则，如果p节点的哈希值ph小于给定哈希值h，则p节点被置为pr
            - 否则，如果p节点的(key==给定k）或者（给定k值不为null && k和p节点的key值equals)，则返回p节点
            - 否则，如果pl为空，则p节点被置为pr
            - 否则，如果pr为空，则p节点被指为pl
            - 否则，判断 `((kc != null || (kc = comparableClassFor(k)) != null) && (dir = compareComparables(kc, k, pk)) != 0)`
                - `kc = comparableClassFor(k)` 如果k实现了Comparable接口，就返回它的类型
                - `dir = compareComparables(kc, k, pk)` 如果pk不为空或者x的类型不是kc，就返回0，否则返回`(Comparable)k).compareTo(x)`的结果
                - 简述，在给定类型和p的key类型都不为空并且给定的key值还实现了Comparable接口的情况下判断两个当前p的key和给定key是否相等，如果不等p就被置为pl，如果相等p就等于pr
            - 否则如果，递归调用该方法在p的右子节点去寻找节点，如果找到的节点不为空就返回该节点
            - 否则，p被置为pl
        3. 如果p不为空，就继续执行步骤2

- `final TreeNode<K,V> getTreeNode(int h, Object k)` 如果当前节点不为根节点，就去寻找根节点。从根节点开始寻找节点，则从当前节点开始向下寻找符合指定哈希值和key的节点并返回。该方法在getNode()，removeNode()，computeIfAbsent(),compute()，merge()中被调用。
	- 源码如下：
	```java
    final TreeNode<K,V> getTreeNode(int h, Object k) {
            return ((parent != null) ? root() : this).find(h, k, null);
        }
    ```

- `static int tieBreakOrder(Object a, Object b)` 在hash值相等的情况下，key没有实现Comparable接口无法用compareTo()方法比较的时候，使用该方法来判定两个key的大小。该方法在treeify()，putTreeVal()中被调用。
    - 源码如下：
    ```java
    static int tieBreakOrder(Object a, Object b) {
                int d;
                if (a == null || b == null ||
                    (d = a.getClass().getName().
                     compareTo(b.getClass().getName())) == 0)
                    d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                         -1 : 1);
                return d;
            }
    ```
    - 步骤如下：
        1. 如果a为空，则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
        2. 如果a不为空，b为空，则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
        3. 如果a不为空，b也不为空，则判断a的类名和b的类名是否相等（字符串比较）splitsplit
            - 如果相等则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
            - 如果不相等，则返回a的类名和b的类名比较的结果

- `final void treeify(Node<K,V>[] tab)` 将此节点所在的链表改在为红黑树。该方法仅在HashMap的treeifyBin()、HashMap.TreeNode中的split()中被调用，源码如下：
```java
final void treeify(Node<K,V>[] tab) {
    	// 定义一个根节点
        TreeNode<K,V> root = null;
        // 定义节点x和next
        // 从当前节点开始遍历
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            // 保存下一个节点的引用
            next = (TreeNode<K,V>)x.next;
            // 将节点的左右子节点设为null，以此清除之前的连接
            x.left = x.right = null;
            // 遍历第一次，创建并保存根节点
            if (root == null) {
                // 根节点的父节点必然为null
                x.parent = null;
                // 红黑树的根节点必然为黑色
                x.red = false;
                // 将当前节点置为根节点
                root = x;
            } else {
                // 以根节点为基础构建左右子节点
                // 获取当前节点的key
                K k = x.key;
                // 获取当前节点的hash
                int h = x.hash;
                // 定义类型变量，用来保存key的类型
                Class<?> kc = null;
                // 以根节点为基础，遍历红黑树
                // 当待插入的节点的哈希值小于等于树节点的哈希值，转向左子树，当哈希值大于树节点的哈希值，转向右子树
                // 如果子树为空，则执行插入操作，如果不为空，则以子树为树节点，继续遍历比较
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    // 如果根节点的哈希值大于x的哈希值h则dir为-1
                    if ((ph = p.hash) > h)
                        dir = -1;
                    // 如果根节点的哈希值小于x的哈希值h，则dir为1
                    else if (ph < h)
                        dir = 1;
                    // 经过上面两步比较，这里哈希值一定是相等的。
                    // 判断待插入节点的key值类型和树节点的key值类型
                    // 如果key的类型不同，则根据类型大小来判断待插入节点和树节点的大小
                    // 如果类型相同则采用key的哈希值来进行比较
                    // 由于HashMap的key具有唯一性，是hash值保障的，因此这一步一定能比较出待插入节点与树节点的大小
                    else if ((kc == null && (kc = comparableClassFor(k)) == null) || (dir = compareComparables(kc, k, pk)) == 0)
                    	dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                    // 根据待插入节点和树节点的大小决定转向左子树还是右子树
                    // 子树为空则平衡插入，子树不为空则执行下一次遍历比较
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    	// 代码执行到这里的时候，已经确定待插入节点会被插入到子树中
                        // 将树节点作为待插入节点的父节点
                        x.parent = xp;
                        if (dir <= 0)
                        	// 待插入节点指定为树节点的左子节点
                            xp.left = x;
                        else
                        	// 待插入节点指定为树节点的右子节点
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        // 构建红黑树结束以后，将TreeNode的链表中红黑树的根节点移做链表头结点
        moveRootToFront(tab, root);
    }
```
	- 此方法的前置条件是，只能被哈希桶中的树节点调用，哈希桶中的树节点与同一桶中的其他节点拥有完整的链表结构。在满足前置条件的情况下遍历链表结构，遍历的代码中可分为两部分，首先构建根节点，其次从根节点开始搜索子树找到节点应该被插入的位置，插入节点并调整结构维持红黑树的性质。
	- 需要说明的是TreeNode实例调用treeify()构建红黑树，但是构建过程中并没有拆解TreeNode实例所在的链表结构。因此，完成红黑树构建以后，每个TreeNode实例实际存在两套结构中：链表、红黑树。这就是为什么TreeNode中也定义了prev成员变量，这可以和HashMap.Node中的next成员变量配合使用，使TreeNode的链表形成双向链表。

- `final Node<K,V> untreeify(HashMap<K,V> map)`，该方法将TreeNode类型节点的链表转换为普通Node链表，并返回普通Node链表的头结点。该方法仅在HashMap.TreeNode的removeTreeNode和split中被调用。
```java
final Node<K,V> untreeify(HashMap<K,V> map) {
			// hd被用来保存头结点的引用，t1用来保存链表遍历过程中的节点
            Node<K,V> hd = null, tl = null;
            // 遍历节点
            for (Node<K,V> q = this; q != null; q = q.next) {
            	// 将TreeNode转换为普通的Node节点
                Node<K,V> p = map.replacementNode(q, null);
                // 如果为遍历的第一次，则保存头结点其引用
                if (tl == null)
                    hd = p;
                else
                	// 将上一个节点的下一个节点指向当前这个节点
                    tl.next = p;
                tl = p;
            }
            // 返回链表的头结点
            return hd;
        }
```
	- 反树化的过程就是将TreeNode转化为Node，这一过程中通过TreeNode中的key、value、hash创建Node节点，丢失了TreeNode作为红黑树的父子节点，作为双向链表结构的前节点。经过反树化以后，TreeNode实例所在的哈希桶中，没有红黑树，也没有双向链表，只有一个Node做节点的单向链表。

- `final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v)`，红黑树版的putVal()方法。该方法在putVal()，computeIfAbsent(),compute(),merge()方法中被调用。
```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            // 如果当前节点的父节点不为空则寻找根节点，否则就以当前节点为根节点
            TreeNode<K,V> root = (parent != null) ? root() : this;
            // 遍历当前节点所在的树形结构
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                // 如果当前节点的哈希值大于待插入节点的哈希值
                if ((ph = p.hash) > h)
                    dir = -1;
                // 否则，如果当前节点的哈希值小于待插入节点的哈希值
                else if (ph < h)
                    dir = 1;
                // 否则，如果当前节点的哈希值等于带插入节点的哈希值
                // 并且当前节点的key和带插入节点的key相等
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                // 否则，哈希值相等但是key值不相等同时key的类型都没有实现Comparable接口
                else if ((kc == null && (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
					// 还没找到节点
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        // 从当前节点的左右两边子树去递归寻找，左边或者右边找到了符合条件的节点，则直接返回该节点
                        if (((ch = p.left) != null && (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null && (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    // 如果左右节点均无法找到，而key的类名相等，则通过key的哈希进行比较
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                // 根据上一步骤比较的结果进行判断，p等于其左子节点还是右子节点，并且其子节点为空，则进行插入
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                	// 获取p节点的下一个节点
                    Node<K,V> xpn = xp.next;
                    // 新建节点
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    // 将新建的节点和p节点相连
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                   	// 将新节点插入到红黑树中
                    // 在将节点插入到链表节点以后，调整节点数组的结构，将数组放到节点数组的最前面
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
```
	- 将一个节点插入到红黑树中，分为两大部分，首先找到节点在红黑树中的位置，然后插入并调整红黑树
	- 找到节点在红黑树中的位置：从根节点开始，比较哈希值大小，如果哈希值小于等于树节点则继续和树节点的左子节点比较，如果大于树节点则继续和右子节点比较，直到树节点的子树为null，无法进行比较时，就找到了节点在红黑树中的位置
	- 插入并调整红黑树：插入分为两部分，先根据key，value,hash创建树节点，然后将新创建的树节点插入到红黑树对应的位置，以及红黑树对应的双向链表中，插入以后执行平衡插入操作并将红黑树的头结点移到双向链表的头部。

- `final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable)`，从红黑树中删除一个节点，movable表示删除节点后是否移动其他节点来调整结构。该方法仅在removeNode()中被调用。
```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
            int n;
            // 如果哈希桶为空或者桶长度为0则直接结束
            if (tab == null || (n = tab.length) == 0)
                return;
			// 用节点的哈希值和哈希桶的大小做位与操作，找到该节点在哈希桶中的位置
            int index = (n - 1) & hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            // 如果待删除节点为哈希桶中的节点，则直接将待删除的节点的下一个节点放到哈希桶中，待删除节点处于游离状态
            if (pred == null)
                tab[index] = first = succ;
            else
            	// 将待删除节点的前节点的下一个节点指向待删除节点的下一个节点，待删除节点处于有利状态
                pred.next = succ;
            // 下一个节点和前节点之间建立关联
            if (succ != null)
                succ.prev = pred;
            // 如果当前节点的下一个节点为null，直接结束
            if (first == null)
                return;
            // 根节点的父节点不为空，就去寻找根节点
            if (root.parent != null)
                root = root.root();
            // 根节点为null或者根节点的右子节点为null或者根节点的左子节点为null，或者根节点的左子节点的左子节点为null
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                // 则从哈希桶的位置开始反树化，然后结束
                tab[index] = first.untreeify(map);  // too small
                return;
            }
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            // 如果待删除节点的左右子节点不为空，则调整结构对子节点重新着色
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                // find successor
                while ((sl = s.left) != null)
                    s = sl;
                // swap colors
                boolean c = s.red; s.red = p.red; p.red = c;
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                // p was s's direct parent
                if (s == pr) {
                    p.parent = s;
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s;
                if ((s.parent = pp) == null)
                    root = s;
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }
            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);
            // detach
            if (replacement == p) {
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
            if (movable)
                moveRootToFront(tab, r);
        }
```

- `final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit)`，该方法将HashMap哈希桶中索引为index的元素，及以其为根节点的红黑树拆分放到放到新的哈希桶tab中。该方法只在HashMap的实例方法resize()中被调用
```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // 将链表分为高位子链表，低位子链表，重新连接节点，并保存在原有链表中的顺序
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            // 遍历当前元素为根节点的红黑树。
            // TreeNode保存了两种结构，一种是链表，持有前后节点的引用
            // 一种是红黑树，持有左右字数以及父节点的引用。
            // 此处遍历是通过链表结构进行遍历。
            // 扩容以后，链表的节点将被分为两部分【这一部分的原理见[HashMap中的方法](http://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法/)扩展部分】
            // 就像上面的例子一样，扩容后，原本在一个链表的节点会被分为两部分
            // 一部分的索引不变，另一部分的索引变成了之前的位与结果+索引
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
            	// 保存当前元素的下一个节点
                next = (TreeNode<K,V>)e.next;
                // 将当前节点的下一个元素置为null，去除所有引用后，方便gc
                e.next = null;
                // 当元素是旧哈希桶的第一个元素时
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                // 当元素不是旧哈希桶第一个元素时
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }
            // 关于这里计算树化的问题。当代码执行到这里的时候lc+hc的数值是大于TREEIFY_THRESHOLD
            // 当lc或hc中任一个数字大于UNTREEIFY_THRESHOLD
            // 如果另一个子链表为空，则必然已经被树化
            // 如果另一个子链表不为空，则有可能需要背树化
            if (loHead != null) {
            	// 红黑树的节点数目小于反树化的阈值
                if (lc <= UNTREEIFY_THRESHOLD)
                	// 进行反树化操作，获得的链表，并放入新哈希桶数组对应的位置
                    tab[index] = loHead.untreeify(map);
                else {
                	// 低位子链表直接放入新的哈希桶数组对应的位置
                    tab[index] = loHead;
                    if (hiHead != null)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
            	// 红黑树的节点数目小于反树化的阈值
                if (hc <= UNTREEIFY_THRESHOLD)
                	// 进行反树化操作，获得的链表，并放入新哈希桶数组对应的位置
                    // 注意这里index+bit这个操作，跟上面分高低位子链表的操作之间的关联
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

- `static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root, TreeNode<K,V> p)`，红黑树方法，所有的调整都是为了适应CLR
```java
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root, TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            // 左旋的节点p不为null，并且有右子节点r，r不为null
            if (p != null && (r = p.right) != null) {
            	// 如果r有左子节点rl，则将其与p的右子节点相关联
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                // 将r的节点指向p的父节点，如果p的父节点为null，则以r为根节点并且将其置为黑色
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                // 如果p的父节点不为null，则将p的父节点指向r
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                // r的左子节点指向p
                r.left = p;
                // p的父节点指向r
                p.parent = r;
            }
            return root;
        }
```
	- 左旋的过程涉及到左旋的点p，p的父节点pp，p的右子节点r，r的左子节点rl。左旋的目的断开旧连接（[将p与父节点pp、右子节点r之间的关系断开],[r与父节点p、左子节点rl之间的关系断开]），建立新连接（[p与新父节点r,新右子节点rl建立互相关联],[r与新父节点pp，新左子节点p建立互相关联]）。
	- 步骤简述：
		- 如果待左旋的节点p为null或者左旋节点的右子节点r为null，则直接返回根节点
		- 待左旋节点p不为null，同时待左旋节点的右子节点r不为null
			- p的右子节点指向r的左子节点rl，rl如果不为null则将其父节点指向p，至此p和rl互相关联
			- r的父节点指向p的父节点pp，判断pp是否为null
				- 如果pp为null，则将根节点指向r，并重新对r着色
				- 如果pp不为null，而p是pp的左子节点或右子节点，则将pp的左子节点或右子节点指向r，至此r和pp互相关联
			- r的左子节点指向p，p的父节点指向r，至此p和r互相关联上



- `static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root, TreeNode<K,V> p)`，右旋基本上和左旋保持对称一致的关系
```java
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root, TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
            	// p的左子节点指向lr
                if ((lr = p.left = l.right) != null)
                	// lr的父节点指向p，完成p与新左子节点lr的互相关联
                    lr.parent = p;
                // l的父节点指向pp
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                // pp的子节点指向l，完成l与新父节点pp的互相
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                // 完成l与新右子节点的互相
                l.right = p;
                p.parent = l;
            }
            return root;
        }
```
	- 右旋包括了四个角色，右旋节点p，p的父节点pp，p的左子节点l，l的右子节点lr。右旋的基本操作是断开旧关联，建立新关联
		- 断开旧关联
			- 断开，p与父节点pp的关联，p与左子节点l的关联
			- 断开，l与父节点p的关联，l与右子节点lr的关联
		- 建立新关联
			- 建立，l与新父节点pp的关联，l与新右子节点p的关联
			- 建立，p与新父节点r的关联，p与新左子节点lr的关联

	- 左旋和右旋归纳总结其实就是断开三个链接，重新建立三个链接，所谓三破三立。分别掌握左旋和右旋的基本结构，三破三立就非常方便完成。

- `static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x)`，HashMap.TreeNode中执行红黑树平衡插入的方法，在调用这个方法之前，待插入节点已经被放置在红黑树中，这个方法的目的在于调整x插入到红黑树后，其父子节点的颜色以保持x插入红黑树以后，红黑树依然能维持它的五个属性。仅在HashMap.TreeNode中的putTreeVal()方法，treeify()方法被调用。x为待插入的节点，root为待插入的红黑树的根节点。
```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
			// 插入的节点只能是红色。因为红黑树性质5要求根节点到任一叶子节点的黑色个数相等
            // 如果插入节点为黑色，则不能满足该性质。关于红黑色及其性质见本文拓展部分。
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
            	// 如果插入一颗空树，则可以直接插入并根据性质2对插入节点着色为黑，即可返回
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                // 如果待插入节点的父节点为黑色或者它的祖父节点为null，那就不必再做调整，直接返回根节点
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                // 如果待插入节点的父节是其父节点的左子节点
                // xp是带插入节点的父节点，xpp是待插入节点的父节点的父节点，xppl是带插入节点的父节点的父节点的左子节点
                // 下面这个判断的意思就是待插入节点的父节点，是待插入节点的祖父节点的左子节点【这块参照文章的拓展部分，关于红黑树的插入的三种情况】
                if (xp == (xppl = xpp.left)) {
                	// 父节点为祖父节点的左子节点，叔父节点为祖父节点的右子节点，叔父节点不为null并且叔父节点为红色
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        // 在对父节点、叔父节点、祖父节点重新着色以后
                        // 以祖父节点为根节点的子树中红黑树的五条性质是符合的
                        // 以祖父节点为新插入的节点进入下一次循环，调整，自下而上直至根节点，完成整颗红黑树的再次平衡
                        x = xpp;
                    }
                    // 叔父节点为黑色或者不存在
                    else {
                    	// 节点插入到父节点右子节点，形成左右结构
                        if (x == xp.right) {
                        	// 进行单左旋转
                            root = rotateLeft(root, x = xp);
                            // 左旋之后，为适应新的节点关系，重新赋值
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        // 到这里，无论插入节点是插入到父节点的左子节点还是右子节点，总会被调整为左左结构
                        // 父节点不为空，需要重新着色
                        if (xp != null) {
                        	// 父节点为黑色
                            xp.red = false;
                            // 祖父节点不为空
                            if (xpp != null) {
                            	// 祖父节点为红色
                                xpp.red = true;
                                // 右旋转
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                // 父节点是祖父节点的右子节点
                else {
                	// 叔父节点不为null，并且为红
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                    	// 插入的节点为左子节点
                        if (x == xp.left) {
                        	// 单右旋转
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                // 左旋转
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```
	- 红黑树的插入需要分条理去理解。首先，应当知道插入的节点必为红色，然后分情况插入：
		- 插入到一棵空树
		- 插入的父节点为黑色
		- 插入的父节点为红色，则父节点必不是根节点，不为根节点就有兄弟节点的可能性。
			- 叔父节点为红色
			- 叔父节点为黑色或不存在
				- 父节点为左子节点
					- 插入的节点为左子节点
					- 插入的节点为右子节点
				- 父节点为右子节点
					- 插入的节点为左子节点
					- 插入的节点为右子节点
	- 根据以上情况进行梳理，梳理每一种结果是否满足红黑树的五条性质，如果不满足就做相应的调整，如果满足就直接结束。红黑树的插入参照本文的扩展部分。

- `static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x)`
```java
        static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                                   TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                if (x == null || x == root)
                    return root;
                else if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (x.red) {
                    x.red = false;
                    return root;
                }
                else if ((xpl = xp.left) == x) {
                    if ((xpr = xp.right) != null && xpr.red) {
                        xpr.red = false;
                        xp.red = true;
                        root = rotateLeft(root, xp);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    if (xpr == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            xpr.red = true;
                            x = xp;
                        }
                        else {
                            if (sr == null || !sr.red) {
                                if (sl != null)
                                    sl.red = false;
                                xpr.red = true;
                                root = rotateRight(root, xpr);
                                xpr = (xp = x.parent) == null ?
                                    null : xp.right;
                            }
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red;
                                if ((sr = xpr.right) != null)
                                    sr.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateLeft(root, xp);
                            }
                            x = root;
                        }
                    }
                }
                else { // symmetric
                    if (xpl != null && xpl.red) {
                        xpl.red = false;
                        xp.red = true;
                        root = rotateRight(root, xp);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    if (xpl == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                        if ((sl == null || !sl.red) &&
                            (sr == null || !sr.red)) {
                            xpl.red = true;
                            x = xp;
                        }
                        else {
                            if (sl == null || !sl.red) {
                                if (sr != null)
                                    sr.red = false;
                                xpl.red = true;
                                root = rotateLeft(root, xpl);
                                xpl = (xp = x.parent) == null ?
                                    null : xp.left;
                            }
                            if (xpl != null) {
                                xpl.red = (xp == null) ? false : xp.red;
                                if ((sl = xpl.left) != null)
                                    sl.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateRight(root, xp);
                            }
                            x = root;
                        }
                    }
                }
            }
        }
```

- `static <K,V> boolean checkInvariants(TreeNode<K,V> t)`，递归检查树节点之间的关系（红黑树和双向链表）是否正确。该方法在moveRootToFront()以及他自身中被调用。
```java
       static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
            TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right, tb = t.prev, tn = (TreeNode<K,V>)t.next;
            // 检查节点t的前节点tb的下一个节点是否指向t
            if (tb != null && tb.next != t)
                return false;
            // 检查节点t的下一个节点tn的前一个节点是否指向t
            if (tn != null && tn.prev != t)
                return false;
            // 检查节点t的父节点tp的左右子节点是否均不为t
            if (tp != null && t != tp.left && t != tp.right)
                return false;
            // 检查t的左子节点tl的父节点是否不等于t或者tl的哈希值大于了t的哈希值
            if (tl != null && (tl.parent != t || tl.hash > t.hash))
                return false;
            // 检查t的右子节点tr的父节点是否不等于t或者tr的哈希值小于t的哈希值
            if (tr != null && (tr.parent != t || tr.hash < t.hash))
                return false;
            // 检查t的颜色为红同时它的子节点是否也是红，性质4
            if (t.red && tl != null && tl.red && tr != null && tr.red)
                return false;
            // 递归检查t的左子节点之间的关系是否也正确
            if (tl != null && !checkInvariants(tl))
                return false;
            // 递归检查t的右子节点之间的关系是否也正确
            if (tr != null && !checkInvariants(tr))
                return false;
            return true;
        }
```
	- 这一块，有一个冗余的检查。虽然moveRootToFront()使用的是assert断言来调用checkInvariants()的方法，但即便在调试阶段，依然有冗余检查。对于单个节点应该只检查自己和子节点以及前后节点的关系，检查父节点的关系和上一步的检查子关系存在重合的可能。

## 扩展

### 二叉搜索树
- 有序数组可以快速的寻找的一个值。但是如果想插入就得找到待插入的数据的位置，然后将其后的所有数据往后挪一个位置，再插入。同样如果打算删除也是如此，找到待删除的数据，删除后，再将其后的数据依次往前挪一个位置。因此，如果有频繁的插入删除操作，选用有序数组显然效率太低。
- 链表可以快速的插入数据，删除数据，只需要将其前后的节点重新指向即可，但是如果打算在链表中查找数据，则不得不依次遍历所有数据，必须从链表头开始访问每一个数据项，直到找到数据为止。
- 树形结构是数组和链表的一个折中方案，既可以像数组那样实现快速的搜索，又能像链表那样快速的插入和删除。其中二叉搜索树就是最常见的树形结构。
- 使用二叉搜索树插入数据需要保证数据的无序性，因为数据一旦有序以后，插入到树形结构中的数据将形成一个链表结构，无法发挥二叉搜索树应有的效能。
- 二叉搜索树有以下特性：
	- 节点的左子节点的数据项小于这个节点数据项
	- 右子节点的数据项大于或等于这个节点数据项
	- 插入一个节点需要根据前面的两个规则进行插入

- 删除操作
	- 待删除节点n没有子节点
		- 找到待删除节点p的父节点np
		- 将np的左子节点或右子节点的引用指向null（剩下的交给gc）
	- 待删除节点n有一个子节点
		- 找到待删除节点n的父节点np
		- 找到待删除节点n的左子节点或右子节点nc
		- 将np的左子节点或右子节点指向nc
		- 将nc的父节点指向np
	- 待删除节点n有两个子节点
		- 寻找待删除节点n的继承者ns
			- 基本思路就是，沿着（待删除节点的右子节点nr -> nr的左子节点nrl -> nrl的左子节点nrll -> ...）路径一路搜索，直到没有左子节点为止
		- 继承节点ns按照上一步的搜索方式寻找出来，则会有两种可能性:
			- ns没有子节点（将ns直接取出，插入到待删除的节点处）
				- 找到ns的父节点，将其左子节点置为null(此时ns被从父节点中切割出来)
				- 获取n节点的父节点np，左子节点nl，右子节点nr
				- 将ns的父节点指向np，左子节点指向nl，右子节点指向nr
				- 将np的左子节点或右子节点指向ns，将nl的父节点指向ns，将nr的父节点指向ns
			- ns有右子节点(分两步走:取出ns,将ns插入到n的位置)
				- 找到ns的父节点nsp，找到ns的右子节点nsr
				- 将nsp的左子节点指向nsr，将nsr的父节点指向nsp(此时ns被从父节点和子节点中切割出来)
				- 获取n节点的父节点np，左子节点nl，右子节点nr
				- 将ns的父节点指向np，左子节点指向nl，右子节点指向nr
				- 将np的左子节点或右子节点指向ns，将nl的父节点指向ns，将nr的父节点指向ns
- 插入操作
	- 待插入的节点p的数据项小于根节点r
		- 转向左子节点
		- 如果左子节点为null，则直接插入
		- 如果左子节点不为null，则以左子节点为根节点继续比较
	- 待插入的节点p的数据项大于等于根节点r
		- 转向右子节点
		- 如果右子节点为null，则直接插入
		- 如果右子节点不为null，则以右子节点为根节点继续比较

### 红黑树
- 在二叉搜索树的基础上，要保证整个树形结构大部分是平衡的，即任一节点的左子树后代数目和右子树的后代数目大致相等，数量不能过于悬殊。红黑树即保证了二叉搜索树的这一平衡性质的树形结构。
- 红黑树的特征：
	1. 节点是红色或黑色
	2. 根节点是黑色
	3. 每个叶节点（NIL节点，空节点）是黑色的
	4. 每个红色节点的两个子节点都是黑色（从每个叶子到根的所有路径上不能有两个连续的红色节点）
	5. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点

- 红黑树的插入
	- 插入的节点必为红色
		- 根据性质5，根节点到叶子节点的路径上黑节点的数量是相同的，如果插入节点为黑色，则会违背该性质
	- 执行插入，分为以下情况：
		- 插入到一颗空树
			- 节点插入的树为空树，则可以直接以插入节点为根节点，但是要将插入节点重新改为黑色，满足性质2
		- 插入的父节点为黑色
			- 节点插入的父节点为黑色，满足所有条件，无需调整
		- 插入的父节点为红色。父节点为红色，根据性质2和性质4，则父节点必然还有父节点。这样又要分作以下情况：
			- 父节点和叔父节点均为红色，无法满足性质4，需要将父节点与叔父节点均设为黑色，祖父节点设为红色
				- 此时，如果祖父节点为根节点则与性质2相违背
				- 祖父节点不为根节点，由于其颜色发生了变化所以需要将其作为一个全新的节点向上递归，调整颜色
			- 如果父节点为红色，而叔父节点为黑色或者不存在
				- 父节点为左子节点
                    - 插入的节点为父节点的左子节点，此时无法满足性质4
                    	- 此时，插入的节点与其父节点构成了左左结构，需要对其父节点进行右旋操作
                    	- 父节点替换祖父节点的位置，祖父节点作为父节点的右子节点，依次将父节点改为黑色，祖父节点改为红色，则满足红黑树的所有性质。
                    - 插入的节点为父节点的右子节点，此时无法满足性质4
                    	- 此时，插入节点与其父节点构成了左右结构，红黑树需要将这个新插入节点左旋转为父节点的左子节点，再参考上一步骤进行操作
                - 父节点为右子节点
                	- 插入的节点为父节点的左子节点
                		- 此时，插入节点与父节点构成右左结构，先转为右右结构，再参照下一部的步骤进行操作
                	- 插入的节点为父节点的右子节点
                    	- 此时，插入节点与父节点构成右右结构，对父节点进行左旋操作
                    	- 父节点为祖父节点，祖父节点为父节点的左自己诶单，一次将福界定啊改为黑色，祖父节点改为红色，则满足红黑树的所有性质


- 红黑树的删除，参照二叉搜索树
- 红黑树平衡修正：对红-黑树进行添加或删除后，可能会破坏其平衡性，进而需要进行修正
	- 左旋
	- 右旋
