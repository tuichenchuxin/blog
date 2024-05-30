---
title: "smart-doc 支持普通 service 接口或者公司内部 rpc 微服务接口"
date: 2024-05-30T17:24:35+08:00
draft: false
---

最近调研了接口文档生成工具，发现了 smart-doc，但是不支持公司内部的微服务，于是在 issue 上相同问题的内容下提问了下

https://github.com/TongchengOpenSource/smart-doc/issues/683

没想到社区很快回复了，并列出了可行思路。

https://github.com/TongchengOpenSource/smart-doc/discussions/795

我看了下思路，又看了下代码，发现实现起来比想象的要简单很多。

于是捣鼓了半天，提交了代码

https://github.com/TongchengOpenSource/smart-doc/pull/797
https://github.com/TongchengOpenSource/smart-doc-maven-plugin/pull/70

很快经过 review 修改后合并了。

哈哈，开心。

以后只要增加 @javadoc 注释，就可以支持普通 service 接口了。