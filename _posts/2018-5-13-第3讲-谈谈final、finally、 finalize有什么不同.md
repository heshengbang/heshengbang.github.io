---
layout: post
title: 谈谈final、finally、 finalize有什么不同
date: 2018-5-13
tags: Java核心技术36讲笔记
---

## 谈谈final、finally、 finalize有什么不同

#### 谈谈 final、finally、 finalize 有什么不同？
- final 可以用来修饰类、方法、变量，分别有不同的意义：
	- final 修饰的 class 代表不可以继承扩展
	- final 的变量是不可以修改的
	- final 的方法是不可以被重写的（override）
- finally 则是 Java 保证重点代码一定要被执行的一种机制。我们可以使用 try-finally 或者 try-catch-finally 来进行类似关闭 JDBC 连接、保证 unlock 锁等动作;
- finalize 是基础类 java.lang.Object 的一个方法，它的设计目的是保证对象在被垃圾收集前完成特定资源的回收。finalize 机制现在已经不推荐使用，并且在 JDK 9 开始被标记为 deprecated;

#### 推荐使用 final 关键字来明确表示我们代码的语义、逻辑意图
- 我们可以将方法或者类声明为 final，这样就可以明确告知别人，这些行为是不许修改的;
- 使用 final 修饰参数或者变量，也可以清楚地避免意外赋值导致的编程错误，甚至，有人明确推荐将所有方法参数、本地变量、成员变量声明成 final;
- final 变量产生了<b>某种程度</b>的不可变（immutable）的效果，所以，可以用于保护只读数据，尤其是在并发编程中，因为明确地不能再赋值 final 变量，有利于减少额外的同步开销，也可以省去一些防御性拷贝的必要;

##### final 也许会有性能的好处
- 很多文章或者书籍中都介绍了可在特定场景提高性能，比如，利用 final 可能有助于 JVM 将方法进行内联，可以改善编译器进行条件编译的能力等等；
- 坦白说，很多类似的结论都是基于假设得出的，比如现代高性能 JVM（如 HotSpot）判断内联未必依赖 final 的提示，要相信 JVM 还是非常智能的；
- 基于以上两点，可以认为在多数情况下，final 字段对性能的影响，并没有考虑的必要；

#### 关闭的连接等资源可以使用finally来完成，但是更推荐在jdk7中添加的try-with-resources 语句，因为通常 Java 平台能够更好地处理异常情况，编码量也要少很多

#### 实现 immutable 的类
- 将 class 自身声明为 final，这样别人就不能扩展来绕过限制了；
- 将所有成员变量定义为 private 和 final，并且不要实现 setter 方法；
- 通常构造对象时，成员变量使用深度拷贝来初始化，而不是直接赋值，这是一种防御措施，因为你无法确定输入对象不被其他人修改；
- 如果确实需要实现 getter 方法，或者其他可能会返回内部状态的方法，使用 copy-on-write 原则，创建私有的 copy；

#### 使用 java.lang.ref.Cleaner 来替换掉原有的 finalize 实现
- Cleaner 的实现利用了幻象引用（PhantomReference），这是一种常见的所谓 post-mortem 清理机制
- 利用幻象引用和引用队列，我们可以保证对象被彻底销毁前做一些类似资源回收的工作,比如：
	- 关闭文件描述符（操作系统有限的资源），它比 finalize 更加轻量、更加可靠
- 每个 Cleaner 的操作都是独立的，它有自己的运行线程，所以可以避免意外死锁等问题
- 实践中，我们可以为自己的模块构建一个 Cleaner，然后实现相应的清理逻辑。下面是 JDK 自身提供的样例程序：
```
public class CleaningExample implements AutoCloseable {
	// A cleaner, preferably one shared within a library
    private static final Cleaner cleaner = <cleaner>;
    static class State implements Runnable {
        State(...) {
            // initialize State needed for cleaning action
        }
        public void run() {
            // cleanup action accessing State, executed at most once
        }
    }
    private final State;
    private final Cleaner.Cleanable cleanable
    public CleaningExample() {
        this.state = new State(...);
        this.cleanable = cleaner.register(this, state);
    }
    public void close() {
        cleanable.clean();
    }
}
```
- 从可预测性的角度来判断，Cleaner 或者幻象引用改善的程度仍然是有限的
	- 如果由于种种原因导致幻象引用堆积，同样会出现问题。
- Cleaner 适合作为一种最后的保证手段，而不是完全依赖 Cleaner 进行资源回收，不然我们就要再做一遍 finalize 的噩梦了
