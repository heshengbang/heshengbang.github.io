---
layout: post
title: SpringMVC个人解读
date: 2018-5-27
tags: Spring
---

**本文基于Spring MVC 4.3.12**

### SpringMVC的工作流程图
![SpringMVC的工作流程图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/SpringMVC的工作流程图.jpg)

- DispatcherServlet工作流程简述
	1. 用户发送请求至前端控制器，DispatcherServlet#doService
	  	- doService()用于暴露出DispatcherServlet特有的请求属性和委托给doDispatch()以进行实际调度
		- doService()中调用doDispatch()处理真正的request转发调度
	2. DispatcherServlet#doDispatch()中寻找确定当前请求的HandlerAdapter
		- 处理器映射器找到具体的处理器(可以根据xml配置、注解进行查找)，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet
	3. 调用HandlerAdapter#handle()，这里会适配调用具体的处理器，后端控制器(XXXController)
		- web后台Controller常见的@RequestMapping()注解使用RequestMappingHandlerAdapter来处理请求
	4. Controller执行完成返回ModelAndView
	5. HandlerAdapter将Controller执行结果ModelAndView返回给DispatcherServlet
	6. DispatcherServlet将ModelAndView传给视图解析器(ViewReslover)
	7. ViewReslover解析后返回具体View
	8. DispatcherServlet根据View进行渲染视图（即将模型数据填充至视图中）
	9. DispatcherServlet响应用户

- doDispatch()核心源码如下：
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;
        //获取当前请求的{@link WebAsyncManager}，或者如果找不到，请创建它并将其与请求关联
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
            	//将请求转换为multipart request请求，并使multipart resolver可用
                //如果没有设置multipart resolver，就使用已经存在的request
                //A HTTP multipart request is a HTTP request that HTTP clients construct to send files and data over to a HTTP Server.
                //It is commonly used by browsers and HTTP clients to upload files to the server.
				processedRequest = checkMultipart(request);
                //如果处理后的request不等于之前的request，那么就设置为true
				multipartRequestParsed = (processedRequest != request);

				// 确定当前请求的处理器
                //getHandler(processedRequest)方法用于按顺序尝试所有处理映射，直至找到并为此请求返回HandlerExecutionChain
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
                	//找不到处理程序 - >设置适当的HTTP响应状态
					noHandlerFound(processedRequest, response);
					return;
				}

				// 由当前处理器确定当前请求的处理器适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// 处理最后修改的头部（如果处理程序支持）
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
                	// 根据提供的最后修改时间戳（由应用程序确定）检查请求的资源是否已被修改
                    // 这也适用于透明地设置“Last-Modified”响应头和HTTP状态的时候
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
                // 执行所有处理器中注册的拦截器的preHandle方法，如果执行结果失败，直接返回，如果成功则下一步
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// 这里完成了真正的请求转发处理.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                // 当前请求的选定处理程序是否可以接受异步处理请求
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
                // 给视图设置默认的名字
				applyDefaultViewName(processedRequest, mv);
                // 执行所有处理器中注册的拦截器的postHandle方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (XXX ex) {
				...
			}
            // 处理 处理选择器或处理器调用的结果，可能是视图或异常都会被解析
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (XXX ex) {
			...
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

- 使用DispatcherServlet需要在web.xml中配置
```xml
<servlet>
	<servlet-name>appServlet</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>xxx/*-servlet.xml</param-value>
		</init-param>
	<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>appServlet</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
```
	- load-on-startup标记容器是否在启动的时候就加载这个servlet，标记容器是否在启动的时候就加载这个servlet
	- DispatcherServlet的类体系结构图如下：
	![DispatcherServlet类体系结构图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/DispatcherServlet类体系结构图.jpg)

- SpringMVC组件说明
	- 前端控制器DispatcherServlet
	  接收请求，响应结果，相当于转发器，中央处理器。有了dispatcherServlet减少了其它组件之间的耦合度。用户请求到达前端控制器，它就相当于mvc模式中的c，dispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，dispatcherServlet的存在降低了组件之间的耦合性。
	- 处理器适配器HandlerAdapter
	  按照特定规则（HandlerAdapter要求的规则）去执行Handler，通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行
	- 处理器映射器HandlerMapping
	  根据请求的url查找Handler，HandlerMapping负责根据用户请求找到Handler处理器，springmvc提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等
	- 处理器Handler
	  Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理
	- 视图解析器View resolver
	  进行视图解析，根据逻辑视图名解析成真正的视图（view）View Resolver负责将处理结果生成View视图，View Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。 springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等
	- 视图View
	  View是一个接口，实现类支持不同的View类型（jsp、freemarker、pdf...）

- MVC的原理图
![springmvc原理图](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/spring/springmvc原理图.png)

	- M-Model 模型（完成业务逻辑：有javaBean构成，service+dao+entity）
	- V-View 视图（做界面的展示  jsp，html……）
	- C-Controller 控制器（接收请求—>调用模型—>根据结果派发页面）