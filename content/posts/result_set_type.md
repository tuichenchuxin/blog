---
title: "是否可以在 jdbc 查询数据的同时获取本次查询的数据量？"
date: 2023-10-13T14:19:38+08:00
draft: false
---

## 背景
在优化全局二级索引的过程中，有一个优化点是如果二级索引的数据量太大，那么带回主表再查可能引起一些问题，例如效率不高，OOM 或者 SQL 超长等问题，因此考虑二级索引表数据量大的时候，直接查询主表。
因此需要查询二级索引表的数据量。
这里考虑能否直接查询二级索引表的同时获取本次查询结果的数据量。

## 实践
### 方式一
通过 ResultSet.TYPE_SCROLL_INSENSITIVE 可以将游标放到最后，然后根据 getRow() 方法获取
```java
    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true", "root", "123456");
        // 设置游标可以滚动，默认是 forward only
        try (PreparedStatement preparedStatement = connection.prepareStatement("select * from t_single", ResultSet.TYPE_SCROLL_INSENSITIVE, ResultSet.CONCUR_READ_ONLY);
                ResultSet resultSet = preparedStatement.executeQuery()) {
            int count = 0;
            // 移动到最后
            if (resultSet.last()) {
                // 获取最后一行的行号
                count = resultSet.getRow();
            }
            System.out.println("total:" + count);
            resultSet.beforeFirst();
            int columnIndex = 1;
            while (resultSet.next()) {
                System.out.println(resultSet.getObject(columnIndex ++));
            }
        }
    }
```

但是为什么之前的很多框架都采用的 count sql，而不用这种方式呢？例如 pageHelper 等
https://cloud.tencent.com/developer/article/1433806 搜索到相关文章，发现 TYPE_SCROLL_INSENSITIVE 会将结果集缓存起来，容易 OOM。那么这种方式可能问题就比较大了。
如果二级索引表查询出的数据量比较大，缓存结果集的风险就比较大了。

https://forums.mysql.com/read.php?39,435275,435275
该主题也在说使用这种resultSet type 会导致性能很差

> How many rows are we talking about here? In general, it's -never- a good idea to select more rows than you need, and use fetch hints to scroll the result set. Pretty much every database now has options like MySQL's LIMIT clause that can (and often will) involve the optimizer to make it so your query examines as few rows as possible.

> The MySQL protocol itself doesn't have the notion of fetch batch sizes, and the driver is optimized for OLTP type queries. Leaving large result sets open while a user paginates is very, very sub-optimal because of the locks and other resources that are needlessly taken.

> You may want to consider re-thinking your overall strategy, because it's more than likely consuming more resources than you think, even on databases other than MySQL.

> -Mark

> Mark Matthews
> Consulting Member Technical Staff - MySQL Enterprise Tools
> Oracle
> http://www.mysql.com/products/enterprise/monitor.html

看官方的回复也不建议在大数据中使用滚动游标的方式。

## 结论
可以在查询结果的同时获取 count，但是性能可能不高，也有占用过多资源的可能。
