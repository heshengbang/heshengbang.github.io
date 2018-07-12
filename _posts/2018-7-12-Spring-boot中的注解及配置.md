---
layout: post
title: Spring-boot/cloud中常见的注解及配置
date: 2018-7-12
tags: Spring
---

### 说明
- 最近到新公司，公司里面用的是spring-boot全家桶，是一套和以前公司完全不同的东西，需要仔细去学习这些东西才能跟上新同事的节奏。所以，立个帖子在这里，记录spring boot/cloud中繁杂的注解及配置，以便以后查询或供他人参考。

### 注解
- `@Configuration`
	- 该注解被用于指示一个类声明一个或多个被@Bean注解的方法，并且这些方法可以由Spring容器处理，以便在运行时生成bean定义和服务bean请求。例如：
	```
        @Configuration
        public class AppConfig {
           @Bean
           public MyBean myBean() {
               // 进行实例化、配置并且返回Bean...
           }
        }
    ```
    - 该注解经常和@Bean配合使用，表明注解的这个类是一个配置类，该类中会有一个或多个被@Bean标注的方法，这些方法会实例化、配置并返回一个Bean实例。
    - 运行时@Configuration注解的类会通过AnnotationConfigApplicationContext或者它的子类解析，简单的例子如下：
    ```
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        ctx.register(AppConfig.class);
        ctx.refresh();
        MyBean myBean = ctx.getBean(MyBean.class);
        // 此时myBean就是一个可以使用的实例对象
    ```

- `@Bean`
	- @Bean明确地指示了一种产生bean的方法，并且将产生的bean交个spring容器管理，该注解常常配合上面提到的@Configuration使用。@Configuration用于注解一个类，表明当前类会配置一个或多个Bean实例，并将Bean交给spring容器管理。@Bean用于注解其中的方法，表明该方法用于产生了一个Bean实例，并且该Bean实例已经交给了spring容器管理。

- `@SpringBootConfiguration`
	- 该注解指示一个类被用来提供Spring boot应用程序（原理类似@Configuration）。该注解可以被用来替换spring标准的@Configuration，这样spring-boot程序就可以自动去寻找配置。
	- 一个spring-boot应用程序应该只包含一个@SpringBootConfiguration并且大多数常用的spring boot应用程序都将从@SpringBootApplication处继承它。

- `@EnableAutoConfiguration`
	- 该注解启用spring application Context的自动配置功能，它会猜测并配置成你可能需要的配置。被@EnableAutoConfiguration注解的类的自动配置，通常基于开发者的classpath下的内容和自定义的bean。例如，如果你有tomcat-embedded.jar 在你的classpath下面，你也许想要一个TomcatServletWebServerFactory（除非你已经自定义了一个ServletWebServerFactory）。
	- 如果使用了@SpringBootApplication，应用程序的上下文的自动配置被自动启用，因此再添加该注解是没有额外效果的。原因可以参见@SpringBootApplication注解的构成。

- `@ComponentScan`
	- 该注解会自动扫描包路径下面的所有被@Controller、@Service、@Repository、@Component 注解的类。
	- @ComponentScan内含一些参数：
		- value 指定扫描的包
		- includeFilters 包含那些过滤
		- excludeFilters 不包含那些过滤
		- useDefaultFilters 默认的过滤规则是开启的
	- 如果开发者要自定义过滤规则的话是要关闭默认过滤规则的，并重写@Filters过滤器接口。
	- @ComponentScan中有一个内部注解@Filters，这个注解中定义的参数classes指定过滤的类，而另一个参数type是指过滤规则，它的值只能为FilterType这个枚举类型中的某一个，枚举类型FilterType含以下内容：
		- FilterType.ANNOTATION：按照注解
        - FilterType.ASSIGNABLE_TYPE：按照给定的类型
        - FilterType.ASPECTJ：使用ASPECTJ表达式
        - FilterType.REGEX：使用正则指定
        - FilterType.CUSTOM：使用自定义规则

- `@SpringBootApplication`
	- 该注解指示一个类中存在一个或多个被@Bean标注的方法，这些方法被用于初始化、配置并产生Bean实例，该注解同时还帮应用启用自动配置和组件扫描的功能。
	- 这是一个十分方便的注解，这个注解产生的原因就是spring-boot为了解决模板化开发spring应用而产生，所以spring中模板化的使用注解也应该被一个注解替换。使用@SpringBootApplication注解等效于同时使用了@Configuration、@EnableAutoConfiguration、@ComponentScan。

- `@RestController`
	- 如果只是使用@RestController注解Controller，则Controller中的方法可以直接返回json数据而无需额外添加@ResponseBody注解，但同时，使用了该注解就无法返回jsp页面或者html，配置的视图解析器 InternalResourceViewResolver将不起作用。return是什么就是什么。
	- 该注解是一个便捷的注解，它和@SpringBootApplication类似，整合了一般spring mvc项目中模板化的两个注解。从源码中就能看出，使用该注解的作用和同时使用@Controller、@ResponseBody类似，因为该注解本身就被@Controller和@ResponseBody所注解。

### 配置

































