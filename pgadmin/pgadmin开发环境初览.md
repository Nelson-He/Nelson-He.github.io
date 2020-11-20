# pgadmin4开发环境初览  

  执行以下python命令，启动pgadmin服务  
		python ./web/pgAdmin4.py  
  控制台输出以下信息，表示可以在浏览器中本地浏览器中打开链接进行操作  
  ![](https://github.com/Nelson-He/Nelson-He.github.io/blob/main/pictures/pgadmin/python_web_pgadmin.png)

  登录界面如下  
  ![](https://github.com/Nelson-He/Nelson-He.github.io/blob/main/pictures/pgadmin/login.png)

  输入账户和密码后，登录系统，界面显示如下：
  ![](https://github.com/Nelson-He/Nelson-He.github.io/blob/main/pictures/pgadmin/index.png)

  pgadmin4是基于JS和python开发的，其中又使用了flask作为web框架、jinja2作为模板引擎。jinja2是flask的默认模板引擎。
  关于flask和jinja2的，可以参考如下资料：  
  flask:  <https://dormousehole.readthedocs.io/en/latest/>  
  jinja2: <https://jinja.palletsprojects.com/en/2.11.x/>  

  在首页会有一个“Add New Server”的快捷按钮，点击就可以创建数据库连接，配置连接参数后点击“Save”按钮保存并连接数据库。  
  ![](https://github.com/Nelson-He/Nelson-He.github.io/blob/main/pictures/pgadmin/setup_server.png)

  输入连接参数后，连接数据库。在连接数据库时，要确保数据库的相关配置文件已经修改正确（postgresql.conf和pg_hba.conf)。
  在数据库中创建一张表，在pgadmin中就可以刷新显示出来，如下
  ![](https://github.com/Nelson-He/Nelson-He.github.io/blob/main/pictures/pgadmin/pgadmin_table_test.png)