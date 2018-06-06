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

- `static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root)` 保证给定根节点一定在链表结构的最前端，也就是hash桶中。思路如下：
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

- `final TreeNode<K,V> find(int h, Object k, Class<?> kc)` 使用给定的哈希值和key及类型从当前点开始找符合指定哈希值和key以及key的类型的节点，源码如下：
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
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }
```
  步骤如下：
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

- `final TreeNode<K,V> getTreeNode(int h, Object k)` 如果当前节点的父节点不为空，就去寻找根节点，如果节点为空，则从当前节点开始向下寻找符合指定哈希值和key的节点并返回

- `static int tieBreakOrder(Object a, Object b)` 在key没有实现Comparable接口无法进行比较时，用该方法来比较key的大小，源码如下：
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
  步骤如下：
    1. 如果a为空，则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
    2. 如果a不为空，b为空，则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
    3. 如果a不为空，b也不为空，则判断a的类名和b的类名是否相等（字符串比较）splitsplit
    	- 如果相等则判断a的哈希值是否小于等于b的哈希值，如果小于等于则返回-1，否则返回1
    	- 如果不相等，则返回a的类名和b的类名比较的结果

- `final void treeify(Node<K,V>[] tab)` 树化此节点所在的节点列表，源码如下：
```java
final void treeify(Node<K,V>[] tab) {
    	// 定义一个根节点
        TreeNode<K,V> root = null;
        // 定义节点x和next
        // 从当前节点开始遍历
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            // next置为当前节点的下一个节点
            next = (TreeNode<K,V>)x.next;
            // 将当前节点的左右子节点置为null
            x.left = x.right = null;
            // 判断根节点是否为null，如果为空则说明根节点尚未构建，则构建根节点
            if (root == null) {
                // 如果根节点为null，则将当前节点的父节点置为null
                x.parent = null;
                // 当前节点颜色置为黑
                x.red = false;
                // 将当前节点置为根节点
                root = x;
            } else {
                // 如果根节点不为null，则证明根节点构建完毕，开始以根节点为基础构建左右子节点
                // 获取当前节点的key
                K k = x.key;
                // 获取当前节点的hash
                int h = x.hash;
                // 定义类型变量，用来保存key的类型
                Class<?> kc = null;
                // 以根节点为基础，遍历寻找x应该插入的位置，主要以x的哈希值与节点的哈希值为比较依据
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    // 如果根节点的哈希值大于x的哈希值h则dir为-1
                    if ((ph = p.hash) > h)
                        dir = -1;
                    // 如果根节点的哈希值小于x的哈希值h，则dir为1
                    else if (ph < h)
                        dir = 1;
                    // 如果哈希值相等且key的类型实现了Comparable接口，则将dir赋值为通过比较key的类型名字比较的结果
                    // 如果key为空或者key均没实现Comparable接口，则将dir赋值为key的哈希值比较或者类名比较的结果
                    else if ((kc == null && (kc = comparableClassFor(k)) == null) || (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                    // 根据上面的比较结果判断该继续从左子节点还是右子节点继续比较
                    // 如果获得的左子树或右子树为空, 则将节点插入当左节点或者右节点
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }
```
  步骤：
	1. （从当前节点开始遍历节点数组）将当前节点赋值为p
	2. 如果红黑树根节点为空，则将p设为红黑树的根节点，p等于自己的下一个节点，进入步骤1
	3. （如果红黑树的根节点不为空，则从根节点遍历红黑树）比较树节点和p的哈希值大小
	4. 如果p的哈希值小于树节点的哈希值则得到树的左子节点
		- 如果左子节点为空，则将p执行平衡插入到红黑树，作为树节点的左子节点，将p=p.next进入步骤2
		- 如果左子节点不为空，则将树节点设为其左子节点，循环到步骤3
	5. 如果p的哈希值大于树节点的哈希值则得到树的右子节点
		- 如果右子节点为空，则将p执行平衡插入到红黑树中，作为树节点的右子节点，将p=p.next进入步骤2
		- 如果右子节点不为空，则进入左子节点继续比较，循环到步骤3
	6. 如果哈希值相等，则判断p的key和树的key的类型
		- 如果p的key没实现Comparable接口，或者，p的key类型等于树节点的key的类型，则判断p的key的哈希值与树节点的key的哈希值的大小
			- 如果p的key的哈希值大，则得到树的左子节点
				- 如果左子节点为空，则将p执行平衡插入到红黑树，作为树节点的左子节点，将p=p.next进入步骤2
				- 如果左子节点不为空，则将树节点设为其左子节点，循环到步骤3
			- 如果树节点的key的哈希值大，则得到树的右子节点
				- 如果右子节点为空，则将p执行平衡插入到红黑树中，作为树节点的右子节点，将p=p.next进入步骤2
				- 如果右子节点不为空，则进入左子节点继续比较，循环到步骤3
		- 否则得到树的左子节点
			- 如果左子节点为空，则将p执行平衡插入到红黑树，作为树节点的左子节点，将p=p.next进入步骤2
			- 如果左子节点不为空，则将树节点设为其左子节点，循环到步骤3

- `final Node<K,V> untreeify(HashMap<K,V> map)`，返回一个非TreeNode的链表。这个方法的作用是将一组红黑树转换为一个链表。
```java
final Node<K,V> untreeify(HashMap<K,V> map) {
            Node<K,V> hd = null, tl = null;
            for (Node<K,V> q = this; q != null; q = q.next) {
                Node<K,V> p = map.replacementNode(q, null);
                if (tl == null)
                    hd = p;
                else
                    tl.next = p;
                tl = p;
            }
            return hd;
        }
```
  步骤：
		1. 从当前节点q开始遍历
		2. 将q从TreeMap转成Node类型的p
			- 如果是遍历的首次，则将头节点赋值给hd
			- 如果不是首次遍历，则将p赋值给t1的下一个值
		3. 将p赋值给t1
		4. q=q.next，进入步骤2
		5. 遍历结束，返回最开始记录的头结点hd

- `final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v)`，红黑树版的putVal()方法
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

- `final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable)`，删除给定节点节点必须在删除之前就存在
```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
            int n;
            // 如果哈希桶为空或者桶长度为0则直接结束
            if (tab == null || (n = tab.length) == 0)
                return;
			// 用节点的哈希值和哈希桶的大小做位与操作，找到该节点在哈希桶中的位置
            int index = (n - 1) & hash;
            // 获取哈希桶中的节点，并将其作为根节点
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            // 获取节点的前一个节点和后一个节点
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            // 如果前一个节点为空
            if (pred == null)
            	// 就将哈希桶中的节点换位当前节点的下一个节点
                tab[index] = first = succ;
            else
            	// 否则前一个节点的下一个节点等于当前节点的下一个节点
                pred.next = succ;
            // 如果下一个节点不为null
            if (succ != null)
            	// 下一个节点的前一个节点等于当前节点的前一个几点
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
            // 获取当前节点的左子节点，右子节点
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            // 如果左右子节点均不为null，执行红黑树的删除操作
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
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

            if (replacement == p) {  // detach
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

- `final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit)`，该方法将map的哈希桶中索引为index的元素，及以其为根节点的红黑树拆分放到放到新的哈希桶tab中
```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // 将链表分为高位子链表，低位子链表，重新连接节点，并保存在原有链表中的顺序
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            // 遍历当前元素为根节点的红黑树。
            // TreeNode保存了两种结构，一种是链表，持有前后节点的引用。一种是红黑树，持有左右字数以及父节点的引用。
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
            // 关于这里计算树化的问题。当代码执行到这里的时候lc+hc的数值是足够被树化的
            // 当lc或hc中任一个数字大于反树化的标准的时候
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
                    // ?
                    if (hiHead != null) // (else is already treeified)
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

- `static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root, TreeNode<K,V> p)`
```java
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;
                p.parent = r;
            }
            return root;
        }
```

- `static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root, TreeNode<K,V> p)`
```java
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }
```

- `static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x)`
```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                if (xp == (xppl = xpp.left)) {
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```

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

- `static <K,V> boolean checkInvariants(TreeNode<K,V> t)`
```java
       static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
            TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
                tb = t.prev, tn = (TreeNode<K,V>)t.next;
            if (tb != null && tb.next != t)
                return false;
            if (tn != null && tn.prev != t)
                return false;
            if (tp != null && t != tp.left && t != tp.right)
                return false;
            if (tl != null && (tl.parent != t || tl.hash > t.hash))
                return false;
            if (tr != null && (tr.parent != t || tr.hash < t.hash))
                return false;
            if (t.red && tl != null && tl.red && tr != null && tr.red)
                return false;
            if (tl != null && !checkInvariants(tl))
                return false;
            if (tr != null && !checkInvariants(tr))
                return false;
            return true;
        }
```

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
	- 节点是红色或黑色
	- 根是黑色
	- 所有叶子（包含空叶子节点nil）都是黑色
	- 如果一个节点是红的，则它的两个子节点都是黑的
	- 从任一节点到其叶子（包含空叶子节点nil）的所有简单路径都包含相同数目的黑色节点

- 红黑树的插入，参照二叉搜索树
- 红黑树的删除，参照二叉搜索树
- 红黑树平衡修正：对红-黑树进行添加或删除后，可能会破坏其平衡性，进而需要进行修正
	- 左旋
	- 右旋

### 源码及注释
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>  {
		// 红黑树的父节点
        TreeNode<K,V> parent;
        // 红黑树的左子节点
        TreeNode<K,V> left;
        // 红黑树的右子节点
        TreeNode<K,V> right;
        // 前一个节点。在节点被删除后需要将前一个节点和当前节点的链接关系取消掉
        TreeNode<K,V> prev;
        // 当前节点是否为红，红黑树的节点不为红就是黑
        boolean red;
        // 构造方法
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        // 返回当前节点所在树的根节点
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
        // 确保给定根节点在hash桶数组中，即链表最前端
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

        // 根据hash值和key值，从根节点开始查找一个节点
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
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }
        // 根据hash和key值，从根节点开始查找TreeNode
        final TreeNode<K,V> getTreeNode(int h, Object k) {
            return ((parent != null) ? root() : this).find(h, k, null);
        }
        // 提供了一种基于hash值的排序，比较两个参数的hash值大小
        static int tieBreakOrder(Object a, Object b) {
            int d;
            if (a == null || b == null ||
                (d = a.getClass().getName().
                 compareTo(b.getClass().getName())) == 0)
                d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                     -1 : 1);
            return d;
        }
        // 将和此节点关联的节点树化，从链表结构转向红黑树
        final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    for (TreeNode<K,V> p = root;;) {
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);
                        TreeNode<K,V> xp = p;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            root = balanceInsertion(root, x);
                            break;
                        }
                    }
                }
            }
            moveRootToFront(tab, root);
        }
        // 返回一组非树形节点来替换和当前节点相连的节点
        final Node<K,V> untreeify(HashMap<K,V> map) {
            Node<K,V> hd = null, tl = null;
            for (Node<K,V> q = this; q != null; q = q.next) {
                Node<K,V> p = map.replacementNode(q, null);
                if (tl == null)
                    hd = p;
                else
                    tl.next = p;
                tl = p;
            }
            return hd;
        }
        // 树版的putVal()方法
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
        // 移除TreeNode
        final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            if (pred == null)
                tab[index] = first = succ;
            else
                pred.next = succ;
            if (succ != null)
                succ.prev = pred;
            if (first == null)
                return;
            if (root.parent != null)
                root = root.root();
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // too small
                return;
            }
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
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

            if (replacement == p) {  // detach
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
        // 如果现在的树形结构太小，就将节点拆分为较高和较低的两部分
        // 该方法只能在resize()方法中被调用，详细见移位和索引部分
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }
            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }

        /* ------------------------------------------------------------ */
        // 红黑树方法，全部由CLR改编
        // 红黑树的左旋操作
        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;
                p.parent = r;
            }
            return root;
        }
        // 红黑树的右旋操作
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }
        // 红黑树的平衡插入
        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                if (xp == (xppl = xpp.left)) {
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
        // 自平衡删除
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
        // 递归检查红黑树的不变性
        static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
            TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
                tb = t.prev, tn = (TreeNode<K,V>)t.next;
            if (tb != null && tb.next != t)
                return false;
            if (tn != null && tn.prev != t)
                return false;
            if (tp != null && t != tp.left && t != tp.right)
                return false;
            if (tl != null && (tl.parent != t || tl.hash > t.hash))
                return false;
            if (tr != null && (tr.parent != t || tr.hash < t.hash))
                return false;
            if (t.red && tl != null && tl.red && tr != null && tr.red)
                return false;
            if (tl != null && !checkInvariants(tl))
                return false;
            if (tr != null && !checkInvariants(tr))
                return false;
            return true;
        }
    }
```

- 从源码中可以看出，TreeNode除了实现Node的方法之外，就是围绕红黑树的性质在构建自己的方法和属性。

