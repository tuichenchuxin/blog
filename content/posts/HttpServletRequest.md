---
title: "HttpServletRequest.setAttribute 不支持并发问题调查"
date: 2024-08-16T09:06:03+08:00
draft: false
---

# 背景
最近发现一个 bug, 有时返回数据，有时返回null。

# 过程
初步调查发现，应该是报错，但是异常被吞了，导致返回 null。 本地调用无法复现，推测可能是并发场景下的报错。

于是通过 postman 来并发调用接口，果然复现了异常。

Request.notifyAttributeAssigned 时由于 context 对象为 null 因此异常。
```log
java.lang.NullPointerException
	at org.apache.catalina.connector.Request.notifyAttributeAssigned(Request.java:1555)
	at org.apache.catalina.connector.Request.setAttribute(Request.java:1541)
	at org.apache.catalina.connector.RequestFacade.setAttribute(RequestFacade.java:540)
	at javax.servlet.ServletRequestWrapper.setAttribute(ServletRequestWrapper.java:293)
	at javax.servlet.ServletRequestWrapper.setAttribute(ServletRequestWrapper.java:293)
	at com.digiwin.athena.knowledgegraph.configuration.KgCacheAspect.around(KgCacheAspect.java:48)
	at sun.reflect.GeneratedMethodAccessor183.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:644)
	at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:633)
	at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:70)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:174)
	at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:185)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:689)
	at com.digiwin.athena.knowledgegraph.service.impl.TaskService$$EnhancerBySpringCGLIB$$73d3828b.postActivityDefinition(<generated>)
	at com.digiwin.athena.knowledgegraph.service.impl.TaskService.lambda$null$98(TaskService.java:3799)
	at java.util.concurrent.CompletableFuture$AsyncRun.run$$$capture(CompletableFuture.java:1640)
	at java.util.concurrent.CompletableFuture$AsyncRun.run(CompletableFuture.java)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:750)
```

那说明有线程 clear context

加了一些 debug 日志条件发现如下现象

46 线程访问完毕后，清除这个对象，但是 45号线程内部的线程池线程 18 正在访问。

为什么 45号内部的线程会访问不属于 45号的对象 2be669ff 而是用 15e2628d

```
http-nio-8085-exec-46 mappingData is clear contexts:org.apache.catalina.mapper.MappingData@15e2628d

nullhttp-nio-8085-exec-45 notifyorg.apache.catalina.mapper.MappingData@2be669ff:StandardEngine[Tomcat].StandardHost[localhost].TomcatEmbeddedContext[]

http-nio-8085-exec-45 pool-18-thread-1 notifyorg.apache.catalina.mapper.MappingData@15e2628d:null

```


```java
	@Nullable
	public static RequestAttributes getRequestAttributes() {
		RequestAttributes attributes = requestAttributesHolder.get();
		if (attributes == null) {
			attributes = inheritableRequestAttributesHolder.get();
		}
		return attributes;
	}
```

一般线程会取到从外部继承的，但是为什么这个跟外部继承的不一致呢？ 

推测是由于内部使用了线程池。线程池的中线程继承的外部 threadLocal 对象。当外部存在并发时，内部调用 inheritableRequestAttributesHolder.get() 可能拿到的并非是实际外部的线程中的 threadLocal。

例如 nio 线程 1 分配了 thread-1
nio 线程 2 分配了 thread-2
nio 线程3 分配了 thread-1
这是 thread-1 拿到的可能是 3 的 threadLocal 对象，那么就会出现错乱。

# 解决方案

既然推测是线程继承导致的，那么就不用使用继承了，直接用自己的 threadLocal 这样一个线程就一个了，应该就可以了。

RequestContextHolder.setRequestAttributes(requestAttributes);