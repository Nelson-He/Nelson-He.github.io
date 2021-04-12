## 1、PostgreSQL 13.2在线创建索引分析

   * PG支持在线创建索引，语法形式为

     ```sql
     CREATE INDEX index_name CONCURRENTLY ON table_name(columns);
     ```

   *  临时表不支持在线创建索引，因为没有其他会话可以使用临时表，即便在临时表上添加了CONCURRENTLY关键字，在执行阶段也会被强制纠正

   * 在线创建索引的流程

     ```shell
     1.开启第一个事务，拿到当前快照snapshot1
     2.等待所有修改过该表的事务结束
     3.扫描该表，第一次创建索引
     4.结束第一个事务
     5.开启第二个事务，拿到当前快照snapshot2
     6.等待所有修改过该表的事务结束
     7.第二次扫描该表，将两次快照之间变更的记录，合并到索引
     8.上一步更新索引结束后，等待snapshot2之前开启的所有事务结束
     9.结束索引创建，索引变为可用
     ```

   *  当前流程中存在如下问题

     ```shell
     1.死锁或唯一索引的唯一性违背时会导致create index失败，残留一个invalid索引。查询会忽略invalid索引，但仍然会导致update时性能损耗
     2.第一次扫描后创建的索引对其他事务生效，在该索引可用之前，其它事务操作时可能会报告违反约束。甚至在索引失败而处于invalid状态时，其唯一性约束仍然有效。
     3.并发创建索引是自排他的，因此同时只能有一个并发索引创建。并且CREATE INDEX CONCURRENTLY不能在事务块中执行。
     4.并发创建索引可能由于长事务的原因造成索引创建一直等待，这个事务可能并非是该表上的事务
     5.不支持在分区表上在线创建索引
     6.不支持在系统表上在线创建索引
     7.不支持为排他约束在线创建索引
     ```

     上面提到的第4点问题需要特别注意，并非只有当前表上的其他事物会阻塞索引创建，其他表上先于在线创建索引事务的事务，也会阻塞索引创建，其示例如下

     ```shell
     --- 会话1
     postgres=# select now();
                   now              
     -------------------------------
      2021-04-08 22:40:06.580691-04
     (1 row)
     
     postgres=# begin;
     BEGIN
     postgres=*# copy table_test_1 from stdin delimiter '|' null '' csv;
     Enter data to be copied followed by a newline.
     End with a backslash and a period on a line by itself, or an EOF signal.
     >> 
     
     --- 会话2
     postgres=# select now();
                   now              
     -------------------------------
      2021-04-08 22:42:01.505194-04
     (1 row)
     
     postgres=# 
     postgres=# create index concurrently table_test_2_index on table_test_2(value1);  ---卡住
     
     --- 查看进程状态
     [pg13@localhost ~]$ ps -ef | grep postgres
     pg13      58525 110424  0 22:37 pts/3    00:00:00 psql -d postgres
     pg13      58526 128019  0 22:37 ?        00:00:00 postgres: pg13 postgres [local] COPY
     pg13      61178  58785  0 22:40 pts/2    00:00:00 psql -d postgres
     pg13      61179 128019  0 22:40 ?        00:00:00 postgres: pg13 postgres [local] CREATE INDEX waiting
     ```

* 数据结构

  ```c
  typedef struct IndexStmt
  {
  	NodeTag		type;
  	char	   *idxname;		/* name of new index, or NULL for default */
  	RangeVar   *relation;		/* relation to build index on */
  	char	   *accessMethod;	/* name of access method (eg. btree) */
  	char	   *tableSpace;		/* tablespace, or NULL for default */
  	List	   *indexParams;	/* columns to index: a list of IndexElem */
  	List	   *indexIncludingParams;	/* additional columns to index: a list of IndexElem */
  	List	   *options;		/* WITH clause options: a list of DefElem */
  	Node	   *whereClause;	/* qualification (partial-index predicate) */
  	List	   *excludeOpNames; /* exclusion operator names, or NIL if none */
  	char	   *idxcomment;		/* comment to apply to index, or NULL */
  	Oid			indexOid;		/* OID of an existing index, if any */
  	Oid			oldNode;		/* relfilenode of existing storage, if any */
  	SubTransactionId oldCreateSubid;	/* rd_createSubid of oldNode */
  	SubTransactionId oldFirstRelfilenodeSubid;	/* rd_firstRelfilenodeSubid of oldNode */
  	bool		unique;			/* is index unique? */
  	bool		primary;		/* is index a primary key? */
  	bool		isconstraint;	/* is it for a pkey/unique constraint? */
  	bool		deferrable;		/* is the constraint DEFERRABLE? */
  	bool		initdeferred;	/* is the constraint INITIALLY DEFERRED? */
  	bool		transformed;	/* true when transformIndexStmt is finished */
  	bool		concurrent;		/* should this be a concurrent index build? */
  	bool		if_not_exists;	/* just do nothing if index already exists? */
  	bool		reset_default_tblspc;	/* reset default_tablespace prior to executing */
  } IndexStmt;
  IndexStmt中添加字段concurrent， 标识是否在线创建索引
  ```

*  函数接口

  ```c
  PreventInTransactionBlock()          /* 确保当前语句不在事务块中执行 */
  DefineIndex()                        /* 索引定义入口函数 */
  makeIndexInfo()                      /* 创建IndexInfo node，在创建索引过程中使用 */
  ComputeIndexAttrs()                  /* 计算每一个索引字段信息 */
  index_create()                       /* 索引创建执行函数 */
  ConstructTupleDescriptor()           /* 构造索引的描述符信息，即索引元信息 */
  heap_create()                        /* 创建索引的relcache，并为其分配物理文件 */
  UpdateIndexRelation()                /* 向pg_index系统表插入索引记录 */
  index_concurrently_build()           /* 在线创建索引的入口 */    
  index_build()                        /* 调用对应索引类型(BTree/Hash/Gin/...)的创建函数来创建索引 */
  validate_index()                     /* 二次扫描heap元组，为第一次索引创建期间新增的heap_tuple创建对应的index tuple */
  table_index_validate_scan()          /* 在线创建索引过程中的第二次表扫描 */
  WaitForOlderSnapshots()              /* 等待比给定事务更老的事务结束 */
  ```

   * 代码流程

     ```
     DefineIndex()函数为创建索引的入口，进入DefineIndex()函数后主要进行如下处理：
     1.判断是否为concurrent模式，若是，则以4级锁（ShareUpdateExclusiveLock）打开基表，否则以5级锁打开（ShareLock）
     2.调用makeIndexInfo创建缓存结构IndexInfo，为index_create准备辅助信息，在concurrent模式下，ii_ReadyForInserts置为false
     3.调用index_create为索引创建系统项，其主要流程如下：
       3.1以3级锁（RowExclusiveLock）模式打开pg_class系统表，后续步骤会向pg_class插入元组
       3.2调用ConstructTupleDescriptor构建索引tuple元信息
       3.3调用GetNewRelFileNode为索引表分配oid（和relfilenode一致）
       3.4调用heap_create为索引表创建relcache的表项，其实际调用RelationBuildLocalRelation进行创建并缓存到relcache中，然后调用                  RelationCreateStorage为索引表创建物理存储文件            
       3.5以8级锁（AccessExclusiveLock）模式锁定索引表
       3.6调用InsertPgClassTuple向pg_class系统表插入待创建的索引表信息
       3.7关闭pg_class系统表，释放相关锁
       3.8调用UpdateIndexRelation向pg_index系统表插入待创建的索引信息
       3.9向pg_depend系统表插入待创建索引引入的依赖信息
       3.10调用index_update_stats更新基表上的索引统计信息
       3.11关闭索引表文件，但仍保留其上的8级锁
     4.关闭打开的基表，但仍保留其上获得的4级表（concurrent模式，非concurrent模式在此步骤前已返回）
     5.获取基表上的会话级4级锁，提交当前为止产生的事务，使得待创建的索引信息对其他事务可见，防止其它事务进行不兼容的HOT更新。在此之后，其他会话便可从   pg_index系统表中查询到该索引信息。     
     6.开启一个新的事务
     7.调用WaitForLockers检测是否仍存在可能会和基表上5级锁冲突的事务，有则等待
     8.调用index_concurrently_build函数在线创建索引
       8.1以4级锁（ShareUpdateExclusiveLock）打开基表
       8.2以3级锁（RowExclusiveLock）打开待创建的索引表
       8.3调用BuildIndexInfo构造IndexInfo辅助结构（步骤2中创建的IndexInfo已经在步骤5提交事务时被释放，因此需要重新构造）
       8.4调用index_build，进而调用索引类型对应的创建函数创建索引，会进行一次heap_scan。  
       8.5关闭基表和索引表，但仍保留其上持有的锁。将索引状态设置为READY状态。
     9.提交事务，使得变更生效，后续所有在基表上的更新需要同步更新刚才创建的索引。
     10.开启一个新的事务
     11.调用WaitForLockers检测是否仍存在可能会和基表上5级锁冲突的事务，有则等待
     12.获取当前活跃事务快照，该快照用来在步骤13中做元组可见性判断
     13.调用validate_index对索引进行效验修正
       12.1以4级锁（ShareUpdateExclusiveLock）打开索引的基表
       12.2以3级锁（RowExclusiveLock）打开索引表
       12.3调用BuildIndexInfo构造IndexInfo辅助结构
       12.4调用tuplesort_begin_datum设置索引元组排序参数，为排序做准备
       12.4调用index_bulk_delete，传入回调函数validate_index_callback，收集所有索引元组的TID信息
       12.5调用tuplesort_performsort对索引进行排序
       12.6调用table_index_validate_scan再次扫描基表，并对索引进行归并排序，将步骤6到步骤9之间产生的新元组（这些元组在步骤6之后的新事务中产生，因       此在步骤6到步骤9之间创建索引时，对创建过程不可见）合并到索引中。具体的执行步骤可以参考函数heapam_index_validate_scan的实现。       
       12.7调用tuplesort_end释放13.4中的准备信息
       12.8依次关闭索引表和基表，但仍保留其上的锁
     14.提交事务，防止其他在线创建索引等待当前事务而产生死锁
     15.开启一个新事务
     16.调用WaitForOlderSnapshots，等待所有比步骤10中新建事务更早的事务结束。
     17.设置索引状态为valid，至此以后，查询便可以使用该索引。
         
     SEE the comments of validate_index() function for more details.  
     ```

     

   * 在线创建索引官方文档描述

     ```
       Creating an index can interfere with regular operation of a database. Normally PostgreSQL locks the table to be indexed against writes and performs the entire index build with a single scan of the table. Other transactions can still read the table, but if they try to insert, update, or delete rows in the table they will block until the index build is finished. This could have a severe effect if the system is a live production database. Very large tables can take many hours to be indexed, and even for smaller tables, an index build can lock out writers for periods that are unacceptably long for a production system.
     
       PostgreSQL supports building indexes without locking out writes. This method is invoked by specifying the CONCURRENTLY option of CREATE INDEX. When this option is used, PostgreSQL must perform two scans of the table, and in addition it must wait for all existing transactions that could potentially modify or use the index to terminate. Thus this method requires more total work than a standard index build and takes significantly longer to complete. However, since it allows normal operations to continue while the index is built, this method is useful for adding new indexes in a production environment. Of course, the extra CPU and I/O load imposed by the index creation might slow other operations.
     
       In a concurrent index build, the index is actually entered into the system catalogs in one transaction, then two table scans occur in two more transactions. Before each table scan, the index build must wait for existing transactions that have modified the table to terminate. After the second scan, the index build must wait for any transactions that have a snapshot (see Chapter 13) predating the second scan to terminate, including transactions used by any phase of concurrent index builds on other tables. Then finally the index can be marked ready for use, and the CREATE INDEX command terminates. Even then, however, the index may not be immediately usable for queries: in the worst case, it cannot be used as long as transactions exist that predate the start of the index build.
     
       If a problem arises while scanning the table, such as a deadlock or a uniqueness violation in a unique index, the CREATE INDEX command will fail but leave behind an “invalid” index. This index will be ignored for querying purposes because it might be incomplete; however it will still consume update overhead. The psql \d command will report such an index as INVALID:
     
                postgres=# \d tab
                       Table "public.tab"
                 Column |  Type   | Collation | Nullable | Default
                --------+---------+-----------+----------+---------
                 col    | integer |           |          |
                Indexes:
                    "idx" btree (col) INVALID
     
       The recommended recovery method in such cases is to drop the index and try again to perform CREATE INDEX CONCURRENTLY. (Another possibility is to rebuild the index with REINDEX INDEX CONCURRENTLY).
     
       Another caveat when building a unique index concurrently is that the uniqueness constraint is already being enforced against other transactions when the second table scan begins. This means that constraint violations could be reported in other queries prior to the index becoming available for use, or even in cases where the index build eventually fails. Also, if a failure does occur in the second scan, the “invalid” index continues to enforce its uniqueness constraint afterwards.
     
       Concurrent builds of expression indexes and partial indexes are supported. Errors occurring in the evaluation of these expressions could cause behavior similar to that described above for unique constraint violations.
     
       Regular index builds permit other regular index builds on the same table to occur simultaneously, but only one concurrent index build can occur on a table at a time. In either case, schema modification of the table is not allowed while the index is being built. Another difference is that a regular CREATE INDEX command can be performed within a transaction block, but CREATE INDEX CONCURRENTLY cannot.
     
       Concurrent builds for indexes on partitioned tables are currently not supported. However, you may concurrently build the index on each partition individually and then finally create the partitioned index non-concurrently in order to reduce the time where writes to the partitioned table will be locked out. In this case, building the partitioned index is a metadata only operation.
     ```
