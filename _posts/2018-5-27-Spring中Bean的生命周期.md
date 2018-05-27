---

layout: post

title: Spring中Bean的生命周期

date: 2018-5-27

tags: Spring

---

### BeanFactory
- 源码先贴为敬，方法的作用见上方注释：
```java
public interface BeanFactory {
	String FACTORY_BEAN_PREFIX = "&";
    //根据名字获取一个bean对象
	Object getBean(String name) throws BeansException;
    //根据名字和指定类型获取一个bean对象，
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
    //根据名字和参数获取一个bean对象
	Object getBean(String name, Object... args) throws BeansException;
    //指定的参数类型获取一个bean对象
	<T> T getBean(Class<T> requiredType) throws BeansException;
    //指定的参数类型和参数获取一个bean对象
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    //是否包含某个bean
	boolean containsBean(String name);
    //给定名称的bean是否是单例
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    //给定名称的bean是否是原型模式，每次通过容器的getBean方法获取prototype定义的Bean时，都将产生一个新的Bean实例
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    //检查具有给定名称的bean是否与指定的类型匹配。
    //更具体地说，检查给定名称的{@link #getBean}调用是否会返回可分配给指定目标类型的对象。
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    //检查具有给定名称的bean是否与指定的类型匹配
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
    //返回给定名称bean的类型
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    //返回给定bean名称的别名，如果有的话
	String[] getAliases(String name);
}
```
- BeanFactory作为一个超级接口，没继承任何接口，与BeanFactroy直接相关的的子接口或子类有12个，关系如下：
![BeanFactory的接口关系图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/BeanFactory的接口关系图.png)

### ApplicationContext
- ApplicationContext作为最常见的Spring容器接口，本文以该接口的实现类中的bean为例介绍Spring bean的生命周期。ApplicationContext接口层次体系结构如下如所示：
![ApplicationContext接口层次体系结构](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/ApplicationContext接口层次体系结构.jpg)
	- MessageSource: 用于解析消息的策略接口，通过参数化的形式对消息进行国际化
	- ListableBeanFactory: BeanFactory接口的扩展实现接口，bean工厂可以枚举所有bean实例，而不是按客户端的要求逐个尝试bean的名称查找。预加载所有bean定义的BeanFactory实现（例如基于XML的工厂）可以实现此接口
	- HierarchicalBeanFactory: BeanFactory的子接口，是Spring 接口层次结构的一部分
	- EnvironmentCapable: 组件指示接口，包含并公开环境应用
	- ResourcePatternResolver: 策略接口，用于将地址模式解析到一个资源对象
	- 封装事件发布功能的接口。 作为ApplicationContext的父接口，为其服务

### Bean的生命周期
- Spring Bean的完整生命周期从创建Spring容器开始，直到最终Spring容器销毁Bean，这其中包含了一系列关键点，流程图如下：
![Bean生命周期流程图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/生命周期流程图.png)
主要涉及到的类或接口如下：
	- BeanFactoryPostProcessor
	```java
    public interface BeanFactoryPostProcessor {
        void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
    }
    ```
	- BeanPostProcessor
	```java
    public interface BeanPostProcessor {
        Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
        Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
    }
    ```
	- InstantiationAwareBeanPostProcessor
	```java
    public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
        Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;
        boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;
        PropertyValues postProcessPropertyValues(
                PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;
    }
    ```
	- BeanNameAware
	```java
    public interface BeanNameAware extends Aware {
        void setBeanName(String name);
    }
    ```
	- BeanFactoryAware
	```java
    public interface BeanFactoryAware extends Aware {
        void setBeanFactory(BeanFactory beanFactory) throws BeansException;
    }
    ```
	- InitializingBean
	```java
    public interface InitializingBean {
        void afterPropertiesSet() throws Exception;
    }
    ```
	- DisposableBean
	```java
    public interface DisposableBean {
        void destroy() throws Exception;
    }
    ```
- Bean生命周期流程简述
	1. 实例化一个Bean，也就是我们通常说的new
	2. 按照Spring上下文对实例化的Bean进行配置，也就是IOC注入
	3. 如果这个Bean实现了BeanNameAware接口，会调用它实现的setBeanName(String beanId)方法，此处传递的是Spring配置文件中Bean的ID
	4. 如果这个Bean实现了BeanFactoryAware接口，会调用它实现的setBeanFactory()，传递的是Spring工厂本身（可以用这个方法获取到其他Bean）
	5. 如果这个Bean实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文，该方式同样可以实现步骤4，但比4更好，因为ApplicationContext是BeanFactory的子接口，有更多的实现方法。
		- 如果是BeanFactory中的Bean，这一步就不需要
	6. 如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessBeforeInitialization(Object obj, String s)方法，BeanPostProcessor经常被用作是Bean内容的更改，并且由于这个是在Bean初始化结束时调用After方法，也可用于内存或缓存技术
	7. 如果这个Bean在Spring配置文件中配置了init-method属性会自动调用其配置的初始化方法
	8. 如果这个Bean关联了BeanPostProcessor接口，将会调用postAfterInitialization(Object obj, String s)方法
	9. 至此，Bean的创建已经完成，Bean已经可以被使用。默认Bean是单例的，因此通过BeanFactory获取的都是同一个实例对象
	10. 当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean接口，会调用其实现的destroy方法
	11. 如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法

- 参与生命周的接口方法分类
	- <b>Bean自身的方法</b>：这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法
	- <b>Bean级生命周期接口方法</b>：这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法
	- <b>容器级生命周期接口方法</b>：这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”
	- <b>工厂后处理器接口方法</b>：这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用


### 扩展

- Spring容器中Bean的5种作用域
	- singleton：单例模式，在整个Spring IoC容器中，使用singleton定义的Bean将只有一个实例
	- prototype：原型模式，每次通过容器的getBean方法获取prototype定义的Bean时，都将产生一个新的Bean实例
	- request：对于每次HTTP请求，使用request定义的Bean都将产生一个新实例，即每次HTTP请求将会产生不同的Bean实例。只有在Web应用中使用Spring时，该作用域才有效
	- session：对于每次HTTP Session，使用session定义的Bean豆浆产生一个新实例。同样只有在Web应用中使用Spring时，该作用域才有效
	- globalsession：每个全局的HTTP Session，使用session定义的Bean都将产生一个新实例。典型情况下，仅在使用portlet context的时候有效。同样只有在Web应用中使用Spring时，该作用域才有效

  其中比较常用的是singleton和prototype两种作用域，如果不指定Bean的作用域，Spring默认使用singleton作用域，它的初始化示意图如下：
![singletonbean初始化示意图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/singletonbean初始化示意图.jpg)

- Spring 循环引用问题
	- singleton对象循环引用：
		- 构造器循环依赖，Spring无法你解决，抛出BeanCurrentlyInCreationException异常
		- 设值注入循环依赖，Spring可以依赖注入
		- 设值与构造混合的循环依赖，Spring可以依赖注入
	- Prototype的循环引用：
		- 构造器依赖注入，Spring无法解决，抛出BeanCurrentlyInCreationException异常
		- 设值注入循环依赖，Spring无法解决，抛出BeanCurrentlyInCreationException异常
		- 设值与构造混合的循环依赖，抛出BeanCurrentlyInCreationException异常
	- singleton对象和prototype对象的混合情况，需要具体分析

- BeanFactory和FactoryBean的区别与联系
	- BeanFactory和FactoryBean都是接口
	- 实现 BeanFactory 接口的类表明此类事一个工厂，作用就是配置、新建、管理各种Bean
	- 实现 FactoryBean 的类表明此类也是一个Bean，类型为工厂Bean（Spring中共有两种bean，一种为普通bean，另一种则为工厂bean）。顾名思义，它也是用来管理Bean的，而它本身由spring管理
	- BeanFactory的源码上面贴过了，这里就不贴了，贴一下FactoryBean的源码：
	```java
    public interface FactoryBean<T> {
		//返回由FactoryBean创建的Bean的实例
		T getObject() throws Exception;
		//确定由FactoryBean创建的Bean的作用域是singleton还是prototype
		Class<?> getObjectType();
		//返回FactoryBean创建的Bean的类型
		boolean isSingleton();
    }
    ```
    - 类实现了FactoryBean接口后，配置到Spring的容器中，通过ApplicationContext或BeanFactory根据factoryBean的名字获取到的是它生产的类的实例。
    	- 例如，AppleFactoryBean被配置到spring上下文中，bean的名字是appleFactoryBean，那么通过BeanFactory或ApplicationContext的`getBean("appleFactoryBean")`获取到的是Apple的一个实例。如果想通过spring拿到appleFactoryBean，需要在名称前加 & 符号`getBean("&appleFactoryBean")`。此处的&符号在BeanFactory的源码中是有提到的。
    	- FactoryBean管理的bean实际上也是由spring进行配置、实例化、管理，因此由FactoryBean管理的bean不能再次配置到spring配置文件中（xml、java类配置、注解均不可以），否则会报BeanIsNotAFactoryException异常