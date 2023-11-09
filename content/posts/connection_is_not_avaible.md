---
title: "Connection_is_not_avaible"
date: 2023-11-09T16:03:41+08:00
draft: false
---
# Connection is not available, request timed out after 30000ms
最近碰到一个这个异常，前端是 xxl-job 任务，后端将 shardingsphere-proxy 作为数据源来使用。

1. 偶发，频率比较高

> 网络上相关解释
从 Hikari 拿连接，但是拿不到，要不就是达到最大连接数了
https://stackoverflow.com/questions/32968530/hikaricp-connection-is-not-available

2. Mysql 连接数上看，两台实例 205， 206 都是没有超过 1000 最大连接数 20000

3. 这个是业务的报错，想拿 connection 拿不到，不确定是 proxy 报的还是业务报的， proxy 无异常日志
根据日志找到了业务的库配置
```
spring.datasource.dynamic.datasource.sharding0.driver-class-name = com.mysql.jdbc.Driver
spring.datasource.dynamic.datasource.sharding0.druid.initial-size = 5
spring.datasource.dynamic.datasource.sharding0.druid.max-active = 100
spring.datasource.dynamic.datasource.sharding0.druid.max-wait = 60000
spring.datasource.dynamic.datasource.sharding0.druid.min-idle = 5
spring.datasource.dynamic.datasource.sharding0.druid.validation-query = select 1
spring.datasource.dynamic.datasource.sharding0.password = root
spring.datasource.dynamic.datasource.sharding0.url = jdbc:mysql://xxx?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8&useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.dynamic.datasource.sharding0.username = root
```
奇怪的是为什么是 druid 连接池，但是却没有 druid 相关信息?
>客户反馈 druid 的链接信息没有生效，实际使用的是 hikari，并且应该是采用了默认配置，那也就是最大连接数是 10

查实修改下连接池配置
```
spring.datasource.dynamic.datasource.sharding0.hikari.minimum-idle = 5
spring.datasource.dynamic.datasource.sharding0.hikari.maximum-pool-size = 100
spring.datasource.dynamic.datasource.sharding0.hikari.connection-timeout = 40000
```
发现报错信息变成了
sharding0 - Connection is not available, request timed out after 40000ms.
>说明参数生效了，另外也说明了的确是用户业务 hikari 连接池 报出的异常信息

难道 100 的连接池还不够，试试 500 ，结果还是不行，难道是客户业务代码有 bug？

还是先证明不是 proxy 的问题吧，考虑监控下发送到 proxy 的连接，看看究竟有多少
> Proxy 的外部连接数，从停止服务的 2 ,启动服务后增长到  52, 启动 job 一直到 job 失败，连接数都是 52

```
# 统计连接到 3307 端口的外部链接数量
netstat -n | grep 3307 | wc -l
```
既然确定是业务的问题，那么就从 业务 hikari 连接池找原因吧，首先打开 hikari 的 debug 日志

```
# 放到 application.properties 中
logging.level.com.zaxxer.hikari.HikariConfig = DEBUG
logging.level.com.zaxxer.hikari = TRACE
```

发现连接池参数没生效，还是 10
难道是写法错误？

尝试修改配置项

Dynamic datasource 的文档居然要收费。。。

https://www.kancloud.cn/tracy5546/dynamic-datasource/2270657

交钱是不可能交钱的。

下载源码后，发现有 spring 的相关提示

原来 dynamic datasource 配置的 hikari 最大连接数是。

我怀疑是为了赚钱，故意的。。
```
spring.datasource.dynamic.datasource.sharding0.hikari.min-idle = 5
spring.datasource.dynamic.datasource.sharding0.hikari.max-pool-size = 30
```
问题解决。