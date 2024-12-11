---
title: "Calcite支持benchmark函数"
date: 2024-11-29T11:38:14+08:00
draft: false
---
# 需求
想支持下 https://dev.mysql.com/doc/refman/8.4/en/information-functions.html#function_benchmark

```sql
mysql> SELECT BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye'));
+---------------------------------------------------+
| BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye')) |
+---------------------------------------------------+
|                                                 0 |
+---------------------------------------------------+
1 row in set (4.74 sec)
```

得先了解下嵌套函数的实现

又折腾了一周 polardbx 的启动，终于能启动了。

试一下 它是否能支持。
嗯，暂不支持，不过错误提示的挺好的。

```
mysql> SELECT BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye'));
ERROR 4998 (HY000): [1900df69ae400000][192.168.200.77:8527][sharding_db]ERR-CODE: [PXC-4998][ERR_NOT_SUPPORT] function BENCHMARK not support yet!
```

接下来做些啥呢？ 看看嵌套函数咋实现的。

这个嵌套函数可以实现

```
mysql> select GREATEST(max(1),min(2));
+--------------------------+
| GREATEST(max(1), min(2)) |
+--------------------------+
|                        2 |
+--------------------------+
1 row in set (0.18 sec)
```
