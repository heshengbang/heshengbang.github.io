---
layout: post
title: RedisTemplate中的opsForXXX
date: 2018-8-11
tags: Redis
---

### Redis的使用
- Redis在Java中通常是被用内存数据库或者缓存，一般会配合Spring框架进行使用。
- 在Spring-boot项目中使用redis仅需要以下步骤即可完成redis的使用：
	- 配置maven依赖。网上有些文章提到只需要`spring-boot-starter-data-redis`这个包，事实上这个包肯定是需要配合Spring-boot的一些包进行使用的。通常应当会包括`spring-boot-starter-parent`作为父项目，还应当包括`spring-boot-starter`，另外就依据你是通过单元测试的方式进行redis功能测试还是编写REST风格的接口对redis进行功能测试了。我是先用单元测试对redis的功能进行测试之后，又编写了REST风格的接口进行功能测试，所以我的pom.xml文件如下：
	```xml
    	<parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>1.5.10.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-redis</artifactId>
            </dependency>
        </dependencies>
    ```
	- 编写配置类。这一步是根据开发者的个人需求进行的，众所周知Spring-boot提供了一套默认的配置以减少开发者的工作，但是事实上很多配置对于开发者只适用于简单的demo而不是实际的生产环境，所以开发者需要去根据自己的环境编写配置（redis服务器地址，端口等）甚至去定制redis的一些初始化工作。我这里只是一个demo所以不需要编写配置类去改变默认的配置，因此就没有代码。Spring-boot集成的redis给出的默认配置如下：
	```
        hostName = 127.0.0.1
        port = 6379
        timeout = 2000
        password = ""
        usePool = true
        useSsl = false
        dbIndex = 5
        poolConfig.maxTotal = 8
        poolConfig.maxIdle = 8
        poolConfig.minIdle = 0
        poolConfig.lifo = true
        poolConfig.fairness = fase
        ...
    ```
	  还有一些默认配置就不写了，自己去看源码，或者debug去看吧。
	- 如果开发者对Spring-boot-starter-data-redis提供的默认配置无法满足自己的需要，又不想将自己的需要的配置写到配置中和代码耦合度变高，可以在spring-boot的application.yaml中去编写配置，这样实现了一种软配置。同样，我这里因为是demo所以不需要改动默认配置，也不需要去application.xml中配置，因此就不配置了。
	- 编写一个spring-boot主程序代码如下:
	```java
        @SpringBootApplication
        public class SpringBootRedisApplication {
            public static void main(String[] args) {
                SpringApplication.run(SpringBootRedisApplication.class, args);
            }
        }
    ```
	- 完成以上步骤后即可对Redis功能进行测试了，简单的测试代码如下：
	```java
        @RunWith(SpringJUnit4ClassRunner.class)
        // SpringBootRedisApplication是spring-boot项目中的主程序
        @SpringBootTest(classes = SpringBootRedisApplication.class)
        public class RedisTest {

            @Autowired
            RedisConnectionFactory factory;

            @Test
            public void testRedis(){
                //得到一个连接
                RedisConnection conn = factory.getConnection();
                conn.set("hello".getBytes(), "world".getBytes());
                System.out.println(new String(conn.get("hello".getBytes())));
            }
        }
    ```
    - 至此，spring-boot对redis的极简化使用即完成。在此基础上，可以根据个人的不同需要，进行配置修正，以测试个人关心的功能点。整个项目需要我们进行创建的就是三个文件，配别是pom.xml、spring-boot主程序、测试文件。哦对了，测试之前请启动本地redis服务器。
- 上面的部分完成了基本的redis使用说明，下面会说一说本文的重点关注部分。RedisTemplate的8种opsForXXX方法。

### RedisTemplate说明
- RedisTemplate
- StringRedisTemplate

### opsForXXX说明
- 说明
开发者使用Redis进行缓存时，通常是不会像上面那样直接使用RedisConnection来进行key-value存储，那样放入redis的键值对，和直接在终端操作命令行是一样的，你写入的key就是key，不会有任何变动。而通过opsForXXX添加到redis中的键值对，键的前面会被添加某些前缀用以区分。事实上，它们都实现了将数据放入redis的缓存中，这里的区别，产生的主要考虑还是在大型应用中缓存多而杂，添加缓存的同时用前缀对其进行分门别类，方便管理，以防止键冲突而导致的内容被覆盖。

- redisTemplate.opsForValue()


- redisTemplate.opsForList()
- redisTemplate.opsForSet()
- redisTemplate.opsForZSet()
- redisTemplate.opsForHash()
- redisTemplate.opsForGeo()
- redisTemplate.opsForCluster()
- redisTemplate.opsForHyperLogLog()






























