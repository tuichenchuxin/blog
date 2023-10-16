---
title: "如果 result set 仅有一个结果集，是否可以用 if(resultset.next()) ?"
date: 2023-10-16T15:13:38+08:00
draft: false
---

今天写代码时遇到一个异常。如下
```
java.sql.SQLException: Streaming result set com.mysql.jdbc.RowDataDynamic@4b76a937 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that you have called .close() on any active streaming result sets before attempting more queries.
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:869)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:865)
	at com.mysql.jdbc.MysqlIO.checkForOutstandingStreamingData(MysqlIO.java:3172)
```
我推测可能是结果集没有关闭，然后想要继续创建结果集。
发现跟之前的代码唯一的区别如下
```java
// 之前
while(resultset.next())

// 之后
if(resultset.next())

```
因为只有一个结果集，所以我考虑直接用下面的方式，结果就报错了。

debug mysql-jdbc 之后，发现 resultSet.next() 方法如果没有更多结果集之后，会关闭流式查询结果集。

```java
// RowDataDynamic.java
    public ResultSetRow next() throws SQLException {
        this.nextRecord();
        if (this.nextRow == null && !this.streamerClosed && !this.moreResultsExisted) {
            // 关闭流
            this.io.closeStreamer(this);
            this.streamerClosed = true;
        }

        if (this.nextRow != null && this.index != Integer.MAX_VALUE) {
            ++this.index;
        }

        return this.nextRow;
    }

```
