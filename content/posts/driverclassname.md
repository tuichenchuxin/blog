---
title: "No suitable driver 问题调查"
date: 2023-08-23T17:55:43+08:00
draft: false
categories: ["Tech"]
---
# 背景
用户使用 shardingsphere 发现 No suitable driver 异常。但是用户的 jar 包中是有该驱动的。
# 调查
发现如果增加这段代码后，可以正常执行
```java
Class.forName("com.mysql.cj.jdbc.Driver");
```
正常而言，按照 JDBC4.0 标准，驱动是通过 spi 机制加载。不需要手动加载。
通过 debug DriverManager 发现其中有个驱动的名称中包含了 `-` 不是合法的 java 名称。
但是会将异常吞掉，所以没有展示出来。
修复名称后，可以正常使用。

# 相关知识
JDBC4.0 是通过 spi 加载。
`-` 不是合法名称，使用 shade 插件替换名称时要注意。
DriverManager 加载多个驱动过程中，如果有任何一个失败，都不会再继续加载，并不会抛出相关异常。