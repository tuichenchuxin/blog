---
title: "单元测试单独运行ok, mvn test 却失败"
date: 2024-11-06T11:37:11+08:00
draft: false
---

# 现象
单独运行单元测试都是通过的，但是 mvn test 一起运行时就报错了。
# 调查过程
开启 debug 模式，执行mvn test 时，发现 threadLocal 对象的值来自于其它 case.

由于同一个模块执行 test 时是单线程串行的。所以应该是 treadLocal 泄漏。

从而发现是测试用例没有清理 treadLocal.

增加清理逻辑后正常。