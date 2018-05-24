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
- `SecurityContextCallableProcessingInterceptor`允许所有和spring mvc的Callable接口整合的支持。当`preProcess(NativeWebRequest, Callable)`被调用时，`CallableProcessingInterceptor`会被建立在`SecurityContextHolder`添加的`SecurityContext`中。在`postProcess(NativeWebRequest, Callable, Object)`方法中，它也通过调用`SecurityContextHolder#clearContext()`来清理`SecurityContextHolder`。

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

### HeaderWriterFilter
该过滤器用于实现将头部添加到当前response中。可以被用作添加一些浏览器保护的过滤器，例如，X-Frame-Options, X-XSS-Protection and X-Content-Type-Options。简单来说，就是可以重写这个过滤器，在过滤器中完成一些安全操作，例如X-Frame-Options, X-XSS-Protection and X-Content-Type-Options之类的。
```java
	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
					throws ServletException, IOException {
		HeaderWriterResponse headerWriterResponse = new HeaderWriterResponse(request,
				response, this.headerWriters);
		try {
			filterChain.doFilter(request, headerWriterResponse);
		}
		finally {
			headerWriterResponse.writeHeaders();
		}
	}

	static class HeaderWriterResponse extends OnCommittedResponseWrapper {
		private final HttpServletRequest request;
		private final List<HeaderWriter> headerWriters;

		HeaderWriterResponse(HttpServletRequest request, HttpServletResponse response,
				List<HeaderWriter> headerWriters) {
			super(response);
			this.request = request;
			this.headerWriters = headerWriters;
		}
		@Override
		protected void onResponseCommitted() {
			writeHeaders();
			this.disableOnResponseCommitted();
		}
		protected void writeHeaders() {
			if (isDisableOnResponseCommitted()) {
				return;
			}
			for (HeaderWriter headerWriter : this.headerWriters) {
				headerWriter.writeHeaders(this.request, getHttpResponse());
			}
		}
		private HttpServletResponse getHttpResponse() {
			return (HttpServletResponse) getResponse();
		}
	}
```

### LogoutFilter
- 注销登录。调用一系列LogoutHandler，这些处理器被要求按指定的队列排序。通常，我们调用的是logout handler是TokenBasedRememberMeServices和SecurityContextLogoutHandler。
- 注销登陆后，将会发生一次重定向，重定向的地址由LogoutSuccessHandler或logoutSuccessUrl的配置决定，具体使用的url由使用哪个构造函数决定。
- logouthandler的注册在spring-security.xml中进行配置，配置好以后将id放入到http配置中的logout部分，例如：
`<logout invalidate-session="true"  logout-url="/j_spring_security_logout" success-handler-ref="logoutSuccessHandler"/>`

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();
			if (logger.isDebugEnabled()) {
				logger.debug("Logging out user '" + auth
						+ "' and transferring to logout destination");
			}
			for (LogoutHandler handler : handlers) {
				handler.logout(request, response, auth);
			}
			logoutSuccessHandler.onLogoutSuccess(request, response, auth);
			return;
		}
		chain.doFilter(request, response);
	}
protected boolean requiresLogout(HttpServletRequest request,
			HttpServletResponse response) {
		return logoutRequestMatcher.matches(request);
	}
```

### UsernamePasswordAuthenticationFilter
处理一次登录表单提交。在spring security 3.0之前调用AuthenticationProcessingFilter去完成这个工作。登录表单必须为这个过滤器准备两个参数：一个用户名和一个密码。默认使用的参数名分别为username和password，在源代码中定义如下：
```java
public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";
```
参数名可以被修改，通过调用set方法:
```java
public void setUsernameParameter(String usernameParameter) {
		this.usernameParameter = usernameParameter;
	}
public void setPasswordParameter(String passwordParameter) {
		this.passwordParameter = passwordParameter;
	}
```
默认情况下，此过滤器响应URL是/login。

通常情况下，使用者需要自己重写这个方法，并在attemptAuthentication()中去实现自己的逻辑。attemptAuthentication()方法会返回一个登录的结果实体对象`UsernamePasswordAuthenticationToken`，该类的继承结构和属性如下：
![UsernamePasswordAuthenticationToken](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/springsecurity/UsernamePasswordAuthenticationToken.jpg)
这个类在Spring security的登录中有非常重要的作用，需要十分注意。

UsernamePasswordAuthenticationFilter的类继承关系图如下：
![UsernamePasswordAuthenticationFilter](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/springsecurity/UsernamePasswordAuthenticationFilter.jpg)

UsernamePasswordAuthenticationFilter类的核心源码如下：
```Java
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}
		String username = obtainUsername(request);
		String password = obtainPassword(request);
		if (username == null) {
			username = "";
		}
		if (password == null) {
			password = "";
		}
		username = username.trim();
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);
		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```
在UsernamePasswordAuthenticationFilter#attemptAuthentication做的工作并不多，但此方法中有一句：
`this.getAuthenticationManager().authenticate(authRequest)`,这其中的getAuthenticationManager()会获取到一个用户认证管理类，而这个类的来源在spring-security中进行配置：
```xml
<beans:bean id="loginFilter"
                class="org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter">
	<beans:property name="filterProcessesUrl" value="/login"></beans:property>
	<beans:property name="authenticationSuccessHandler" ref="authenticationSuccessHandler"></beans:property>
	<beans:property name="authenticationFailureHandler" ref="authenticationFailureHandler"></beans:property>
	<beans:property name="authenticationManager" ref="authenticationManager"></beans:property>
	<beans:property name="sessionAuthenticationStrategy" ref="sas"/>
</beans:bean>
<authentication-manager alias="authenticationManager">
	<authentication-provider ref="authenticationProvider"/>
</authentication-manager>
```
这一块几乎就是spring security登录认证授权的核心部分，它的流程的大致情况如下图所示：
![spring security登录认证](https://github.com/heshengbang/heshengbang.github.io/raw/master/images/springsecurity/springsecurity登录验证流程.jpg)

简述一下过程：
- 由UsernamePasswordAuthentication执行attemptAuthention()
- 在attemptAuthention()中会调用ProviderManager或其衍生的子类的authenticate()方法
- 在ProviderManage的authenticate()方法会调用DaoAuthenticationProvider中继承自AbstractUserDetailsAuthenticationProvider中实现的authenticate()方法，该方法中调用了DaoAuthenticationProvider中实现的retrieveUser()方法
- retrieveUser()方法中调用UserDetailService中的loadUserByUsername()方法，该方法中返回一个UserDetail
- UserDetail这个对象返回到AbstractUserDetailsAuthenticationProvider的authenticate()方法中，在该方法中的核心代码如下：
```java
try {
	preAuthenticationChecks.check(user);
    //在该方法中完成用户名和密码的校验
	additionalAuthenticationChecks(user,(UsernamePasswordAuthenticationToken) authentication);
	} catch (AuthenticationException exception) {
		if (cacheWasUsed) {
			cacheWasUsed = false;
			user = retrieveUser(username,(UsernamePasswordAuthenticationToken) authentication);
			preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user,(UsernamePasswordAuthenticationToken) authentication);
		} else {
			throw exception;
		}
	}
	postAuthenticationChecks.check(user);
```
该方法中完成了用户的用户名和密码校验，执行完毕后就获得一个UsernamePasswordAuthenticationToken实体对象，这个对象中包含了用户已经被认证的flag。加入用户名密码错误，则会在其中直接判处错误。
