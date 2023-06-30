---
title: "使用 sqlline 连接 sharding_jdbc"
date: 2023-06-30T16:02:42+08:00
draft: false
---

https://github.com/julianhyde/sqlline
sqlline 是一个通过 jdbc 的连接工具。

1. 下载代码并编译
```
git clone git://github.com/julianhyde/sqlline.git
cd sqlline
./mvnw package
```

2. 将 shardingsphere-jdbc 打包，之后最好再使用 sharde PLUGIN 打成一个 jar 包

3. 修改 sqlline bin 目录下的 sqlline 脚本，将相关依赖添加进去包括数据库驱动、shardingsphere shade jar、 sqlline-VERSION-jar-with-dependencies.jar
```
#!/bin/bash
# sqlline - Script to launch SQL shell on Unix, Linux or Mac OS

BINPATH=$(dirname $0)
exec java -cp "$BINPATH/../target/*":"/Users/chenchuxin/Documents/sqlline-sqlline-1.12.0/bin/*" sqlline.SqlLine "$@"

# End sqlline
```

4. 使用 sqlline
```
 sqlline -d org.apache.shardingsphere.driver.ShardingSphereDriver -u jdbc:shardingsphere:absolutepath:/Users/chenchuxin/Documents/GitHub/sw-test/src/main/resources/META-INF/oracle.yaml
```


