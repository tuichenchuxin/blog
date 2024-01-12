---
title: "Antlr4 支持中文空格"
date: 2024-01-12T16:05:10+08:00
draft: false
---
Oracle 是可以支持 中文空格的（\u3000）。因此 shardingSphere 的解析也需要支持。

查询 antlr4 的定义，首先我们遇到中文空格要像遇到英文空格一样跳过。
```
WS
    : [ \t\r\n\u3000] + ->skip
    ;
```
其次还需要将自定义的字符中排除 \u3000
没有找到正确的排除方法。因此考虑在范围定义中跳过 \u3000
```
IDENTIFIER_
    : [A-Za-z\u0080-\u2FFF\u3001-\uFF0B\uFF0D-\uFFFF]+[A-Za-z_$#0-9\u0080-\u2FFF\u3001-\uFF0B\uFF0D-\uFFFF]*
    ;
```
完美解决