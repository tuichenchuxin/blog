---
title: "redis learn"
date: 2025-03-28T09:20:48+08:00
draft: false
---

# 契机
最近在项目中修改了个 redis 的超时问题，发现对 redis 的 api 以及实现原理都一知半解。这确实说不过去啊。

所以准备把 redis 学一学。最好是能深入到源码级别。

学习方式依然是准备找到开源社区，参与进来。

# 查找容易参与的相关项目

现在找到了一个可行的方向，就是在 calcite 去贡献一些 redis 的连接器的实现。

这样有助于我了解 calcite 和 redis 的一些操作协议。 从 redis 的连接和命令来学习 redis 就跟学习 mysql 类似吧。

关于 redis 本身的实现，我觉得学习会比较困难，首先是用 c 语言写的，其次是 redis 的启动核心实现的逻辑等等，这些还需要进一步安排学习。

# 启动 redis + calcite

查找可以贡献的点。

官网里有 https://calcite.apache.org/docs/redis_adapter.html 
Future plan: More Redis features need to be further refined: for example HyperLogLog and Pub/Sub.

那我感觉可以朝这个方向努力下

这个暂时先等等，后续等我的 mongoadapter 被合并了之后，我再找点儿 redis adapter 来尝试下。

# build-your-own-x 
找到了这个项目，看起来是集成了许多项目的初始指导。

我找到了个 rust 创建 mini-redis 客户端的项目。

https://tokio.rs/tokio/tutorial/setup

来尝试一下哈

看了半天，发现是教你如何使用 tokio 的，跟 redis 的关系不大，不过tokio 也可以学一学吧，所以就把它看完吧。

实在看不动了，先记录吧，下次学习 tokio 多线程之类的时候，可以看看 https://tokio.rs/tokio/tutorial/async

还是继续搞一下 calcite + redis 的项目吧，这个有及时的反馈，应该还是能搞得下去的。

# calcite redis

## redis 的相关操作熟悉
我去，redis 的命令真多呢。

先来粗略的看一下这个 redis 教程 https://redis.com.cn/redis-intro.html

连接 redis
```
docker run -it --rm redis redis-cli -h host.docker.internal -p 6379
```

## redis HyperLogLog

计算基数（非重复数据），一个 key 的内存是固定的，内存占用特别少。

可以计算总数，精度99%以上，无法返回数据。

考虑如何整合到 calcite 上


现在考虑 hyperLogLog 一共就三个操作。

```
PFADD
PFCOUNT
PFMERGE
```

现在两个思路
1. 一个key 对应一个表，表里只有一条数据 estimate （基数）

2. 定义schema 的时候，匹配到多个key，把这些 key以及对应的 estimate 作为两列查询
    > 这个需要看是否支持模糊匹配到对应的 key

考虑先按照第二种方式来调研一下

现在看来，是可以通过配置模糊匹配的方式来定义表

```
{
  "version": "1.0",
  "defaultSchema": "redis",
  "schemas": [
    {
      "name": "redis",
      "type": "custom",
      "factory": "org.apache.calcite.adapter.redis.RedisSchemaFactory",
      "operand": {
        "host": "localhost",
        "database": "0",
        "tables": [
          {
            "name": "uv_stats",          // 表名
            "dataFormat": "hyperloglog", // 标识为 HLL 类型
            "keyPattern": "uv:*"         // 匹配的 Redis 键模式
          }
        ]
      }
    }
  ]
}
```

大概效果如下
```sql
select * from uv_status;

KEYS | CARDINAL_NUMBER
uv:20240910 | 2
uv:20240911 | 3
```

### 那么接下来就是考虑实现啦

我提了一个 jira 到 calcite 来支持这个，并给出了相关设计。接下来就是等社区的回复啦。

https://issues.apache.org/jira/browse/CALCITE-6956

