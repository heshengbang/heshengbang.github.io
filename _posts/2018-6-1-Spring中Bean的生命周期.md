---

layout: post

title: Spring中Bean的生命周期

date: 2018-6-1

tags: Spring

---

### Spring Bean的生命周期
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
- 为Bean注入属性
- 调用BeanNameAware的setBeanName方法
	- 给Bean的名字赋值。如果配置了Bean名字就赋值给定的名字，如果没有就将实现类首字母小写
- 调用BeanFactoryAware的setBeanFactory方法
- 执行BeanPostProcessor的postProcessBeforeInitialization方法
- 执行InitializingBean的afterPropertiesSet方法
- 调用配置文件的Bean的init-method属性执行执行的初始化方法
- 执行BeanPostProcess的postProcessAfterInitialization方法
- 执行InstantiationAwareBeanPostProcessor的postProcessAfterInitialization方法
- Bean至此，初始化成功，进入可用状态
- 调用DisposibleBean的destroy方法
- 调用配置文件中指定的destroy-method属性的初始化方法