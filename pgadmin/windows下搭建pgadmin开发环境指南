# windows下pgadmin开发环境搭建
**本文总结了参照pgadmin\pkg\win32\README.txt中指令在windows10平台上搭建pgadmin开发环境的过程，读者可先自行参考READEME进行安装，在出错的情况下以本文作为参照**

## 预备条件
1. 安装Qt 5.14.2版本： <https://www.qt.io/download-qt-installer>
	* 选择MSVC 2017 64bit option.

2. 安装visual studio 2017 pro版本：<https://my.visualstudio.com/Downloads?q=Visual%20Studio%202017>
	* 选择桌面开发C++选项

3. 安装巧克力包管理器：<https://blog.csdn.net/qq_38652871/article/details/103093447>

4. 安装需要使用的各种命令行工具，命令如下：  
		choco install -y  bzip2 cmake diffutils gzip git innosetup nodejs-lts python strawberryperl wget yarn

5. 升级pip  
		pip install --upgrade pip

6. 安装virtualenv：  
		pip install virtualenv

7. 安装XXCOPY：<http://www.xxcopy.com/> 
   * 如果不需要本地出windows安装包，可不需要安装该包
   * 需要使用XXCOPY替代Make.bat中的XCOPY，因为XCOPY有260字符长度的限制，因此在打包过程中会出现***内存不足***的错误。
   * 关于如何取消win10字符长度限制，可参考<https://blog.csdn.net/lujisheng/article/details/78936904>，但是该设置并不能使XCOPY支持超过260的路径，只能说系统支持了该能力，而由于XCOPY编译的原因并不支持。

## 后面所有章节的操作需要在vs2017的命令行提示符下进行
1. 选择 **适用于 VS 2017 的 x64 本机工具命令提示**  
2. 创建工作目录  
		mkdir c:\build64
3. 下载zlib源码，并进行编译构建  
		wget https://zlib.net/zlib-1.2.11.tar.gz  
		tar -zxvf zlib-1.2.11.tar.gz  
		cd zlib-1.2.1  
		cmake -DCMAKE_INSTALL_PREFIX=C:/build64/zlib -G "Visual Studio 15 2017 Win64" .  
		msbuild RUN_TESTS.vcxproj /p:Configuration=Release
		msbuild INSTALL.vcxproj /p:Configuration=Release
		copy C:\build64\zlib\lib\zlib.lib C:\build64\zlib\lib\zdll.lib
	
    * 【注意】 此处msbuild的顺序和README.txt中的不一致，若按照README中的顺序执行则第一个msbuild会报错

4. 下载OpenSSL源码，并编译构建  
		wget https://www.openssl.org/source/openssl-1.1.1g.tar.gz
		tar -zxvf openssl-1.1.1g.tar.gz
		cd openssl-1.1.1g
		perl Configure VC-WIN64A no-asm --prefix=C:\build64\openssl no-ssl2 no-ssl3 no-comp
		nmake
		nmake test
		nmake install

5. 下载PostgreSQL源码并编译构建
		wget https://ftp.postgresql.org/pub/source/v12.3/postgresql-12.3.tar.gz
		tar -zxvf postgresql-12.3.tar.gz
		cd postgresql-12.3\src\tools\msvc

		>> config.pl echo # Configuration arguments for vcbuild.
		>> config.pl echo use strict;
		>> config.pl echo use warnings;
		>> config.pl echo.
		>> config.pl echo our $config = {
		>> config.pl echo 	asserts   =^> 0,        # --enable-cassert
		>> config.pl echo 	ldap      =^> 1,        # --with-ldap
		>> config.pl echo 	extraver  =^> undef,    # --with-extra-version=^<string^>
		>> config.pl echo 	gss       =^> undef,    # --with-gssapi=^<path^>
		>> config.pl echo 	icu       =^> undef,    # --with-icu=^<path^>
		>> config.pl echo 	nls       =^> undef,    # --enable-nls=^<path^>
		>> config.pl echo 	tap_tests =^> undef,    # --enable-tap-tests
		>> config.pl echo 	tcl       =^> undef,    # --with-tcl=^<path^>
		>> config.pl echo 	perl      =^> undef,    # --with-perl
		>> config.pl echo 	python    =^> undef,    # --with-python=^<path^>
		>> config.pl echo 	openssl   =^> 'C:\build64\openssl',    # --with-openssl=^<path^>
		>> config.pl echo 	uuid      =^> undef,    # --with-ossp-uuid
		>> config.pl echo 	xml       =^> undef,    # --with-libxml=^<path^>
		>> config.pl echo 	xslt      =^> undef,    # --with-libxslt=^<path^>
		>> config.pl echo 	iconv     =^> undef,    # (not in configure, path to iconv)
		>> config.pl echo 	zlib      =^> 'C:\build64\zlib'     # --with-zlib=^<path^>
		>> config.pl echo };
		>> config.pl echo.
		>> config.pl echo 1;
		>> buildenv.pl echo $ENV{PATH} = "C:\\build64\\openssl\\bin;C:\\build64\\zlib\\bin;$ENV{PATH}";
		
		perl build.pl Release
		perl vcregress.pl check
		perl install.pl C:\build64\pgsql
		copy C:\build64\zlib\bin\zlib.dll C:\build64\pgsql\bin"
		copy C:\build64\openssl\bin\libssl-1_1-x64.dll C:\build64\pgsql\bin"
		copy C:\build64\openssl\bin\libcrypto-1_1-x64.dll C:\build64\pgsql\bin"  

## 安装淘宝npm源
		npm install -g cnpm 
		cnpm config set registry https://registry.npm.taobao.org --global
		cnpm config set strict-ssl false

## 修改适配编译构建脚本Make.bat(如果不需要本地出windows安装包，以下步骤1到步骤5可不执行)
1. 删除脚本第35行的签名工具调用
2. 修改适配第55行~第61行的环境变量,修改为自己环境相关的
3. 修改第199行、202行的yarn安装调用为cnpm安装调用，如下
		CALL cnpm install || EXIT /B 1     -- 199行
		CALL cnpm run bundle || EXIT /B 1  -- 202行
4. 将204行的XCOPY修改为XXCOPY，原因如上所述，修改后如下
		XXCOPY /S /I /E /H /Y "%WD%\web" "%BUILDROOT%\web" > nul || EXIT /B 1
5. 将211行的FOR循环删除替换为如下两个DEL删除（在编译过程中发现FOR循环好像死循环了，一直在执行，无法退出，没有深究其原因）
		DEL /a /f /s /q  "%BUILDROOT%\web\*.pyc" 1> nul 2>&1
		DEL /a /f /s /q  "%BUILDROOT%\web\*.pyo" 1> nul 2>&1

## 适配web目录下的package.json文件和webpack.config.js文件
1. 将package.json文件中的第116行和120行命令中的yarn修改为cnpm，修改后的命令如下
		"webpacker:watch": "cnpm run webpack --config webpack.config.js --progress --watch",   -- 116行
		"bundle": "cross-env NODE_ENV=production cnpm run bundle:dev",                         -- 120行

2. 将webpack.config.js文件中的第195行到211行之间的装载器注释掉，如下
		//{
		//  loader: 'image-webpack-loader',
		//  query: {
		//    bypassOnDebug: true,
		//    mozjpeg: {
		//      progressive: true,
		//    },
		//    gifsicle: {
		//      interlaced: false,
		//    },
		//    optipng: {
		//      optimizationLevel: 7,
		//    },
		//    pngquant: {
		//      quality: '75-90',
		//      speed: 3,
		//    },
		//  },
		//}  


## 搭建本地开发环境
1. 安装JS依赖
		cd pgadmin4\web
		cnpm install
		cnpm run bundle

2. 创建虚拟机环境
		cd pgadmin4
		python -m venv venv
		pip install -r web\regression\requirements.txt
		pip install sphinx

3. 启动环境
		python .\web\pgAdmin4.py 

## 编译安装包
 在pgadmin目录下执行make，即可在dist目录下编译生成安装包
