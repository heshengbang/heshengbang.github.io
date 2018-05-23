---
layout: post
title: Spring Security登录过滤链分析
date: 2018-5-23
tags: Spring
---

## Spring Security登录过滤链分析

### Spring Security登录过滤链总览
- org.springframework.security.web.context.SecurityContextPersistenceFilter
- org.springframework.security.web.session.ConcurrentSessionFilter*
- org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter
- org.springframework.security.web.header.HeaderWriterFilter
- org.springframework.security.web.authentication.logout.LogoutFilter
- org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter*
- org.springframework.security.web.savedrequest.RequestCacheAwareFilter
- org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter
- org.springframework.security.web.authentication.AnonymousAuthenticationFilter
- org.springframework.security.web.session.SessionManagementFilter
- org.springframework.security.web.access.ExceptionTranslationFilter
- org.springframework.security.access.intercept.AbstractSecurityInterceptor*
- org.springframework.security.web.access.intercept.FilterSecurityInterceptor

以上过滤器，从上之下，依次过滤。标*的是用户可能需要根据自己具体实现重写的。

### SecurityContextPersistenceFilter
- 在每次请求之前，检查是否有SecurityContext被放入的标识
`static final String FILTER_APPLIED = "__spring_security_scpf_applied";`
	- 如果有，则进入过滤链执行，执行结束返回,不执行任何其他代码
- 如果没有，则放入标识
- 从SecurityContextRepository中获取配置好的SecurityContext填充到SecurityContextHolder
- 进入过滤链执行
- 过滤链执行结束后回到当前这个过滤器，将SecurityContext获取到
- 从SecurityContextHolder中清除这些信息
- 将上一步获取的SecurityContext存储SecurityContextRepository

默认使用HttpSessionSecurityContextRepository。
核心代码如下：
```java
HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
		SecurityContext contextBeforeChainExecution = repo.loadContext(holder);
		try {
			SecurityContextHolder.setContext(contextBeforeChainExecution);
			chain.doFilter(holder.getRequest(), holder.getResponse());
		}
		finally {
			SecurityContext contextAfterChainExecution = SecurityContextHolder
					.getContext();
			SecurityContextHolder.clearContext();
			repo.saveContext(contextAfterChainExecution, holder.getRequest(),
					holder.getResponse());
			request.removeAttribute(FILTER_APPLIED);

			if (debug) {
				logger.debug("SecurityContextHolder now cleared, as request processing completed");
			}
		}
```
其中`repo.loadContext(holder)`的实现细节如下：
```java
public SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder) {
		HttpServletRequest request = requestResponseHolder.getRequest();
		HttpServletResponse response = requestResponseHolder.getResponse();
		HttpSession httpSession = request.getSession(false);
		SecurityContext context = readSecurityContextFromSession(httpSession);
		if (context == null) {
			if (logger.isDebugEnabled()) {
				logger.debug("No SecurityContext was available from the HttpSession: "
						+ httpSession + ". " + "A new one will be created.");
			}
			context = generateNewContext();
		}
		SaveToSessionResponseWrapper wrappedResponse = new SaveToSessionResponseWrapper(
				response, request, httpSession != null, context);
		requestResponseHolder.setResponse(wrappedResponse);
		if (isServlet3) {
			requestResponseHolder.setRequest(new Servlet3SaveToSessionRequestWrapper(
					request, wrappedResponse));
		}
		return context;
	}
```
可以查看`HttpSession`类中和SecurityContextRepository配置相关的选项。

- 这个过滤器只会在每次请求执行一次，以此来解决servlet容器的兼容性问题
- 这个过滤器必须被执行在任何认证进程机制之前
- 认证进程机制（Authentication processing mechanisms）例如:BASIC、CAS进程过滤器等等，它们期望在它们执行时，`SecurityContextHolder`包含一个合法可用的`SecurityContext`

### ConcurrentSessionFilter
- 该过滤器是并发Session处理器所需要的过滤器
- 该过滤器包含两个功能
	- 首先它为每次请求调用`{@link org.springframework.security.core.session.SessionRegistry#refreshLastRequest(String)}`，这样被注册的Session都会有一个正确的最近更新日期/时间
	- 其次，它为每次请求，从SessionRegistry中获取一个`org.springframework.security.core.session.SessionInformation`，并且检查session是否被标记为过期。如果它已经被标记为过期，那么配置的登出处理器（logout handlers）将被调用，就像LogoutFilter做的那样，典型的，使session无效。指向配置好的过期url的重定向将会发生，并且session无效之后会导致`HttpSessionDestroyedEvent`被注册在web.xml中的`HttpSessionEventPublisher`发布
- 基本操作
	- 通过SessionRegistry获取SessionInformation
		- 如果SessionInformation为空则进入过滤链
	- 如果SessionInformation不为空，则判断是否过期
		- 如果不过期，则刷新SessionRegistry中的最近一次请求时间
	- 如果过期，则执行logout操作
		- 从SecurityContextHolder中获取Authentication
		- 挨个在LogoutHandler中处理Authentication
	- 获取过期跳转的url
		- 如果过期url为空则返回一段文字，并刷新response的缓冲区
	- 如果过期跳转url不为空，则重定向到过期url
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		HttpSession session = request.getSession(false);
		if (session != null) {
			SessionInformation info = sessionRegistry.getSessionInformation(session
					.getId());
			if (info != null) {
				if (info.isExpired()) {
					doLogout(request, response);

					String targetUrl = determineExpiredUrl(request, info);

					if (targetUrl != null) {
						redirectStrategy.sendRedirect(request, response, targetUrl);

						return;
					}
					else {
						response.getWriter().print(
								"This session has been expired (possibly due to multiple concurrent "
										+ "logins being attempted as the same user).");
						response.flushBuffer();
					}

					return;
				}
				else {
					sessionRegistry.refreshLastRequest(info.getSessionId());
				}
			}
		}
		chain.doFilter(request, response);
	}

    private void doLogout(HttpServletRequest request, HttpServletResponse response) {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
	//org.springframework.security.web.authentication.logout.SecurityContextLogoutHandler
		for (LogoutHandler handler : handlers) {
			handler.logout(request, response, auth);
		}
	}
```
- 使用者可以根据自己的实际业务需要，重写此过滤器，自定义session失效时间，处理非法请求

### WebAsyncManagerIntegrationFilter
该过滤器通过使用
`SecurityContextCallableProcessingInterceptor#beforeConcurrentHandling(org.springframework.web.context.request.NativeWebRequest, Callable)`来整合SecurityContext和spring web的WebAsyncManager，在callable中填充SecurityContext。

- WebAsyncManager是管理异步请求的核心类，一般作为SPI，通常不直接由应用程序类使用
- SPI 全称为 (Service Provider Interface) ,是JDK内置的一种服务提供发现机制
- `SecurityContextCallableProcessingInterceptor`允许所有和spring mvc的Callable接口支持的整合。当`preProcess(NativeWebRequest, Callable)`被调用时，`CallableProcessingInterceptor`会被建立在`SecurityContextHolder`添加的`SecurityContext`中。在`postProcess(NativeWebRequest, Callable, Object)`方法中，它也通过调用`SecurityContextHolder#clearContext()`来清理`SecurityContextHolder`。

核心代码如下：
```java
protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		SecurityContextCallableProcessingInterceptor securityProcessingInterceptor = (SecurityContextCallableProcessingInterceptor) asyncManager
				.getCallableInterceptor(CALLABLE_INTERCEPTOR_KEY);
		if (securityProcessingInterceptor == null) {
			asyncManager.registerCallableInterceptor(CALLABLE_INTERCEPTOR_KEY,
					new SecurityContextCallableProcessingInterceptor());
		}

		filterChain.doFilter(request, response);
	}
```
虽然看了源码和注释，但是还是没搞懂这个类是用来干嘛的。

