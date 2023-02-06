---
title: "PgAmdin4 展示 DDL 语句逻辑分析"
date: 2022-04-13T19:31:40+08:00
draft: true
---
## PgAmdin4 展示 DDL 语句
通过 PgAdmin4 可以获取 table 的 DDL 语句
 ![](pgAmdin4-DDL.png)
 ```sql
 -- Table: public.t_order_0

-- DROP TABLE IF EXISTS public.t_order_0;

CREATE TABLE IF NOT EXISTS public.t_order_0
(
    order_id integer NOT NULL,
    user_id integer NOT NULL,
    status character varying(45) COLLATE pg_catalog."default",
    CONSTRAINT t_order_0_pkey PRIMARY KEY (order_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.t_order_0
    OWNER to postgres;

COMMENT ON TABLE public.t_order_0
    IS 'haha';

COMMENT ON COLUMN public.t_order_0.order_id
    IS 'haha';
 ```

## PgAdmin4 是如何展示对应的 DDL 语句的呢
https://github.com/postgres/pgadmin4
翻阅源码发现 DDL 语句的展示，主要是通过以下步骤来获取 SQL 语句的。
  - 查询表基础信息（区分 pg 不同版本）
  - 获取相关权限信息
  - 查询列相关信息
    - 检查 of_type 和 继承表，如果存在需要获取
    - 获取该表的所有列信息
    - 对列做格式化
    - 查询 constriant 信息，并添加到 列上
  - 检查  table is partitions
  - 查询 index 信息
  - 查询 ROW SECURITY POLICY
  - 查询 Triggers
  - 查询 Rules
相关代码如下
```python
    def sql(self, gid, sid, did, scid, tid):
        """
        This function will creates reverse engineered sql for
        the table object

         Args:
           gid: Server Group ID
           sid: Server ID
           did: Database ID
           scid: Schema ID
           tid: Table ID
        """
        main_sql = []

        status, res = self._fetch_table_properties(did, scid, tid)
        if not status:
            return res

        if len(res['rows']) == 0:
            return gone(gettext(self.not_found_error_msg()))

        data = res['rows'][0]

        return BaseTableView.get_reverse_engineered_sql(
            self, did=did, scid=scid, tid=tid, main_sql=main_sql, data=data,
            add_not_exists_clause=True)
```

``` python
    def get_reverse_engineered_sql(self, **kwargs):
        """
        This function will creates reverse engineered sql for
        the table object

         Args:
           kwargs
        """
        did = kwargs.get('did')
        scid = kwargs.get('scid')
        tid = kwargs.get('tid')
        main_sql = kwargs.get('main_sql')
        data = kwargs.get('data')
        json_resp = kwargs.get('json_resp', True)
        diff_partition_sql = kwargs.get('diff_partition_sql', False)
        if_exists_flag = kwargs.get('add_not_exists_clause', False)

        # Table & Schema declaration so that we can use them in child nodes
        schema = data['schema']
        table = data['name']
        is_partitioned = 'is_partitioned' in data and data['is_partitioned']

        # Get Reverse engineered sql for Table
        self._get_resql_for_table(did, scid, tid, data, json_resp, main_sql,
                                  add_not_exists_clause=if_exists_flag)
        # Get Reverse engineered sql for Table
        self._get_resql_for_index(did, tid, main_sql, json_resp, schema,
                                  table, add_not_exists_clause=if_exists_flag)

        # Get Reverse engineered sql for ROW SECURITY POLICY
        self._get_resql_for_row_security_policy(scid, tid, json_resp,
                                                main_sql, schema, table)

        # Get Reverse engineered sql for Triggers
        self._get_resql_for_triggers(tid, json_resp, main_sql, schema, table)

        # Get Reverse engineered sql for Compound Triggers
        self._get_resql_for_compound_triggers(tid, main_sql, schema, table)

        # Get Reverse engineered sql for Rules
        self._get_resql_for_rules(tid, main_sql, table, json_resp)

        # Get Reverse engineered sql for Partitions
        partition_main_sql = ""
        if is_partitioned:
            sql = render_template("/".join([self.partition_template_path,
                                            self._NODES_SQL]),
                                  scid=scid, tid=tid)
            status, rset = self.conn.execute_2darray(sql)
            if not status:
                return internal_server_error(errormsg=rset)

            self._get_resql_for_partitions(data, rset, json_resp,
                                           diff_partition_sql, main_sql, did)

        sql = '\n'.join(main_sql)

        if not json_resp:
            return sql, partition_main_sql
        return ajax_response(response=sql.strip('\n'))
```


## pgAdmin 展示表结构反向 SQL 执行了哪些 SQL 语句呢？
- table propeties
```sql
SELECT rel.oid, rel.relname AS name, rel.reltablespace AS spcoid,rel.relacl AS relacl_str,
  (CASE WHEN length(spc.spcname::text) > 0 OR rel.relkind = 'p' THEN spc.spcname ELSE
    (SELECT sp.spcname FROM pg_catalog.pg_database dtb
    JOIN pg_catalog.pg_tablespace sp ON dtb.dattablespace=sp.oid
    WHERE dtb.oid = 16384::oid)
  END) as spcname,
  (CASE rel.relreplident
          WHEN 'd' THEN 'default'
          WHEN 'n' THEN 'nothing'
          WHEN 'f' THEN 'full'
          WHEN 'i' THEN 'index'
  END) as replica_identity,
  (select nspname FROM pg_catalog.pg_namespace WHERE oid = 2200::oid ) as schema,
  pg_catalog.pg_get_userbyid(rel.relowner) AS relowner, rel.relkind,
  (CASE WHEN rel.relkind = 'p' THEN true ELSE false END) AS is_partitioned,
  rel.relhassubclass, rel.reltuples::bigint, des.description, con.conname, con.conkey,
	EXISTS(select 1 FROM pg_catalog.pg_trigger
			JOIN pg_catalog.pg_proc pt ON pt.oid=tgfoid AND pt.proname='logtrigger'
			JOIN pg_catalog.pg_proc pc ON pc.pronamespace=pt.pronamespace AND pc.proname='slonyversion'
			WHERE tgrelid=rel.oid) AS isrepl,
	(SELECT count(*) FROM pg_catalog.pg_trigger WHERE tgrelid=rel.oid AND tgisinternal = FALSE) AS triggercount,
	(SELECT ARRAY(SELECT CASE WHEN (nspname NOT LIKE 'pg\_%') THEN
            pg_catalog.quote_ident(nspname)||'.'||pg_catalog.quote_ident(c.relname)
            ELSE pg_catalog.quote_ident(c.relname) END AS inherited_tables
    FROM pg_catalog.pg_inherits i
    JOIN pg_catalog.pg_class c ON c.oid = i.inhparent
    JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace
    WHERE i.inhrelid = rel.oid ORDER BY inhseqno)) AS coll_inherits,
  (SELECT count(*)
		FROM pg_catalog.pg_inherits i
      JOIN pg_catalog.pg_class c ON c.oid = i.inhparent
      JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace
		WHERE i.inhrelid = rel.oid) AS inherited_tables_cnt,
	(CASE WHEN rel.relpersistence = 'u' THEN true ELSE false END) AS relpersistence,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'fillfactor=([0-9]*)') AS fillfactor,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'parallel_workers=([0-9]*)') AS parallel_workers,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'toast_tuple_target=([0-9]*)') AS toast_tuple_target,
	(substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_enabled=([a-z|0-9]*)'))::BOOL AS autovacuum_enabled,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_vacuum_threshold=([0-9]*)') AS autovacuum_vacuum_threshold,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_vacuum_scale_factor=([0-9]*[.]?[0-9]*)') AS autovacuum_vacuum_scale_factor,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_analyze_threshold=([0-9]*)') AS autovacuum_analyze_threshold,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_analyze_scale_factor=([0-9]*[.]?[0-9]*)') AS autovacuum_analyze_scale_factor,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_vacuum_cost_delay=([0-9]*)') AS autovacuum_vacuum_cost_delay,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_vacuum_cost_limit=([0-9]*)') AS autovacuum_vacuum_cost_limit,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_freeze_min_age=([0-9]*)') AS autovacuum_freeze_min_age,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_freeze_max_age=([0-9]*)') AS autovacuum_freeze_max_age,
	substring(pg_catalog.array_to_string(rel.reloptions, ',') FROM 'autovacuum_freeze_table_age=([0-9]*)') AS autovacuum_freeze_table_age,
	(substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_enabled=([a-z|0-9]*)'))::BOOL AS toast_autovacuum_enabled,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_vacuum_threshold=([0-9]*)') AS toast_autovacuum_vacuum_threshold,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_vacuum_scale_factor=([0-9]*[.]?[0-9]*)') AS toast_autovacuum_vacuum_scale_factor,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_analyze_threshold=([0-9]*)') AS toast_autovacuum_analyze_threshold,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_analyze_scale_factor=([0-9]*[.]?[0-9]*)') AS toast_autovacuum_analyze_scale_factor,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_vacuum_cost_delay=([0-9]*)') AS toast_autovacuum_vacuum_cost_delay,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_vacuum_cost_limit=([0-9]*)') AS toast_autovacuum_vacuum_cost_limit,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_freeze_min_age=([0-9]*)') AS toast_autovacuum_freeze_min_age,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_freeze_max_age=([0-9]*)') AS toast_autovacuum_freeze_max_age,
	substring(pg_catalog.array_to_string(tst.reloptions, ',') FROM 'autovacuum_freeze_table_age=([0-9]*)') AS toast_autovacuum_freeze_table_age,
	rel.reloptions AS reloptions, tst.reloptions AS toast_reloptions, rel.reloftype,
	CASE WHEN typ.typname IS NOT NULL THEN (select pg_catalog.quote_ident(nspname) FROM pg_catalog.pg_namespace WHERE oid = 2200::oid )||'.'||pg_catalog.quote_ident(typ.typname) ELSE typ.typname END AS typname,
	typ.typrelid AS typoid, rel.relrowsecurity as rlspolicy, rel.relforcerowsecurity as forcerlspolicy,
	(CASE WHEN rel.reltoastrelid = 0 THEN false ELSE true END) AS hastoasttable,
	(SELECT pg_catalog.array_agg(provider || '=' || label) FROM pg_catalog.pg_seclabels sl1 WHERE sl1.objoid=rel.oid AND sl1.objsubid=0) AS seclabels,
	(CASE WHEN rel.oid <= 13756::oid THEN true ElSE false END) AS is_sys_table
	-- Added for partition table
    , (CASE WHEN rel.relkind = 'p' THEN pg_catalog.pg_get_partkeydef(16410::oid) ELSE '' END) AS partition_scheme FROM pg_catalog.pg_class rel
  LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
  LEFT OUTER JOIN pg_catalog.pg_description des ON (des.objoid=rel.oid AND des.objsubid=0 AND des.classoid='pg_class'::regclass)
  LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
  LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
  LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
AND NOT rel.relispartition
  AND rel.oid = 16410::oid ORDER BY rel.relname;
```
- 表总数
```
'SELECT COUNT(*)::text FROM public.t_order_1;'
```
- 权限
```
SELECT 'relacl' as deftype, COALESCE(gt.rolname, 'PUBLIC') grantee, g.rolname grantor,
    pg_catalog.array_agg(privilege_type) as privileges, pg_catalog.array_agg(is_grantable) as grantable
FROM
  (SELECT
    d.grantee, d.grantor, d.is_grantable,
    CASE d.privilege_type
		WHEN 'CONNECT' THEN 'c'
		WHEN 'CREATE' THEN 'C'
		WHEN 'DELETE' THEN 'd'
		WHEN 'EXECUTE' THEN 'X'
		WHEN 'INSERT' THEN 'a'
		WHEN 'REFERENCES' THEN 'x'
		WHEN 'SELECT' THEN 'r'
		WHEN 'TEMPORARY' THEN 'T'
		WHEN 'TRIGGER' THEN 't'
		WHEN 'TRUNCATE' THEN 'D'
		WHEN 'UPDATE' THEN 'w'
		WHEN 'USAGE' THEN 'U'
		ELSE 'UNKNOWN'
	END AS privilege_type
  FROM
    (SELECT rel.relacl
        FROM pg_catalog.pg_class rel
          LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
          LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
          LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
          LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
        WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
            AND rel.oid = 16410::oid
    ) acl,
    (SELECT (d).grantee AS grantee, (d).grantor AS grantor, (d).is_grantable
        AS is_grantable, (d).privilege_type AS privilege_type FROM (SELECT
        aclexplode(rel.relacl) as d
        FROM pg_catalog.pg_class rel
          LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
          LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
          LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
          LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
        WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
            AND rel.oid = 16410::oid
        ) a ORDER BY privilege_type) d
    ) d
  LEFT JOIN pg_catalog.pg_roles g ON (d.grantor = g.oid)
  LEFT JOIN pg_catalog.pg_roles gt ON (d.grantee = gt.oid)
GROUP BY g.rolname, gt.rolname
```
reverse engine for table
```
SELECT 'relacl' as deftype, COALESCE(gt.rolname, 'PUBLIC') grantee, g.rolname grantor,
    pg_catalog.array_agg(privilege_type) as privileges, pg_catalog.array_agg(is_grantable) as grantable
FROM
  (SELECT
    d.grantee, d.grantor, d.is_grantable,
    CASE d.privilege_type
		WHEN 'CONNECT' THEN 'c'
		WHEN 'CREATE' THEN 'C'
		WHEN 'DELETE' THEN 'd'
		WHEN 'EXECUTE' THEN 'X'
		WHEN 'INSERT' THEN 'a'
		WHEN 'REFERENCES' THEN 'x'
		WHEN 'SELECT' THEN 'r'
		WHEN 'TEMPORARY' THEN 'T'
		WHEN 'TRIGGER' THEN 't'
		WHEN 'TRUNCATE' THEN 'D'
		WHEN 'UPDATE' THEN 'w'
		WHEN 'USAGE' THEN 'U'
		ELSE 'UNKNOWN'
	END AS privilege_type
  FROM
    (SELECT rel.relacl
        FROM pg_catalog.pg_class rel
          LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
          LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
          LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
          LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
        WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
            AND rel.oid = 16410::oid
    ) acl,
    (SELECT (d).grantee AS grantee, (d).grantor AS grantor, (d).is_grantable
        AS is_grantable, (d).privilege_type AS privilege_type FROM (SELECT
        aclexplode(rel.relacl) as d
        FROM pg_catalog.pg_class rel
          LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
          LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
          LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
          LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
        WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
            AND rel.oid = 16410::oid
        ) a ORDER BY privilege_type) d
    ) d
  LEFT JOIN pg_catalog.pg_roles g ON (d.grantor = g.oid)
  LEFT JOIN pg_catalog.pg_roles gt ON (d.grantee = gt.oid)
GROUP BY g.rolname, gt.rolname
```
获取列
```
SELECT att.attname as name, att.atttypid, att.attlen, att.attnum, att.attndims,
		att.atttypmod, att.attacl, att.attnotnull, att.attoptions, att.attstattarget,
		att.attstorage, att.attidentity,
		pg_catalog.pg_get_expr(def.adbin, def.adrelid) AS defval,
		pg_catalog.format_type(ty.oid,NULL) AS typname,
        pg_catalog.format_type(ty.oid,att.atttypmod) AS displaytypname,
		pg_catalog.format_type(ty.oid,att.atttypmod) AS cltype,
        CASE WHEN ty.typelem > 0 THEN ty.typelem ELSE ty.oid END as elemoid,
		(SELECT nspname FROM pg_catalog.pg_namespace WHERE oid = ty.typnamespace) as typnspname,
        ty.typstorage AS defaultstorage,
		description, pi.indkey,
	(SELECT count(1) FROM pg_catalog.pg_type t2 WHERE t2.typname=ty.typname) > 1 AS isdup,
	CASE WHEN length(coll.collname::text) > 0 AND length(nspc.nspname::text) > 0  THEN
	  pg_catalog.concat(pg_catalog.quote_ident(nspc.nspname),'.',pg_catalog.quote_ident(coll.collname))
	ELSE '' END AS collspcname,
	EXISTS(SELECT 1 FROM pg_catalog.pg_constraint WHERE conrelid=att.attrelid AND contype='f' AND att.attnum=ANY(conkey)) As is_fk,
	(SELECT pg_catalog.array_agg(provider || '=' || label) FROM pg_catalog.pg_seclabels sl1 WHERE sl1.objoid=att.attrelid AND sl1.objsubid=att.attnum) AS seclabels,
	(CASE WHEN (att.attnum < 1) THEN true ElSE false END) AS is_sys_column,
	(CASE WHEN (att.attidentity in ('a', 'd')) THEN 'i' WHEN (att.attgenerated in ('s')) THEN 'g' ELSE 'n' END) AS colconstype,
	(CASE WHEN (att.attgenerated in ('s')) THEN pg_catalog.pg_get_expr(def.adbin, def.adrelid) END) AS genexpr, tab.relname as relname,
	(CASE WHEN tab.relkind = 'v' THEN true ELSE false END) AS is_view_only,
	seq.*
FROM pg_catalog.pg_attribute att
  JOIN pg_catalog.pg_type ty ON ty.oid=atttypid
  LEFT OUTER JOIN pg_catalog.pg_attrdef def ON adrelid=att.attrelid AND adnum=att.attnum
  LEFT OUTER JOIN pg_catalog.pg_description des ON (des.objoid=att.attrelid AND des.objsubid=att.attnum AND des.classoid='pg_class'::regclass)
  LEFT OUTER JOIN (pg_catalog.pg_depend dep JOIN pg_catalog.pg_class cs ON dep.classid='pg_class'::regclass AND dep.objid=cs.oid AND cs.relkind='S') ON dep.refobjid=att.attrelid AND dep.refobjsubid=att.attnum
  LEFT OUTER JOIN pg_catalog.pg_index pi ON pi.indrelid=att.attrelid AND indisprimary
  LEFT OUTER JOIN pg_catalog.pg_collation coll ON att.attcollation=coll.oid
  LEFT OUTER JOIN pg_catalog.pg_namespace nspc ON coll.collnamespace=nspc.oid
  LEFT OUTER JOIN pg_catalog.pg_sequence seq ON cs.oid=seq.seqrelid
  LEFT OUTER JOIN pg_catalog.pg_class tab on tab.oid = att.attrelid
WHERE att.attrelid = 16410::oid
    AND att.attnum > 0
    AND att.attisdropped IS FALSE
    ORDER BY att.attnum;
```
```
SELECT t.main_oid, pg_catalog.ARRAY_AGG(t.typname) as edit_types
FROM
(SELECT pc.castsource AS main_oid, pg_catalog.format_type(tt.oid,NULL) AS typname
FROM pg_catalog.pg_type tt
    JOIN pg_catalog.pg_cast pc ON tt.oid=pc.casttarget
    WHERE pc.castsource IN (23,1043)
    AND pc.castcontext IN ('i', 'a')
UNION
SELECT tt.typbasetype AS main_oid, pg_catalog.format_type(tt.oid,NULL) AS typname
FROM pg_catalog.pg_type tt
WHERE tt.typbasetype  IN (23,1043)
) t
GROUP BY t.main_oid;
```
```
SELECT 'attacl' as deftype,
    COALESCE(gt.rolname, 'PUBLIC') grantee,
    g.rolname grantor,
    pg_catalog.array_agg(privilege_type order by privilege_type) as privileges,
    pg_catalog.array_agg(is_grantable) as grantable
FROM
  (SELECT
    d.grantee, d.grantor, d.is_grantable,
    CASE d.privilege_type
        WHEN 'CONNECT' THEN 'c'
        WHEN 'CREATE' THEN 'C'
        WHEN 'DELETE' THEN 'd'
        WHEN 'EXECUTE' THEN 'X'
        WHEN 'INSERT' THEN 'a'
        WHEN 'REFERENCES' THEN 'x'
        WHEN 'SELECT' THEN 'r'
        WHEN 'TEMPORARY' THEN 'T'
        WHEN 'TRIGGER' THEN 't'
        WHEN 'TRUNCATE' THEN 'D'
        WHEN 'UPDATE' THEN 'w'
        WHEN 'USAGE' THEN 'U'
        ELSE 'UNKNOWN'
    END AS privilege_type
  FROM
    (SELECT attacl
        FROM pg_catalog.pg_attribute att
        WHERE att.attrelid = 16410::oid
        AND att.attnum = 1::int
    ) acl,
    (SELECT (d).grantee AS grantee, (d).grantor AS grantor, (d).is_grantable
        AS is_grantable, (d).privilege_type AS privilege_type FROM (SELECT
        pg_catalog.aclexplode(attacl) as d FROM pg_catalog.pg_attribute att
        WHERE att.attrelid = 16410::oid
        AND att.attnum = 1::int) a) d
    ) d
  LEFT JOIN pg_catalog.pg_roles g ON (d.grantor = g.oid)
  LEFT JOIN pg_catalog.pg_roles gt ON (d.grantee = gt.oid)
GROUP BY g.rolname, gt.rolname
ORDER BY grantee
```
```
SELECT cls.oid,
    cls.relname as name,
    indnkeyatts as col_count,
    CASE WHEN length(spcname::text) > 0 THEN spcname ELSE
        (SELECT sp.spcname FROM pg_catalog.pg_database dtb
        JOIN pg_catalog.pg_tablespace sp ON dtb.dattablespace=sp.oid
        WHERE dtb.oid = 16384::oid)
    END as spcname,
    CASE contype
        WHEN 'p' THEN desp.description
        WHEN 'u' THEN desp.description
        WHEN 'x' THEN desp.description
        ELSE des.description
    END AS comment,
    condeferrable,
    condeferred,
    conislocal,
    substring(pg_catalog.array_to_string(cls.reloptions, ',') from 'fillfactor=([0-9]*)') AS fillfactor
FROM pg_catalog.pg_index idx
JOIN pg_catalog.pg_class cls ON cls.oid=indexrelid
LEFT OUTER JOIN pg_catalog.pg_tablespace ta on ta.oid=cls.reltablespace
LEFT JOIN pg_catalog.pg_depend dep ON (dep.classid = cls.tableoid AND dep.objid = cls.oid AND dep.refobjsubid = '0' AND dep.refclassid=(SELECT oid FROM pg_catalog.pg_class WHERE relname='pg_constraint') AND dep.deptype='i')
LEFT OUTER JOIN pg_catalog.pg_constraint con ON (con.tableoid = dep.refclassid AND con.oid = dep.refobjid)
LEFT OUTER JOIN pg_catalog.pg_description des ON (des.objoid=cls.oid AND des.classoid='pg_class'::regclass)
LEFT OUTER JOIN pg_catalog.pg_description desp ON (desp.objoid=con.oid AND desp.objsubid = 0 AND desp.classoid='pg_constraint'::regclass)
WHERE indrelid = 16410::oid
AND contype='u'
ORDER BY cls.relname
```
```
SELECT c.oid, conname as name, relname, nspname, description as comment,
       pg_catalog.pg_get_expr(conbin, conrelid, true) as consrc,
       connoinherit, NOT convalidated as convalidated, conislocal
    FROM pg_catalog.pg_constraint c
    JOIN pg_catalog.pg_class cl ON cl.oid=conrelid
    JOIN pg_catalog.pg_namespace nl ON nl.oid=relnamespace
LEFT OUTER JOIN
    pg_catalog.pg_description des ON (des.objoid=c.oid AND
                           des.classoid='pg_constraint'::regclass)
WHERE contype = 'c'
    AND conrelid = 16410::oid
```
```
SELECT cls.oid,
    cls.relname as name,
    indnkeyatts as col_count,
    amname,
    CASE WHEN length(spcname::text) > 0 THEN spcname ELSE
        (SELECT sp.spcname FROM pg_catalog.pg_database dtb
        JOIN pg_catalog.pg_tablespace sp ON dtb.dattablespace=sp.oid
        WHERE dtb.oid = 16384::oid)
    END as spcname,
    CASE contype
        WHEN 'p' THEN desp.description
        WHEN 'u' THEN desp.description
        WHEN 'x' THEN desp.description
        ELSE des.description
    END AS comment,
    condeferrable,
    condeferred,
    substring(pg_catalog.array_to_string(cls.reloptions, ',') from 'fillfactor=([0-9]*)') AS fillfactor,
    pg_catalog.pg_get_expr(idx.indpred, idx.indrelid, true) AS indconstraint
FROM pg_catalog.pg_index idx
JOIN pg_catalog.pg_class cls ON cls.oid=indexrelid
LEFT OUTER JOIN pg_catalog.pg_tablespace ta on ta.oid=cls.reltablespace
JOIN pg_catalog.pg_am am ON am.oid=cls.relam
LEFT JOIN pg_catalog.pg_depend dep ON (dep.classid = cls.tableoid AND dep.objid = cls.oid AND dep.refobjsubid = '0' AND dep.refclassid=(SELECT oid FROM pg_catalog.pg_class WHERE relname='pg_constraint') AND dep.deptype='i')
LEFT OUTER JOIN pg_catalog.pg_constraint con ON (con.tableoid = dep.refclassid AND con.oid = dep.refobjid)
LEFT OUTER JOIN pg_catalog.pg_description des ON (des.objoid=cls.oid AND des.classoid='pg_class'::regclass)
LEFT OUTER JOIN pg_catalog.pg_description desp ON (desp.objoid=con.oid AND desp.objsubid = 0 AND desp.classoid='pg_constraint'::regclass)
WHERE indrelid = 16410::oid
AND contype='x'
ORDER BY cls.relname
```
Table 结束，接下来是 index
```
SELECT DISTINCT ON(cls.relname) cls.oid, cls.relname as name,
(SELECT (CASE WHEN count(i.inhrelid) > 0 THEN true ELSE false END) FROM pg_inherits i WHERE i.inhrelid = cls.oid) as is_inherited
FROM pg_catalog.pg_index idx
    JOIN pg_catalog.pg_class cls ON cls.oid=indexrelid
    JOIN pg_catalog.pg_class tab ON tab.oid=indrelid
    LEFT OUTER JOIN pg_catalog.pg_tablespace ta on ta.oid=cls.reltablespace
    JOIN pg_catalog.pg_namespace n ON n.oid=tab.relnamespace
    JOIN pg_catalog.pg_am am ON am.oid=cls.relam
    LEFT JOIN pg_catalog.pg_depend dep ON (dep.classid = cls.tableoid AND dep.objid = cls.oid AND dep.refobjsubid = '0' AND dep.refclassid=(SELECT oid FROM pg_catalog.pg_class WHERE relname='pg_constraint') AND dep.deptype='i')
    LEFT OUTER JOIN pg_catalog.pg_constraint con ON (con.tableoid = dep.refclassid AND con.oid = dep.refobjid)
WHERE indrelid = 16410::OID
    AND conname is NULL
    ORDER BY cls.relname
```
ROW SECURITY POLICY
```
'SELECT
    pl.oid AS oid,
    pl.polname AS name
FROM
    pg_catalog.pg_policy pl
WHERE
    pl.polrelid	 = 16410
ORDER BY
    pl.polname;'
```
trigger
```sql 
SELECT t.oid, t.tgname as name, t.tgenabled AS is_enable_trigger
FROM pg_catalog.pg_trigger t
    WHERE NOT tgisinternal
    AND tgrelid = 16410::OID
    ORDER BY tgname;
```
rules
```sql
SELECT
    rw.oid AS oid,
    rw.rulename AS name,
    CASE WHEN rw.ev_enabled != \'D\' THEN True ELSE False END AS enabled,
    rw.ev_enabled AS is_enable_rule

FROM
    pg_catalog.pg_rewrite rw
WHERE
    rw.ev_class = 16410
ORDER BY
    rw.rulename
```