# PostgreSQL优化器分析--扫描与连接

扫描和连接是PostgreSQL执行查询的基本操作，所有的查询指令都涉及扫描操作，多表查询或子查询时则涉及连接操作。

## **扫描**

PostgreSQL中常用的扫描类型有：

* 顺序扫描（seq-scan）
* 索引扫描（index-scan）
* 位图索引扫描（bitmap-index scan/bitmap-heap scan）
* 仅索引扫描（index-only scan）

顺序扫描，顾名思义，就是顺序的一条一条的访问表元组。顺序扫描是表扫描中最基本的扫描方法，若表上不存在索引，或者即使存在索引，但是优化器认为顺序扫描的代价比较低时（比如select * 返回全量数据），就会选择顺序扫描的方法。

```sql
postgres=# explain select * from table_scan_test;
                                QUERY PLAN                                
--------------------------------------------------------------------------
 Seq Scan on table_scan_test  (cost=0.00..15406.00 rows=1000000 width=13)
(1 row)

postgres=# 
```

索引扫描，即通过扫描索引的方式先获取到元组的TID，而后通过元组的TID快速定位并获取元组数据。由于索引是排序后的数据，因此在进行where限定查询时，能够快速返回查询数据。需要注意，虽然索引是排序的，但是数据是未排序的，通过TID获取数据时，仍然需要对数据页进行随机读取操作。同时，若多个字段上存在联合索引，则在查询的where限定条件中需要从左到右的包含字段，否则也不会走索引扫描。

```sql
postgres=# \d+ scan_test
                                      Table "public.scan_test"
 Column |       Type        | Collation | Nullable | Default | Storage  | Stats target | Description 
--------+-------------------+-----------+----------+---------+----------+--------------+-------------
 id     | bigint            |           |          |         | plain    |              | 
 name   | character varying |           |          |         | extended |              | 
Indexes:
    "scan_test_id_index" btree (id)
    "scan_test_id_name_index" btree (id, name)

--- 索引扫描（此示例是在创建联合索引之前运行，若创建联合索引后，该查询会走仅索引扫描）
postgres=# explain select * from scan_test where id = 1111;
                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Index Scan using scan_test_id_index on scan_test  (cost=0.42..8.44 rows=1 width=13)
   Index Cond: (id = 1111)
(2 rows)

--- 虽然存在联合索引，但由于限定条件不包含索引字段的首字段，因此走顺序扫描
postgres=# explain select * from scan_test where name = '1111';
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Gather  (cost=1000.00..11617.63 rows=33 width=13)
   Workers Planned: 2
   ->  Parallel Seq Scan on scan_test  (cost=0.00..10614.33 rows=14 width=13)
         Filter: ((name)::text = '1111'::text)
(4 rows)

postgres=# 
```

位图索引扫描，是索引扫描的一种变体，也算是一种改良版。在索引扫描中，通过查询索引获取到满足限定条件的元组的TID后，在获取该TID对应的元组前，需要先读取该TID所在的page，然后才能返回TID对应的元组。若查询过程中满足条件的元组数比较多，则需要频繁进行数据页的读取，尤其是数据分布比较分散时，可能对于同一个数据页需要进行多次随机读取操作。

为了解决这种问题，引入了位图索引扫描，其解决办法是，先将满足条件的元组的TID对应的bit位置1，这样若某一个元组满足条件，则该元组对应的页面就会被标记，然后顺序获取标记过的页面，对于该页面顺序扫描并检查是否满足条件，若满足则返回，否则继续扫描。也就说，位图索引扫描将索引扫描中的随机读取数据页的行为，变成了顺序读取的行为。在执行计划中，位图索引扫描必然是Bitmap Index Scan和Bitmap Heap Scan的联合体。

```sql
postgres=# explain select * from table_scan_test where id = 1111;
                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on table_scan_test  (cost=5.20..365.65 rows=100 width=13)
   Recheck Cond: (id = 1111)
   ->  Bitmap Index Scan on table_scan_test_index_id_name  (cost=0.00..5.17 rows=100 width=0)
         Index Cond: (id = 1111)
(4 rows)

postgres=# 
```

仅索引扫描也是索引扫描的一种变体。若表的索引信息，已经包含了查询所需的所有信息，则优化器会使用仅索引扫描，避免访问数据页。当然，仅索引扫描并不是再也不会访问数据页了，由于索引页中并不包含当前元组的事务信息，即不能通过索引项判断当前元组的可见性，因此在必要时还是会访问数据页来判断元组的可见性。更多关于索引扫描和仅索引扫描的描述，可以参考[这里](https://medium.com/pgmustard/index-only-scans-in-postgres-a1603f551ee)

```sql
postgres=# explain select * from table_scan_test where id = 9999 and name ='1111';
                                                QUERY PLAN                                                 
-----------------------------------------------------------------------------------------------------------
 Index Only Scan using table_scan_test_index_id_name on table_scan_test  (cost=0.42..8.45 rows=1 width=13)
   Index Cond: ((id = 9999) AND (name = '1111'::text))
(2 rows)

postgres=# 
```

## **连接**

PostgreSQL中基本的连接类型有：

* 嵌套循环连接（nested-loop join）
* 归并连接（merge join）
* 哈希连接（hash join）

预置条件：

```sql
postgres=# create table table1(id1 int, id2 int);
CREATE TABLE
postgres=# create table table2(id1 int, id2 int);
CREATE TABLE
postgres=# insert into table1 values(generate_series(1,10000),3);
INSERT 0 10000
postgres=# insert into table2 values(generate_series(1,1000),3);
INSERT 0 1000
postgres=# analyze;
ANALYZE
```

嵌套循环连接是最基本和最简单的连接类型，所有的连接条件和连接类型都可以使用嵌套循环连接来实现，其实现机制是：循环遍历外层表，对于外层表的每条元组，都遍历扫描内层表，寻找匹配的元组。伪代码示例如下

```c
for each tuple r in table1
    for each tuple s in table2
        if r and s match the join condition
           emit output tuple (r,s)
```

具体的查询示例如下

```sql
postgres=# explain select * from table1, table2;
                              QUERY PLAN                              
----------------------------------------------------------------------
 Nested Loop  (cost=0.00..125162.50 rows=10000000 width=16)
   ->  Seq Scan on table1  (cost=0.00..145.00 rows=10000 width=8)
   ->  Materialize  (cost=0.00..20.00 rows=1000 width=8)
         ->  Seq Scan on table2  (cost=0.00..15.00 rows=1000 width=8)
(4 rows)

postgres=# 
```

归并连接仅仅可以用于等值条件的连接，在连接时分为两个阶段：首先对于需要连接的两个表，依据其连接键先进行排序；然后再迭代遍历排序后的数据以寻找匹配的元组。伪代码示例如下

```c
sort(table1)
sort(table2)
for each tuple r in table1
    for each tuple s in table2
        if (r.key = s.key)
            emit output tuple (r,s)
         if (r.key > s.key)
            continue;
         else
            break;
```

具体的查询示例如下：

```sql
postgres=# set enable_nestloop=off;
SET
postgres=# 
postgres=# set enable_hashjoin = off;
SET
postgres=# 
postgres=# explain select * from table1 t1, table2 t2 where t1.id1 = t2.id1;
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Merge Join  (cost=874.21..894.21 rows=1000 width=16)
   Merge Cond: (t1.id1 = t2.id1)
   ->  Sort  (cost=809.39..834.39 rows=10000 width=8)
         Sort Key: t1.id1
         ->  Seq Scan on table1 t1  (cost=0.00..145.00 rows=10000 width=8)
   ->  Sort  (cost=64.83..67.33 rows=1000 width=8)
         Sort Key: t2.id1
         ->  Seq Scan on table2 t2  (cost=0.00..15.00 rows=1000 width=8)
(8 rows)

postgres=# 
```

和归并连接类似，哈希连接也分为两个阶段：构建阶段和探测阶段。其构建阶段和哈希阶段的功能如下伪码所示：

```c
Build Phase:
    for each tuple r in inner relation table2
        insert r into hash table HashTab with key r.key

Probe Phase:
    for each tuple s in outer relation table1
        for each tuple r in bucket HashTab[s.key]
            if (s.key = r.key)
                emit output tuple (r,s)
```

具体的查询示例如下：

```sql
postgres=# set enable_nestloop = off;
SET
postgres=# set enable_mergejoin = off;
SET
postgres=# explain select * from table1 t1, table2 t2 where t1.id1 = t2.id1;
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Hash Join  (cost=27.50..220.00 rows=1000 width=16)
   Hash Cond: (t1.id1 = t2.id1)
   ->  Seq Scan on table1 t1  (cost=0.00..145.00 rows=10000 width=8)
   ->  Hash  (cost=15.00..15.00 rows=1000 width=8)
         ->  Seq Scan on table2 t2  (cost=0.00..15.00 rows=1000 width=8)
(5 rows)

postgres=# 
```

【参考链接】

[http://www.interdb.jp/pg/pgsql03.html](http://www.interdb.jp/pg/pgsql03.html)

[https://severalnines.com/database-blog/overview-join-methods-postgresql](https://severalnines.com/database-blog/overview-join-methods-postgresql)
