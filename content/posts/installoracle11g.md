---
title: "mac docker 安装 oracle 11g"
date: 2022-04-25T18:12:49+08:00
draft: false
---
```
docker pull registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
```
```
docker run --privileged --restart=always --name oracle_11g -p 1521:1521 -d registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
```
```
docker exec -it 容器ID /bin/bash
```
```
source /home/oracle/.bash_profile
```
```
sqlplus nologconnect as sysdba
```
```sql
create user oracle identified by oracle#123;
alter user system identified by system;
alter user system identified by 123456;
grant connect,resource,dba to oracle;
```
参考：https://baijiahao.baidu.com/s?id=1709232831349390844&wfr=spider&for=pc