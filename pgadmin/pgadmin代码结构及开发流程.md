# pgadmin代码结构及开发流程浅述  
## 主要目录结构  
  pgadmin以flask作为web框架，以jinja2作为模板引擎。前端采用js语言开发，后端采用python语言开发，在前后端之间通过url请求进行通信。
  其顶层代码目录结构树如下：  
  
      DEPENDENCIES
      Dockerfile
      docs
      .gitignore
      LICENSE
      Make.bat
      Makefile
      pkg 
      .pycodestyle
      README
      requirements.txt
      runtime
      tools
      web

其中web目录是pgadmin的源码目录，在使用pycharm导入工程进行编译调试时，应该以web目录作为源码目录。Make.bat是windows平台下的打包脚本，Makefile则是Linux系列平台下的打包脚本。如果
在二次开发后需要针对对应平台出安装包，则需要修改对应的打包脚本。   
web目录的目录结构如下所示：  

    babel.cfg
    config.py
    .editorconfig
    .eslintignore
    .eslintrc.js
    karma.conf.js
    migrations
    package.json
    pgadmin
    pgAdmin4.py
    pgAdmin4.wsgi
    pgadmin.themes.json
    regression
    setup.py
    webpack.config.js
    webpack.shim.js
    webpack.test.config.js
    yarn.lock
  
pgAdmin4.py是pgadmin的main文件，在针对pgadmin进行调试时，选中该文件后启动调试即可。webpack.config.js和package.json是js的依赖和打包相关的配置。  
config.py是pgadmin的运行环境相关的配置，包括监听的端口号等都可以在该文件中配置。官方不建议修改此文件，建议新增config_local.py，将需要重新配置的参数值配置在该文件中，
pgadmin在加载配置的时候，首先加载config.py，然后加载config_local.py，后者会覆盖前者的设置值。  
pgadmin目录是pgadmin的核心目录，所有的功能实现都在pgadmin目录中，其目录结构如下：  

    about
    authenticate
    browser
    dashboard
    feature_tests
    help
    __init__.py
    messages.pot
    misc
    model
    preferences
    redirects
    settings
    setup
    static
    templates
    tools
    translations
    utils
    
  简述下最重要的几个目录：  
  browser：几乎所有数据库相关的操作代码都在该目录下，该目录下的代码层级结构和数据库的层级结构一致，即group -> server -> database -> scehma -> table -> column    
  dashboard： pgadmin登录后首页仪表盘相关的代码在该目录下，包括仪表盘相关的js文件和sql模板文件  
  tools: 该目录下存放了pgadmin页面上提供的工具的代码，包括sql编辑器等  
  static: 该目录下存放了pgadmin全局需要使用的一些js文件和其他资源文件  
  
## 代码组织关系  
  大家可以发现，每一级目录下面一般都会存在如下两个目录和一个文件：  
  
    --static  
    --templates  
    --__init__.py   
    
  其中__init.py__在python中的作用大家都知道，就是将该目录变成一个可导入使用的包。在pgadmin中，除了工具外其他所有的后端功能，基本都在__init__.py中实现。  
  在查看代码时，可以针对某一功能点，直接查看对应的目录下的__init__.py即可，比如我们想看首页dashboard上的数据是怎么来的，直接查看dashboard目录下的__init__.py即可。  
  templates下存放jinjia2引擎需要使用的模板文件，包括html/css/sql等模板文件。  
  在templates下的sql文件，又会按照不同的版本号进行存放，pgadmin匹配模板文件的过程为：  
  依据所连接数据库的版本号，逐级递减的在版本目录下寻找对应模板文件，若找到，则返回。最终在版本目录下未找到时，则返回default目录下的模板文件。  
  static目录下存放前端界面需要使用的代码，包括js代码，css文件等。  
  
## 一个简单demo  
  如何针对pgadmin进行二次开发，对于没有任何前端开发基础的我来说，最简单的方法就是‘抄袭’，即研究下当前的功能是怎么实现的，然后照葫芦画瓢。  
  下面以我添加支持约束启停功能为例，简单概述下代码流程。  
### 找葫芦  
  使用pgadmin的约束，发现社区版本身支持约束的删除（drop），而启停是和删除完全并列的两个功能，因此只要找到删除功能的代码点，就可以知道启停如何添加了。  
  1. 在代码中全局搜索删除功能上的标签‘Delete/Drop’，发现对应的标签在node.js文件中定义，这就是我们需要添加enable和disable的地方。  
  2. 根据前面所述的代码目录结构，我们逐级往下找，找到约束功能的后端实现在web\pgadmin\browser\server_groups\servers\databases\schemas\tables\constraints目录下，
  这就是需要增加后端代码和sql模板的地方。  
### 画瓢  
  前面已经找到了葫芦，紧接着开始画瓢就行。
  * 在node.js中增加label。  
  1. 由于enable和disable标签只能出现在约束上，在其他页签上右击时不能出现，因此需要有个控制机制。研究约束删除，发现删除标签是否出现，受变量canDelete控制，
  因此引入两个变量canEnable和canDisable，控制是否在右键时显示启停标签。默认false，即不显示。  
  
			/******************************************************************  
			** This function determines the given item can be enabled or not.  
			**  
			** Override this, when a node can not enabled.  
			**/  
			canEnable: false,  
			/******************************************************************  
			** This function determines the given item can be disabled or not.  
			**  
			** Override this, when a node can not be disabled.  
			**/  
			canDisable: false,  
      
   2. 添加标签对应的处理逻辑，以约束启用（enable为例），代码如下：  
   
  			// Enable the selected object
			enable_obj: function(args, item) {
			var input = args || {
				'url': 'DELETE',
				},
				obj = this,
				t = pgBrowser.tree,
				i = input.item || item || t.selected(),
				d = i && i.length == 1 ? t.itemData(i) : undefined;
			
			if (!d)
				return;
			
			/*
			* Make sure - we're using the correct version of node
			*/
			obj = pgBrowser.Nodes[d._type];
			var objName = d.label;
			
			var msg, title;
			msg = gettext('Are you sure you want to enable %s "%s"?', obj.label.toLowerCase(), d.label);
			title = gettext('Enable %s?', obj.label);
			if (!(_.isFunction(obj.canEnable) ?
				obj.canEnable.apply(obj, [d, i]) : obj.canEnable)) {
				Alertify.error(
				gettext('The %s "%s" cannot be enabled.', obj.label, d.label),
				10
				);
				return;
			}
			Alertify.confirm(title, msg,
				function() {
				$.ajax({
					url: obj.generate_url(i, 'enable', d, true),
					type: 'put',
				})
					.done(function(res) {
					if (res.success == 0) {
						pgBrowser.report_error(res.errormsg, res.info);
					} else {
						pgBrowser.removeTreeNode(i, true);
					}
					return true;
					})
					.fail(function(jqx) {
					var errmsg = jqx.responseText;
					/* Error from the server */
					if (jqx.status == 417 || jqx.status == 410 || jqx.status == 500) {
						try {
						var data = JSON.parse(jqx.responseText);
						errmsg = data.info || data.errormsg;
						} catch (e) {
						console.warn(e.stack || e);
						}
					}
					pgBrowser.report_error(
						gettext('Error enabling %s: "%s"', obj.label, objName), errmsg);
			
					});
				},
				null
			).set('labels', {
				ok: gettext('Yes'),
				cancel: gettext('No'),
			}).show();
			},
      
  3. 针对约束标签，使能启用和禁用label，找到constraints目录下的static目录，在对应的约束定义中添加功能，以check约束为例，在check_constraint.js中添加    

			canEnable: true,  
			canDisable: true,   

  * 增加后端处理逻辑，以check约束启用为例，    
  1. 增加enable.sql模板文件  

			ALTER TABLE {{ conn|qtIdent(data.schema, data.table) }}
				ENABLE CONSTRAINT {{ conn|qtIdent(data.name) }};
        
  2. 在__init__.py中增加处理逻辑  

			@check_precondition  
			def enable_constraint(self, gid, sid, did, scid, tid, cid):  
				"""  
				Validate check constraint.  
				Args:  
					gid: Server Group Id  
					sid: Server Id  
					did: Database Id  
					scid: Schema Id  
					tid: Table Id  
					cid: Check Constraint Id  
			  
				Returns:  
				"""  
				data = {}  
				try:  
					data['schema'] = self.schema  
					data['table'] = self.table  
					sql = render_template(  
						"/".join([self.template_path, 'get_name.sql']), cid=cid)  
					status, res = self.conn.execute_scalar(sql)  
					if not status:  
						return internal_server_error(errormsg=res)  
			  
					data['name'] = res  
					sql = render_template(  
						"/".join([self.template_path, 'enable.sql']), data=data)  
					status, res = self.conn.execute_dict(sql)  
					if not status:  
						return internal_server_error(errormsg=res)  
			  
					return make_json_response(  
						success=1,  
						info=_("Check constraint enabled."),  
						data={  
							'id': cid,  
							'tid': tid,  
							'scid': scid,  
							'did': did  
						}  
					)  
				except Exception as e:  
					return internal_server_error(errormsg=str(e))  
       
  3. 配置路由关系。由于pgadmin前端和后端之间是通过url交互的，因此需要配置正确的路由关系。此处我们借鉴的是删除功能，再次借鉴删除的路由，配置如下  

			'enable': [{'put': 'enable_constraint'}],  
			'disable': [{'put': 'disable_constraint'}],  
      
  至此，pgadmin的开发流程就简述完了。  
# 总结  
  由于在开发pgadmin之前没有前端基础，而且python语言也不熟，因此开发过程中都是边查阅资料，边模仿已有实现，不断研究，不断尝试。  
