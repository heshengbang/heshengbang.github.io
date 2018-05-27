---
layout: post
title: Spring AOP的实现原理
date: 2018-5-27
tags: Spring
---

### AOP
- AOP（Aspect Orient Programming），我们一般称为面向方面（切面）编程，作为面向对象的一种补充，用于处理系统中分布于各个模块的横切关注点，比如事务管理、日志、缓存等等。
- AOP实现的基本原理是AOP框架自动创建的AOP代理。
- AOP代理主要分为静态代理和动态代理：
	- 静态代理的代表为AspectJ
	- 动态代理则以Spring AOP为代表

### 使用AspectJ的编译时增强实现AOP
- AspectJ是静态代理的增强，所谓的静态代理就是AOP框架会在编译阶段生成AOP代理类，因此也称为编译时增强。例如：
```java
public class Hello {
    public void sayHello() {
        System.out.println("hello");
    }
     public static void main(String[] args) {
        Hello h = new Hello();
        h.sayHello();
    }
}
//使用AspectJ增强
public aspect TxAspect {
    void around():call(void Hello.sayHello()){
        System.out.println("方法执行前...");
        proceed();
        System.out.println("方法执行后...");
    }
}
```

- 通过反编译查看class文件，可以看出：AspectJ的静态代理在编译阶段将Aspect织入Java字节码中，运行的时候使用的就是经过增强之后的AOP对象

### 使用Spring AOP
- 与AspectJ的静态代理不同，Spring AOP使用的动态代理
	- 所谓的动态代理就是说AOP框架不会去修改字节码，而是在内存中临时为方法生成一个AOP对象
	- 这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法
- Spring AOP中的动态代理主要有两种方式，Spring AOP中的动态代理主要有两种方式
	- JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口，JDK动态代理的核心是InvocationHandler接口和Proxy类
	- 目标类没有实现接口，那么Spring AOP会选择使用CGLIB来动态代理目标类
	- CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类。<b>注意，CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的</b>

### 总结
- AspectJ在编译时就增强了目标对象
- Spring AOP的动态代理则是在每次运行时动态的增强，生成AOP代理对象
- 区别在于生成AOP代理对象的时机不同
	- 相对来说AspectJ的静态代理方式具有更好的性能
	- 但是AspectJ需要特定的编译器进行处理，而Spring AOP则无需特定的编译器处理