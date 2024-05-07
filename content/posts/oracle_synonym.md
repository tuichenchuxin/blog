---
title: "Oracle 同义词"
date: 2024-04-17T15:42:13+08:00
draft: false
---

# ORACLE 同义词的定义及使用

```sql
CREATE SYNONYM TEST FOR DM.TM_WGG_ATM_GTW_MON;
```

- 同义词就是别名，与视图类似，都是不占实际存储空间，相当于是访问数据库对象的另外一种方式。
- 公用同义词和私用同义词

## 同义词的作用
- 多用户协同开发，不再需要使用 user.tableA 而是直接使用同义词即可。
- 简化 sql 语句

## 如何实现不同用户使用相同 SQL 查询

用户 encrypt
```sql
create synonym TEST_SYNONYM FOR AB01;

grant select on TEST_SYNONYM to encrypt_1;
```

用户 encrypt_1
```sql
## 创建一个指向 encrypt.TEST_SYNONYM 的同义词
create synonym TEST_SYNONYM for encrypt.TEST_SYNONYM;
```

这样两个用户都可以使用

```sql 
select * from TEST_SYNONYM;
```


# ShardingSphere 中同义词加载的初步想法

- 查询用户 schema 下能够访问到的所有同义词，包含私有和公有的同义词
- 找到同义词指向的对象
- 找到对象指向的对象

通过如下 SQL 可以找到最终指向的对象。但是如果指向的是其它用户下的 SYNONYM ，似乎查不出来
```sql
SELECT *
         FROM USER_DEPENDENCIES
         where REFERENCED_TYPE in ('TABLE', 'VIEW')
START WITH NAME IN ('ENCRYPT_1_B_TEST_1')
CONNECT BY (PRIOR REFERENCED_NAME) = NAME AND (SELECT SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA') FROM DUAL) = (PRIOR REFERENCED_OWNER)
         ORDER BY NAME, REFERENCED_NAME;
```