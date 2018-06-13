---

layout: post

title: Hibernate的原理和缓存机制

date: 2018-6-12

tags: Hibernate

---

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
			applyQuerySettingsAndHints( query );
			return query;
		}
		catch (RuntimeException e) {
			throw exceptionConverter.convert( e );
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
- 上一节的代码示例中我们能看到，对于一次查询过程来说，session是必不可少的，这个session在查询的时候还会被检查状态是否为打开。Hibernate Session默认使用的是SessionImpl类。
- 在hibernate乃至大多数ORM框架中，对象都有三种状态，分别是：瞬时态、持久太、游离态。
	- 瞬时状态：刚创建的对象还没有被Session持久化、缓存中不存在这个对象的数据并且数据库中没有这个对象对应的数据为瞬时状态这个时候是没有OID。
	- 持久状态：对象经过Session持久化操作，缓存中存在这个对象的数据为持久状态并且数据库中存在这个对象对应的数据为持久状态这个时候有OID。
	- 游离状态：当Session关闭，缓存中不存在这个对象数据而数据库中有这个对象的数据并且有OID为游离状态。
  OID为了在系统中能够找到所需对象，我们需要为每一个对象分配一个唯一的表示号。在关系数据库中我们称之为关键字，而在对象术语中，则叫做对象标识(Object identifier-OID).通常OID在内部都使用一个或多个大整数表示，而在应用程序中则提供一个完整的类为其他类提供获取、操作。三种状态之间的转化关系图如下：
![hibernate中对象三种状态转化图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/hibernate/hibernate中对象三种状态转化图.png)

  当对象处于持久化状态时，它一直位于 Session 的缓存中，对它的任何操作在事务提交时都将同步到数据库，因此，对一个已经持久的对象调用 save() 或 update() 方法是没有意义的。


### N+1问题
- 使用Hibernate或mybatis之类的ORM框架中，普遍存在N+1问题。例如，以Hibernate为例，存在数据表department与实体对象Department相对应，使用以下代码进行数据查询：

### 一级缓存
### 二级缓存
### 查询缓存

### 参考
- [hibernate缓存机制详细分析](https://www.cnblogs.com/xiaoluo501395377/p/3377604.html)
- [hibernate工作原理](http://www.cnblogs.com/bile/p/4030575.html)
- [Hibernate中的三种数据持久状态和缓存机制](http://www.cnblogs.com/hcl22/p/6100191.html)