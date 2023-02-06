---
title: "Docker"
date: 2022-06-22T18:12:49+08:00
draft: false
---
https://yeasy.gitbook.io/docker_practice/
# 使用 dockerFile 创建镜像
```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```
# 容器的使用
