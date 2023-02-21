---
title: "ShardingSphere PostgreSQL openGauss \\d 支持方案"
date: 2022-11-04T18:09:40+08:00
draft: false
---
# PG \\d 支持

# \\d 的现状

\\d 实际执行的语句如下

```
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'i' THEN 'index' WHEN 'I' THEN 'global partition index' WHEN 'S' THEN 'sequence' WHEN 'L' THEN 'large sequence' WHEN 'f' THEN 'foreign table' WHEN 'm' THEN 'materialized view'  WHEN 'e' THEN 'stream' WHEN 'o' THEN 'contview' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  c.reloptions as "Storage"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','m','S','L','f','e','o','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'db4ai'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
      AND c.relname not like 'matviewmap\_%'
      AND c.relname not like 'mlog\_%'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
```

查询结果如下

```
Schema |         Name         | Type  |  Owner  |             Storage
--------+----------------------+-------+---------+----------------------------------
 public | new_t_single_view    | view  | gaussdb |
 public | student10w           | table | gaussdb | {orientation=row,compression=no}
 public | t_broadcast_table    | table | gaussdb | {orientation=row,compression=no}
 public | t_broadcast_view_new | view  | gaussdb |
 public | t_encrypt            | table | gaussdb | {orientation=row,compression=no}
 public | t_order_0            | table | gaussdb | {orientation=row,compression=no}
 public | t_order_1            | table | gaussdb | {orientation=row,compression=no}
 public | t_order_view_new_0   | view  | gaussdb |
 public | t_order_view_new_1   | view  | gaussdb |
 public | t_single             | table | gaussdb | {orientation=row,compression=no}
(10 rows)
```

涉及的系统表有

`pg_catalog.pg_class`  `pg_catalog.pg_namespace`

涉及的函数有

`pg_catalog.pg_get_userbyid(c.relowner)`  `pg_catalog.pg_table_is_visible(c.oid)`

# 方案设计

## 方案概述

利用之前的[ShardingSphere 内置数据库设计](https://tuichenchuxin.github.io/posts/shardingspheresysdatabase/)
在系统库中增加对应的表，表结构和字段类型同原生的数据库一致，并做<strong>数据收集并装饰</strong>，通过 federation 执行<strong>内存查询</strong>

## 治理中心结构
![](/img/pgdzk.jpeg)

## 表的数据收集

```
SELECT 
  -- 数据库收集值
  n.nspname as "Schema",
  -- 数据库收集值
  c.relname as "Name",
  -- 数据库收集值
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'i' THEN 'index' WHEN 'I' THEN 'global partition index' WHEN 'S' THEN 'sequence' WHEN 'L' THEN 'large sequence' WHEN 'f' THEN 'foreign table' WHEN 'm' THEN 'materialized view'  WHEN 'e' THEN 'stream' WHEN 'o' THEN 'contview' END as "Type",
  -- 通过 calcite 注册函数，返回 ss 中的用户信息
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  -- 数据库收集值
  c.reloptions as "Storage"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','m','S','L','f','e','o','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'db4ai'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
      AND c.relname not like 'matviewmap\_%'
      AND c.relname not like 'mlog\_%'
      -- 通过 calcite 注册函数，mock 返回 true
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
```

需要收集 `pg_class` 和 `pg_namespace` 的信息，并做一些值的装饰，例如 分片表需要替换表名。

`pg_class` 表

[https://opengauss.org/zh/docs/3.1.0/docs/Developerguide/PG_CLASS.html](https://opengauss.org/zh/docs/3.1.0/docs/Developerguide/PG_CLASS.html)

黄色表示 \d 使用到的字段

`pg_namespace` 表

[https://opengauss.org/zh/docs/3.1.0/docs/Developerguide/PG_NAMESPACE.html](https://opengauss.org/zh/docs/3.1.0/docs/Developerguide/PG_NAMESPACE.html)

## 关键问题及解决方法

1. 查询语句中的函数无法下推，如何使用

通过 federation 注册函数解决 `pg_table_is_visible(c.oid)` mock 返回 true，收集数据时就只收集 true 的部分；

`pg_get_userbyid` mock 返回 ss 中配置的用户；

1. pg_class 数据量比较大，只收集 \d 使用到的数据行内容。
2. 查询中需要根据 pg_class 的 relnamespace 关联到 pg_namespace 中的 oid， 但是由于分布式系统，因此可能存在 pg_namespace oid 相同但是 nspname 却不同。

需要新建一张系统 oid 映射关系表，通过 ss 生成 id 映射不同实例的 oid，在表存储时，需要存储 ss 生成的 oid。（需要进一步设计）

1. 针对 \d+ 是否能够支持

```
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'i' THEN 'index' WHEN 'I' THEN 'global partition index' WHEN 'S' THEN 'sequence' WHEN 'L' THEN 'large sequence' WHEN 'f' THEN 'foreign table' WHEN 'm' THEN 'materialized view'  WHEN 'e' THEN 'stream' WHEN 'o' THEN 'contview' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  -- 该值暂时无法支持显示，需要获取大小
  pg_catalog.pg_size_pretty(pg_catalog.pg_table_size(c.oid)) as "Size",
  c.reloptions as "Storage",
  -- 该值依赖 pg_catalog.pg_description 表的收集以及 oid 的映射维护
  -- select description from pg_catalog.pg_description where objoid = $1 and classoid = (select oid from pg_catalog.pg_class where relname = $2 and relnamespace = 11) and objsubid = 0
  pg_catalog.obj_description(c.oid, 'pg_class') as "Description"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','m','S','L','f','e','o','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'db4ai'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
      AND c.relname not like 'matviewmap\_%'
      AND c.relname not like 'mlog\_%'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
```

## 附录

### 系统函数分析

`pg_get_userbyid`

对应源码为

```
/* ----------
 * pg_get_userbyid    - Get a user name by roleid and
 *            fallback to 'unknown (OID=n)'
 * ----------
 */
Datum
pg_get_userbyid(PG_FUNCTION_ARGS)
{
   Oid          roleid = PG_GETARG_OID(0);
   Name      result;
   HeapTuple  roletup;
   Form_pg_authid role_rec;

   /*
    * Allocate space for the result
    */
   result = (Name) palloc(NAMEDATALEN);
   memset(NameStr(*result), 0, NAMEDATALEN);

   /*
    * Get the pg_authid entry and print the result
    */
   roletup = SearchSysCache1(AUTHOID, ObjectIdGetDatum(roleid));
   if (HeapTupleIsValid(roletup))
   {
      role_rec = (Form_pg_authid) GETSTRUCT(roletup);
      *result = role_rec->rolname;
      ReleaseSysCache(roletup);
   }
   else
      sprintf(NameStr(*result), "unknown (OID=%u)", roleid);

   PG_RETURN_NAME(result);
}
```

可以用以下语句获取

```
<strong>select</strong> oid,rolname <strong>from</strong> pg_authid where oid = 1234
```

`pg_table_is_visible(c.oid)` 函数对应源码如下

```
/*
 * RelationIsVisible
 *    Determine whether a relation (identified by OID) is visible in the
 *    current search path.  Visible means "would be found by searching
 *    for the unqualified relation name".
 */
bool
RelationIsVisible(Oid relid)
{
   HeapTuple  reltup;
   Form_pg_class relform;
   Oid          relnamespace;
   bool      visible;

   reltup = SearchSysCache1(<em>RELOID</em>, ObjectIdGetDatum(relid));
   if (!HeapTupleIsValid(reltup))
      elog(ERROR, "cache lookup failed for relation %u", relid);
   relform = (Form_pg_class) GETSTRUCT(reltup);

   recomputeNamespacePath();

   /*
    * Quick check: if it ain't in the path at all, it ain't visible. Items in
    * the system namespace are surely in the path and so we needn't even do
    * list_member_oid() for them.
    */
   relnamespace = relform->relnamespace;
   if (relnamespace != PG_CATALOG_NAMESPACE &&
      !list_member_oid(activeSearchPath, relnamespace))
      visible = false;
   else
   {
      /*
       * If it is in the path, it might still not be visible; it could be
       * hidden by another relation of the same name earlier in the path. So
       * we must do a slow check for conflicting relations.
       */
      char      *relname = NameStr(relform->relname);
      ListCell   *l;

      visible = false;
      foreach(l, activeSearchPath)
      {
         Oid          namespaceId = lfirst_oid(l);

         if (namespaceId == relnamespace)
         {
            /* Found it first in path */
            visible = true;
            break;
         }
         if (OidIsValid(get_relname_relid(relname, namespaceId)))
         {
            /* Found something else first in path */
            break;
         }
      }
   }

   ReleaseSysCache(reltup);

   return visible;
}
```

##

