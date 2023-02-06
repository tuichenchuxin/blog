---
title: "Add Yat Test for ShardingSphere"
date: 2022-06-22T17:44:41+08:00
draft: true
---

# 构建 yat
下载源码
``` 
git clone https://gitee.com/opengauss/Yat.git
``` 
构建 yat
```
cd Yat/yat-master
chmod +x gradlew
./gradlew pack
```
```
cd pkg
chmod +x install
./install -F
```
根据源码中的 dockerFile 构建 dockerImage


# 使用 yat 测试 测试 shardingSphere proxy
利用构建的 yat image 运行相关测试
需要挂在到对应目录
```
docker run --name yat0 -i -t -v /Users/chenchuxin/Documents/GitHub/Yat/openGaussBase:/root/openGaussBase -w /root/openGaussBase --entrypoint=bash --privileged=true yat-v1
```
修改 yat 项目 openGaussBase/conf 下的 configure.yml 文件
```
yat.limit.case.size.max: '200K'
yat.limit.case.count.max: 100000
yat.limit.case.depth.max: 10
yat.limit.case.name.pattern: .*
```
修改 yat 项目 openGaussBase/conf 下的 nodes.yml 文件
```
default:
  host: 'host.docker.internal'
  db:
    url: 'jdbc:opengauss://host.docker.internal:3307/sharding_db?loggerLevel=OFF'
    driver: 'org.opengauss.Driver'
    username: 'root'
    password: 'root'
    port: 3307
    name: 'sharding_db'
  ssh:
    port: 22
    username: root
    password: '********'
```
创建 conf/env.sh
```
touch conf/env.sh
```
在 openGaussBase/schedule 目录创建 schedule.schd 文件，增加你需要测试的 case
```
test: SQL/DDL/view/Opengauss_Function_DDL_View_Case0034
```
进入 docker bash 模式 运行 yat suite
```
[+] [2022-06-22 09:06:15] [05.394 s] [v] SQL/DDL/view/Opengauss_Function_DDL_View_Case0034 .... : ok
################# Testing Result 1/1 Using Time PT5.46011S At 2022-06-22 09:06:20 ##################
Run test suite finished
```
也可能出现相关异常，可参考 https://gitee.com/opengauss/Yat/blob/master/yat-master/docs/index.md#yat-quick-start 来解决