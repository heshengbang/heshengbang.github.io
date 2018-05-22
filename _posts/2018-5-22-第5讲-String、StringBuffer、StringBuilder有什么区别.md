---
layout: post
title: String、StringBuffer、StringBuilder有什么区别
date: 2018-5-22
tags: Java核心技术36讲笔记
---

## String、StringBuffer、StringBuilder有什么区别

### 理解 Java 的字符串，String、StringBuffer、StringBuilder 有什么区别？
- String 是 Java 语言非常基础和重要的类，提供了构造和管理字符串的各种基本逻辑。它是典型的 Immutable 类，被声明成为 final class，所有属性也都是 final 的。也由于它的不可变性，类似拼接、裁剪字符串等动作，都会产生新的 String 对象。由于字符串操作的普遍性，所以相关操作的效率往往对应用性能有明显影响。
- StringBuffer 是为解决上面提到拼接产生太多中间对象的问题而提供的一个类，它是 Java 1.5 中新增的，我们可以用 append 或者 add 方法，把字符串添加到已有序列的末尾或者指定位置。StringBuffer 本质是一个线程安全的可修改字符序列，它保证了线程安全，也随之带来了额外的性能开销，所以除非有线程安全的需要，不然还是推荐使用它的后继者，也就是 StringBuilder。
- StringBuilder 在能力上和 StringBuffer 没有本质区别，但是它去掉了线程安全的部分，有效减小了开销，是绝大部分情况下进行字符串拼接的首选。

![String/StringBuffer/StringBuilder关系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/String、StringBuffer、StringBuilder关系图.jpg)

### 关键点
- String是Immutable的，不当的字符串操作会产生大量临时字符串和线程安全方面的问题
- StringBuffer是如何在String基础上实现高效的字符串拼接和线程安全的
- StringBuilder是如何在StringBuffer基础上在非多线程环境下减小资源开销的
- JVM对象缓存机制的理解和使用（字符串常量池）
- 如何通过优化String、StringBuffer、StringBuilder的使用优化Java代码
- String及相关类的演进历程（包括Java 9中的巨大变化）

### 字符串设计和实现考量
- String 是 Immutable 类的典型实现，原生的保证了基础线程安全，因为你无法对它内部数据进行任何修改，这种便利甚至体现在拷贝构造函数中，由于不可变，Immutable 对象在拷贝时不需要额外复制数据
- StringBuffer的线程安全是通过在各种修改数据的方法上加synchronized关键字实现的，非常直白。其实，这种简单粗暴的实现方式，非常适合我们常见的线程安全类实现，不必纠结于 synchronized 性能之类的，有人说“过早优化是万恶之源”，但考虑可靠性、正确性和代码可读性才是大多数应用开发最重要的因素。
- 为了实现修改字符序列的目的，StringBuffer 和 StringBuilder 底层都是利用可修改的（char，JDK 9 以后是 byte）数组，二者都继承了 AbstractStringBuilder，里面包含了基本操作，区别仅在于最终的方法是否加了 synchronized
- 字符数组内部，构建时初始字符串长度加 16（这意味着，如果没有构建对象时输入最初的字符串，那么初始值就是 16）。我们如果确定拼接会发生非常多次，而且大概是可预计的，那么就可以指定合适的大小，避免很多次扩容的开销。扩容会产生多重开销，因为要抛弃原有数组，创建新的（可以简单认为是倍数）数组，还要进行 arraycopy
- 通常情况下，我们没必要刻意去使用append方法，可以直接使用字符串进行拼接操作，因为在编译阶段，所有的拼接最终会被编译为append操作
- 在 JDK 8 中，字符串拼接操作会自动被 javac 转换为 StringBuilder 操作，而在 JDK 9 里面则是因为 Java 9 为了更加统一字符串操作优化，提供了 StringConcatFactory，作为一个统一的入口。javac 自动生成的代码，虽然未必是最优化的，但普通场景也足够了，你可以酌情选择

### 字符串缓存
- String 在 Java 6 以后提供了 intern() 方法，目的是提示 JVM 把相应字符串缓存起来，以备重复使用。
- 在我们创建字符串对象并调用 intern() 方法的时候，如果已经有缓存的字符串，就会返回缓存里的实例，否则将其缓存起来。一般来说，JVM 会将所有的类似“abc”这样的文本字符串，或者字符串常量之类缓存起来。、
- 缓存的字符串是存在所谓 PermGen 里的，也就是臭名昭著的“永久代”，这个空间是很有限的，也基本不会被 FullGC 之外的垃圾收集照顾到，因此如果使用不当，OOM 就会经常光顾
- 在后续版本中，这个缓存被放置在堆中，这样就极大避免了永久代占满的问题，甚至永久代在 JDK 8 中被 MetaSpace（元数据区）替代了。而且，默认缓存大小也在不断地扩大中，从最初的 1009，到 7u40 以后被修改为 60013
- `-XX:+PrintStringTableStatistics`可用于打印出缓存的大小，`-XX:StringTableSize=N`用于手动调整缓存的大小，通常情况下这是不必要的。
- Intern 是一种显式地排重机制，但是它也有一定的副作用，因为需要开发者写代码时明确调用，一是不方便，每一个都显式调用是非常麻烦的；另外就是我们很难保证效率，应用开发阶段很难清楚地预计字符串的重复情况，有人认为这是一种污染代码的实践
- 在 Oracle JDK 8u20 之后，推出了一个新的特性，也就是 G1 GC 下的字符串排重。它是通过将相同数据的字符串指向同一份数据来做到的，是 JVM 底层的改变，并不需要 Java 类库做什么修改。该功能目前是默认关闭的，你需要使用下面参数开启，并且记得指定使用 G1 GC：`-XX:+UseStringDeduplication`
- 在运行时，字符串的一些基础操作会直接利用 JVM 内部的 Intrinsic 机制，往往运行的就是特殊优化的本地代码，而根本就不是 Java 代码生成的字节码。
	- Intrinsic 可以简单理解为，是一种利用 native 方式 hard-coded 的逻辑，算是一种特别的内联，很多优化还是需要直接使用特定的 CPU 指令。

### String 自身的演化
-  Java 的字符串，在历史版本中，它是使用 char 数组来存数据的，这样非常直接。但是 Java 中的 char 是两个 bytes 大小，拉丁语系语言的字符，根本就不需要太宽的 char，这样无区别的实现就造成了一定的浪费。密度是编程语言平台永恒的话题，因为归根结底绝大部分任务是要来操作数据的。
-  在 Java 6 的时候，Oracle JDK 就提供了压缩字符串的特性，但是这个特性的实现并不是开源的，而且在实践中也暴露出了一些问题，所以在最新的 JDK 版本中已经将它移除了
-  在 Java 9 中，我们引入了 Compact Strings 的设计，对字符串进行了大刀阔斧的改进。
	-  将数据存储方式从 char 数组，改变为一个 byte 数组加上一个标识编码的所谓 coder，并且将相关字符串操作类都进行了修改。
	-  所有相关的 Intrinsic 之类也都进行了重写，以保证没有任何性能损失
	-  在极端情况下，字符串也出现了一些能力退化，比如最大字符串的大小。
	-  原来 char 数组的实现，字符串的最大长度就是数组本身的长度限制，但是替换成 byte 数组，同样数组长度下，存储能力是退化了一倍的！但这是存在于理论中的极限，还没有发现现实应用受此影响。