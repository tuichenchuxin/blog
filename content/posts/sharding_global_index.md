---
title: "分片表全局二级索引"
date: 2023-09-26T10:20:20+08:00
draft: true
---

```yaml
databaseName: sharding_db

dataSources:
  ds_0:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: 123456
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    
rules:
- !SHARDING
  tables:
    t_order:
      # 全局索引，多个逗号分隔
      globalIndexNames: 
        - user_id_idx
      actualDataNodes: ds_0.t_order_${0..1}
      databaseStrategy:
        none:
      tableStrategy:
        standard:
          shardingColumn: order_id
          shardingAlgorithmName: t_order_inline

  # 分片全局索引规则
  globalIndexes:
    user_id_idx:
      # 可选配置，多个字段用英文逗号分隔，* 表示覆盖全部字段，对应聚簇索引
      coveringColumns:
        - order_id
      actualDataNodes: ds_0.user_id_idx_${0..1}
      databaseStrategy:
        none:
      tableStrategy:
        standard:
          shardingColumn: user_id
          shardingAlgorithmName: user_id_idx_inline

  shardingAlgorithms:
    user_id_idx_inline:
      type: INLINE
      props:
        algorithm-expression: user_id_idx_${user_id % 2}
    t_order_inline:
      type: INLINE
      props:
        algorithm-expression: t_order_${order_id % 2}
```
### 新功能开发增强
- 计划采用最终一致性的方案来进行设计，保留原有双写的强一致性方案
- 增强查询能力
- jdbc 接入二级索引
- 隐藏二级索引表
- 超过一定数据量之后，就直接使用原表查询
- 开启事务，就走主表

### 相关改造点
#### 创建索引
