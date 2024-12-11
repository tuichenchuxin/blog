---
title: "idea 修改配置，模拟分析 OOM"
date: 2024-12-11T16:26:15+08:00
draft: false
---

# 背景
写了个功能，线上一跑，pod 就重启了。估计是 OOM 了。

# 调查过程
于是本地准备模拟一下场景，分析下大对象。

修改运行时参数
```
-Xms200m
-Xmx200m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/Users/chenchuxin/Documents/dgw/km-deployer-service/
```
![](/img/DA2B35CED19477FA7744087DC1F0444A.png)

这样可以得到 hprof 文件，idea 就可以分析

![](/img/759F5584B8D11999A990F25B4EC66029.png)

很容易发现大对象是 bson 文件，看起来是 1000 一批还是太多了，因为是 document 对象，所以可能存储很多内容。

调下批次处理数量就可以了。