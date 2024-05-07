---
title: "Spring 核心知识"
date: 2024-03-10T12:11:39+08:00
draft: false
---

# 注解源码解析
- [x] @Configuration 如何扫描注解，并且构建 bean
- [x] @ComponentScan 定义 filter, 源码查看如何扫描注释，使用过滤规则
- [x] @Bean 源码查看如何 parser bean 注解信息，如何 实例化，并调用自定义的 init 和 destroy。
- [x] 从 IOC 容器中获取 Bean 的过程
- [x] @Import 注解与 bean 注解类似，不过使用起来更灵活些
- [x] @PropertySource 如何解析自定义文件，并加入到 spring 环境中