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
## 10. 源代码  
  
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

