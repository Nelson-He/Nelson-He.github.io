# postgresql插件支持鉴权失败锁账户
  需求如下：  
  * 1. 插件支持鉴权失败后锁定账户，除非账户解锁，否则无法再次登录连接  
  * 2. 锁定账户前可以尝试的登录次数需要可配  

## 初始思路  
  * 由于PG自身支持通过ALTER ROLE XXX NOLOGIN的方式锁定账户，因此参考ALTER ROLE的方式实现了一版账户锁定，其中锁账户的代码如下：  

			static void auth_lock(const Port *port)  
			{  
				Datum		new_record[Natts_pg_authid];  
				bool		new_record_nulls[Natts_pg_authid];  
				bool		new_record_repl[Natts_pg_authid];  
				Relation	pg_authid_rel;  
				TupleDesc	pg_authid_dsc;  
				HeapTuple	tuple,  
							new_tuple;  
				Oid			roleid;  
					  
				/*  
				* Scan the pg_authid relation to be certain the user exists.  
				*/  
				pg_authid_rel = heap_open(AuthIdRelationId, RowExclusiveLock);
				pg_authid_dsc = RelationGetDescr(pg_authid_rel);
			
				tuple = SearchSysCache1(AUTHNAME, CStringGetDatum(port->user_name));
				if (!HeapTupleIsValid(tuple))
					ereport(ERROR,
						(errcode(ERRCODE_UNDEFINED_OBJECT),
						errmsg("role \"%s\" does not exist", port->user_name)));
									
				roleid = HeapTupleGetOid(tuple);
			
				/*
				* Build an updated tuple, perusing the information just obtained
				*/
				MemSet(new_record, 0, sizeof(new_record));
				MemSet(new_record_nulls, false, sizeof(new_record_nulls));
				MemSet(new_record_repl, false, sizeof(new_record_repl));
				
				new_record[Anum_pg_authid_rolcanlogin - 1] = BoolGetDatum(false);
				new_record_repl[Anum_pg_authid_rolcanlogin - 1] = true;
			
				new_tuple = heap_modify_tuple(tuple, pg_authid_dsc, new_record,
											new_record_nulls, new_record_repl);
				CatalogTupleUpdate(pg_authid_rel, &tuple->t_self, new_tuple);
			
				InvokeObjectPostAlterHook(AuthIdRelationId, roleid, 0);
			
				ReleaseSysCache(tuple);
				heap_freetuple(new_tuple);
			
				heap_close(pg_authid_rel, RowExclusiveLock);
			
			}

  编译后调测，发现数据库报错，原因是全局变量MyDatabaseId未赋值，因此无法进行表扫描操作。  
  查看数据库backend进程的初始化函数InitPostgres，发现鉴权机制的代码PerformAuthentication在打开数据库对象之前，因此无法在鉴权阶段进行表操作。  
  为了实现PG的鉴权锁定功能，决定探一探PG的鉴权机制。  

## 初探PG鉴权机制  
  大家知道，在客户端连接数据库的过程中，会经历两个阶段：  
  * 1. startup 阶段，客户端尝试创建连接并发送授权信息，如果一切正常，服务端会反馈状态信息，连接成功创建，随后进入 normal 阶段。  
  * 2. normal 阶段，客户端发送请求至服务端，服务端执行命令并将结果返回给客户端。客户端请求结束后，可以主动发送消息断开连接。  
  而鉴权就发生在startup阶段。  
  鉴权功能的整体入口为PerformAuthentication函数，在加载完pg_hba.conf和pg_ident.conf文件后，会依据文件中配置的鉴权方法进行鉴权。  
  鉴权的核心功能在ClientAuthentication中实现，在该函数的最后，有一个钩子ClientAuthentication_hook，用来执行一些自定义的鉴权操作。我们需要插件化支持鉴权锁定账户，
  就要打这个钩子的主意。这里可以参考社区已有的插件auth_delay，其实现功能很简单，就是在鉴权失败后休眠一段时间，以防止暴力破解。  
  鉴权的处理流程大致如下：  

			client                            server(postmaster)  
			
			startup     --------------->       
			                                  fork (child_process1)  
			                                  ClientAuthentication(1)  
			            <---------------      auth_request  
			password    --------------->      fork (child_process2)  
			                                  
			                                  ClientAuthentication(2)  
                                       
  在startup阶段，PG会fork前后两个子进程处理鉴权相关的事宜。如果pg_hba.conf文件中没有客户端的连接配置信息，则只需要fork一个子进程就会报错（即上图中的child_process1处理阶段）。  
  真正的密码效验在child_process2进程的处理阶段。这里一定得注意。因为钩子是在ClientAuthentication函数的尾部，而前后两个进程都会执行该钩子，如果不注意此处的差异，则每次登陆时钩子
  都会执行两次，导致次数递减错误。  
  而在startup阶段无法操作数据库，在normal阶段才可以操作数据库，这样的设计可以更安全的保护数据库。
  
## 实现方案    
  初步了解PG的鉴权机制后，借鉴pg_hba.conf的处理机制，新增控制文件，存放用户鉴权信息。大体的实现方案为：  
  * 1. 不再依赖内核的alter role机制， 在鉴权失败后，通过ereport FATAL的方式报错，同时刷新用户鉴权信息文件。  
  * 2. 假如用户设置最大次数为5次，则4次失败，第5次成功后，需要重置用户鉴权信息，这样用户可以重新鉴权5次。  
  * 3. 若当天用户鉴权4次失败后未再进行鉴权，则第二天仍然可以鉴权5次。  
  * 4. 用户鉴权失败锁定后，需要提供手段解锁账户，因此新增系统函数实现解锁功能。  
  
## 插件支持  
  实现上面的方案，并进行插件化。PG所有的插件代码在contrib目录下，contrib是contribute的缩写，意指大家贡献的结果。插件化时需要提供如下脚本：  
  * Makefile   编译脚本，控制插件代码如何编译和安装  
  * control    插件控制文件，在PG中执行create extension时需要使用，控制插件的版本、使用权限等
  * sql        实现的系统函数的定义文件
  
  上面三个文件均可以在拷贝已有文件的基础上修改完成。完成后执行make && make install, 就可以在数据库中使用create extension的方式启用该插件。  
  
