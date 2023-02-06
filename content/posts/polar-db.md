---
title: "Polar Db"
date: 2022-06-14T18:09:13+08:00
draft: true
---

https://zhuanlan.zhihu.com/p/515688555

# mac 开发环境启动 galaxysql
https://hub.docker.com/r/polardbx/polardb-x

```
docker pull polardbx/polardb-x
```

```
docker run -d --name some-dn-and-gms --env mode=dev -p 4886:4886 -p 32886:32886 polardbx/polardb-x
```

```
docker exec -it 41d8a027 bash
mysql -h127.0.0.1 -P4886 -uroot -padmin -D polardbx_meta_db_polardbx -e "select passwd_enc from storage_info where inst_kind=2"
```
获取密码后修改 server.properties

```yaml
serverPort=8528
managerPort=3406
rpcPort=9090
charset=utf-8
processors=4
processorHandler=16
processorKillExecutor=128
timerExecutor=8
managerExecutor=256
serverExecutor=1024
idleTimeout=
trustedIps=127.0.0.1
slowSqlTime=1000
maxConnection=20000
allowManagerLogin=1
allowCrossDbQuery=true
galaxyXProtocol=1
metaDbAddr=127.0.0.1:4886
metaDbXprotoPort=32886
metaDbUser=my_polarx
# 前文查看的存储节点密码
metaDbPasswd=qEJWtCdgsOIie4j4mKP2Bvg2dsFHzdIhTaqMiq86N1QQU1HHL7olKb60pxz5hp/4
#?? E2+jB0*0&gM9)9$4+6)E4@1$lO9%G8+jA4_
metaDbName=polardbx_meta_db_polardbx
instanceId=polardbx-polardbx
```
注释掉 CobarServer.java 中  tryStartCdcManager(); 代码

启动配置如下
```xml
<component name="ProjectRunConfigurationManager">
  <configuration default="false" name="TddlLauncher" type="Application" factoryName="Application" singleton="false" nameIsGenerated="true">
    <envs>
      <env name="dnPasswordKey" value="asdf1234ghjk5678" />
    </envs>
    <option name="MAIN_CLASS_NAME" value="com.alibaba.polardbx.server.TddlLauncher" />
    <module name="polardbx-server" />
    <option name="VM_PARAMETERS" value="-Dserver.conf=$PROJECT_DIR$/polardbx-server/src/main/conf/server.properties" />
    <extension name="coverage">
      <pattern>
        <option name="PATTERN" value="com.alibaba.polardbx.server.*" />
        <option name="ENABLED" value="true" />
      </pattern>
    </extension>
    <method v="2">
      <option name="Make" enabled="true" />
    </method>
  </configuration>
</component>
```
连接 polar-dbx cn
```
mysql -h127.0.0.1 -P8528 -upolardbx_root -p123456
```
# polardb-x debug ddl
 create ddl 一直不能成功原来是因为，docker 镜像中没有启动 cdc，DDL 流程中涉及到 notify cdc，所以注释 CdcManager.notifyDdl 方法中的代码可以执行成功。
 这篇文档写得挺好，说明了 ddl 执行的流程
 https://zhuanlan.zhihu.com/p/515688555