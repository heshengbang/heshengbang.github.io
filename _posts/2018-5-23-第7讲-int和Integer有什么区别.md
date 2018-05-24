---
layout: post
title: int和Integer有什么区别
date: 2018-5-23
tags: Java核心技术36讲笔记
---

## int和Integer有什么区别

### int和Integer有什么区别
- Java的8个原始数据类型（Primitive Types，boolean、byte 、short、char、int、float、double、long）
- Java 语言虽然号称一切都是对象，但原始数据类型是例外
- Integer 是 int 对应的包装类，它有一个 int 类型的字段存储数据，并且提供了基本操作，比如数学运算、int 和字符串之间转换等
- 在 Java 5 中，引入了自动装箱和自动拆箱功能（boxing/unboxing），Java 可以根据上下文，自动进行转换，极大地简化了相关编程
- 构建 Integer 对象的传统方式是直接调用构造器，直接 new 一个对象。但是根据实践，我们发现大部分数据操作都是集中在有限的、较小的数值范围，因而，在 Java 5 中新增了静态工厂方法 valueOf，在调用它的时候会利用一个缓存机制，带来了明显的性能改进。按照 Javadoc，这个值默认缓存是 -128 到 127 之间。

### 关键点
- 自动装箱/自动拆箱是发生在什么阶段，编译期还是运行时？
- 使用静态工厂方法valueOf会使用到缓存机制，那么自动装箱的时候，缓存机制起作用吗？
- 原始数据类型与对象在应用时具体会产生哪些差异？
- Integer 源码

### 理解自动装箱、拆箱
- 自动装箱实际上算是一种语法糖，可以简单理解为 Java 平台为我们自动进行了一些转换，保证不同的写法在运行时等价，它们发生在编译阶段，也就是生成的字节码是一致的
- 自动装箱就是在编译器使用Integer.valueOf将int转为Integer，因此自动装箱时，缓存机制依然起作用
- 缓存机制并不是只有 Integer 才有，同样存在于其他的一些包装类，比如：
	- Boolean，缓存了 true/false 对应实例，确切说，只会返回两个常量实例 Boolean.TRUE/FALSE
	- Short，同样是缓存了 -128 到 127 之间的数值
	- Byte，数值有限，所以全部都被缓存
	- Character，缓存范围 '\u0000' 到 '\u007F'

### 原始数据类型与对象在应用时具体会产生哪些差异
- 原则上，建议避免无意中的装箱、拆箱行为，尤其是在性能敏感的场合，创建 10 万个 Java 对象和 10 万个整数的开销可不是一个数量级的，不管是内存使用还是处理速度，光是对象头的空间占用就已经是数量级的差距
- 使用原始数据类型、数组甚至本地代码实现等，在性能极度敏感的场景往往具有比较大的优势，用其替换掉包装类、动态数组（如 ArrayList）等可以作为性能优化的备选项
- 一些追求极致性能的产品或者类库，会极力避免创建过多对象。当然，在大多数产品代码里，并没有必要这么做，还是以开发效率优先。以我们经常会使用到的计数器实现为例，下面是一个常见的线程安全计数器实现。
```java
class Counter {
    private final AtomicLong counter = new AtomicLong();
    public void increase() {
        counter.incrementAndGet();
    }
}
```
如果利用原始数据类型，可以将其修改为
```java
class CompactCounter {
    private volatile long counter;
    private static final AtomicLongFieldUpdater<CompactCounter> updater = AtomicLongFieldUpdater.newUpdater(CompactCounter.class, "counter");
    public void increase() {
        updater.incrementAndGet(this);
    }
}
```

### Integer源码分析
- Integer 的缓存范围虽然默认是 -128 到 127，但是在特别的应用场景，比如我们明确知道应用会频繁使用更大的数值，我们可以通过指定JVM参数来设置缓存上限：`-XX:AutoBoxCacheMax=N`
- 这些实现，都体现在java.lang.Integer源码之中，并实现在 IntegerCache 的静态初始化块里:
```
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];
        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =                VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            ...
            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }
        ...
  }
```

- 我们在分析字符串的设计实现时，提到过字符串是不可变的，保证了基本的信息安全和并发编程中的线程安全。如果你去看包装类里存储数值的成员变量“value”，你会发现，不管是 Integer 还 Boolean 等，都被声明为“private final”，所以，它们同样是不可变类型！
- Integer 等包装类，定义了类似 SIZE 或者 BYTES 这样的常量，这反映了什么样的设计考虑呢？可能在不同的平台，比如 32 位或者 64 位平台，存在非常大的不同。那么，在 32 位 JDK 或者 64 位 JDK 里，数据位数会有不同吗？或者说，这个问题可以扩展为，我使用 32 位 JDK 开发编译的程序，运行在 64 位 JDK 上，需要做什么特别的移植工作吗？
	- <b>这种移植对于 Java 来说相对要简单些，因为原始数据类型是不存在差异的，这些明确定义在Java 语言规范里面，不管是 32 位还是 64 位环境，开发者无需担心数据的位数差异。</b>
	- <b>对于应用移植，虽然存在一些底层实现的差异，比如 64 位 HotSpot JVM 里的对象要比 32 位 HotSpot JVM 大（具体区别取决于不同 JVM 实现的选择），但是总体来说，并没有行为差异，应用移植还是可以做到宣称的“一次书写，到处执行”，应用开发者更多需要考虑的是容量、能力等方面的差异。</b>

### 原始类型线程安全
#### 原始数据类型操作是不是线程安全的呢？
- 原始数据类型的变量，显然要使用并发相关手段，才能保证线程安全。如果有线程安全的计算需要，建议考虑使用类似 AtomicInteger、AtomicLong 这样的线程安全类。
- 值得注意的是，部分比较宽的数据类型，比如 float、double，甚至不能保证更新操作的原子性，可能出现程序读取到只更新了一半数据位的数值！

### Java 原始数据类型和引用类型局限性
- 原始数据类型和 Java 泛型并不能配合使用
这是因为 Java 的泛型某种程度上可以算作伪泛型，它完全是一种编译期的技巧，Java 编译期会自动将类型转换为对应的特定类型，这就决定了使用泛型，必须保证相应类型可以转换为 Object，而原始数据类型却根本不是对象。

- 无法高效地表达数据，也不便于表达复杂的数据结构，比如 vector 和 tuple
Java 的对象都是引用类型，如果是一个原始数据类型数组，它在内存里是一段连续的内存，而对象数组则不然，数据存储的是引用，对象往往是分散地存储在堆的不同位置。这种设计虽然带来了极大灵活性，但是也导致了数据操作的低效，尤其是无法充分利用现代 CPU 缓存机制

Java 为对象内建了各种多态、线程安全等方面的支持，但这不是所有场合的需求，尤其是数据处理重要性日益提高，更加高密度的值类型是非常现实的需求。针对这些方面的增强，目前正在 OpenJDK 领域紧锣密鼓地进行开发，有兴趣的话你可以关注相关工程：http://openjdk.java.net/projects/valhalla/

### 对象的内存结构是什么样的吗？对象的大小是如何计算的？
- 数据存放地址
	- 原始数据类型存放在Java虚拟机栈中的局部变量表中；
	- 原始数据类型数组也是一种对象，因此存放在堆中；
	- 对象存放在堆中

- 数据大小计算
	- 整型
        - byte,1字节
        - short,2字节
        - int, 4字节
        - long,8字节
	- 浮点类型
        - float,4字节
        - double,8字节
    - char类型
    	- char,unicode,2字节
    - boolean类型
    	- boolean,大小没有明确指定。
    - 以上见数据类型的定义，见[官方文档](ttp://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html)
> boolean: The boolean data type has only two possible values: true and false. Use this data type for simple flags that track true/false conditions. This data type represents one bit of information, but its "size" isn't something that's precisely defined.

<b>
- 注意：
    - 此处的1字节是指1byte，1byte=8bit,bit译为中文是位
    - 简单来说：byte类型是1字节，8位
    - short是2字节，16位
    - int是4字节，32位
    - long是8字节，64位
    - float是4字节，32位
    - double是8字节，64位
    - char是2字节，16位
</b>

- 数据的内存大小占用（以64位Hotspot JVM为例）
	- Java对象的内存布局：对象头（Header），实例数据（Instance Data）和对齐填充（Padding）
		- 对象头在32位系统上占用8bytes，64位系统上占用16bytes
		- reference类型在32位系统上每个占用4bytes, 在64位系统上每个占用8bytes

	- 指针压缩
		- 对象占用的内存大小受到VM参数UseCompressedOops的影响
		- 对对象头的影响
			- 开启（-XX:+UseCompressedOops）对象头大小为12bytes（64位机器）
```java
class A {
	int a;
}
```
A对象占用内存情况：
①关闭指针压缩： 16+4=20不是8的倍数，所以+padding/4=24
②开启指针压缩： 12+4=16已经是8的倍数了，不需要再padding。
		- 对reference类型的影响
			- 64位机器上reference类型占用8个字节，开启指针压缩后占用4个字节
```java
static class B2 {
        int b2a;
        Integer b2b;
}
```
B2对象占用内存情况：
①关闭指针压缩： 16+4+8=28不是8的倍数，所以+padding/4=32
②开启指针压缩： 12+4+4=20不是8的倍数，所以+padding/4=24
		- 对数组对象的影响
			- 64位机器上，数组对象的对象头占用24个字节，启用压缩之后占用16个字节。之所以比普通对象占用内存多是因为需要额外的空间存储数组的长度。
new Integer[0]占用的内存大小，长度为0，即是对象头的大小：
①未开启压缩：24bytes
②开启压缩后：16bytes
		- 对复杂对象的影响就是整合了以上几种情况的复合而已。

	- 原生类型(primitive type)的内存占用如下

		Primitive Type | Memory Required(bytes)
        -|:-:
        boolean | 1字节
        byte | 1字节
        short | 2字节
        char | 2字节
        int | 4字节
        float | 4字节
        long | 8字节
        double | 8字节

	- 复杂类型，举例说明如下：
```java
static class B {
        int a;
        int b;
    }
static class C {
        int ba;
        B[] as = new B[3];

        C() {
            for (int i = 0; i < as.length; i++) {
                as[i] = new B();
            }
        }
    }
```
![对象内存大小占用计算示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/对象内存大小占用计算示意图.png)
	- 主要是三部分构成：C对象本身的大小+数组对象的大小+B对象的大小。
        - 未开启压缩：(16(头) + 4(属性) + 8(引用) + 4(padding)) + (24(头) + 8x3(引用)) +(16(头)+4x2(属性))*3 = 152bytes
        - 开启压缩：(12 + 4 + 4 +4(padding)) + (16 + 4*3 +4(数组对象padding)) + (12+8+4（B对象padding）)*3= 128bytes
