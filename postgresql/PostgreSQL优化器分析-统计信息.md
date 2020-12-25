# PostgreSQL优化器分析--统计信息与行数评估

PostgreSQL的优化器是一种CBO的优化器，即基于代价的优化。那么在优化器选择最优执行路径时，如何来确定某一条执行路径的优劣呢？其实现是计算每一条路径的代价，选择代价最低的路径作为最优路径。而在计算代价的过程中，统计信息发挥了重要的作用。  可以说，统计信息很大程度上决定了优化器将要选择的执行路径。

比如有一张雇员信息表employees，其id字段是主键：

```sql
create table employees(
id  bigint primary key,
name varchar(64),
location text);
```

向里面随机插入10000条数据：

  ```sql
insert into employees select id, to_char(id, 'FM0000'), to_char(random()*100, 'FM0000') from generate_series(1, 10000) id order by random();
  ```

随后我们很快在这张表上搜索一下所有id小于10的雇员信息，查看它的执行计划，如下

```sql
postgres=# explain select * from employees where id < 10;
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..94.40 rows=811 width=186)
   Filter: (id < 10)
(2 rows)
```

What？What? What? 看见这样的执行计划信息，估计所有人的内心都有上万只羊驼跑过：这PG也太不靠谱了，id上明明有个索引可以用来加速，为啥子还是顺序扫描呢？Plus，id<10的数据怎么算也不会超过10，怎么执行计划中预估的行数这么多，这也太随意了吧。

为了解开上面的谜团，帮PG甩开这个锅，我们有必要深入的探一探统计信息与行评估的关系。

## 系统视图

和统计信息相关的几个重要系统视图如下，关于每个视图的具体作用可以参考相应链接，此处不再赘述

* [pg_stat_database](https://www.postgresql.org/docs/12/monitoring-stats.html#PG-STAT-DATABASE-VIEW)
* [pg_stat_user_tables](https://www.postgresql.org/docs/12/monitoring-stats.html#PG-STAT-ALL-TABLES-VIEW)
* [pg_stat_user_indexes](https://www.postgresql.org/docs/12/monitoring-stats.html#PG-STAT-ALL-INDEXES-VIEW)
* [pg_statio_user_tables](https://www.postgresql.org/docs/12/monitoring-stats.html#PG-STATIO-ALL-TABLES-VIEW)
* [pg_stat_bgwriter](https://www.postgresql.org/docs/12/monitoring-stats.html#PG-STAT-BGWRITER-VIEW) 
* [pg_stats](https://www.postgresql.org/docs/12/view-pg-stats.html)

## 影响参数

先将影响统计信息的几个重要参数罗列出来，然后再来看下这些参数是如何影响统计信息的。

* [autovacuum](https://www.postgresql.org/docs/12/runtime-config-autovacuum.html)                                           是否启用autovacuum进程
* [autovacuum_naptime](https://www.postgresql.org/docs/12/runtime-config-autovacuum.html)                            两次autovacuum执行的休眠间隔，即本次启动距上次启动的时间间隔
* [autovacuum_analyze_threshold](https://www.postgresql.org/docs/12/runtime-config-autovacuum.html)            autovacuum进程执行analyze操作的阈值
* [autovacuum_analyze_scale_factor](https://www.postgresql.org/docs/12/runtime-config-autovacuum.html)       autovacuum进程执行analyze操作的阈值系数
* [track_activities](https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-ACTIVITIES)                                       是否监控统计服务器执行的命令
* [track_counts](https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-COUNTS)                                          是否统计表和索引的访问次数
* [track_functions](https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-FUNCTIONS)                                      是否统计用户函数的使用信息
* [track_io_timing](https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-IO-TIMING)                                      是否监控统计块设备的IO状态信息

上面的这些参数影响autovacuum和stats collector进程的运行，而这两个进程又直接决定了统计信息的准确与否。以autovacuum_naptime参数为例，在naptime之间产生的元组更新或删除就可能会统计不准。

回过头来再看上面的执行计划，猜测是由于insert插入时间短，且恰好处于autovacuum进程休眠期间，导致该表的统计信息还没有来得及被统计分析，统计信息不准，进而导致执行计划选择不准。为了证明这点，我们删除表后重建，在插入数据后快速查询pg_stat_user_tables视图，查询信息如下

```sql
postgres=# select * from pg_stat_user_tables where relname = 'employees';
-[ RECORD 1 ]-------+----------
relid               | 49257
schemaname          | public
relname             | employees
seq_scan            | 1
seq_tup_read        | 0
idx_scan            | 0
idx_tup_fetch       | 0
n_tup_ins           | 10000
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 10000
n_dead_tup          | 0
n_mod_since_analyze | 10000
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0
```

上面的查询结果显示，last_analyze和last_autoanalyze都是空，说明没有执行过相关操作，这证实了我们的猜测。

再次重复执行上面的查询，结果显示如下

```sql
postgres=# explain select * from employees where id < 10;
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=4.35..30.34 rows=9 width=18)
   Recheck Cond: (id < 10)
   ->  Bitmap Index Scan on employees_pkey  (cost=0.00..4.35 rows=9 width=0)
         Index Cond: (id < 10)
(4 rows)
postgres=# select * from pg_stat_user_tables where relname = 'employees';
-[ RECORD 1 ]-------+------------------------------
relid               | 49257
schemaname          | public
relname             | employees
seq_scan            | 1
seq_tup_read        | 0
idx_scan            | 1
idx_tup_fetch       | 1
n_tup_ins           | 10000
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 10000
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 2020-12-25 14:17:12.416294+08
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 1
```

可以看到，由于autovacuum的执行，更新了employees表的统计信息，因此这次的查询计划符合我们的预期。

## 最频值与直方图边界

上面的示例说明了统计信息在PG最优路径的选择时有重要的影响。那么，统计信息具体是怎么影响最优路径的选择的呢？前面已经说过，PG是基于代价的最优路径选择，因此在选择seq_scan还是index_scan时，判断的依据就是哪种路径的代价最低。seq_scan的代价比较好理解，就是顺序读取page，而index_scan的代价则涉及一个选择率的问题。什么是选择率呢，就是本次index_scan将要扫描获取的tuple占总tuple的的占比。当然，这个信息是估计而来，而估计的依据就是统计信息。

首先引入两个基本概念：

最频值：列数据中最常出现的值列表

直方图边界：将列数据的所有取值近乎平均的分配到每个区间的区间列表

以上面示例中的employees表为例，我们查看该表的location字段的最频值信息和直方图信息，如下

```sql
postgres=# select most_common_vals, most_common_freqs from pg_stats where tablename='employees' and attname='location';
-[ RECORD 1 ]-----+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
most_common_vals  | {0024,0053,0068,0058,0044,0092,0054,0056,0083,0026,0063,0090,0002,0066,0079,0095,0023,0078,0041,0004,0037,0048,0017,0043,0076,0008,0086,0099,0014,0025,0040,0077,0027,0071,0073,0006,0018,0081,0051,0080,0087,0009,0029,0057,0065,0050,0070,0085,0003,0062,0067,0088,0011,0015,0028,0060,0075,0019,0020,0032,0059,0022,0036,0039,0096,0007,0049,0069,0013,0098,0001,0030,0033,0046,0052,0012,0016,0035,0047,0010,0061,0064,0082,0005,0055,0072,0074,0097,0084,0091,0038,0034,0089,0094,0045,0021,0031,0093,0042,0100}
most_common_freqs | {0.0125,0.0123,0.0119,0.0118,0.0117,0.0117,0.0116,0.0116,0.0116,0.0113,0.0113,0.0113,0.0112,0.0112,0.0112,0.0112,0.0111,0.0111,0.011,0.0109,0.0109,0.0109,0.0108,0.0108,0.0108,0.0107,0.0107,0.0107,0.0106,0.0106,0.0106,0.0106,0.0105,0.0105,0.0105,0.0104,0.0104,0.0104,0.0103,0.0103,0.0103,0.0102,0.0102,0.0102,0.0102,0.0101,0.0101,0.0101,0.01,0.01,0.01,0.01,0.0099,0.0099,0.0099,0.0099,0.0099,0.0098,0.0098,0.0098,0.0097,0.0096,0.0096,0.0096,0.0096,0.0095,0.0095,0.0095,0.0094,0.0094,0.0093,0.0093,0.0093,0.0093,0.0093,0.0092,0.0092,0.0092,0.0092,0.0091,0.0091,0.0091,0.0091,0.009,0.009,0.009,0.009,0.009,0.0089,0.0088,0.0087,0.0086,0.0085,0.0085,0.0084,0.0082,0.0077,0.007,0.0064,0.0045}
postgres=# select histogram_bounds from pg_stats where tablename='employees' and attname='location';
 histogram_bounds 
------------------
 
(1 row)

postgres=# 
```

可以看到，这里只有MCV信息，并没有histogram_bound信息。这是为什么呢？因为我们插入数据时，location列上的数据为random()*100，也就是说，location的取值最多有100种，而PG在采样时，其采样的桶大小也是100，因此所有值的统计信息都可以在MCV中存储，自然就没有使用histogram_bound的必要了。

为了演示，我们清空employees表后，插入100万条数据，然后执行如上两个查询，其结果如下：

```sql
postgres=# insert into employees select id, to_char(id, 'FM0000'), to_char(random()*2000, 'FM0000') from generate_series(1, 1000000) id;
INSERT 0 1000000
postgres=# 
postgres=# select most_common_vals, most_common_freqs from pg_stats where tablename='employees' and attname='location';
-[ RECORD 1 ]-----+---------------------------------------------------------------------------------
most_common_vals  | {0594,0608,1279,1552,0820,0068,1071}
most_common_freqs | {0.000933333,0.000933333,0.000933333,0.000933333,0.0009,0.000866667,0.000866667}

postgres=# select histogram_bounds from pg_stats where tablename='employees' and attname='location';
-[ RECORD 1 ]----+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
histogram_bounds | {0000,0020,0040,0061,0081,0102,0121,0142,0161,0180,0199,0218,0236,0255,0276,0295,0314,0334,0356,0375,0396,0414,0437,0459,0478,0498,0516,0533,0555,0575,0597,0619,0642,0661,0683,0701,0721,0742,0761,0779,0798,0818,0840,0861,0882,0901,0921,0943,0964,0984,1004,1023,1043,1062,1083,1106,1127,1147,1169,1188,1209,1228,1250,1271,1293,1313,1332,1351,1372,1391,1409,1428,1448,1468,1486,1506,1526,1547,1566,1587,1609,1629,1650,1670,1690,1710,1728,1748,1766,1784,1805,1824,1845,1862,1883,1902,1921,1940,1961,1979,2000}

postgres=# 
```

## 选择率

前面说过，选择率就是本次扫描需要返回的tuple数占总tuple数的占比。那么选择率是如何计算的呢？假如有类似如下的查询，PG是怎么评估它需要返回的tuple数呢？

```sql
select * from employees where location = 'XXXX';
select * from employees where location < 'XXXX';
```

答案是：对于等值条件，PG使用最频值直接计算对应的选择率；对于非等值条件，则依据最频值和直方图列表计算而来。需要解释的是，直方图列表和最频值的统计信息是相辅相成的，两者之间并没有覆盖，而是共同组成了统计信息，也就是说，直方图列表中的字段的出现频率加上最频值的频率，就是所有数字的出现频率，即1。

* 示例1：

```sql
select * from employees where location = '0608';
```



由于该查询是等值条件，且等值处于MCV列表中，则其选择率就是该MCV对应的MCF，即

​	                                            *Selectivity = MCF['0608'] = 0.000933333* 

我们在数据库中执行explain进行佐证：

```sql
postgres=# explain select * from employees where location = '0608';
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=19.66..2546.76 rows=933 width=18)
   Recheck Cond: (location = '0608'::text)
   ->  Bitmap Index Scan on employees_index_location  (cost=0.00..19.42 rows=933 width=0)
         Index Cond: (location = '0608'::text)
(4 rows)

postgres=# select 1000000 * 0.000933333;
   ?column?    
---------------
 933.333000000
(1 row)
```

* 示例2： 

```sql
select * from employees where location = '0133';
```

该查询是等值条件，但是其等值并不在MCV列表中，这时候PG是怎么计算选择率的呢，总不能去使用直方图区间吧？PG对于这种场景的选择率计算简单直接，其计算公式为

​                                                 *Selectivity = (1 - sum(MCF)) / (distinct_val - sizeof(MCV))*

即选择率等于1减去所有最频值的出现频率之和，再除以该字段所有不同值的的取值个数与最频值个数的差。

我们在数据库中执行explain进行佐证：

```sql
postgres=# select n_distinct, most_common_vals, most_common_freqs from pg_stats where tablename='employees' and attname='location';
 n_distinct |           most_common_vals           |                                most_common_freqs                                 
------------+--------------------------------------+----------------------------------------------------------------------------------
       2001 | {0594,0608,1279,1552,0820,0068,1071} | {0.000933333,0.000933333,0.000933333,0.000933333,0.0009,0.000866667,0.000866667}
(1 row)

postgres=# select (1 - 0.000933333 - 0.000933333 - 0.000933333 - 0.000933333 - 0.0009 - 0.000866667 - 0.000866667) / (2001 - 7) * 1000000;
         ?column?         
--------------------------
 498.31160180541625000000
(1 row)

postgres=# explain select * from employees where location = '0133';
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=12.28..1543.22 rows=498 width=18)
   Recheck Cond: (location = '0133'::text)
   ->  Bitmap Index Scan on employees_index_location  (cost=0.00..12.16 rows=498 width=0)
         Index Cond: (location = '0133'::text)
(4 rows)

postgres=# 
```

* 示例3

```sql
select * from employees where location < '0603';
```

该查询非等值条件，因此需要结合MCV信息和histogram_bound信息来计算选择率。对于这种情况，PG计算选择率的公式为

​                                     *Selectivity = mcv_selectivity + histogram_selectivity \* histogram_fraction*

即选择率等于MCV值列表中，小于条件值的MCV的频率之和，加上该条件值落入histogram_bound区间的频率，其中

​                                    *histogram_fraction = 1 -  sum(MCF)*

即直方图区间在整体选择率中的占比等于1减去最频值的占比。

​                                    *histogram_selectivity  = (bound_num(<const_val) + (const_val - bound_low)/(bound_high - bound_low)) / 100*

即直方图区间的选择率等于：小于常量值的区间个数加上常量值在落入的区间的占比，最后再除以总区间个数

具体到这个例子

```sql
Selectivity = sum(0.000933333, 0.000866667) + ((30 + ('0603' - '0597')/('0619 - 0597')) / 100) * 
				((1 - sum(0.000933333,0.000933333,0.000933333,0.000933333,0.0009,0.000866667,0.000866667)))
            = 0.30259990929272727272456281818
```

在数据库中执行进行佐证

```sql
postgres=# explain select * from employees where location < '0603';
                                          QUERY PLAN                                           
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=5704.32..15880.59 rows=304502 width=18)
   Recheck Cond: (location < '0603'::text)
   ->  Bitmap Index Scan on employees_index_location  (cost=0.00..5628.19 rows=304502 width=0)
         Index Cond: (location < '0603'::text)
(4 rows)
```

这里和我们的预估有点出入，为了一探究竟，我们深入源码去分析下。选择率的计算函数为scalarineqsel()，查看该函数的实现，可以看到，该函数中调用了mcv_selectivity()和ineq_histogram_selectivity()来分别计算MCV和histogram_bound部分的选择率

```c
/*
 * If we have most-common-values info, add up the fractions of the MCV
 * entries that satisfy MCV OP CONST.  These fractions contribute directly
 * to the result selectivity.  Also add up the total fraction represented
 * by MCV entries.
 */
mcv_selec = mcv_selectivity(vardata, &opproc, constval, true,
							&sumcommon);

/*
 * If there is a histogram, determine which bin the constant falls in, and
 * compute the resulting contribution to selectivity.
 */
hist_selec = ineq_histogram_selectivity(root, vardata,
										&opproc, isgt, iseq,
										constval, consttype);

```

MCV的计算简单直接，不会和我们的理解产生偏差，因此我们深入研究下直方图区间的计算，进入ineq_histogram_selectivity函数，发现有如下代码块

```c
/*
 * Convert the constant and the two nearest bin boundary
 * values to a uniform comparison scale, and do a linear
 * interpolation within this bin.
 */
if (convert_to_scalar(constval, consttype, sslot.stacoll,
					  &val,
					  sslot.values[i - 1], sslot.values[i],
					  vardata->vartype,
					  &low, &high))
```

也就是说，这里会将字符串这种类型，转换为相对应的标量类型，然后用转换后的标量类型来做计算。具体到我们这个例子，转换后的结果为

```sql
val = 0.64768799606055327
high = 0.66046034767588024
low = 0.6341643296443249
```

再由于我们的过滤条件是<，不是<=，因此不包含=，而在直方图区间中，包含=时的选择率为

```c
eq_selec = 1 / (distinct_val - sizeof(mc_val)) = 1/(2001 - 7) = 0.00050150451354062187
```

因此，修正后的选择率计算如下

```sql
Selectivity = sum(0.000933333, 0.000866667) + ((30 + (0.64768799606055327 - 0.6341643296443249)/(0.66046034767588024 - 0.6341643296443249)) / 100 - 0.00050150451354062187) * ((1 - sum(0.000933333,0.000933333,0.000933333,0.000933333,0.0009,0.000866667,0.000866667)))
            = 0.30450180288733740666190632436
```

由上可见，修正后的选择率计算数值，和执行计划中输出的一致。



【参考链接】

[https://www.postgresql.org/docs/12/row-estimation-examples.html](https://www.postgresql.org/docs/12/row-estimation-examples.html)

[http://www.interdb.jp/pg/pgsql03.html](http://www.interdb.jp/pg/pgsql03.html)

