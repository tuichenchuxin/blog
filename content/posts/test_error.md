---
title: "本地测试可以运行，但是环境上 mockito java.lang.VerifyError"
date: 2024-11-06T18:16:08+08:00
draft: false
---

# 问题描述
```
org.mockito.exceptions.base.MockitoException: 

Mockito cannot mock this class: class com.digiwin.athena.datamap.service.inner.DataMapPickService.

If you're not sure why you're getting this error, please open an issue on GitHub.


Java               : 1.8
JVM vendor name    : Oracle Corporation
JVM vendor version : 25.11-b03
JVM name           : Java HotSpot(TM) 64-Bit Server VM
JVM version        : 1.8.0_11-b12
JVM info           : mixed mode
OS name            : Mac OS X
OS version         : 10.16


You are seeing this disclaimer because Mockito is configured to create inlined mocks.
You can learn about inline mocks and their limitations under item #39 of the Mockito class javadoc.

Underlying exception : java.lang.IllegalArgumentException: Could not create type
Caused by: java.lang.IllegalArgumentException: Could not create type
Caused by: java.lang.VerifyError
```

一直报错，但是本地却没有。

# 调查过程
查询发现 Caused by: java.lang.VerifyError 可能是由于 jvm 验证到内存和class 可能不同，报出来的问题。

于是下载对应版本，本地复现，确实能复现该问题。

那么是哪个类不一致呢。

```
org.mockito.exceptions.base.MockitoException: 
Mockito cannot mock this class: class org.springframework.data.mongodb.core.MongoTemplate.

If you're not sure why you're getting this error, please open an issue on GitHub.


Java               : 1.8
JVM vendor name    : Oracle Corporation
JVM vendor version : 25.11-b03
JVM name           : Java HotSpot(TM) 64-Bit Server VM
JVM version        : 1.8.0_11-b12
JVM info           : mixed mode
OS name            : Mac OS X
OS version         : 10.16


You are seeing this disclaimer because Mockito is configured to create inlined mocks.
You can learn about inline mocks and their limitations under item #39 of the Mockito class javadoc.

Underlying exception : java.lang.IllegalArgumentException: Could not create type

	at org.mockito.junit.jupiter.MockitoExtension.beforeEach(MockitoExtension.java:153)
	at java.util.ArrayList.forEach(ArrayList.java:1234)
	at java.util.ArrayList.forEach(ArrayList.java:1234)
	Suppressed: java.lang.NullPointerException
		at org.mockito.junit.jupiter.MockitoExtension.afterEach(MockitoExtension.java:184)
		... 2 more
Caused by: java.lang.IllegalArgumentException: Could not create type
	at net.bytebuddy.TypeCache.findOrInsert(TypeCache.java:170)
	at net.bytebuddy.TypeCache$WithInlineExpunction.findOrInsert(TypeCache.java:399)
	at net.bytebuddy.TypeCache.findOrInsert(TypeCache.java:190)
	at net.bytebuddy.TypeCache$WithInlineExpunction.findOrInsert(TypeCache.java:410)
	at org.mockito.internal.creation.bytebuddy.TypeCachingBytecodeGenerator.mockClass(TypeCachingBytecodeGenerator.java:40)
	at org.mockito.internal.creation.bytebuddy.InlineDelegateByteBuddyMockMaker.createMockType(InlineDelegateByteBuddyMockMaker.java:396)
	at org.mockito.internal.creation.bytebuddy.InlineDelegateByteBuddyMockMaker.doCreateMock(InlineDelegateByteBuddyMockMaker.java:355)
	at org.mockito.internal.creation.bytebuddy.InlineDelegateByteBuddyMockMaker.createMock(InlineDelegateByteBuddyMockMaker.java:334)
	at org.mockito.internal.creation.bytebuddy.InlineByteBuddyMockMaker.createMock(InlineByteBuddyMockMaker.java:56)
	at org.mockito.internal.util.MockUtil.createMock(MockUtil.java:99)
	at org.mockito.internal.MockitoCore.mock(MockitoCore.java:88)
	at org.mockito.Mockito.mock(Mockito.java:2037)
	at org.mockito.internal.configuration.MockAnnotationProcessor.processAnnotationForMock(MockAnnotationProcessor.java:74)
	at org.mockito.internal.configuration.MockAnnotationProcessor.process(MockAnnotationProcessor.java:28)
	at org.mockito.internal.configuration.MockAnnotationProcessor.process(MockAnnotationProcessor.java:25)
	at org.mockito.internal.configuration.IndependentAnnotationEngine.createMockFor(IndependentAnnotationEngine.java:44)
	at org.mockito.internal.configuration.IndependentAnnotationEngine.process(IndependentAnnotationEngine.java:72)
	at org.mockito.internal.configuration.InjectingAnnotationEngine.processIndependentAnnotations(InjectingAnnotationEngine.java:73)
	at org.mockito.internal.configuration.InjectingAnnotationEngine.process(InjectingAnnotationEngine.java:47)
	at org.mockito.MockitoAnnotations.openMocks(MockitoAnnotations.java:81)
	at org.mockito.internal.framework.DefaultMockitoSession.<init>(DefaultMockitoSession.java:43)
	at org.mockito.internal.session.DefaultMockitoSessionBuilder.startMocking(DefaultMockitoSessionBuilder.java:83)
	... 3 more
Caused by: java.lang.VerifyError
	at sun.instrument.InstrumentationImpl.retransformClasses0(Native Method)
	at sun.instrument.InstrumentationImpl.retransformClasses(InstrumentationImpl.java:144)
	at org.mockito.internal.creation.bytebuddy.InlineBytecodeGenerator.triggerRetransformation(InlineBytecodeGenerator.java:280)
	at org.mockito.internal.creation.bytebuddy.InlineBytecodeGenerator.mockClass(InlineBytecodeGenerator.java:217)
	at org.mockito.internal.creation.bytebuddy.TypeCachingBytecodeGenerator.lambda$mockClass$0(TypeCachingBytecodeGenerator.java:47)
	at org.mockito.internal.creation.bytebuddy.TypeCachingBytecodeGenerator$$Lambda$342/1424439581.call(Unknown Source)
	at net.bytebuddy.TypeCache.findOrInsert(TypeCache.java:168)
	... 24 more
```

debug 也没有太多的发现。

如果去掉这个依赖，就可以规避这个问题。
```
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-inline</artifactId>
      <version>${mockito.version}</version>
      <scope>test</scope>
    </dependency>
```

在 mockito github 上也找到了类似的 issue
```
https://github.com/mockito/mockito/issues/2079
```

不过没有后续了。

# 暂时结论

先去掉 mockito-inline jar 包了，不用 mockStatic 方式。

定位起来这个问题比较麻烦，盲点比较多，mockito 框架可以进入我的待贡献清单了。

