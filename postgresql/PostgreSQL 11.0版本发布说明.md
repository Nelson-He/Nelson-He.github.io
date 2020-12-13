# PostgreSQL 11.0版本发布说明
  * 原始链接：  
  [E.3. 版本11Release Notes](http://postgres.cn/docs/11/release-11.html)  
  [PostgreSQL 11.0 正式版版本发布说明](http://www.postgres.cn/v2/release/v/53)  
  
  PostgreSQL 11 重点对系统性能进行提升，特别是在对大数据集和高计算负载的情况下进行了增强。尤其是PostgreSQL 11 对表分区的功能进行了重大的改变和提升，增加了内置事务管理的存储过程，提升了并行查询能力和并行数据定义能力，也引入了JIT编译来加速查询中的表达式的计算执行。  
  
  ### 增强的健壮性和分区表性能的提升  
  PostgreSQL 11 在目前版本已有了按值列表或是按范围作为分区键值的分区表功能外，又增加了按哈希键值分区的功能，也称之为“Hash分区”。 PostgreSQL 11 还通过在分区功能中使用外部数据封装器postgres_fdw的功能，也进一步提升了数据聚合能力。

  为了帮助管理分区，PostgreSQL 11 引入了将不含有分区键值的记录自动转入缺省分区的功能，并增加了在（主表）执行创建主键、外键、索引和触发器时，会将这些操作全部自动复制给所有分区表的功能。另外PostgreSQL 11现在也支持当记录中的分区键值字段被更新后，会自动将该记录移至新的正确的分区表中的功能。

  PostgreSQL 11 版本通过使用新的分区消除策略来提升查询分区表的性能。另外，PostgreSQL 11现在在分区表上也支持流行的“UPSERT”功能，这可以帮助用户在处理应用数据时，简化应用程序的开发，减少网络负载。
  
  ### 支持内置事务的存储过程  
  在PostgreSQL 11版本前，函数是不能控制它们自己的事务的。PostgreSQL 11版本增加了SQL 标准存储过程的特性，并且支持在过程中执行完整的事务管理功能。这个新特性允许开发人员创建更多更高级的服务端应用，比如涉及大量数据的增量导入功能。

SQL 过程是使用 CREATE PROCEDURE 指令创建，执行时使用 CALL 指令，现在服务端的过程语言有 PL/pgSQL、 PL/Perl、 PL/Python 和 PL/Tcl。  

  ### 并行查询的增强  
  PostgreSQL 11提升了并行查询的性能，通过更加有效的分区数据扫描，在并行顺序扫描和哈希聚合方面性能有了更大的改进。即使是组成UNION的查询子句不能并行处理时，PostgreSQL现在也可以对使用 UNION 的SELECT查询并行处理。

PostgreSQL 11 也对几种数据集的定义指令增加了并行处理功能，最显著的就是通过 CREATE INDEX 指令创建的B-Tree索引。其他几种支持并行化操作的还有 CREATE TABLE .. AS 、 SELECT INTO 、 CREATE MATERIALIZED VIEW 等创建表和物化视图的操作。  

  ### 表达式的 (JIT) 编译  
  PostgreSQL 11 版本引入了JIT编译来加速查询中的表达式的计算和执行。JIT表达式的编译使用LLVM项目编译器的架构来提升在WHERE条件、指定列表、聚合、投影以及一些内部操作的表达式的编译执行。

要使用JIT 编译，用户需要安装LLVM相关的依赖包，并在系统 中启用JIT编译，可通过在PostgreSQL的配置文件中设置 jit = on ，或是在 PostgreSQL 当前会话中执行 SET jit = on 指令均可启用JIT。

  ### 一般性的用户体验提升  
  改变 ALTER TABLE .. ADD COLUMN .. DEFAULT .. 指令在带有非 NULL 缺省值时需要重写整个表的操作，新的变化对这样的指令带来了极大的性能量级提升；
  “覆盖索引”操作， 允许用户在创建一个索引通过使用 INCLUDE 选项来增加额外字段，这样会对无B-tree索引列的查询来使用Index-Only 的扫描有很大好处；
  更多的窗口函数功能，包括允许 RANGE 来使用 PRECEDING / FOLLOWING 和 GROUPS，窗口的非包含功能等；
  给PostgreSQL的命令行程序psql增加了关键字quit 和 exit ，以让用户（按各种习惯）更加容易退出这个程序。
  
## 1. 服务器  
### 1.1 分区  
  * 允许创建基于键列哈希的分区  
  * 支持分区表上的索引  
  分区表上的“索引”不是一种跨整个分区表的物理索引，而是一种自动创建表的每个分区上类似索引的模板。

  如果分区键是索引列集的一部分，分区索引可声明为UNIQUE。它将代表跨整个分区表的有效唯一性约束，即使每个物理索引仅强制其自己分区内的唯一性。

  新命令[LTER INDEX ATTACH PARTITION](http://postgres.cn/docs/11/sql-alterindex.html)使得分区表分区上已有索引与相配的索引模板关联。这样为已存在的分区表设置新的分区索引提供了灵活性。  
  * 允许分区表上的外键  
  * 允许分区表上的FOR EACH ROW触发器  
  创建分区表上的触发器，会自动创建所有已存在的和未来的分区上的触发器。这也允许分区表上可延迟的唯一约束  
  * 允许分区表有缺省分区  
  缺省分区将存储与任何其他已定义分区不匹配的行，作相应的搜索  
  * 改变分区键列的UPDATE语句现在会使得受影响的行移动到适当的分区  
  * 允许分区表上的INSERT, UPDATE和COPY，将行正确地路由到外部分区。 该功能由postgres_fdw外部表支持。  
  * 允许查询处理时更快的分区排除。 这将加速对有许多分区的分区表的访问。
  * 允许查询执行期间分区消除。 以前，分区消除仅在计划期间发生，意味着许多连接和准备的查询不能使用分区消除。 
  * 分区表之间等值连接时，允许匹配的分区直接连接。该特性缺省是禁用的，可修改[enable_partitionwise_join](http://postgres.cn/docs/11/runtime-config-query.html#GUC-ENABLE-PARTITIONWISE-JOIN)来启用  
  * 允许分区表上的聚集函数对每个分区独立评估而后归并结果。该特性缺省是禁用的，可修改[enable_partitionwise_aggregate](http://postgres.cn/docs/11/runtime-config-query.html#GUC-ENABLE-PARTITIONWISE-AGGREGATE)来启用  
  * 允许[postgres_fdw](http://postgres.cn/docs/11/postgres-fdw.html)将聚集下推到那些作为分区的外部表  
### 1.2 并行查询  
  * 允许并行构建btree索引  
  * 允许使用共享哈希表并行执行哈希连接  
  * 允许UNION并行运行每个SELECT，如果单个SELECT无法并行化的话  
  * 允许使用并行工作者进行更有效的分区扫描  
  * 允许将LIMIT传给并行工作者。这样允许工作者减少返回结果，使用有针对性的索引扫描。    
  * 允许单次评估查询，例如WHERE子句聚集查询和目标列表中的函数并行化  
  * 增加服务器参数[parallel_leader_participation](http://postgres.cn/docs/11/runtime-config-query.html#GUC-PARALLEL-LEADER-PARTICIPATION)用来控制领导者是否执行子计划。缺省启用，意味着领导者要执行子计划  
  * 允许命令CREATE TABLE ... AS, SELECT INTO和CREATE MATERIALIZED VIEW并行化  
  * 改进有许多并行工作者时顺序扫描的性能  
  * 在EXPLAIN中增加并行工作者排序活动的报告  
  
## 2. 库备份和流复制  
## 3. 实用工具命令  
## 4. 数据类型  
## 5. 函数  
## 6. 服务器端语言  
## 7. 客户端接口  
## 8. 客户端应用程序  
## 9. 服务器应用程序  
  * 为[pg_basebackup](http://postgres.cn/docs/11/app-pgbasebackup.html)增加一创建命名复制槽的选项  
  使用WAL流式方法（--wal-method=stream）时，选项--create-slot可以用来创建命名复制槽(--slot)  
  * 允许[initdb](http://postgres.cn/docs/11/app-initdb.html)设置对数据目录的组读取访问权限 
  在initdb时可以通过--allow-group-access选项来实现该功能。管理员也可以在运行initdb之前对空数据目录设置组权限。服务器变量data_directory_mode允许读取数据目录组权限。  
  * 新增[pg_verify_checksums](http://postgres.cn/docs/11/pgverifychecksums.html)工具以便离线时验证数据库校验和  
  * 允许[pg_resetwal](http://postgres.cn/docs/11/app-pgresetwal.html)通过--wal-segsize修改WAL大小  
  * 在pg_resetwal和pg_controldata中增加长选项  
  * 为pg_receivewal增加--no-sync选项，控制阻止同步WAL写  
## 10. 源代码  
  * 增加了PGXS支持，以便安装头文件  
  这样可支持创建依赖其他模块的扩展模块。以前没有容易的方法让依赖模块找到所引用的模块的头文件。现有的几个定义数据类型的contrib模块已经调整，安装了相关文件。PL/Perl和PL/Python现在也安装了它们的头文件，以支持语言转换模块的创建
  * 安装errcodes.txt，以允许扩展访问PostgreSQL已知的错误码  
  * 将文档转成了DocBook XML格式，为了与以前分支兼容，文件名仍使用sgml扩展名  
  * 在适用平台上使用stdbool.h定义bool类型。这样可以消除需要引用stdbool.h的扩展模块的编码危险  
  * 重新设计初始化系统目录内容的方法。初始数据现在用Perl数据结构表示，使之更易进行自动化处理   
  * 防止扩展中创建带引号的值列表的GUC参数。该功能目前不支持，因为即使在扩展加载之前也需要参数属性的信息  
  * 增加[SCRAM](http://postgres.cn/docs/11/auth-password.html)认证中使用通道绑定的功能。  
  通道绑定的目的是防止中间人攻击，但SCRAM无法阻止，除非它能被强制激活。不幸的是，在libpq中无法做到。期望在未来的libpq版本和不是由libpql构建的接口如JDBC中支持它  
  * 允许后台工作者连接到通常不允许被连接的数据库  
  * ARMv8上，增加硬件CRC计算的支持  
  * 通过使用OID加速对内置函数的查找。以前的二分查找被替换为查找数组  
  * 加速查询结果的构造  
  * 改进系统缓存的访问速度  
  * 增加代内存分配器以对顺序的分配/释放进行优化。这也减少了逻辑解码的内存使用  
  * 拉通VACUUM和ANALYZE对pg_class.reltuples的计算，使其计算结果一致  
  * 升级perltidy的版本到20170521
  
## 11. 附加模块  
* 允许[pg_prewarm](http://postgres.cn/docs/11/pgprewarm.html)启动时恢复以前的共享缓冲区内容  
  该功能实现的是让pg_prewarm在服务器运行期间和关闭时偶尔将共享缓冲区的关系和块数量的数据保存到磁盘  
* 在[pg_trgm](http://postgres.cn/docs/11/pgtrgm.html)中增加函数strict_word_similarity()以计算整个单词的相似性  
  当前pg_trgm中已有函数word_similarity()，但其设计是为了找到单词相似的部分，而strict_word_similarity()则是计算整个单词的相似性  
* 允许[btree_gin](http://postgres.cn/docs/11/btree-gin.html)对bool, bpchar, name和uuid数据类型进行索引
* 允许[cube](http://postgres.cn/docs/11/cube.html)和[seg](http://postgres.cn/docs/11/seg.html)扩展使用GiST索引执行仅索引扫描  
* 允许用~>操作符接收负的立方体坐标, 这对在查找降序的坐标时做KNN-GiST搜索很有用  
* [unaccent](http://postgres.cn/docs/11/unaccent.html)扩展中增加越南语处理  
* 改进[amcheck](http://postgres.cn/docs/11/amcheck.html)，检查每个堆元组有无索引入口  
* [adminpack](http://postgres.cn/docs/11/adminpack.html)使用新的缺省文件系统访问角色  
  以前，只有超级用户才能调用adminpack中的函数；现在根据角色的权限来确认访问权限。
* 将pg_stat_statement的查询ID扩展至64位  
  这将大大减少查询ID出现哈希冲突的可能性。现在查询ID可能显示为负值。
* 移除contrib/start-scripts/osx脚本，因为其不再建议使用（contrib/start-scripts/macos代替）  
* 移除chkpass扩展，该扩展不再视为是一种有用的安全工具或如何编写扩展的示例  

