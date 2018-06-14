---

layout: post

title: Hibernate的原理和缓存机制

date: 2018-6-12

tags: Hibernate

---

本文基于hibernate-core-5.2.2.Final。

### Hibernate的一次查询过程
- 以查询部门为例，Department和数据库中的department表相映射，它的一次查询代码如下：
```java
	String sql = "from " + Department.class.getName();
    // 创建一个查询对象
	Query query = session.createQuery(sql);
    // 设置结果从哪里开始取值，对应sql语句中的offset，即偏移量
	query = query.setFirstResult(0);
    // 设置结果取多少条数据，对应sql语句中的limit
	query = query.setMaxResults(10);
    // 此时执行查询操作，查询出的是Department的代理数据对象
    // 是一个采用动态代理技术在Department类基础上生成的对象
    // 等到后面实际需要这个数据对象的时候，再去查询每个数据对象的各个字段内容
    // 值得注意的是，这里的Iterator是hibernate自己实现的一个迭代器IteratorImpl
	Iterator<Department> employees = query.iterate();
	while (employees.hasNext()) {
    	// 此时才针对每个数据对象去数据库中查询内容
		System.out.println(employees.next().getDeptName());
	}
```
其中`session.createQuery(sql);`的执行代码是`org.hibernate.internal.AbstractSharedSessionContract#createQuery(String queryString)`，源码如下：
```java
public QueryImplementor createQuery(String queryString) {
		// 检查session是否打开
		checkOpen();
        // 检查事务的同步状态
		checkTransactionSynchStatus();
		delayedAfterCompletion();
		try {
        	// 新建一个不可变的QueryImpl对象
			final QueryImpl query = new QueryImpl(
					this,
					getQueryPlan( queryString, false ).getParameterMetadata(),
					queryString
			);
            // 给查询语句设置注释？
			query.setComment( queryString );
            // 调用org.hibernate.internal.SessionImpl中的applyQuerySettingsAndHints执行查询操作
			applyQuerySettingsAndHints(query);
			return query;
		}
		catch (RuntimeException e) {
			throw exceptionConverter.convert(e);
		}
	}
```

- 执行以上代码的一次查询，一共查询出四条数据。控制台打出了如下日志：
```
    Hibernate: select department0_.DEPT_ID as col_0_0_ from DEPARTMENT department0_ limit ?
    Hibernate: select department0_.DEPT_ID as DEPT_ID1_0_0_, department0_.DEPT_NAME as DEPT_NAM2_0_0_, department0_.DEPT_NO as DEPT_NO3_0_0_, department0_.LOCATION as LOCATION4_0_0_ from DEPARTMENT department0_ where department0_.DEPT_ID=?
    ACCOUNTING
    Hibernate: select department0_.DEPT_ID as DEPT_ID1_0_0_, department0_.DEPT_NAME as DEPT_NAM2_0_0_, department0_.DEPT_NO as DEPT_NO3_0_0_, department0_.LOCATION as LOCATION4_0_0_ from DEPARTMENT department0_ where department0_.DEPT_ID=?
    RESEARCH
    Hibernate: select department0_.DEPT_ID as DEPT_ID1_0_0_, department0_.DEPT_NAME as DEPT_NAM2_0_0_, department0_.DEPT_NO as DEPT_NO3_0_0_, department0_.LOCATION as LOCATION4_0_0_ from DEPARTMENT department0_ where department0_.DEPT_ID=?
    SALES
    Hibernate: select department0_.DEPT_ID as DEPT_ID1_0_0_, department0_.DEPT_NAME as DEPT_NAM2_0_0_, department0_.DEPT_NO as DEPT_NO3_0_0_, department0_.LOCATION as LOCATION4_0_0_ from DEPARTMENT department0_ where department0_.DEPT_ID=?
    OPERATIONS
```
  其中`query.iterate();`执行结束以后，第一条日志记录打出。Iterator中存放了四条Department的代理对象数据，当调用这个代理对象的get方法的时候，才会真正的去执行查询这个对象在数据库中每个字段的值。


### Hibernate工作原理
- 一次完整的Hibernate查询源码如下所示：
```java
public class QueryObjectDemo {
        public static void main(String[] args) {
            // 新建configuration实例对象，调用configure()方法对其进行装配
            Configuration configuration = new Configuration().configure();
            // 根据configuration实例获取SessionFactory
            SessionFactory sessionFactory = configuration.buildSessionFactory();
            // 从SessionFactory中获取一个session实例
            Session session = sessionFactory.getCurrentSession();
            // 在session中开始一次事务，只有对数据库的更新(add/update/delete)操作才需要事务
            // 事务是对数据库最基本的操作单元，四大特性是ACID
            session.beginTransaction();
            String sql = "from " + Department.class.getName();
            // 在session中创建一次查询
            Query query = session.createQuery(sql);
            // 设置查询结果的偏移量offset
            query = query.setFirstResult(0);
            // 设置查询结果的数量limit
            query = query.setMaxResults(10);
            // 执行查询
            Iterator<Department> employees = query.iterate();
            while (employees.hasNext()) {
                System.out.println(employees.next().getDeptName());
            }
            // 同样只有更新操作才需要事务
            session.getTransaction().commit();
            // 关闭Session，一级缓存失效
            session.close();
            // 关闭SessionFactory，二级缓存和查询缓存失效
            sessionFactory.close();
        }
}
```
  以上源代码展示了一次查询过程从前到后的过程，其中比较引人关注的还是新建的Configuration对象，如何通过调用configure方法来装配配置文件中的各项内容。简单来说，Configuration中持有一个StandardServiceRegistryBuilder对象实例，当调用configure方法时，若指定了地址和名字则去获取指定的配置文件，如果没有，就会获取默认的配置文件名，获取到配置文件并解析配置文件然后再将这些配置装配到StandardServiceRegistryBuilder对象实例中。下面展示`org.hibernate.cfg.Configuration`通过调用configure()进行装配的源码如下：
```java
// used to build SF
private StandardServiceRegistryBuilder standardServiceRegistryBuilder;
public Configuration configure() throws HibernateException {
		return configure( StandardServiceRegistryBuilder.DEFAULT_CFG_RESOURCE_NAME );
	}
public Configuration configure(String resource) throws HibernateException {
		standardServiceRegistryBuilder.configure( resource );
		// todo : still need to have StandardServiceRegistryBuilder handle the "other cfg.xml" elements.
		//		currently it just reads the config properties
		properties.putAll( standardServiceRegistryBuilder.getSettings() );
		return this;
	}
```
  下面展示`org.hibernate.boot.registry.StandardServiceRegistryBuilder`的部分相关源码：
```java
	// The default resource name for a hibernate configuration xml file.
	public static final String DEFAULT_CFG_RESOURCE_NAME = "hibernate.cfg.xml";
    public StandardServiceRegistryBuilder configure(String resourceName) {
		return configure( configLoader.loadConfigXmlResource( resourceName ) );
	}
    public StandardServiceRegistryBuilder configure(LoadedConfig loadedConfig) {
		aggregatedCfgXml.merge( loadedConfig );
		settings.putAll( loadedConfig.getConfigurationValues() );

		return this;
	}
    public Map getSettings() {
		return settings;
	}
```
  源码再往深处挖就是具体的获取配置文件，根据文件的类型进行解析，获取jdbc相关配置。这里就不一一展示，如果想看更详细的源码可以直接去薅hibernate-core包。解析xml在大多数开源框架中都会用到，各家的实现大致步骤基本一致，但是在细节上各有不同，有兴趣的可以挨个去看看hibernate如何解析hibernate.cfg.xml，spring如何解析spring-context等等。

- 从上一步源码我们能推算出一次查询的大致参与角色及去作用如下：
	- 新建Configuration，通过configure()方法去获取到其配置
	- 使用上一步的configuration实例对象，通过调用buildSessionFactory()中获取到SessionFactory对象实例
	- 从上一步的SessionFactory实例获取到Session实例对象，有两种方式
		- 获取一个新Session，调用openSession()方法获得
		- 获取当前已经存在的Session，调用getCurrentSession()方法获得
	- 在Session中开始一个事务，通过以下两种方式，实际上没区别:
		- session.beginTransaction();
		- session.getTransaction().begin();
	- 在session中创建一个Query实例对象，并给其指定各种查询属性，例如offset和limit以及是否启用查询缓存
	- 获取查询结果，常见的三种方式如下：
		- 通过list()获取一个List对象
		- 通过iterate()获取一个Iterator对象
		- getResultList()同样获取到一个List对象

- Hibernate中每个配置文件对应一个configuration对象。在极端情况下，不使用任何配置文件，也可以创建Configuration对象。
	- org.hibernate.cfg.Configuration实例代表一个应用程序到SQL数据库的映射配置，Configuration提供了一个buildSessionFactory()方法，该方法可以产生一个不可变的SessionFactory对象。
	- 可以直接实例化Configuration来获取一个实例，并为它指定一个Hibernate映射文件，如果映射文件在类加载路径中，则可以使用addResource()方法来添加映射定义文件。
	- hibernate支持几种不同的配置方式，这几种配置方式对应创建Configuration对象的方式也有所不同。通常有几种配置Hibernate的方式：
		- 使用hibernate.properties文件作为配置文件
		- 使用hibernate.cfg.xml文件作为配置文件
		- 不使用任何的配置文件，以编码方式来创建Configuration对象
	- Configuration对象的唯一作用就是创建SessionFactory实例，所以它才被设计成为启动期间对象，而一旦SessionFactory对象创建完成，它就被丢弃

- 从Configuration中获取到的SessionFactory是hibernate中的关键对象。SessionFactory是单个数据库映射关系经过编译后的内存镜像，它也是线程安全的。它是生成Session的工厂，本身要应用到ConnectionProvider，该对象可以在进程和集群的级别上，为那些事务之间可以重用的数据提供可选的二级缓存。
	- 连接提供者(ConnectionProvider)是生成JDBC的连接的工厂，同时具备连接池的作用，通过抽象，将底层的DataSource和DriverManager隔离开。这个对象无需应用程序直接访问，仅在应用程序需要扩展时使用。

- SessionFactory中可以获取到一个session实例，Session。是应用程序和持久存储层之间交互操作的一个单线程对象。它也是Hibernate持久化操作的关键对象，所有的持久化对象必须在Session的管理下才能够进行持久化操作。此对象的生存周期很短，其隐藏了JDBC连接，也是Transaction 的工厂。Session对象有一个一级缓存，现实执行Flush之前，所有的持久化操作的数据都在缓存中Session对象中。对于一次crud过程来说，session是必不可少的，这个session在查询的时候还会被检查状态是否为打开。Hibernate Session默认使用的是SessionImpl类。
	- Session中隐藏了事务工厂(TransactionFactory)和事务(Transaction)的实现细节。
	- TransactionFactory是生成Transaction对象实例的工厂，该对象也无需应用程序的直接访问，hibernate实现了它的访问细节，从中获取到Transaction。
	- Transaction代表一次原子操作，它具有数据库事务的概念。但它通过抽象，将应用程序从底层的具体的JDBC、JTA和CORBA事务中隔离开。在某些情况下，一个Session 之内可能包含多个Transaction对象。虽然事务操作是可选的，但是所有的持久化操作都应该在事务管理下进行，即使是只读操作。

- hibernate通过session对对象进行操作，在hibernate乃至大多数ORM框架中，对象都有三种状态，分别是：瞬时态、持久太、游离态。
	- 瞬时状态：刚创建的对象还没有被Session持久化、缓存中不存在这个对象的数据并且数据库中没有这个对象对应的数据为瞬时状态这个时候是没有OID。
	- 持久状态：对象经过Session持久化操作，缓存中存在这个对象的数据为持久状态并且数据库中存在这个对象对应的数据为持久状态这个时候有OID。
	- 游离状态：当Session关闭，缓存中不存在这个对象数据而数据库中有这个对象的数据并且有OID为游离状态。

  OID是为了在系统中能够找到所需对象，我们需要为每一个对象分配一个唯一的表示号。在关系数据库中我们称之为关键字，而在对象术语中，则叫做对象标识(Object identifier-OID).通常OID在内部都使用一个或多个大整数表示，而在应用程序中则提供一个完整的类为其他类提供获取、操作。三种状态之间的转化关系图如下：
![hibernate中对象三种状态转化图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/hibernate/hibernate中对象三种状态转化图.png)

  当对象处于持久化状态时，它一直位于 Session 的缓存中，对它的任何操作在事务提交时都将同步到数据库，因此，对一个已经持久的对象调用 save() 或 update() 方法是没有意义的。


### N+1问题
- 在前面查询过程中，我们可以看到一次查询，所得结果是四条数据，但是前后有五条日志记录。这是因为第一条查询日志，是查询所有数据条目，此时返回的是一个包装在Department类之上的代理类，当用户通过这个代理类的某些方法去访问对象的属性的时候，hibernate才会真正的通过这个对象的oid去数据库查询它各个字段的真实内容。第一次查询返回N条记录，当这N条记录的某个字段被访问的时候，又会发出N次查询去获取这N条记录所代表的真实对象，因此一共发出了N+1次查询，这就是所谓的N+1问题。

- 众所周知，访问数据库是一项非常好资源耗时间的工作，访问N条记录却需要N+1次查询，这显然不是一个最优选项。因此，各个ORM框架都通过缓存来解决N+1问题。


### 一级缓存
- 无论通过query.iterate()还是query.list()得到了数据对象的代理对象，通过访问这些代理对象而获取到真实的数据对象，这条保存了真实的数据对象将被放入hibernate内置的一级缓存之中。

- hibernate的一级缓存是session级别的，所以如果session关闭后，缓存就没了，此时就会再次发sql去查数据库。


### 二级缓存
- hibernate没有内置二级缓存，但是它支持通过配置其他缓存组件来实现二级缓存，只需要加入缓存的组件包即可，hibernate常见的二级缓存是ehcache。

- 首先，将ehcache的依赖包加入classpath或者maven项目的pom.xml文件中

- 其次，通过在hibernate.config.xml使用以下配置来配置二级缓存:
```xml
    <!-- 开启二级缓存 -->
    <property name="hibernate.cache.use_second_level_cache">true</property>
    <!-- 二级缓存的提供类 在hibernate4.0版本以后我们都是配置这个属性来指定二级缓存的提供类-->
    <property name="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</property>
    <!-- 二级缓存配置文件的位置 -->
    <property name="hibernate.cache.provider_configuration_file_resource_path">ehcache.xml</property>
```

- 因为ehcache需要配置ehcache.xml来使用缓存，所以添加ehcache.xml配置如下:
```xml
    <ehcache>
        <!--指定二级缓存存放在磁盘上的位置-->
        <diskStore path="user.dir"/>
        <!--我们可以给每个实体类指定一个对应的缓存，如果没有匹配到该类，则使用这个默认的缓存配置。各个选项的具体含义可通过ehcache.xml的官方文档查看-->
        <defaultCache
            maxElementsInMemory="10000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="true"
        />
        <!--可以给每个实体类指定一个配置文件，通过name属性指定，要使用类的全名-->
        <cache name="com.hsb.hibernate.Department"
            maxElementsInMemory="10000"
            eternal="false"
            timeToIdleSeconds="300"
            timeToLiveSeconds="600"
            overflowToDisk="true"
        />
    </ehcache>
```

- 针对具体的映射类来配置，是否使用二级缓存。
	- 配置式:
        ```xml
            <hibernate-mapping package="com.hsb.hibernate.Department">
                <class name="Department" table="department">
                    <cache usage="read-only"/>
                    <id name="id" type="int" column="id">
                        <generator class="native"/>
                    </id>
                    <property name="name" column="name" type="string"></property>
                    <property name="sex" column="code" type="string"></property>
                    <many-to-one name="address" column="rid" fetch="join"></many-to-one>
                </class>
            </hibernate-mapping>
        ```
    - 注解式：在类名上面添加`@Cache(usage=CacheConcurrencyStrategy.READ_ONLY)`，例如:
        ```java
        @Entity
        @Table(name="department")
        @Cache(usage=CacheConcurrencyStrategy.READ_ONLY)
        public class Department
        {
            ...
        }
        ```
    - 二级缓存的使用策略一般有这几种：read-only、nonstrict-read-write、read-write、transactional。通常使用二级缓存都是将其配置成read-only，即我们应当在那些不需要进行修改的实体类上使用二级缓存，否则如果对缓存进行读写的话，性能会变差，这样设置缓存就失去了意义。

- 二级缓存是sessionFactory级别的缓存，这区别于一级缓存。启用二级缓存后，系统会在session关闭的情况下，去访问二级缓存，确定二级缓存中是否有想要访问的数据，如果有，则返回数据，如果没有则发出sql查询。

- 二级缓存只会缓存数据对象，如果之前访问的不是整个数据对象，而是数据对象的某一部分字段，那这些数据将不会被加入到二级缓存中。一旦session关闭，下次用户再次访问此类数据，还是会发出sql查询。

- 通过二级缓存可以部分解决N+1问题。当用户访问一次对象以后，即便session被关闭，后续无论用户访问多少次对象，都不再进行查询，因为数据对象在二级缓存中。在对象存在大量重复查询的情况下，二级缓存可以显著的提高查询效率。

- 由于二级缓存不会缓存hql的查询语句，但是大多数时候，使用hibernate无可避免的要使用hql，因此对hql的查询语句缓存是十分有必要的。此时，我们就需要查询缓存。


### 查询缓存
- 配置查询缓存，只需要在hibernate.cfg.xml中加入一条配置即可：
```xml
    <property name="hibernate.cache.use_query_cache">true</property>
```

- 在查询hql语句时要使用查询缓存，就需要在查询语句后面设置这样一个方法:
```java
List<Department> ls = session.createQuery("from Department where name like ?")
		//开启查询缓存，查询缓存也是SessionFactory级别的缓存
		.setCacheable(true)
		.setParameter(0, "%系统%")
		.setFirstResult(0).setMaxResults(10).list();
```
  同时需要再Department类上加注释@Cacheable

- 查询缓存也是sessionFactory级别的缓存

- hql查询语句完全相同时，连参数设置都要相同，此时查询缓存才有效。此外，在没配置二级缓存的情况下，查询缓存只会缓存对象id，因此如果要发挥查询缓存的效用，应尽量保证二级缓存的配置。换句话说，在二级缓存无法满足需要的情况下，再考虑查询缓存。

### 参考
- [hibernate缓存机制详细分析](https://www.cnblogs.com/xiaoluo501395377/p/3377604.html)
- [hibernate工作原理](http://www.cnblogs.com/bile/p/4030575.html)
- [Hibernate中的三种数据持久状态和缓存机制](http://www.cnblogs.com/hcl22/p/6100191.html)