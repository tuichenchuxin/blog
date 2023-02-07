---
title: "How to get postgres create table sql"
date: 2022-04-08T14:09:16+08:00
draft: false
---
## 获取 postgres 的建表语句的几种方法
### 使用 pg_dump
可以使用 pg_dump 直接查看建表语句
以本机 docker 的 postgres 为例
```
docker run -it --rm  postgres pg_dump -h host.docker.internal -p 5432 -U postgres -d demo_ds_0 -s -t t_order_0
```
结果展示
``` sql
--
-- PostgreSQL database dump
--

-- Dumped from database version 14.2 (Debian 14.2-1.pgdg110+1)
-- Dumped by pg_dump version 14.2 (Debian 14.2-1.pgdg110+1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: t_order_0; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.t_order_0 (
    order_id integer NOT NULL,
    user_id integer NOT NULL,
    status character varying(45)
);


ALTER TABLE public.t_order_0 OWNER TO postgres;

--
-- Name: TABLE t_order_0; Type: COMMENT; Schema: public; Owner: postgres
--

COMMENT ON TABLE public.t_order_0 IS 'haha';


--
-- Name: COLUMN t_order_0.order_id; Type: COMMENT; Schema: public; Owner: postgres
--

COMMENT ON COLUMN public.t_order_0.order_id IS 'haha';


--
-- Name: t_order_0 t_order_0_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.t_order_0
    ADD CONSTRAINT t_order_0_pkey PRIMARY KEY (order_id);


--
-- PostgreSQL database dump complete
--
```


### 使用自定义函数
创建函数如下
``` sql 
CREATE OR REPLACE FUNCTION tabledef(oid) RETURNS text
LANGUAGE sql STRICT AS $$
/* snatched from https://github.com/filiprem/pg-tools */
WITH attrdef AS (
    SELECT
        n.nspname,
        c.relname,
        pg_catalog.array_to_string(c.reloptions || array(select 'toast.' || x from pg_catalog.unnest(tc.reloptions) x), ', ') as relopts,
        c.relpersistence,
        a.attnum,
        a.attname,
        pg_catalog.format_type(a.atttypid, a.atttypmod) as atttype,
        (SELECT substring(pg_catalog.pg_get_expr(d.adbin, d.adrelid, true) for 128) FROM pg_catalog.pg_attrdef d
            WHERE d.adrelid = a.attrelid AND d.adnum = a.attnum AND a.atthasdef) as attdefault,
        a.attnotnull,
        (SELECT c.collname FROM pg_catalog.pg_collation c, pg_catalog.pg_type t
            WHERE c.oid = a.attcollation AND t.oid = a.atttypid AND a.attcollation <> t.typcollation) as attcollation,
        a.attidentity,
        a.attgenerated
    FROM pg_catalog.pg_attribute a
    JOIN pg_catalog.pg_class c ON a.attrelid = c.oid
    JOIN pg_catalog.pg_namespace n ON c.relnamespace = n.oid
    LEFT JOIN pg_catalog.pg_class tc ON (c.reltoastrelid = tc.oid)
    WHERE a.attrelid = $1
        AND a.attnum > 0
        AND NOT a.attisdropped
    ORDER BY a.attnum
),
coldef AS (
    SELECT
        attrdef.nspname,
        attrdef.relname,
        attrdef.relopts,
        attrdef.relpersistence,
        pg_catalog.format(
            '%I %s%s%s%s%s',
            attrdef.attname,
            attrdef.atttype,
            case when attrdef.attcollation is null then '' else pg_catalog.format(' COLLATE %I', attrdef.attcollation) end,
            case when attrdef.attnotnull then ' NOT NULL' else '' end,
            case when attrdef.attdefault is null then ''
                else case when attrdef.attgenerated = 's' then pg_catalog.format(' GENERATED ALWAYS AS (%s) STORED', attrdef.attdefault)
                    when attrdef.attgenerated <> '' then ' GENERATED AS NOT_IMPLEMENTED'
                    else pg_catalog.format(' DEFAULT %s', attrdef.attdefault)
                end
            end,
            case when attrdef.attidentity<>'' then pg_catalog.format(' GENERATED %s AS IDENTITY',
                    case attrdef.attidentity when 'd' then 'BY DEFAULT' when 'a' then 'ALWAYS' else 'NOT_IMPLEMENTED' end)
                else '' end
        ) as col_create_sql
    FROM attrdef
    ORDER BY attrdef.attnum
),
tabdef AS (
    SELECT
        coldef.nspname,
        coldef.relname,
        coldef.relopts,
        coldef.relpersistence,
        string_agg(coldef.col_create_sql, E',\n    ') as cols_create_sql
    FROM coldef
    GROUP BY
        coldef.nspname, coldef.relname, coldef.relopts, coldef.relpersistence
)
SELECT
    format(
        'CREATE%s TABLE %I.%I%s%s%s;',
        case tabdef.relpersistence when 't' then ' TEMP' when 'u' then ' UNLOGGED' else '' end,
        tabdef.nspname,
        tabdef.relname,
        coalesce(
            (SELECT format(E'\n    PARTITION OF %I.%I %s\n', pn.nspname, pc.relname,
                pg_get_expr(c.relpartbound, c.oid))
                FROM pg_class c JOIN pg_inherits i ON c.oid = i.inhrelid
                JOIN pg_class pc ON pc.oid = i.inhparent
                JOIN pg_namespace pn ON pn.oid = pc.relnamespace
                WHERE c.oid = $1),
            format(E' (\n    %s\n)', tabdef.cols_create_sql)
        ),
        case when tabdef.relopts <> '' then format(' WITH (%s)', tabdef.relopts) else '' end,
        coalesce(E'\nPARTITION BY '||pg_get_partkeydef($1), '')
    ) as table_create_sql
FROM tabdef
$$;
```

### 通过 pgAdmin4 获取
抓包后，发现 pgAdmin4 实际查询的 sql 如下
``` sql
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
    , (CASE WHEN rel.relkind = 'p' THEN pg_catalog.pg_get_partkeydef(16396::oid) ELSE '' END) AS partition_scheme FROM pg_catalog.pg_class rel
  LEFT OUTER JOIN pg_catalog.pg_tablespace spc on spc.oid=rel.reltablespace
  LEFT OUTER JOIN pg_catalog.pg_description des ON (des.objoid=rel.oid AND des.objsubid=0 AND des.classoid='pg_class'::regclass)
  LEFT OUTER JOIN pg_catalog.pg_constraint con ON con.conrelid=rel.oid AND con.contype='p'
  LEFT OUTER JOIN pg_catalog.pg_class tst ON tst.oid = rel.reltoastrelid
  LEFT JOIN pg_catalog.pg_type typ ON rel.reloftype=typ.oid
WHERE rel.relkind IN ('r','s','t','p') AND rel.relnamespace = 2200::oid
AND NOT rel.relispartition
  AND rel.oid = 16396::oid ORDER BY rel.relname;
  ```
  https://github.com/postgres/pgadmin4
  