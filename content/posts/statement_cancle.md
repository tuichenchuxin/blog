---
title: "jdbc Statement 无法取消的问题"
date: 2023-12-21T18:06:56+08:00
draft: false
categories: ["database"]
---

## 问题描述
今天遇到一个奇怪的问题，我将运行中的 statement 信息缓存到内存集合中，通过另一个线程调用 statement.cancle() 接口，发现无论如何都取消不了，需要等到 statment 运行完，才能取消，可是这个取消已经没有意义了。

我的代码如下：
```java
        for (Statement each : process.getDistributedJoinProcessStatements().values()) {
            if (null != each) {
                if (!each.isClosed()) {
                    each.cancel();
                }
            }
        }
```

我发现 debug 无论如何也走不下去，于是我不断点，每次卡住的时候就 dump 出结果
```log
"pool-12-thread-1@5031" prio=5 tid=0x2e nid=NA waiting for monitor entry
  java.lang.Thread.State: BLOCKED
	 waiting for ShardingSphere-Command-0@6041 to release lock on <0x2198> (a com.mysql.jdbc.JDBC4Connection)
	  at com.mysql.jdbc.StatementImpl.isClosed(StatementImpl.java:2487)
	  at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.isClosed(HikariProxyPreparedStatement.java:-1)
	  at org.apache.shardingsphere.mode.manager.cluster.coordinator.subscriber.ProcessListChangedSubscriber.killProcessListId(ProcessListChangedSubscriber.java:98)
	  - locked <0x2196> (a org.apache.shardingsphere.mode.manager.cluster.coordinator.subscriber.ProcessListChangedSubscriber)
	  at jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(NativeMethodAccessorImpl.java:-1)
	  at jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
	  at jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	  at java.lang.reflect.Method.invoke(Method.java:568)
	  at com.google.common.eventbus.Subscriber.invokeSubscriberMethod(Subscriber.java:87)
	  at com.google.common.eventbus.Subscriber$SynchronizedSubscriber.invokeSubscriberMethod(Subscriber.java:144)
	  - locked <0x2197> (a com.google.common.eventbus.Subscriber$SynchronizedSubscriber)
	  at com.google.common.eventbus.Subscriber$1.run(Subscriber.java:72)
	  at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
	  at com.google.common.eventbus.Subscriber.dispatchEvent(Subscriber.java:67)
	  at com.google.common.eventbus.Dispatcher$PerThreadQueuedDispatcher.dispatch(Dispatcher.java:108)
	  at com.google.common.eventbus.EventBus.post(EventBus.java:212)
	  at org.apache.shardingsphere.infra.util.eventbus.EventBusContext.post(EventBusContext.java:51)
	  at org.apache.shardingsphere.mode.manager.cluster.coordinator.registry.GovernanceWatcherFactory$$Lambda$558/0x0000000800fabe28.accept(Unknown Source:-1)
	  at java.util.Optional.ifPresent(Optional.java:178)
	  at org.apache.shardingsphere.mode.manager.cluster.coordinator.registry.GovernanceWatcherFactory.lambda$watch$0(GovernanceWatcherFactory.java:55)
	  at org.apache.shardingsphere.mode.manager.cluster.coordinator.registry.GovernanceWatcherFactory$$Lambda$504/0x0000000800f81bc0.onChange(Unknown Source:-1)
	  at org.apache.shardingsphere.mode.repository.cluster.zookeeper.ZookeeperRepository.lambda$watch$0(ZookeeperRepository.java:368)
	  at org.apache.shardingsphere.mode.repository.cluster.zookeeper.ZookeeperRepository$$Lambda$510/0x0000000800f85b90.childEvent(Unknown Source:-1)
	  at org.apache.curator.framework.recipes.cache.TreeCacheListenerWrapper.sendEvent(TreeCacheListenerWrapper.java:71)
	  at org.apache.curator.framework.recipes.cache.TreeCacheListenerWrapper.event(TreeCacheListenerWrapper.java:42)
	  at org.apache.curator.framework.recipes.cache.CuratorCacheListenerBuilderImpl$2.lambda$event$0(CuratorCacheListenerBuilderImpl.java:149)
	  at org.apache.curator.framework.recipes.cache.CuratorCacheListenerBuilderImpl$2$$Lambda$557/0x0000000800fab580.accept(Unknown Source:-1)
	  at java.util.ArrayList.forEach(ArrayList.java:1511)
	  at org.apache.curator.framework.recipes.cache.CuratorCacheListenerBuilderImpl$2.event(CuratorCacheListenerBuilderImpl.java:149)
	  at org.apache.curator.framework.recipes.cache.CuratorCacheImpl.lambda$putStorage$6(CuratorCacheImpl.java:287)
	  at org.apache.curator.framework.recipes.cache.CuratorCacheImpl$$Lambda$522/0x0000000800f8f1c0.accept(Unknown Source:-1)
	  at org.apache.curator.framework.listen.MappingListenerManager.lambda$forEach$0(MappingListenerManager.java:92)
	  at org.apache.curator.framework.listen.MappingListenerManager$$Lambda$85/0x0000000800d0a120.run(Unknown Source:-1)
	  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	  at java.lang.Thread.run(Thread.java:833)
```

原来是被 mysql 连接对象锁住了。
debug 时，idea 会调用 toString 也会上锁，所以卡住。
因此直接调用 cancle 不判断就没有这个问题了。
```java
        for (Statement each : process.getDistributedJoinProcessStatements().values()) {
            each.cancel();
        }
```

另外 cancle 内部其实是创建一个新连接，然后发送 kill 语句来实现的。
```java
    public void cancel() throws SQLException {
        if (!this.statementExecuting.get()) {
            return;
        }

        if (!this.isClosed && this.connection != null && this.connection.versionMeetsMinimum(5, 0, 0)) {
            Connection cancelConn = null;
            java.sql.Statement cancelStmt = null;

            try {
                cancelConn = this.connection.duplicate();
                cancelStmt = cancelConn.createStatement();
                cancelStmt.execute("KILL QUERY " + this.connection.getIO().getThreadId());
                this.wasCancelled = true;
            } finally {
                if (cancelStmt != null) {
                    cancelStmt.close();
                }

                if (cancelConn != null) {
                    cancelConn.close();
                }
            }

        }
    }
```