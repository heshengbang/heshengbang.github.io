---
layout: post
title: synchronized底层实现与锁的升级降
date: 2018-6-14
tags: Java核心技术36讲笔记
---

### synchronized 底层如何实现？什么是锁的升级、降级？
- synchronized 代码块是由一对儿 monitorenter/monitorexit 指令实现的，Monitor 对象是同步的基本实现单元。每个对象的对象头里面都内置了一个Monitor，通过monitor对对象实现线程安全的访问。
- 在 Java 6 之前，Monitor 的实现完全是依靠操作系统内部的互斥锁，因为需要进行用户态到内核态的切换，所以同步操作是一个无差别的重量级操作。