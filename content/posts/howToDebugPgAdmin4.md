---
title: "How To Debug PgAdmin4"
date: 2022-04-14T11:37:36+08:00
draft: false
categories: ["Tech"]
---

## 下载源码
https://github.com/postgres/pgadmin4
## 安装环境

```
brew install node
```
```
brew install yarn
```
```
cd runtime
```
```
yarn install
```
```
node_modules/nw/nwjs/nwjs.app/Contents/MacOS/nwjs .
```
```
sudo mkdir "/var/log/pgadmin"
```
```
sudo chmod a+wrx "/var/log/pgadmin"
```
```
sudo mkdir "/var/lib/pgadmin"
```
```
sudo chmod a+wrx "/var/lib/pgadmin"
```
```
pip install --upgrade pip 
```
```
pip install psycopg2-binary
```
```
make install-node
```
最后运行 pgAdmin4.py
过程中会有一些问题，参考 https://github.com/postgres/pgadmin4 readme. 和 stack overflow
