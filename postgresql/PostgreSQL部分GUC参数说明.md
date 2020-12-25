* max_stack_depth  -- 最大栈深度，超级用户设置
  指定进程调用栈的最大安全执行深度，默认值为2MB。最小值为100KB，最大值为OS的限制值-512KB。OS限制可通过ulimit -s指令查看。之所以需要预留512KB的buffer，是因为堆栈深度不是在服务器的每个例程中检查的，而是只在关键可能递归的例程（如表达式求值）中检查。如果业务查询特别复杂，导致执行时超过了该参数限制，则可适当调大该参数值。默认值2MB能满足常见的业务处理。

* wal_buffers  -- WAL缓存，重启生效  
  WAL数据刷盘前的缓存大小。默认值-1标识由系统自己计算，计算的结果大小在64kB - WAL_SEGMENT_SIZE之间，其计算公式如下：
  ```
  static int
  XLOGChooseNumBuffers(void)
  {
  	int			xbuffers;
  
  	xbuffers = NBuffers / 32;
  	if (xbuffers > (wal_segment_size / XLOG_BLCKSZ))
  		xbuffers = (wal_segment_size / XLOG_BLCKSZ);
  	if (xbuffers < 8)
  		xbuffers = 8;
  	return xbuffers;
  }
  ```
  如果自动计算的数值不合适，可以手动修改，手动设置时该值不能小于32kB。
  由于每次事务提交时，WAL日志都需要刷盘，因此该参数设置过大并不会对写性能有多大的提升。如果在一台非常繁忙的系统上会有大量的事务同时提交，可以调高该参数值，否则不建议手动修改该参数。

* work_mem  -- 工作内存，普通用户设置  
  限定排序和哈希表操作在使用临时文件前可以使用的最大内存，是影响PostgreSQL性能的重要参数之一。 合适的work_mem大小能够保证这些操作在内存中完成。 使用排序操作的场景有ORDER BY，DISTINCT和merge join，使用哈希表的场景有hash连接、hash-aggregate以及基于hash处理的in子查询。
  需要注意，由于PG支持并行查询，因此在一个复杂查询中，可能同时有多个并行排序操作，而每个操作都可以使用多达work_mem的本地内存。对于多个不同的连接亦是如此。所以该参数值需要结合系统的max_connections参数值一起考虑。过大的设置会使系统在高并发时承受内存不够的风险。
  默认值为4MB，最小值64kB。设置时若未指定单位，则默认单位为kB。 work_mem的计算公式如下：
  ```
  max(64kb, (total_mem-shared_buffers)/(max_connections*3)/max_parallel_workers_per_gather)
  ```
  【优化建议】：
  全局使用系统默认的work_mem参数值，在需要进行如上复杂处理的查询会话中，动态调大该参数值，以提升性能。  

* maintenance_work_mem  --维护工作内存，普通用户设置    
  限定针对数据库的维护操作或语句可以使用的最大本地内存，是影响PostgreSQL性能的重要参数之一。在VACUUM、CREATE INDEX和ALTER TABLE ADD FOREIGN KEY操作时用到。和work_mem的使用场景不同的是，该参数的使用场景不会有太高的并发度，因此参数值可以设置的比work_mem打的多。调大该值，可以缩短VACUUM数据库和从dump文件中恢复数据库需要的时间。
  默认值64MB，最小值1MB。 建议设置为：  
  ```
  max((total_mem/63963136*1024),65536)
  ```

* max_connections  -- 最大连接数，重启生效  
  PostgreSQL可接受的最大并发连接数。 如上work_mem参数中所提及，max_connections对系统整体的内存资源消耗会产生影响。 设置过大的max_connections可能会产生如下问题：  
  （1） 数据库存在过载风险：  当活跃的并发连接比较多时，系统可能忙于在CPU和IO等待之间切换，而不能有效进行业务处理，导致系统性能降低甚至瘫痪，只能重启恢复（类似于DOS攻击）。  
  （2） 内存资源不够用： 买个数据库连接都可以使用worm_mem大小的私有内存来执行查询，当数据库连接过多时，内存资源不够的风险加大。  
  默认值100。建议值可参考如下计算公式进行配置：  
  ```
  max_connections < max(num_cores, parallel_io_limit) / (session_busy_ratio * avg_parallelism)
  ```
	* num_cores is the number of cores available  
	* parallel_io_limit is the number of concurrent I/O requests your storage subsystem can handle  
	* session_busy_ratio is the fraction of time that the connection is active executing a statement in the database  
	* avg_parallelism is the average number of backend processes working on a single query.  
 
* max_files_per_process -- 进程最大文件数，重启生效  
  单个服务器进程可以同时打开的最大文件数。如果数据库有非常多的表和索引，并且每张表都会被频繁访问，则可以将该值调大一些，以降低打开关闭文件的次数。
  若遇到“Too many open files”这样的错误，表明该参数值设置过大，需要调小该参数值。  
  该参数值的设置，不能超过系统限制(ulimit -n)。

* max_logical_replication_workers  -- 最大逻辑复制工作者，重启生效
  指定逻辑复制工作者的最大数目，包括apply工作者和表同步工作者。
  逻辑复制工作者进程是从工作者进程池中获取的，因此也受限于max_worker_processes参数控制。
  默认值4。  

* max_parallel_workers  -- 最大并行工作者，普通用户设置  
  设置系统在并行查询操作中可使用的最大工作者进程数目。通常该参数和max_parallel_maintenance_workers、max_parallel_workers_per_gather配合调整使用。
  该参数值的设置与CPU、内存以及IO等都有关。 一般情况下，该参数值设置为与CPU核数相同的数值，即
  ```
  max_parallel_workers = cpu_core_num
  ```
  默认值8。

* max_parallel_maintenance_workers  -- 最大并行维护工作者，普通用户设置  
  设置单一命令可以使用的最大并行工作者进程，当前(pg13)仅支持CREATE INDEX(bTree)和VACUUM(不带FULL)命令。 工作者进程也是从max_worker_processes创建的进程池中获取，并且受max_parallel_workers参数限制。需要注意的是，实际获取到的工作者进程数可能少于预期的进程数。
  与并行查询不同的是，所有的并行维护工作者进程共享maintenance_work_mem的资源限制，即所有的并行工作者消耗的内存总和不能超过该限制。
  默认值2，设置为0值可以禁用命令启用工作者进程。
  一般情况下，该参数依据如下公式进行调整：
  ```
  max_parallel_maintenance_workers = min(4, ceil(cpu_core_num / 2)) 
  ```

* max_parallel_workers_per_gather -- 合并节点最大工作者，普通用户设置  
  设置单个Gather或者Gather Merge节点可使用的最大工作者进程。工作者进程也是从max_worker_processes创建的进程池中获取，并且受max_parallel_workers参数限制。需要注意的是，实际获取到的工作者进程数可能少于预期的进程数。
  另外，并行查询可能会消耗比普通查询多得多的资源，因为每个工作者进程都是独立的工作，对系统的影响犹如多个不同的会话连接一样，这点在设置诸如work_mem这样的系统参数时一定要注意。
  默认值为2，设置为0可以禁用并行查询。
  一般情况下，该参数依据如下公式进行调整：  
  ```
  max_parallel_workers_per_gather = min(4, ceil(cpu_core_num / 2)) 
  ```

  【参考链接】：
  [https://www.postgresql.org/docs/13/runtime-config-resource.html](https://www.postgresql.org/docs/13/runtime-config-resource.html)  
  [https://www.citusdata.com/blog/2020/10/08/analyzing-connection-scalability/#memory-usage](https://www.citusdata.com/blog/2020/10/08/analyzing-connection-scalability/#memory-usage)
  [https://www.cybertec-postgresql.com/en/tuning-max_connections-in-postgresql/](https://www.cybertec-postgresql.com/en/tuning-max_connections-in-postgresql/)
  [https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)  
  [https://info.crunchydata.com/blog/optimize-postgresql-server-performance](https://info.crunchydata.com/blog/optimize-postgresql-server-performance)  
  [https://aws.amazon.com/cn/blogs/database/a-case-study-of-tuning-autovacuum-in-amazon-rds-for-postgresql/](https://aws.amazon.com/cn/blogs/database/a-case-study-of-tuning-autovacuum-in-amazon-rds-for-postgresql/)  
  [https://www.2ndquadrant.com/en/blog/performance-limits-of-logical-replication-solutions/](https://www.2ndquadrant.com/en/blog/performance-limits-of-logical-replication-solutions/)  
  
  【PG参数在线工具】：  
  [https://pgtune.leopard.in.ua/#/](https://pgtune.leopard.in.ua/#/)  
  [https://github.com/le0pard/pgtune/blob/master/webpack/selectors/configuration.js](https://github.com/le0pard/pgtune/blob/master/webpack/selectors/configuration.js)  
  
  [PG调优工具]
  [https://github.com/jfcoz/postgresqltuner](https://github.com/jfcoz/postgresqltuner)


  
  
