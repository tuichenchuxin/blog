---
title: "DataTTL 相关调研"
date: 2023-02-06T11:46:34+08:00
draft: false
---

# TiDB
## 使用

创建表时可以设置数据过期时间，然后后台定时任务删除相关数据

产品文档：https://docs.pingcap.com/zh/tidb/dev/time-to-live

## 功能列表

| 功能 | 备注 |
| --- | --- |
| create 语句中增加 TTL 属性，指定表中数据的过期时间 | |
| create 语句中增加注释信息(兼容 MySQL)，包含 TTL 属性 | |
| alter 语句修改 TTL 属性 | |
| TTL 可以配合表中其它列的属性来使用 | |
| 可以指定 TTL 任务时间间隔 | |


# PolarDB-X

创建 TTL 表，按照时间分区，定期删除和创建相关分区表

产品文档：
https://help.aliyun.com/document_detail/403528.html
## 功能列表

| 功能 | 备注 |
| -----  | ------- |
| 对按照时间进行 range 分区的表，定时失效过期分区，定时提前创建分区 | 仅用在自动模式下的分区表上 |
| 支持通过 DDL 语句来定义相关分区的 TTL | TTL表支持的时间分区列类型为：date、datetime； 所有的唯一键（包括主键）必须包含TTL表的local partition by range时间分区列；所有的唯一键（包括主键）必须包含TTL表的local partition by range时间分区列 |
| 支持查看分区信息以及过期时间等 information_schema.local_partitions |  |
| 支持校验物理表的物理分区的完整性 | |
| 支持 Alter 语句手动创建新分区表/删除过期分区表 | |
| 支持普通表和 TTL 表互相转换 | |
| 支持创建、查看、删除 TTL 定时任务 | |

# Google Spanner

创建表时可以设置行删除策略，后台任务扫描表并删除相关行

产品文档：https://cloud.google.com/spanner/docs/ttl?hl=zh-cn

## 功能列表
| 功能 | 备注 |
| --- | --- |
| create 语句中增加行删除策略 | |
| alter 语句修改行删除策略 | |
| 查看表的 TTL 删除情况 | |


