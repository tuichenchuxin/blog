---
title: "ShardingSphere 中的分布式事务"
date: 2024-03-28T15:53:24+08:00
draft: false
---

# 以配置 XA 事务为例，查看整体流程

## transaction rule 的初始化

```java
public TransactionRule(final TransactionRuleConfiguration ruleConfig, final Map<String, ShardingSphereDatabase> databases) {
    configuration = ruleConfig;
    defaultType = TransactionType.valueOf(ruleConfig.getDefaultType().toUpperCase());
    providerType = ruleConfig.getProviderType();
    props = ruleConfig.getProps();
    // 创建 transactionManager 
    resource = new AtomicReference<>(createTransactionManagerEngine(databases));
    attributes = new RuleAttributes();
}
```

XA transactionManager 内部管理着 XA DataSource 也一并初始化(mysql 提供的 com.mysql.cj.jdbc.MysqlXADataSource)
```java
public final class XAShardingSphereTransactionManager implements ShardingSphereTransactionManager {
    
    private final Map<String, XATransactionDataSource> cachedDataSources = new CaseInsensitiveMap<>();
    
    private XATransactionManagerProvider xaTransactionManagerProvider;
```

## 以 insert 为例，查看全卷事务如何管理

其实就是使用初始化时使用的 transactionManager 来 begine 和 commit.

```java
private <T> T doExecuteWithImplicitCommitTransaction(final ImplicitTransactionCallback<T> callback) throws SQLException {
        T result;
        // 获取上述初始化的 xatransactionManagerEngine
        BackendTransactionManager transactionManager = new BackendTransactionManager(databaseConnectionManager);
        try {
            transactionManager.begin();
            result = callback.execute();
            transactionManager.commit();
            // CHECKSTYLE:OFF
        } catch (final Exception ex) {
            // CHECKSTYLE:ON
            transactionManager.rollback();
            String databaseName = databaseConnectionManager.getConnectionSession().getDatabaseName();
            throw SQLExceptionTransformEngine.toSQLException(ex, ProxyContext.getInstance().getContextManager().getMetaDataContexts().getMetaData().getDatabase(databaseName).getProtocolType());
        }
        return result;
    }
```

## 总结

从 shardingSphere 角度来看，其实对于分布式事务而言，主要提供了一些封装和框架流程的调用。内部的 transactionManager 实际由其它开源组件提供。
例如 narayana 和 atomics。

narayana 和 atomics 其实提供了 transactionManager 角色和功能， shardingSphere 做了一些封装。
各个数据库，承担的是 resourceManager 角色和功能。 TM 发起 prepare， RM 准备提交，达到万事俱备只欠东风的状态。
TM 发现大家都完成了之后，发起提交命令，如果有人失败了，那么就整体回滚。