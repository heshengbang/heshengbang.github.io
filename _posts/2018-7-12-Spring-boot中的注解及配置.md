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

- `@Value`
	- 该注释用于注解属性、方法、构造函数，用于给一个受影响的"变量"一个默认值表达式。

- `@Bean`
	- @Bean明确地指示了一种产生bean的方法，并且将产生的bean交个spring容器管理，该注解常常配合上面提到的@Configuration使用。@Configuration用于注解一个类，表明当前类会配置一个或多个Bean实例，并将Bean交给spring容器管理。@Bean用于注解其中的方法，表明该方法用于产生了一个Bean实例，并且该Bean实例已经交给了spring容器管理。

- `@RefreshScope`
	- 被该注解标注的类对应的bean将在运行时被刷新。被这种方式注解的bean可以在运行时被刷新，并且任何使用它们的组件将在下一次方法调用中获得新的bean实例，该实例已经完全初始化并注入所有依赖项。

- `@ConfigurationProperties`
	- 该注解用来注解Bean将获取到外部化的配置。如果开发者想绑定一些来自外部配置的属性值(例如，来自application.properties文件的一些属性配置)，即可将该注解添加到一个类上或者被@Bean注解的方法上。例如：
	```
    	// 实体对象结构如下
        public class People {
            private String name;
            private Integer age;
            private List<String> address;
            private Phone phone;
        }
        public class Phone {
            private String number;
        }
        // application.properties中的配置如下
        com.example.demo.name=${aaa:hi}
        com.example.demo.age=11
        com.example.demo.address[0]=北京
        com.example.demo.address[1]=上海
        com.example.demo.address[2]=广州
        com.example.demo.phone.number=1111111
    ```
      有两种方式可以使用@ConfigurationProperties来完成外部配置文件的属性注入
    ```
        //加在类上,需要和@Component注解,结合使用
        @Component
        @ConfigurationProperties(prefix = "com.example.demo")
        public class People {
            private String name;
            private Integer age;
            private List<String> address;
            private Phone phone;
        }
        // 通过@Bean的方式进行声明
        @SpringBootApplication
        public class DemoApplication {
            @Bean
            @ConfigurationProperties(prefix = "com.example.demo")
            public People people() {
                return new People();
            }
            public static void main(String[] args) {
                SpringApplication.run(DemoApplication.class, args);
            }
        }
    ```
	- 值得注意的是，该注解和@Value不同，因为标注的是外部属性，所以注解中的参数使用SpEL表达式不会生效。
	- 该注解中内含的参数value和prefix互为别名，其值均为外部配置文件中定义的键值对中键的某一部分

- `@AliasFor`
	- 该注解被用来为另一个注解的属性声明别名。

- `@EnableAutoConfiguration`
	- 该注解启用spring application Context的自动配置功能，它会猜测并配置成你可能需要的配置。被@EnableAutoConfiguration注解的类的自动配置，通常基于开发者的classpath下的内容和自定义的bean。例如，如果你有tomcat-embedded.jar 在你的classpath下面，你也许想要一个TomcatServletWebServerFactory（除非你已经自定义了一个ServletWebServerFactory）。
	- 如果使用了@SpringBootApplication，应用程序的上下文的自动配置被自动启用，因此再添加该注解是没有额外效果的。原因可以参见@SpringBootApplication注解的构成。

- `@RequestBody`
	- 该注解表明一个方法的参数应该被绑定在web请求的content里面，根据请求的内容和类型来解析方法参数。此外，可以通过@Valid注解来验证参数的正确性。

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

- `@RestController`
	- 如果只是使用@RestController注解Controller，则Controller中的方法可以直接返回json数据而无需额外添加@ResponseBody注解，但同时，使用了该注解就无法返回jsp页面或者html，配置的视图解析器 InternalResourceViewResolver将不起作用。return是什么就是什么。
	- 该注解是一个便捷的注解，它和@SpringBootApplication类似，整合了一般spring mvc项目中模板化的两个注解。从源码中就能看出，使用该注解的作用和同时使用@Controller、@ResponseBody类似，因为该注解本身就被@Controller和@ResponseBody所注解。

- `@GetMapping`
	- 该注解将通过get方式发送的http请求映射到指定的处理方法上。
	- 具体来说，@GetMapping是一个复合注释，它的作用类似于@RequestMapping(method = RequestMethod.GET)，是一种快捷的注释方式。

- `@PostMapping`
	- 该注解将通过post方式发送的http请求映射到指定的处理方法上。
	- 具体来说，@PostMapping是一个复合注释，它的作用类似于@RequestMapping(method = RequestMethod.POST)，是一种快捷的注释方式。

- `@PathVariable`
	- 该注解表明方法的参数被绑定在URI模板中。在Servlet程序中，它支持@RequestMapping注释的方法。
	- 用法如下：
	```
        @GetMapping("/hello/{name}")
        public String index(@PathVariable String name) {
            return "hello! " + name;
        }
    ```
      通过该注释，可以从请求的url中获取到方法所需的参数。

- `@Import`
	- 该注解用于导入一个或多个被@Configuration注解的配置类。它提供的功能约等于spring xml文件中使用的import元素导入的被@Configuration锁注解的配置类。
	- 该注解的用法可参见@EnableEurekaServer的源码

- `@SpringBootApplication`
	- 该注解指示一个类中存在一个或多个被@Bean标注的方法，这些方法被用于初始化、配置并产生Bean实例，该注解同时还帮应用启用自动配置和组件扫描的功能。
	- 这是一个十分方便的注解，这个注解产生的原因就是spring-boot为了解决模板化开发spring应用而产生，所以spring中模板化的使用注解也应该被一个注解替换。使用@SpringBootApplication注解等效于同时使用了@Configuration、@EnableAutoConfiguration、@ComponentScan。

- `@SpringBootConfiguration`
	- 该注解指示一个类被用来提供Spring boot应用程序（原理类似@Configuration）。该注解可以被用来替换spring标准的@Configuration，这样spring-boot程序就可以自动去寻找配置。
	- 一个spring-boot应用程序应该只包含一个@SpringBootConfiguration并且大多数常用的spring boot应用程序都将从@SpringBootApplication处继承它。

- `@EnableEurekaServer`
	- 使用该注解可以激活Eureka服务器的相关配置
	- 该注解的源码如下：
	```
        @Target(ElementType.TYPE)
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Import(EurekaServerMarkerConfiguration.class)
        public @interface EnableEurekaServer {

        }
    ```
    - 从源码能看出，该注解导入了EurekaServerMarkerConfiguration这个类，而这个类恰巧就是被@Configuration注解的配置类，这也印证了@Import注解的用法。

- `@EnableDiscoveryClient`
	- 使用该注解以启用Eureka Discovery客户端。
	- 该注解的源码如下：
	```
        @Target(ElementType.TYPE)
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Inherited
        @Import(EnableDiscoveryClientImportSelector.class)
        public @interface EnableDiscoveryClient {

            /**
             * If true, the ServiceRegistry will automatically register the local server.
             */
            boolean autoRegister() default true;
        }
    ```
    - 其中值得重点关注的是通过@Import导入的EnableDiscoveryClientImportSelector配置类，其次是注解的参数autoRegister。
    - EnableDiscoveryClientImportSelector帮助导入作为DiscoveryClient可能需要的类。
    - autoRegister默认为true。该参数如果为true，则该注解所标注的服务将被自动的注册到本地的Eureka服务器，如果是false则需要手动注册。
    - 值得注意的是，该注解通常被用来注解那些会使用 注册在Eureka服务器上的服务 的那些服务。当然，它们自身也会被注册在Eureka服务器上，并且也可能是一个服务提供者。

- `@EnableEurekaClient`
	- 该注解很便捷的帮助客户端启用Eureka Discovery的配置，使用这个注解可以帮助你发现并确定你想要使用的Eureka服务。它所做的就是打开Discovery功能，并使用自动配置的方式去找开发者可用的Eurka服务类。
	- 该注解的源码如下：
	```
        @Target(ElementType.TYPE)
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Inherited
        @EnableDiscoveryClient
        public @interface EnableEurekaClient {

        }
    ```
    - 从源码中就能看出，该注解的定义中也被@EnableDiscoveryClient所标注，这意味着，它所提供的功能和@EnableDiscoveryClient其实有很大一部分类似。但是，它作为一个单独的注解存在必有其意义，那就是用来注解服务提供者。使用这个注解标注的类，会注册在Eureka服务器上，但是它们本身并不会去消费Eureka服务器上注册的那些服务。

- `@SpringBootConfiguration`
	- 该注解指示一个类被用来提供Spring boot应用程序（原理类似@Configuration）。该注解可以被用来替换spring标准的@Configuration，这样spring-boot程序就可以自动去寻找配置。
	- 一个spring-boot应用程序应该只包含一个@SpringBootConfiguration并且大多数常用的spring boot应用程序都将从@SpringBootApplication处继承它。

- `@EnableFeignClients`
	- 该注解将会扫描代码中被@FeignClient 注解的接口，这些接口被声明为Feign Client。配置组件扫描指令，可以直接使用@Configuration注解某个类来完成。
	- 该注解在定义是通过@Import注解导入了FeignClientsRegistrar类来处理那些被扫描使用了@FeignClient注解的类，将它们注册到FeignClient列表中。

- `@FeignClient`
	- 当某个接口使用了该注解后，那就应该有一个Restful风格的客户端被创建。可以通过配置@EnableFeignClients来讲该注解标注的接口注册到FeignClient列表中，在Eureka Server中去寻找对应的服务，并且将请求路径映射到被@FeignClient标注的接口上。其他地方调用该注解标注的接口，该注解将通过Eureka Server使用restful风格，找到对应的服务上面的处理方法去处理，然后返回结果或者直接结束。
	- 该注解中有两个参数value/name，这两个参数虽然为不同的参数，但是它们互为别名，因此其实是同一个东西。这个参数用来指定外部服务，即用来真实处理接口中方法的远程服务的名字。这个名字在spring-boot的application.properties中被定义，而这个服务通常被注册在Eureka之类的服务注册发现服务器上。

### 配置

































