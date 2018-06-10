---
layout: post
title: synchronized和ReentrantLock
date: 2018-6-10
tags: Java核心技术36讲笔记
---

### synchronized 和 ReentrantLock 有什么区别？有人说 synchronized 最慢，这话靠谱吗？
- synchronized 是 Java 内建的同步机制，所以也有人称其为 Intrinsic Locking，它提供了互斥的语义和可见性，当一个线程已经获取当前锁时，其他试图获取的线程只能等待或者阻塞在那里，关于synchronized更详细的内容，[点击查看另一篇文章](http://www.heshengbang.tech/2018/05/Java的synchronized关键字/)
- 在 Java 5 以前，synchronized 是仅有的同步手段，在代码中， synchronized 可以用来修饰方法，也可以使用在特定的代码块儿上，本质上 synchronized 方法等同于把方法全部语句用 synchronized 块包起来
- ReentrantLock，通常翻译为再入锁，是 Java 5 提供的锁实现，它的语义和 synchronized 基本相同。再入锁通过代码直接调用 lock() 方法获取，代码书写也更加灵活。
- ReentrantLock 提供了很多实用的方法，能够实现很多 synchronized 无法做到的细节控制，比如可以控制 fairness，也就是公平性，或者利用定义条件等。但是，编码中也需要注意，必须要明确调用 unlock() 方法释放，不然就会一直持有该锁。
- synchronized 和 ReentrantLock 的性能不能一概而论，早期版本 synchronized 在很多场景下性能相差较大，在后续版本进行了较多改进，在低竞争场景中表现可能优于 ReentrantLock。

### 知识点
- 什么是线程安全
- synchronized基本使用，案例
- ReentrantLock基本使用，案例
- synchronized的原理和底层实现
- ReentrantLock的原理和底层实现
- 锁膨胀、降级、偏斜锁、自旋锁、轻量级锁、重量级锁分别代表什么含义
- 并发包中 java.util.concurrent.lock 各种不同实现和案例分析

### 线程安全
- 基本特征:
	- 原子性，简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制实现
	- 可见性，是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上，volatile 就是负责保证可见性的
	- 有序性，是保证线程内串行语义，避免指令重排等