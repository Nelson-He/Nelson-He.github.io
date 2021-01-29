### 1. PostgreSQL中的索引和约束，怎么区分是系统创建的还是用户手工创建的。

在postgresql中，虽然有明确的视图来区分系统表上的索引和用户表上的索引，但是目前没有明确的手段来区分一个索引是用户手动创建的还是系统自动创建的。

### 2. 索引的组成列中，索引列的升序和降序如何查询得知

索引列的升序和降序信息，没有一个明确的字段来存储，只是在pg_index的indoption字段中有比特位来存储。关于具体的信息，正在查询资料。

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

