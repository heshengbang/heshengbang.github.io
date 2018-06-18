---

layout: post

title: Spring中Bean的生命周期

date: 2018-6-1

tags: Spring

---

### Spring Bean的生命周期（全）
- 实例化BeanFactoryPostProcessor实现类
- 执行BeanFactoryPostProcessor的postProcessBeanFactory方法
	- 在Application Context标准初始化以后，修改它内部的BeanFactory
	- 所有Bean的定义都会被加载，但是还没有Bean被初始化
	- 这可用于重写Bean定义或者给Bean添加新属性，甚至可以帮助运行时修改Bean定义
- 实例化BeanPostProcessor实现类
- 实例化InstantiationAwareBeanPostProcessor实现类
- 执行InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法
	- 在Bean被初始化之前调用该方法，返回Bean的代理对象而不是真实想要创建的对象，防止Bean被执行默认的初始化方法
- 执行Bean的构造方法
- 执行InstantiationAwareBeanPostProcessor的postProcessPropertyValues方法
	- 在Bean的属性被设值之前，将那些被作为属性配置好的Bean初始化并赋值
- 给Bean中各个成员变量需要依赖的引用对象
- 如果实现了BeanNameAware接口，调用BeanNameAware的setBeanName方法
	- 给Bean的名字赋值。如果配置了Bean名字就赋值给定的名字，如果没有就将实现类首字母小写
- 如果实现了BeanFactoryAware接口，调用setBeanFactory方法
- 如果实现了ApplicationContextAware接口，调用setApplicationContext方法
- 如果实现了BeanPostProcessor接口，则调用postProcessBeforeInitialization方法
- 如果实现了BeanInitializing接口，则调用afterPropertiesSet方法
- 调用配置文件的Bean的init-method属性执行执行的初始化方法
- 如果实现了BeanPostProcessor接口，则调用postProcessAfterInitialization方法
- 执行InstantiationAwareBeanPostProcessor的postProcessAfterInitialization方法
- Bean至此，初始化成功，进入可用状态
- 调用DisposibleBean的destroy方法
- 调用配置文件中指定的destroy-method属性的初始化方法

### Spring in Action中简述的生命周期（从Bean被实例化开始的）
- Spring对Bean进行实例化
- Spring将值和Bean的引用注入进Bean对应的属性
- 如果Bean实现了BeanNameAware接口，Spring将Bean的ID传递给setBeanName()接口方法
- 如果Bean实现了BeanFactoryAware接口，Spring将调用setBeanFactory()接口方法，将BeanFactory容器实例传入
- 如果Bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext()接口方法，将ApplicationContext的实例的引用传入
- 如果Bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessBeforeInitialization()接口方法
- 如果Bean实现了InitializingBean接口，Spring将调用它们的afterPropertiesSet()接口方法（如果Bean使用init-method的初始化方法，该方法也会被调用）
- 如果Bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessAfterInitialization()接口方法
- Bean创建完毕，驻留在ApplicationContext中
- 如果Bean实现了DisposableBean接口，Spring将调用它的destroy接口方法（如果Bean使用destroy-method，该方法也会被调用）

##### 注：目前网络上有不同的Spring Bean生命周期解释，是因为有的是从Bean的实例化开始解释，有的是从BeanFactory创建开始解释