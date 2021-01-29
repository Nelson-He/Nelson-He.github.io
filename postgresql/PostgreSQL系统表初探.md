### 1. PostgreSQL中的索引和约束，怎么区分是系统创建的还是用户手工创建的。

在postgresql中，虽然有明确的视图来区分系统表上的索引和用户表上的索引，但是目前没有明确的手段来区分一个索引是用户手动创建的还是系统自动创建的。

### 2. 索引的组成列中，索引列的升序和降序如何查询得知

索引列的升序和降序信息，没有一个明确的字段来存储，只是在pg_index的indoption字段中有比特位来存储。

关于indoption字段的作用，在pgindex.h中有如下解释和定义

```c
/*
 * Index AMs that support ordered scans must support these two indoption
 * bits.  Otherwise, the content of the per-column indoption fields is
 * open for future definition.
 */
#define INDOPTION_DESC			0x0001	/* values are in reverse order */
#define INDOPTION_NULLS_FIRST	0x0002	/* NULLs are first instead of last */
```

查看调用点，发现在该bit位置1时为DESC，否则默认为ASC，如下

```c
/* if it supports sort ordering, report DESC and NULLS opts */
if (opt & INDOPTION_DESC)
{
	appendStringInfoString(&buf, " DESC");
	/* NULLS FIRST is the default in this case */
	if (!(opt & INDOPTION_NULLS_FIRST))
		appendStringInfoString(&buf, " NULLS LAST");
}
else
{
	if (opt & INDOPTION_NULLS_FIRST)
		appendStringInfoString(&buf, " NULLS FIRST");
}
```

* **综上信息，结合pg系统表，我们可以得到查询索引是升序还是降序的sql如下**

```sql
SELECT
      t.relname AS tablename,
      i.relname AS indexname, pg_attribute.attname AS colname,
      k AS col_order,
      CASE WHEN i.indoption[k-1] & 1 = 1 THEN 'DESC' ELSE 'ASC' END AS descasc,
      CASE WHEN (i.indoption[k-1] & 2 = 2) THEN 'NULLS FIRST' ELSE 'NULLS LAST' END AS nulls
    FROM (
      SELECT
        pg_class.relname,
        pg_index.indrelid, pg_index.indclass, pg_index.indoption,
        unnest(pg_index.indkey) AS k
      FROM pg_index
      INNER JOIN pg_class ON pg_index.indexrelid = pg_class.oid
      WHERE relname = 'cities_pkey'  ---index_name
    ) i
    INNER JOIN pg_opclass on (pg_opclass.oid = i.indclass[k-1])
    INNER JOIN pg_am ON (pg_opclass.opcmethod = pg_am.oid)
    INNER JOIN pg_class t ON i.indrelid = t.oid
    INNER JOIN pg_attribute ON (pg_attribute.attrelid = i.indrelid AND pg_attribute.attnum = k);
```

下面进行验证，我们创建两个表和两个索引，一个升序，一个降序，然后分别查询

```sql
---测试desc
postgres=# create table test_desc(id1 bigint, id2 bigint);
CREATE TABLE
postgres=# insert into test_desc select random() * i, random() * i from generate_series(1,10000) i;
INSERT 0 10000
postgres=# 
postgres=# create index test_desc_index on test_desc(id1 desc);
CREATE INDEX
postgres=# 
postgres=# SELECT
postgres-#       t.relname AS tablename,
postgres-#       i.relname AS indexname, pg_attribute.attname AS colname,
postgres-#       k AS col_order,
postgres-#       CASE WHEN i.indoption[k-1] & 1 = 1 THEN 'DESC' ELSE 'ASC' END AS descasc,
postgres-#       CASE WHEN (i.indoption[k-1] & 2 = 2) THEN 'NULLS FIRST' ELSE 'NULLS LAST' END AS nulls
postgres-#     FROM (
postgres(#       SELECT
postgres(#         pg_class.relname,
postgres(#         pg_index.indrelid, pg_index.indclass, pg_index.indoption,
postgres(#         unnest(pg_index.indkey) AS k
postgres(#       FROM pg_index
postgres(#       INNER JOIN pg_class ON pg_index.indexrelid = pg_class.oid
postgres(#       WHERE relname = 'test_desc_index'  ---index_name
postgres(#     ) i
postgres-#     INNER JOIN pg_opclass on (pg_opclass.oid = i.indclass[k-1])
postgres-#     INNER JOIN pg_am ON (pg_opclass.opcmethod = pg_am.oid)
postgres-#     INNER JOIN pg_class t ON i.indrelid = t.oid
postgres-#     INNER JOIN pg_attribute ON (pg_attribute.attrelid = i.indrelid AND pg_attribute.attnum = k);
 tablename |    indexname    | colname | col_order | descasc |    nulls    
-----------+-----------------+---------+-----------+---------+-------------
 test_desc | test_desc_index | id1     |         1 | DESC    | NULLS FIRST
(1 row)

postgres=# 
---测试asc
postgres=# create table test_asc(id1 bigint, id2 bigint);
CREATE TABLE
postgres=# insert into test_asc select random() * i, random() * i from generate_series(1,10000) i;
INSERT 0 10000
postgres=# create index test_asc_index on test_desc(id1 asc);
CREATE INDEX
postgres=# 
postgres=# SELECT
postgres-#       t.relname AS tablename,
postgres-#       i.relname AS indexname, pg_attribute.attname AS colname,
postgres-#       k AS col_order,
postgres-#       CASE WHEN i.indoption[k-1] & 1 = 1 THEN 'DESC' ELSE 'ASC' END AS descasc,
postgres-#       CASE WHEN (i.indoption[k-1] & 2 = 2) THEN 'NULLS FIRST' ELSE 'NULLS LAST' END AS nulls
postgres-#     FROM (
postgres(#       SELECT
postgres(#         pg_class.relname,
postgres(#         pg_index.indrelid, pg_index.indclass, pg_index.indoption,
postgres(#         unnest(pg_index.indkey) AS k
postgres(#       FROM pg_index
postgres(#       INNER JOIN pg_class ON pg_index.indexrelid = pg_class.oid
postgres(#       WHERE relname = 'test_asc_index'  ---index_name
postgres(#     ) i
postgres-#     INNER JOIN pg_opclass on (pg_opclass.oid = i.indclass[k-1])
postgres-#     INNER JOIN pg_am ON (pg_opclass.opcmethod = pg_am.oid)
postgres-#     INNER JOIN pg_class t ON i.indrelid = t.oid
postgres-#     INNER JOIN pg_attribute ON (pg_attribute.attrelid = i.indrelid AND pg_attribute.attnum = k);
 tablename |   indexname    | colname | col_order | descasc |   nulls    
-----------+----------------+---------+-----------+---------+------------
 test_desc | test_asc_index | id1     |         1 | ASC     | NULLS LAST
(1 row)

postgres=# 
```



### 3. 创建表时的表空间如何确定
 PG 中有系统视图pg_tables存储了表相关的信息，可以查询该视图获得表的tablespace信息，如下
```sql
postgres=# select case when tablespace is null then 'pg_default' else tablespace end from pg_tables where tablename='cities';
 tablespace 
------------
 pg_default
(1 row)

postgres=# 
```

之所以需要判空，是因为在存储为默认表空间时，pg_class中的reltablespace字段会被置为0，因此在视图关联时无法关联到，pg_tables的定义如下

```sql
 SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    pg_get_userbyid(c.relowner) AS tableowner,
    t.spcname AS tablespace,
    c.relhasindex AS hasindexes,
    c.relhasrules AS hasrules,
    c.relhastriggers AS hastriggers,
    c.relrowsecurity AS rowsecurity
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_tablespace t ON t.oid = c.reltablespace
  WHERE c.relkind = ANY (ARRAY['r'::"char", 'p'::"char"]);
```

