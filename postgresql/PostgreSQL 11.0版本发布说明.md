# PostgreSQL 11.0版本发布说明
  * 原始链接：  
  [E.3. 版本11Release Notes](http://postgres.cn/docs/11/release-11.html)  
  [PostgreSQL 11.0 正式版版本发布说明](http://www.postgres.cn/v2/release/v/53)  
  [Release 11 Appendix E. Release Notes](https://www.postgresql.org/docs/11/release-11.html)
  
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
  新命令[ALTER INDEX ATTACH PARTITION](http://postgres.cn/docs/11/sql-alterindex.html)使得分区表分区上已有索引与相配的索引模板关联。这样为已存在的分区表设置新的分区索引提供了灵活性。  
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
### 1.3 索引  
  * 允许B-树索引包含不是搜索键或唯一约束的，但可用于仅索引扫描读的列  
  该功能可用[CREATE INDEX](http://postgres.cn/docs/11/sql-createindex.html)的新的INCLUDE子句来启用。它利用构造“覆盖索引”来优化特定类型的查询。可包含一些列，即使其数据类型B-树不支持
  * 改进索引不断增长时的性能  
  * 改进哈希索引扫描的性能  
  * 为哈希、GiST和GIN索引添加谓词锁。这样减少了序列化模式事务中序列化冲突的可能性  
#### 1.3.1 [SP-Gist](http://postgres.cn/docs/11/spgist.html)  
  * 增加前缀匹配操作符text ^@ text，由SP-GiST支持此项功能。 这类似于btree索引时使用var LIKE 'word%'，但它更有效  
  * 允许使用SP-GiST索引多边形  
  * 允许SP-GiST使用叶子键的有损表示  
### 1.4 优化器
  * 改进统计信息最频值的选择  
  以前，最频值(MCV)的标识是基于他们与所有列值比较的频度。现在MCV的选择是基于他们与非MCV值比较的频度。这样就改进了均匀和不均匀分布下的算法的健壮性  
  * 改进>=和<=的选择性评估  
  以前，此类案例如>和<分别使用同样的选择性评估，除非比较常数是MCV。该变化对使用BETWEEN在小区间查询特别有用  
  * 在等效的地方将var =var简化为var IS NOT NULL。这会带来更好的选择性评估  
  * 改进优化器对EXISTS和NOT EXISTS查询的行计数估计  
  * 使优化器负责评估代价和HAVING子句的选择性  
### 1.5 一般性能  
  * 增加对查询计划的某些部分的[即时编译](http://postgres.cn/docs/11/jit.html)（JIT），以改进执行性能。该特性需LLVM可用。当前不是默认启用，即使在构建中支持也不是  
  * 允许在可能时位图扫描执行仅索引扫描  
  * 更新VACUUM时空闲空间映射。这样允许空闲空间能更快地重用  
  * 允许VACUUM避免不必要的索引扫描  
  * 改进多并发事务的提交性能  
  * 减少那些在目标列表中使用集返回函数的查询的内存使用  
  * 改进聚集计算的速度  
  * 允许[postgres_fdw](http://postgres.cn/docs/11/postgres-fdw.html)将使用连接的UPDATE和DELETE命令推送到外部服务器。以前，仅推送非连接的UPDATE和DELETE命令  
  * 增加Windows上的大页支持。该功能由huge_pages配置参数来控制  
### 1.6 监控  
  * 在log_statement_stats,log_parser_stats,log_planner_stats和log_executor_stats的输出中显示内存使用情况  
  * 增加pg_stat_activity.backend_type列以显示后台工作者类型。该类型在ps输出中也可见  
  * 使log_autovacuum_min_duration记录那些被同时删除而忽略掉的表   
#### 1.6.1 信息模式  
  * 增加与表约束和触发器相关的information_schema列  
  特别是现在triggers.action_order,triggers.action_reference_old_table和triggers.action_reference_new_table都填有数据，而以前则总为空。此外，现在table_constraints.enforced虽然存在，但仍未填入有用数据  
### 1.7 认证  
  * 允许服务器在搜索+绑定模式中指定更为复杂的[LDAP](http://postgres.cn/docs/11/auth-ldap.html)规格要求。特别是ldapsearchfilter允许使用LDAP属性组合进行模式匹配    
  * 允许LDAP认证使用加密的LDAP。 们已经支持TLS上的LDAP，这通过用ldaptls=1来实现。新的TLS LDAP是加密LDAP，用ldapscheme=ldaps或ldapurl=ldaps:// 来启用 
  * 改进LDAP错误日志  
### 1.8 权限  
  * 增加启用文件系统访问的[默认角色](http://postgres.cn/docs/11/default-roles.html#DEFAULT-ROLES-TABLE)  
  特别地，这些新角色是：pg_read_server_files,pg_write_server_files和pg_execute_server_program。这些角色现在还控制着谁可以使用服务器端COPY和file_fdw扩展。以前，仅超级用户才能使用这些功能，这仍然是默认行为  
  * 允许用GRANT/REVOKE权限许可来控制文件系统函数的访问，而不是由超级用户检查来控制  
  特别地，修改了这些函数：pg_ls_dir(),pg_read_file(),pg_read_binary_file(),pg_stat_file()  
  * 使用GRANT/REVOKE控制对lo_import()和lo_export()的访问  
  以前，仅超级用户才有权访问这些函数。编译时选项ALLOW_DANGEROUS_LO_FUNCTIONS已被移除  
  * 防止对[postgres_fdw](http://postgres.cn/docs/11/postgres-fdw.html)表进行无密码访问时，使用视图拥有者而非会话拥有者  
  PostgreSQL仅允许超级用户对postgres_fdw表进行无密码的访问，例如通过peer访问。以前，会话拥有者必须为超级用户才允许访问；现在以检查视图拥有者代之  
  * 修复视图上SELECT FOR UPDATE的无效的锁权限检查问题  
### 1.9 服务器配置  
  * 增加服务器参数[ssl_passphrase_command](http://postgres.cn/docs/11/runtime-config-connection.html#GUC-SSL-PASSPHRASE-COMMAND)以允许提供SSL密钥文件的密语  
  此外增加[ssl_passphrase_command_supports_reload](http://postgres.cn/docs/11/runtime-config-connection.html#GUC-SSL-PASSPHRASE-COMMAND-SUPPORTS-RELOAD)以详细说明SSL配置是否应重新载入，且服务器配置重新载入时是否调用ssl_passphrase_command  
  * 增加存储参数[toast_tuple_target](http://postgres.cn/docs/11/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS)，以在考虑用TOAST存储前控制最小的元组长度。缺省的TOAST阈值不变
  * 允许以字节为单位制定内存和文件大小相关的服务器选项。新单位后缀为“B”。已有的单位是“kB”, “MB”, “GB”和“TB”。  
### 1.10 [预写日志](http://postgres.cn/docs/11/wal.html)（WAL）  
  * 允许initdb时设置WAL文件大小。 以前，缺省值16MB只能在编译时改变   
  * 仅为单个检查点保留WAL数据。 以前，为两个检查点保留WAL  
  * 为提高压缩率，以零填充强制切换的WAL段文件的未使用部分   
  
## 2. 库备份和流复制  
  * 使用流复制时，复制TRUNCATE活动  
  * 将准备事务信息传给逻辑复制订阅者  
  * 将非日志表、临时表和pg_internal.init文件从流式库备份中排除  
  * 允许在流式库备份时验证堆页面的校验和  
  * 允许复制槽以编程方式前进，而不是被订阅者消费。这样允许在没必要消费内容时，复制槽有效前进。此功能通过pg_replication_slot_advance()实现  
  * 将时间线信息加到[backup_label](http://postgres.cn/docs/11/continuous-archiving.html#BACKUP-LOWLEVEL-BASE-BACKUP)文件中。另增WAL时间线与backup_label匹配的检查  
  * 将主机和端口连接信息加到pg_stat_wal_receiver系统视图中  
  
## 3. 实用工具命令  
  * 允许ALTER TABLE用非空缺省值添加列，而不需重写表。当缺省值为常数时启用该项功能  
  * 允许用视图所基于的表来锁定视图  
  * 允许ALTER INDEX为表达式索引设置统计信息收集目标。在psql中，\d+现在显示索引的统计信息目标  
  * 允许一条VACUUM或ANALYZE命令中指定多个表。此外，VACUUM中提及的任何表若使用列（字段）列表，则必须提供ANALYZE关键词；以前此类情况下ANALYZE是隐含在内的  
  * 把括号括起来的选项语法增加到ANALYZE。这类似于VACUUM支持的语法  
  * 增加CREATE AGGREGATE选项以指定聚集的终结函数的行为。这有助于允许用户定义聚集函数的优化、用作窗口函数  
  
## 4. 数据类型  
  * 允许创建域数组。这也允许array_agg()用在域上  
  * 支持符合类型上的域。也允许PL/Perl, PL/Python和PL/Tcl处理复合域函数参数和结果。亦改进了PL/Python的域处理  
  * 增加从JSONB标量到数字和布尔数据类型的类型转换  
  
## 5. 函数  
  * 增加SQL:2011指定的所有[窗口函数](http://postgres.cn/docs/11/sql-select.html#SQL-WINDOW)帧选项  
  * 增加SHA-2哈希函数家族。特别地，增加sha224(),sha256(), sha384(),sha512()  
  * 增加对64位非加密哈希函数的支持  
  * 允许to_char()和to_timestamp()指定自UTC的时区偏移的小时和分钟。 这以TZH和TZM格式规范来实施 
  * 增加搜索函数[websearch_to_tsquery()](http://postgres.cn/docs/11/functions-textsearch.html#TEXTSEARCH-FUNCTIONS-TABLE)，支持一种类似于Web搜索引擎使用的查询语法  
  * 增加[json(b)\_to_tsvector()](http://postgres.cn/docs/11/functions-textsearch.html#TEXTSEARCH-FUNCTIONS-TABLE)函数以创建用于匹配JSON/JSONB值的文本搜索查询  
  
## 6. 服务器端语言  
  * 增加SQL级过程，在过程中能够开始和提交其自身的事务  
  用新命令[CREATE PROCEDURE](http://postgres.cn/docs/11/sql-createprocedure.html)创建过程，用[CALL](http://postgres.cn/docs/11/sql-call.html)来调用  
  新命令ALTER/DROP ROUTINE可以改变/删除所有类似程序的对象，包括过程、函数和聚集。而且在CREATE OPERATOR和CREATE TRIGGER时，现在写FUNCTION比写PROCEDURE好，因为引用的对象必须是函数而非过程。但是，为了兼容性，仍然接受过去的语法  
  * 将事务控制增加到PL/pgSQL, PL/Perl, PL/Python, PL/Tcl和SPI服务器端语言中  
  事务控制仅在顶部事务层过程和仅包含其它DO和CALL块的嵌套DO和CALL块中可用  
  * 增加将PL/pgSQL复合类型变量定义为非空、常数或带初始值的功能  
  * 允许PL/pgSQL处理同一会话中发生在第一次和后来函数执行之间对复合类型的修改。以前这些情况产生错误  
  * 增加jsonb_plpython扩展以转换JSONB到/自PL/Python类型  
  * 增加jsonb_plperl扩展以转换JSONB到/自PL/Perl类型  
  
## 7. 客户端接口  
  * 修改libpq，以禁用默认的压缩。现代的OpenSSL版本中，压缩已被禁用，故与这些库一起的libpq设置无效  
  * 将DO CONTINUE选项加到ecpg的WHENEVER语句中。这会产生C的continue语句，特定条件发生时将导致返回含有它的循环的顶部  
  * 增加ecpg模式以启用Oracle Pro\*C风格的字符数组处理方式。该模式用-C来启用  
  
## 8. 客户端应用程序  
### 8.1 psql  
  * 增加psql命令\gdesc以显示查询结果中的列的名字和类型  
  * 增加psql变量以报告查询活动或错误。这些新变量是ERROR,SQLSTATE, ROW_COUNT,LAST_ERROR_MESSAGE和LAST_ERROR_SQLSTATE  
  * 允许psql测试一个变量是否存在。语法:{?variable_name}允许在\if语句中测试变量是否存在  
  * 允许环境变量PSQL_PAGER控制psql的分页  
  该项功能允许psql的缺省分页器可指定为单独环境变量，与其它应用程序分页器分开。如果PSQL_PAGER未设置，PAGER仍然起作用  
  * 使psql的\d+命令总显示表分区信息  
  以前如果无分区，分区表不会显示分区信息。现在也要显示哪些分区是如何划分的
  * 确保psql提示输入密码时报告确切的用户名  
  以前嵌入在URI中的-U和用户名的组合会导致错误的报告。当指定--password时，也会在密码提示之前禁止（显示）用户名  
  * 允许没有先前输入时，使用quit和exit退出psql  
  此外输入缓冲区非空时，若在一行中单独使用quit和exit，则打印如何退出的提示。对于help也添加了类似的提示  
  * 在一行中单独输入了\q但被忽略时，让psql提示使用CTRL+D。例如，当\q是在字符串中给出时，该命令并不退出  
  * 改进ALTER INDEX RESET/SET的制表符  
  * 增加允许psql基于服务器版本适配其制表符自动补全查询的基础架构。以前对老版本服务器的制表符自动补全查询会失败  
### 8.2 pgbench  
  * 在pgbench中增加对NULL、布尔数和一些函数与操作符的表达式支持  
  * 在pgbench中增加\if条件支持  
  * 允许在pgbench变量名中使用非ASCII字符  
  * 增加pgbench选项--init-steps以控制初始化步骤的执行  
  * 将一种近似于Zipfian分布的随机数发生器添加到pgbench  
  * 允许在pgbench中设置随机数种子  
  * 允许pgbench用pow()和power()来做幂运算  
  * 将哈希函数添加到pgbench  
  * 当使用--latency-limit和--rate时，使pgbench统计更准确   
  
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

