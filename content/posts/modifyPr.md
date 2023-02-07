---
title: "How to Modify Pr"
date: 2022-05-12T17:56:24+08:00
draft: false
---

Step 1: 克隆项目

Step 2: 添加新的远程仓库
为了修改他人 Fork 的仓库，你需要将其添加到自己的远程仓库列表中
```
$ git remote add sirmin https://github.com/SirMin/shardingsphere.git
```
现在，当你执行 git remote -v 指令时，就可以看到他人 Fork 的仓库，出现在你的远程仓库列表中：
```
% git remote -v
origin	https://github.com/tuichenchuxin/shardingsphere.git (fetch)
origin	https://github.com/tuichenchuxin/shardingsphere.git (push)
sirmin	https://github.com/SirMin/shardingsphere.git (fetch)
sirmin	https://github.com/SirMin/shardingsphere.git (push)
upstream	https://github.com/apache/shardingsphere.git (fetch)
upstream	https://github.com/apache/shardingsphere.git (push)
```
Step 3: 拉取新的远程仓库
```
$ git fetch sirmin
```
Step 4: 切换到对应的分支
你需要确认一下，贡献者提出 Pull Request 时所用的分支（如果你不确定他们使用的哪个分支，请看 Pull Request 顶部的信息）：

在本地，给该分支起一个不重复的名字，例如 EvanOne0616-patch，然后切换到贡献者提出 Pull Request 所用的分支：

```
$ git checkout -b sirmin-patch sirmin/sirmin/resultSet

```

Step 5: 提交修改，推送远程

```
$ git commit -m "Fix the wrong spelling of the word"
$ git push sirmin HEAD:sirmin/resultSet
```