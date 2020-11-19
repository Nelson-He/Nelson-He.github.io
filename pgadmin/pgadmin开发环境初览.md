# pgadmin4开发环境初览  

  执行以下python命令，启动pgadmin服务  
		python ./web/pgAdmin4.py  
  控制台输出以下信息，表示可以在浏览器中本地浏览器中打开链接进行操作  

  登录界面如下

  输入账户和密码后，登录系统，界面显示如下：

  pgadmin4是基于JS和python开发的，其中又使用了flask作为web框架、jinja2作为模板引擎。jinja2是flask的默认模板引擎。
  关于flask和jinja2的，可以参考如下资料：  
  flask:  <https://dormousehole.readthedocs.io/en/latest/>  
  jinja2: <https://jinja.palletsprojects.com/en/2.11.x/>