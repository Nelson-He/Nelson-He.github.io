# PostgreSQL 11.0版本发布说明
  * 原始链接：[E.3. 版本11](http://postgres.cn/docs/11/release-11.html)  
  
## 1. 服务器  
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

