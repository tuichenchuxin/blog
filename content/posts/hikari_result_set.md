---
title: "Streaming result set com.mysql.cj.protocol.a.result.ResultsetRowsStreaming@3e72fd68 is still active."
date: 2024-03-22T08:53:52+08:00
draft: false
---

# 背景
今天写一个并发查询的功能，但是报了如下的错误
```
java.sql.SQLException: Streaming result set com.mysql.cj.protocol.a.result.ResultsetRowsStreaming@3e72fd68 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that you have called .close() on any active streaming result sets before attempting more queries.
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:129)
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)
	at com.mysql.cj.jdbc.StatementImpl.executeSimpleNonQuery(StatementImpl.java:1241)
	at com.mysql.cj.jdbc.StatementImpl.setupStreamingTimeout(StatementImpl.java:632)
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeQuery(ClientPreparedStatement.java:947)
	at com.zaxxer.hikari.pool.ProxyPreparedStatement.executeQuery(ProxyPreparedStatement.java:52)
	at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.executeQuery(HikariProxyPreparedStatement.java)
```

# 探索

奇怪的是，虽然是并发，但是我都使用了独立的 hikari 连接，并且每次查询完都主动 close 了连接和 statments，当然这里要注意下，随时使用了全新的 hikari 连接，但是因为它有连接池，所以其它线程依然可以重复使用空闲的数据库连接。
```java
 @Override
public void close() throws SQLException {
    Collection<SQLException> result = new LinkedList<>();
    for (Statement each : statements) {
        try {
            each.cancel();
            each.close();
        } catch (final SQLException ex) {
            if (isClosed(each)) {
                continue;
            }
            result.add(ex);
        }
    }
    for (Connection each : connections) {
        try {
            each.close();
        } catch (final SQLException ex) {
            result.add(ex);
        }
    }
    if (result.isEmpty()) {
        return;
    }
    SQLException ex = new SQLException();
    result.forEach(ex::setNextException);
    throw ex;
}
```

为什么会出现这个问题呢？难道是没有实际关掉 ResultSet 导致的么, 于是我又增加了如下代码

```java
for (ResultSet each : resultSets) {
    try {
        each.close();
    } catch (final SQLException ex) {
        if (isClosed(each)) {
            continue;
        }
        result.add(ex);
    }
}
```

报错信息变了

```
java.util.concurrent.ExecutionException: java.lang.NullPointerException: Cannot invoke "Object.toString()" because "type" is null
```

这原来是真正的问题，查询过程中出现了异常。

当我修改了如下问题后，即使没有关闭 resultSet 仅关闭 statment 也不会再有异常了。

所以可以推测，仅关闭 statment 并不会关闭 resultSet.

那么为什么无异常的情况下，不主动关闭 resultSet 也无报错呢。

推测是由于异常了，所以没有调用 while（resultSet.next()） 因此没有关。

那我们可以修改 while 为 if 来验证

可以看到异常又重现了

```
java.sql.SQLException: Streaming result set com.mysql.cj.protocol.a.result.ResultsetRowsStreaming@3d0961f9 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that you have called .close() on any active streaming result sets before attempting more queries.
```

# 结论
1. 仅关闭 statement 不能关闭 resultset 
2. resultSet next() 方法需要循环调用到结束，可以自动关闭流式结果集
3. 最好主动关闭 resultSet.