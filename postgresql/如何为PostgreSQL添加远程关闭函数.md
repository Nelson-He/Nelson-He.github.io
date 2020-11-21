# 如何为PostgreSQL添加远程关闭函数  
  在后台运行数据库时，常使用pg_ctl命令来控制数据库的启停等操作   
  
		pg_ctl start   # 启动数据库  
		pg_ctl stop    # 关闭数据库  
		pg_ctl restart # 重启数据库  
  或者在psql客户端连接数据库时，可以使用元命令\!来执行shell命令操作数据库，如下   
  
  		postgres=#  
		postgres=# \\! pg_ctl restart -mi  
  但是当通过libpq协议连接数据库时，是没有办法对数据库进行启停操作的。其实虽然通过libpq没有办法实现启动，但是却可以实现数据库的停止和重启，方法就是在数据库内核中自定义相关的
系统函数，然后通过libpq连接到数据库执行该系统函数即可。  
  下面通过添加远程停库的功能来演示这种方法。  
  首先在数据库内核中新增数据库停止的方法，如下  
  
		/* These codes are just used to demenstate the realization. */  
		static pgpid_t  
		get_pgpid(bool is_status_request)  
		{
			FILE	   *pidf;  
			long		pid;  
			char pid_file[MAXPGPATH];  
		  
			snprintf(pid_file, MAXPGPATH, "%s/postmaster.pid", DataDir);  
		  
			pidf = fopen(pid_file, "r");  
			if (pidf == NULL)  
			{  
				return -1;  
			}  
			if (fscanf(pidf, "%ld", &pid) != 1)  
			{  
				fclose(pidf);
				return -1;
			}
			
			fclose(pidf);
			return (pgpid_t) pid;
		}
		
		void pg_ctl_stop(FunctionCallInfo fcinfo) 
		{
			pgpid_t		pid;
		
			pid = get_pgpid(false);
		
			if (pid <= 0)				/* no pid file */
			{
				ereport(ERROR, (errmsg("postmaster does not exist")));
			}
		
			if (kill((pid_t) pid, SIGQUIT) != 0) /* use SIGQUIT to shutdown the database directlly */
			{
				ereport(ERROR, (errmsg("kill the postmaster failed")));
			}	
		}  
  在pg_proc.dat中注册上面新增的函数，如下  
  
		{ oid => '4035', descr => 'kill the database from client',  
		proname => 'pg_ctl_stop', prorettype => 'void',  
		proargtypes => '', prosrc => 'pg_ctl_stop' },  
  编译并安装数据库，由于新增了系统函数，所以需要重新initdb。  
  
		initdb -D /opt/local/pgsql/data -E utf-8  
  启动数据库，此时就可以远程连接到数据库上，执行select pg_ctl_stop();来停止数据库。  
  使用python的psycopg2库来连接数据库，该库用来在命令行模式下通过libpq来验证数据库操作相当方便。  
  
		>>> import psycopg2  
		>>> conn = psycopg2.connect(database="postgres", user="postgres", password="", host="127.0.0.1", port="5432")  
		>>> cusor = conn.cursor()  
		>>> cusor.execute("select pg_ctl_stop();")  
  执行后在另一个连接数据库的终端中可以看到，数据库已经被关闭，如下  
  
		2020-11-21 22:13:54.886 CST [17085] LOG:  received immediate shutdown request  
		2020-11-21 22:13:54.889 CST [17124] WARNING:  terminating connection because of crash of another server process
		2020-11-21 22:13:54.889 CST [17124] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
		2020-11-21 22:13:54.889 CST [17124] HINT:  In a moment you should be able to reconnect to the database and repeat your command.
		2020-11-21 22:13:54.889 CST [17090] WARNING:  terminating connection because of crash of another server process
		2020-11-21 22:13:54.889 CST [17090] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
		2020-11-21 22:13:54.889 CST [17090] HINT:  In a moment you should be able to reconnect to the database and repeat your command.
		2020-11-21 22:13:54.910 CST [17085] LOG:  database system is shut down



