---
title: "数据不一致问题调查"
date: 2024-08-22T16:08:26+08:00
draft: false
---

# 背景
还是老问题，线上接口多次请求数据不一致。

# 过程
通过本地程序并发测试，本地无法复现。

并发测试线上接口，可以复现。

查看日志，发现日志打印不全。不要使用 log.error("xxxx:{}", e.getMessage())

修改日志, 打出异常堆栈。
```java
log.error("xxxx", e)
```

发现在并发中，调用会报空指针异常。
```java
HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
```
当前线程是没有设置的，但是代码里会调用继承的, 这里本地并发测试无法复现，但是线上会出现。
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
这个内部的原因暂时还未知，需要了解下继承 threadLoacal 和内部线程流转。