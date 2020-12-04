# pgadmin代码结构及开发流程浅述  
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

其中web目录是pgadmin的源码目录，在使用pycharm导入工程进行编译调试时，应该以web目录作为源码目录。
Make.bat是windows平台下的打包脚本，Makefile则是Linux系列平台下的打包脚本。web目录的目录结构如下所示：  

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
  browser：几乎所有数据库相关的操作代码都在该目录下，该目录下的代码层级结构和数据库的层级结构一致，即group -> server -> database -> scehma -> table -> column。
  dashboard： pgadmin登录后首页仪表盘相关的代码在该目录下，包括仪表盘相关的js文件和sql模板文件，
  tools: 该目录下存放了pgadmin页面上提供的工具的代码，包括sql编辑器等
  static: 该目录下存放了pgadmin全局需要使用的一些js文件和其他资源文件
  
