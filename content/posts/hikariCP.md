---
title: "探索 HikariCP"
date: 2023-12-15T09:14:20+08:00
draft: false
---

项目地址： https://github.com/brettwooldridge/HikariCP

   "Simplicity is prerequisite for reliability."
         - Edsger Dijkstra

最先强调的就是简单，扁平，性能，做了 字节码级别的优化。
配置简单，很多东西选择不做，例如 statement 缓存。

pool size 比想象的要小的多，这样更能拥有更好的性能。因为单 cpu 并非并行，上下文切换需要时间，另外查询的位置需要来回切换等等。
不过 IO wait 比较大时，那么并发时能够提升的。
另外对于大事务和短事务，比较难以协调，这时候最好搞两个 pool。

## HikariCP 怎么学习
考虑编译起来，然后运行 test 查看代码逻辑

- [x] testSealed1

通过 IDEA 运行时发现总会报
Error occurred during initialization of boot layer FindException: Module not found
后来发现 mvn clean 之后，可以运行，但是有部分代码是会在编译阶段被更改，因此还得 install 之后在跑
但是 idea 运行之前会 build 然后就改变了 target 下部分代码。
所以需要勾选运行前不 build 即可解决问题。

- [x] testSealed2

- [x] testSealed3

- [x] testSealedAccessibleMethods

创建 hikari datasoure 的时候会设置 sealed, 如果已经 sealed 了，那么后续就不允许改了。

- [x] testIdleTimeout

 看一下 idle 为什么被释放。
```java
 @Test
   public void testIdleTimeout() throws InterruptedException, SQLException
   {
      HikariConfig config = newHikariConfig();
      config.setMinimumIdle(5);
      config.setMaximumPoolSize(10);
      config.setConnectionTestQuery("SELECT 1");
      config.setDataSourceClassName("org.h2.jdbcx.JdbcDataSource");
      config.addDataSourceProperty("url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1");

      System.setProperty("com.zaxxer.hikari.housekeeping.periodMs", "1000");

      try (HikariDataSource ds = new HikariDataSource(config)) {
         getUnsealedConfig(ds).setIdleTimeout(3000);

         System.clearProperty("com.zaxxer.hikari.housekeeping.periodMs");

         SECONDS.sleep(1);

         HikariPool pool = getPool(ds);

         assertEquals("Total connections not as expected", 5, pool.getTotalConnections());
         assertEquals("Idle connections not as expected", 5, pool.getIdleConnections());

         try (Connection connection = ds.getConnection()) {
            Assert.assertNotNull(connection);

            MILLISECONDS.sleep(1500);
            // 获取了一个 connection, 从 connectionBag 中借了一个，由于最小的空闲线程数是 5，因此又自动补充了一个
            assertEquals("Second total connections not as expected", 6, pool.getTotalConnections());
            assertEquals("Second idle connections not as expected", 5, pool.getIdleConnections());
         }
         // 还没超过设置的 3000 ms,所以总空闲连接就变成了 6 个。
         assertEquals("Idle connections not as expected", 6, pool.getIdleConnections());

         MILLISECONDS.sleep(3000);
         // 超时释放了
         assertEquals("Third total connections not as expected", 5, pool.getTotalConnections());
         assertEquals("Third idle connections not as expected", 5, pool.getIdleConnections());
      }
   }
```
看一下是谁补充了 connectionBag.
```java
this.houseKeeperTask = houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 100L, housekeepingPeriodMs, MILLISECONDS);
```
debug 发现是 houseKeeperTask ,定时任务，探测到最小空闲连接已经小于 4 了，因此又补充了。

那么又是谁释放了呢？

依然是 houseKeeperTask 发现空闲连接已经大于最小空闲连接配置，并且时间已经超过 配置的 3000 ms,所以就释放了。
```java 
            if (idleTimeout > 0L && config.getMinimumIdle() < config.getMaximumPoolSize()) {
               logPoolState("Before cleanup ");
               final var notInUse = connectionBag.values(STATE_NOT_IN_USE);
               var maxToRemove = notInUse.size() - config.getMinimumIdle();
               for (PoolEntry entry : notInUse) {
                  if (maxToRemove > 0 && elapsedMillis(entry.lastAccessed, now) > idleTimeout && connectionBag.reserve(entry)) {
                     closeConnection(entry, "(connection has passed idleTimeout)");
                     maxToRemove--;
                  }
               }
               logPoolState("After cleanup  ");
            }
```

- [x] testIdleTimeout2
- [x] testConcurrentClose
- [x] testPoolSizeAboutSameSizeAsThreadCount
   > 测试 poolSize 的大小基本上跟线程数差不多
- [x] testSlowConnectionTimeBurstyWork
   > 测试快速的任务处理下，连接创建比较慢，但是实际使用的connection 很少
- [x] testRaceCondition
   > 测试30s,不停创建连接和驱逐连接
- [x] testAutoCommit
   > 测试设置 autoCommit 为false ,关闭连接后，代理的原真实连接还是会恢复成原状
- [x] testTransactionIsolation
   > 测试设置 isolation，关闭连接后，恢复（dirtyBits 通过位运算来提高性能）
- [x] testIsolation
   > 看起来就是测试设置 isolation 成功
- [x] testReadOnly
   > 测试设置 readOnly, 关闭连接后，恢复
- [x] testCatalog
   > 测试设置 catalog, 关闭连接后，恢复
- [x] testCommitTracking
   > 测试 isCommitStateDirty 在 autoCommit = false 时，有 SQL 执行时，就变成 true, rollback 或者 commit 就恢复
- [x] testException1
   > 测试异常之后，能够自动检查 exception 并且关闭连接
- [x] testUseAfterStatementClose
   > 测试关闭 statement 之后，不允许再使用 statement
- [x] testUseAfterClose
   > 测试关闭连接后使用
- [x] testLastErrorTimeout
   > 测试获取 connection 时，isConnectionDead 不会主动抛出异常
   