---
title: "接口管理工具调研"
date: 2024-05-27T09:25:11+08:00
draft: false
---

# 背景

由于后端接口众多，接口文档的维护不好，因此需要调研相关接口管理工具，来规范化接口文档，提高开发和沟通效率。

# 工具

## Swagger

### 介绍
  - swagger 通过代码注解将相关注释内容通过 swagger-ui 体现到接口文档上，文档会随着项目启动而实时变化。
  - 国内目前多数使用的为 springfox-swagger2（最近版本停留在 2020-07 月 https://github.com/springfox/springfox）
  - 最新的版本已经改为 springdoc（https://github.com/springdoc/springdoc-openapi）

### 使用
由于国内主要使用的 springfox-swagger2， 所以这里就主要介绍下 swagger2 注解的使用。

 https://juejin.cn/post/6993166706398986277


## Smart-doc

### 介绍
  - Smart-doc 是基于java 标准注释来生成对应的文档接口
  - 运行时代码无 smart-doc 相关内容，需要利用 mvn 插件生成对应接口文档
  - 文档不会在项目启动时实时更新，但是可以在编译期更新
  - Github: https://github.com/TongchengOpenSource/smart-doc

### 使用
https://smart-doc-group.github.io/#/zh-cn/start/guide

### 集成
https://smart-doc-group.github.io/#/zh-cn/start/quickstart

## 优劣势分析
Swagger: 
优点：使用广泛，代码中已经集成，方便前端调试
缺点：需要侵入代码，通过运行时反射机制实现，代码库不会更新

Smart-doc:
优点：无代码侵入，运行时不受影响（可以理解为分析和生成文档的工具），只写注释即可，开源库更新及时
缺点：调试能力稍弱于 swagger

# 问题记录
无论 swagger 还是 smart-doc 都无法创建我们项目内部封装的 rpc 接口。

幸运的是，通过 github 联系上了 smart-doc 的相关 Maintainer shalousun，在他的提议下，我按照他的建议提了个 PR 来支持普通的 service 接口。

通过 @javadoc 注解来标注该 service 接口需要显示出来。

https://github.com/TongchengOpenSource/smart-doc/pull/797

希望这个 pr 最终能够被合并进去。
