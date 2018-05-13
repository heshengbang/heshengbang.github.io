---
layout: post
title: Exception和Error有什么区别
date: 2018-5-13
tags: Java核心技术36讲笔记
---

## Exception和Error有什么区别

#### Throwable：Java 中只有 Throwable 类型的实例才可以被抛出（throw）或者捕获（catch），它是异常处理机制的基本组成类型。
 - Error
	- Error 是指在正常情况下，不大可能出现的情况，绝大部分的 Error 都会导致程序（比如 JVM 自身）处于非正常的、不可恢复状态（如 OutOfMemoryError 之类）
 - Exception
	- Exception 是程序正常运行中，可以预料的意外情况，可能并且应该被捕获，进行相应处理。
	- 可检查（checked）异常
		- 可检查异常在源代码里必须显式地进行捕获处理，这是编译期检查的一部分
	- 不检查（unchecked）异常
		- 不检查异常就是所谓的运行时异常，通常是可以编码避免的逻辑错误，并不会在编译期强制要求
			- NullPointerException
			- ArrayIndexOutOfBoundsException

#### 异常的层次结构
- Object
	- Throwable
		- Error
			- LinkageError:
				- NoClassDefFoundError:
				- UnsatisfiedLinkError:
				- ExceptionInInitializerError:
				- ClassCircularityError:
				- ClassFormatError:
				- IncompatibleClassChangeError:
				- VerifyError:
			- VirtualMachineError:
				- OutOfMemoryError:
				- StackOverflowError:
				- UnknownError:
				- InternalError:
			- Exception
				- Checked Exception
					- IOException:
				- Unchecked Exception
					- RuntimeException
						- NullPointerException：
						- ClassCastException：
						- SecurityException：

#### NoClassDefFoundError 和 ClassNotFoundException 有什么区别？


#### 捕获异常时应遵循的原则：
- 尽量不要捕获类似 Exception 这样的通用异常，而是应该捕获特定异常
- 不要生吞（swallow）异常，即将异常捕获后既不抛出也不写入到日志中

#### 性能方面
- try-catch 代码段会产生额外的性能开销，或者换个角度说，它往往会影响 JVM 对代码进行优化，所以建议仅捕获有必要的代码段，尽量不要一个大的 try 包住整段的代码；与此同时，利用异常控制代码流程，也不是一个好主意，远比我们通常意义上的条件语句（if/else、switch）要低效。
- Java 每实例化一个 Exception，都会对当时的栈进行快照，这是一个相对比较重的操作。如果发生的非常频繁，这个开销可就不能被忽略了