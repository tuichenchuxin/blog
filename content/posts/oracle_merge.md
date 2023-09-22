---
title: "Oracle merge insert 问题"
date: 2023-09-22T15:45:09+08:00
draft: false
---

```sql
create table ENCRYPT.TEST1
(
    A   VARCHAR2(200)  PRIMARY KEY,
    B   VARCHAR2(200),
    C   VARCHAR2(200)
)
/
create table ENCRYPT.TEST2
(
    A   VARCHAR2(200)  PRIMARY KEY,
    B   VARCHAR2(200),
    C   VARCHAR2(200)
)
/

insert into TEST1 values (1,2,3);
insert into TEST1 values (2,3,4);

insert into TEST2 values (4,2,3);

merge into TEST2 fi using TEST1 se on (fi.A = se.A and fi.A = 3)
when matched then update set fi.B = se.B, fi.C = se.C
when not matched then insert (A, B, C) values (1,2,3);
```

执行如上 SQL 发现一直报
ORA-00001: unique constraint (ENCRYPT.SYS_C007033) violated


原来是因为我理解错了 merge insert 的含义。
这是将 TEST2 的数据痛 TEST1 进行比对，如果 TEST1 中的数据满足 ON 的条件，那么会进行 update，否则会进行 insert.
因为 TEST1 中不符合条件的有两条，所以会依次执行 insert (1,2,3) 所以就冲突了。（相当于 insert 执行了两次）
