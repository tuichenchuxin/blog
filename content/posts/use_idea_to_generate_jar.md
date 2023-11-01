---
title: "使用idea创建的 jar 文件都是 .java 而不是 .class"
date: 2023-11-01T15:38:48+08:00
draft: false
---

通过如下步骤打包后发现 jar 包中的文件都是 .java 文件，而不是 .class 文件
![](/img/20231101-155254.jpeg)
![](/img/20231101-155302.jpeg)
![](/img/20231101-155306.jpeg)

经过了一下午的尝试，无论是添加.class 文件还是添加 文件夹，编译后的 jar 文件中都是.java 文件，不是.class 文件。
所以最后还是直接用 maven 吧，无需添加任何插件，直接 mvn clean install 即可。