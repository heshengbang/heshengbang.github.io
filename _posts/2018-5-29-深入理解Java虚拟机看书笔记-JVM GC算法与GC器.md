---
layout: post
title: JVM GC算法与GC器
date: 2018-5-29
tags: 看书笔记
---

### 本文目录
- 垃圾收集算法
    - 标记-清除算法
    - 复制算法
    - 标记整理算法
    - 分代收集算法
- 垃圾收集器
	- Serial收集器
	- ParNew收集器
	- Parallel Scavenge收集器
	- Serial Old收集器
	- Parallel Old收集器
	- CMS收集器
	- G1收集器

## 垃圾收集算法

### 标记-清除算法
- JVM最基础的垃圾收集算法，是标记-清除(Mark-Sweep)算法。基本思路是：标记出所有需要回收的对象，在标记完成以后统一回收所有被标记的对象。
- 之所以说标记-清除算法是最基础的收集算法，是因为后续的所有收集算法都是基于标记-清除算法的思路，只不过对其不足进行改进而得到的。
- 示意图
![标记清除算法示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/标记清除算法示意图.jpg)

#### 标记
- 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链时，它将会被第一次标记并且进行筛选
	- 筛选的条件是此对象是否有必要执行finalize()方法，如果对象没有覆盖finalize()方法或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为”没有必要执行“
	- 对象如果被判定为有必要执行finalize()方法，那么对象将被放置到F-Queue的队列之中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它
- 在执行finalize()方法执行过程中，如果对象没有将自己与引用链上的其他对象建立联系，它将第二次被标

#### 清除
- 通过垃圾收集器，完成垃圾收集，内存释放

#### 缺点
- 效率不高：标记和清除两个过程的效率都不高
- 空间问题：标记清除以后会产生大量不连续的内存碎片，内存碎片过多会导致程序运行过程中分配较大的对象时，无法找到满足的连续内存空间而触发GC

### 复制算法
- 复制算法是为了解决标记清除算法效率不高的问题而产生的
- 复制算法将可用内存按容量划分为大小相等的两块，每次使用其中的一块。当其中一块内存用完后，将还存活的对象复制到另外一块上面，然后将已使用过的内存空间一次清理掉
![复制算法示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/复制算法示意图.jpg)

- 现在商用的虚拟机都采用了这种算法，只不过不是按照1:1的比例来划分区域，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间。每次使用Eden和其中一块survivor，当回收时，将Eden和survivor中还存活的对象一次性复制到另外一块survivor，然后清理掉Eden和刚才用过的survivor区域
- Hotspot虚拟机默认的Eden和survivor的大小比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%，只有10%的内存会被浪费。当超过10%内存大小的对象存活时，会有其他一些内存区域（老年代）进行担保(Handle Promotion)

#### 优点
- 每次都是对整个半区进行内存回收，内存分配也不用考虑内存碎片的问题。只要移动堆顶指针，按顺序分配内存即可

#### 缺点
- 每次使用只能使用内存区域的一半，代价相对过高

### 标记整理算法
- 复制算法在对象存活率比较高时，就会进行比较多的复制操作，效率会大大降低。如果不想浪费50%的内存空间，就需要进行额外的空间进行分配担保，以应对内存中所有对象100%存活的极端情况，所以一般老年代不会选用复制算法。
- 根据老年代的特点，我们一般使用标记-整理算法，它的标记思路和标记-清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存
- 示意图
![标记整理算法示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/标记整理算法示意图.jpg)

### 分代收集算法
- 根据对象存活周期的不同将内存划分为几块，一般是将java堆分为新生代和老年代，然后根据各个年代的特点采用最适当的收集算法
    - 新生代中，发现每次垃圾收集时都有大批对象死去，只有少量存活，因此采用复制算法
    - 老年代中，因为对象存活率高，没有额外空间对它进行分配担保，因此必须采用标记-清理或者标记-整理算法进行回收
- 当前商业虚拟机的垃圾收集都采用分代收集算法

## 垃圾收集器
- 垃圾收集器总览
![hotspot垃圾收集器总览](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/hotspot垃圾收集器总览.jpg)

### Serial收集器
- 最基本的、历史最悠久的垃圾收集器。Serial收集器是新生代的收集器，也是单线程的垃圾收集器，在它执行垃圾收集工作的时候，必须暂停其他所有工作线程，直到垃圾收集结束
- Serial/Serial Old收集器运行示意图。新生代用Serial，老年代选用Serial Old。
![Serial/Serial Old收集器运行示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/serial垃圾收集器示意图.jpg)
- 优点：简单而高效（相比其他单线程垃圾收集器）
- 缺点：暂停其他所有工作线程（stop the world）
- 应用场合：client模式下默认的新生代收集器

### ParNew收集器
- Serial收集器的多线程版本，除了使用多线程进行垃圾收集之外，其余行为包括Serial收集器可用的所有控制参数、收集算法、Stop the world、对象分配规则、回收策略等都与Serial收集器完全一样
- ParNew/Serial Old收集器运行示意图。新生代用ParNew，老年代选用Serial Old。
![ParNew/Serial Old垃圾收集器算法示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/parnew垃圾收集器算法示意图.jpg)
- ParNew收集器是运行在Server模式下的虚拟机中首选的新生代垃圾收集器，因为它和Serial收集器是唯二能和CMS收集器配合工作的。JDK1.5中当使用CMS收集老年代时，新生代只能选择Serial或者ParNew收集器
- 单CPU的情况下，ParNew收集器的效果不会比Serial收集器效果好，因为线程交互是需要开销的。超过2个CPU并在CPU越来越多的情况下，ParNew的多线程收集优势才会逐渐显露出来

### Parallel Scavenge收集器
- Parallel Scavenge收集器没有使用传统的GC收集器代码框架，而是独立实现
- Parallel Scavenge收集器是一个新生代收集器，使用复制算法，采用并行的多线程收集器
- 关注点
	- CMS收集器的关注点是缩短工作线程暂停时间
		- 暂停时间越短，就越适合与用户交互的程序，良好的响应速度能提升用户体验
	- Parallel Scavenge收集器的目的是达到一个可控制的吞吐量
		- 吞吐量就是CPU用于运行用户代码的时间和CPU总消耗时间的比值，即吞吐量 = 运行用户代码时间 / (运行用户代码时间 + 垃圾手机时间)，例如：虚拟机运行了100分钟，垃圾回收花掉1分钟，那么吞吐量就是99%
		- 高吞吐量可以高效率的利用CPU，尽快的完成程序的运算任务，适合在后台运算而不需要太多的交互任务
	- Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的`-XX:MaxGCPauseMillis`参数以及直接设置吞吐量大小的`-XX:GCTimeRatio`
		- `-XX:MaxGCPauseMillis`参数允许的值是一个大于0的毫秒数，收集器尽可能保证内存回收花费的时间不超过设定值
		- `-XX:GCTimeRatio`参数的值应当是一个大于0且小于100的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数
		  例如：如果把这个参数设为19，那么允许的最大GC时间就占总时间的5%(即1/(1+19))，默认值为99，就是允许最大1%的垃圾收集时间
- 因为与吞吐量密切相关，Parallel Scavenge收集器也被称为“吞吐量优先”的收集器

### Serial Old收集器
- Serial Old收集器是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。
- Serial/Serial Old收集器运行示意图。新生代用Serial，老年代选用Serial Old。
![Serial Old垃圾收集器示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/serial垃圾收集器示意图.jpg)
- Serial Old收集器主要意义也是给client模式下的虚拟机使用，在Server模式下，它有两大使用途径：①在JDK 1.5以及之前的版本中与Parallel Scavenge配合使用②作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用

### Parallel Old收集器
- parallel old收集器是parallel scavenge收集器在老年代的版本。使用多线程和标记-整理算法。
- parallel scavenge从JDK1.6开始被提供，在此之前，新生代的parallel scavenge收集器一直处于比较尴尬的地位。因为如果新生代选择了parallel scavenge收集器，老年代除了Serial Old收集器外别无他选。由于老年代Serial Old收集器在服务器端应用性能上的拖累，使用了parallel scavenge收集器也未必能在整体应用上获得吞吐量最大的效果。在Parallel Old收集器出现后，吞吐量优先收集器就有了比较名副其实的应用组合。在注重吞吐量以及CPU的敏感场合，都可以优先考虑Parallel Scavenge和Parallel Old收集器组合
- parallelscavenge垃圾收集器示意图。新生代选用Parallel Scavenge收集器，老年代选用Parallel Old收集器。
![Serial Old垃圾收集器示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/parallelscavenge垃圾收集器示意图.jpg)

### CMS收集器
- CMS(Concurrent Mark Sweep)收集器是一种以获取最短回收停顿时间为目标的收集器

#### 基本概述
- 目前很大一部分Java应用集中在互联网站或者B/S系统的服务端上，这类应用尤其重视服务的相应速度，希望系统停顿时间最短，给用户带来较好的交互体验。

#### 基本实现
- CMS收集器采用收集-清理算法实现，运行过程分为四个步骤：
    - 初始标记：标记GC Roots能直接关联到的对象，速度快，时间段
    - 并发标记：进行GC Roots Tracing，时间长
    - 重新标记：修正并发标记期间因用户程序继续运行而导致的标记产生变动的那一部分对象的标记记录，时间比初始标记长一点，但是比并发标记时间段
    - 并发清除：
- 并发标记和重新标记都需要stop the world
- CMS收集器收集过程中，耗时最长的是并发标记和并发清理，而初始标记和重新标记的时间太短到可以忽略不计，因而总体上来说，可以认为CMS收集器可以与用户线程并发执行
- CMS收集器示意图
![Serial Old垃圾收集器示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/concurrentMarksweep收集器示意图.jpg)

#### 缺点
- CMS对CPU资源敏感，在并发收集阶段虽然不会导致用户线程停顿，但是会占用一部分线程导致应用程序变慢，总吞吐量降低。当CPU的数量不足4个时，CMS收集器对用户程序的影响就可能变得很大
- CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次full gc。所谓浮动垃圾，就是指CMS并发处理过程中，用户线程还在运行着进而产生的新垃圾，这一部分垃圾出现在重新标记以后，CMS无法在本次收集中处理掉它们，因而只能在下一次GC去清理
- 最后一个缺点是因为它采用的标记-清理算法，这会导致大量的内存碎片，使用户程序无法分配大的内存空间，进而导致full gc

### G1收集器
- G1是面向服务端应用的垃圾收集器，HotSpot团队对它的期望是，未来能够替换掉JDK1.5中的CMS收集器

#### 概述
- G1收集器没有使用传统的GC收集器代码框架，而是独立实现
- 在G1之前的收集器进行收集的范围都是整个新生代或者老年代，而G1可以横跨新生代和老年代
- G1收集器与其他收集器采用了不同的收集理念，它将java堆划分为相等的独立区域（Region），虽然也保留了新生代和老年代的概念，但新生代和老年代不再是物理隔离的，它们只是一部分Regio的集合
- G1收集器能够建立可预测的停顿时间的原因是它有计划的避免在整个Java堆中进行全区域的垃圾收集，G1跟踪各个Region里面垃圾堆的价值大小（回收所获得的空间大小以及回收所需的经验值），在后台维护一个优先级列表，每次根据允许的收集时间，优先回收价值最大的Region（G1，即Garbage-First，这也是G1收集器名字的来源）
- 使用Region划分的内存空间以及有优先级的区域回收方式，如此能够保证G1收集器在有限的时间内可以获取尽可能高的收集效率
- G1的实现细节是通过维护一个remember set来避免整堆扫描，G1将内存划分为很多相等的region，每一个region都对应一个remember set，当虚拟机发现应用程序对Reference类型的数据进行写操作时，会产生一个write barrier暂停写操作，检查Reference引用的对象是否处于不同的Region之中：
	- 如果是，则通过CardTable将引用信息记录到被引用对象所属的Region对应的remember set之中
  在进行GC时，在GC根节点的枚举范围中加入Remember set记录的引用地址，就能保证不进行全堆扫描也不会遗漏

#### 特点
- 并发并行：
	- G1能利用多CPU、多核环境下的硬件优势，使用多个CPU或CPU核心来缩短stop-the-world的停顿时间
	- 简而言之，其他垃圾收集器原本需要停止java工作线程来完成GC，G1收集器可以通过并发的方式使java工作线程继续执行
- 分代收集：
	- G1和其他收集器一样，仍然采用了分代收集的概念
	- G1和其他收集器不同之处在于，它独立可以完成整个GC堆的管理，它能够采用不同的方式去处理新创建的对象和已经存活了一段时间以及熬过很多次GC的旧对象以获取更好的收集效果。而其他收集器是需要相互配合的，可以往前翻看看收集器这一节开篇那张图
- 空间整合：
	- 与CMS采用的标记-清除算法不同，G1整体上采用了标记-整理算法，而在局部区域来看，又是采用了复制算法来实现，这两种方式无论哪一种都不会产生内存碎片，能为后续程序提供可用的大内存
- 可预测的停顿：
	- G1相比CMS可以降低停顿时间，G1和CMS都关注停顿时间的降低，但是G1除了停顿时间降低之外还能建立可预测的停顿时间模型，即：能够让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集收集上的时间不超过N毫秒

#### 步骤（不含维护remember set的操作）
![G1收集器运行示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/javabasic/G1收集器运行示意图.jpg)

- 初始标记（Initial Marking），同CMS的初始标记
	- 标记GC Roots能够关联到的对象并修改TAMS(Next Top at Mark Start)的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建对象，这一阶段需要停顿工作线程，但耗时很短
- 并发标记(Concurrent Marking)，同CMS的并发标记
	- 从GCRoots开始对堆中对象进行可达性分析，找出存活的对象，这阶段耗时很长但可与用户程序并发执行
- 最终标记（Final Marking）
	- 为了修正在并发标记期间，因用户程序继续运作而导致标记产生变化的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remember Set中，这阶段需要停顿线程，但可以并发执行
- 筛选回收（Live data Counting and Evacuating）
	- 首先，对各个Region的回收价值和成本来进行排序
	- 根据用户所期望的GC停顿时间来制定回收计划
	- 可以与用户线程并发执行，但只回收一部分Region，时间是用户可控的，停顿用户线程可以大幅提升收集效率


## 扩展
### 并发和并行
- 并行(Parallel)：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态
- 并发(Concurrent)：指用户线程与垃圾收集线程同时执行（但不一定是并行的，可能会交替执行），用户程序在继续运行，而垃圾收集程序运行与另一个CPU上























