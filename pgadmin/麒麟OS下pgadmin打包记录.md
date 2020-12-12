* (1) 新增postgres用户，在postgres用户下编译安装PostgreSQL, 此处安装目录为/opt/local/pgsql，安装完成后initdb数据库。  

			./configure  --prefix=/opt/local/pgsql
			make
			make install
			mkdir /opt/local/pgsql/data
			initdb -D /opt/local/pgsql/data -E utf-8
      
* (2) 修改环境变量：  

			export PATH=/opt/local/pgsql/bin:$PATH
			export PGDATA=/opt/local/pgsql/data
      
* (3) 安装依赖包  

			yum install -y expect fakeroot httpd-devel qt5-qtbase-devel python3-devel nodejs python3-sphinx rpm-build rpm-sign yum-utils openssl-devel 
      
* (4) 安装yarn和cnpm  

			npm install -g cnpm --registry=https://registry.npm.taobao.org  
			npm install -g yarn    
      
* (5) 修改权限，同时将qmake路径添加到postgres用户的PGPATH中：  

			chmod 755 -R /usr/lib/node_modules
			export PATH=/usr/lib64/qt5/bin/:$PATH  
     
* (6) 修改pkg/linux/build-functions.sh脚本，将其中yarn命令修改为cnpm，如下  

			cnpm install  
			cnpm run bundle  
      
* (7) 修改脚本 pkg/redhat/build.sh，删除脚本中关于centos版本号==7的判断条件，修改后的build.sh脚本如下  

			#!/bin/bash
			
			# Exit when any command fails
			set -e
			
			# Debugging shizz
			trap 'last_command=$current_command; current_command=$BASH_COMMAND' DEBUG
			trap 'if [ $? -ne 0 ]; then echo "\"${last_command}\" command filed with exit code $?."; fi' EXIT
			
			OS_VERSION=$(cat /etc/os-release | grep "^VERSION_ID=" | awk -F "=" '{ print $2 }' | sed 's/"//g')
			OS_NAME=$(cat /etc/os-release | grep "^ID=" | awk -F "=" '{ print $2 }' | sed 's/"//g')
			OS_ARCH=$(arch)
			
			# Make sure we get the right libpq
			export PATH=/opt/local/pgsql/bin:$PATH
			
			# Common Linux build functions
			source pkg/linux/build-functions.sh
			
			# Assemble the "standard" installation footprint
			_setup_env $0 "redhat"
			_cleanup "rpm"
			_setup_dirs
			_create_python_virtualenv "redhat"
			_build_runtime
			_build_docs "redhat"
			_copy_code
			
			#
			# Server package
			#
			
			# Create the Redhat packaging stuffs for the server
			echo "Creating the server package..."
			
			cat << EOF > "${BUILDROOT}/server.spec"
			%global __requires_exclude_from ^/.*$
			%global __provides_exclude_from ^/.*$
			
			# Bytecompiling Python 3 doesn't work on RHEL/CentOS 7, so make it a no-op
			
			%define __python /bin/true
			
			%undefine __brp_mangle_shebangs
			%undefine __brp_ldconfig
			
			Name:		${APP_NAME}-server
			Version:	${APP_LONG_VERSION}
			Release:	1%{?dist}
			Summary:	The core server package for pgAdmin.
			License:	PostgreSQL
			URL:		https://www.pgadmin.org/
			
			Requires:	python3, postgresql-libs >= 11
			
			%description
			The core server package for pgAdmin. pgAdmin is the most popular and feature rich Open Source administration and development platform for PostgreSQL, the most advanced Open Source database in the world.
			
			%build
			
			%install
			cp -rfa %{pga_build_root}/server/* \${RPM_BUILD_ROOT}
			
			%files
			/usr/pgadmin4/*
			EOF
			
			# Build the Redhat package for the server
			rpmbuild --define "pga_build_root ${BUILDROOT}" -bb "${BUILDROOT}/server.spec"
			
			#
			# Desktop package
			#
			
			# Create the Redhat packaging stuffs for the desktop
			echo "Creating the desktop package..."
			
			cat << EOF > "${BUILDROOT}/desktop.spec"
			%global __requires_exclude_from ^/.*$
			%global __provides_exclude_from ^/.*$
			
			%undefine __brp_mangle_shebangs
			%undefine __brp_ldconfig
			
			Name:		${APP_NAME}-desktop
			Version:	${APP_LONG_VERSION}
			Release:	1%{?dist}
			Summary:	The desktop user interface for pgAdmin.
			License:	PostgreSQL
			URL:		https://www.pgadmin.org/
			Requires:	${APP_NAME}-server, qt5-qtbase, qt5-qtbase-gui
			
			%description
			The desktop user interface for pgAdmin. pgAdmin is the most popular and feature rich Open Source administration and development platform for PostgreSQL, the most advanced Open Source database in the world.
			
			%build
			
			%install
			cp -rfa %{pga_build_root}/desktop/* \${RPM_BUILD_ROOT}
			
			%files
			/usr/pgadmin4/bin/*
			/usr/pgadmin4/share/*
			/usr/share/applications/*
			EOF
			
			# Build the Redhat package for the server
			rpmbuild --define "pga_build_root ${BUILDROOT}" -bb "${BUILDROOT}/desktop.spec"
			
			
			#
			# Web package
			#
			
			# Create the Redhat packaging stuffs for the web
			echo "Creating the web package..."
			
			cat << EOF > "${BUILDROOT}/web.spec"
			%global __requires_exclude_from ^/.*$
			%global __provides_exclude_from ^/.*$
			
			%undefine __brp_mangle_shebangs
			%undefine __brp_ldconfig
			
			Name:		${APP_NAME}-web
			Version:	${APP_LONG_VERSION}
			Release:	1%{?dist}
			BuildArch:	noarch
			Summary:	The web interface for pgAdmin, hosted under Apache HTTPD.
			License:	PostgreSQL
			URL:		https://www.pgadmin.org/
			
			Requires:	${APP_NAME}-server, httpd, pgadmin4-python3-mod_wsgi
			
			%description
			The web interface for pgAdmin, hosted under Apache HTTPD. pgAdmin is the most popular and feature rich Open Source administration and development platform for PostgreSQL, the most advanced Open Source database in the world.
			
			%build
			
			%install
			cp -rfa %{pga_build_root}/web/* \${RPM_BUILD_ROOT}
			
			%files
			/usr/pgadmin4/bin/*
			/etc/httpd/conf.d/*
			EOF
			
			mkdir -p "${WEBROOT}/etc/httpd/conf.d"
			cp "${SOURCEDIR}/pkg/redhat/pgadmin4.conf" "${WEBROOT}/etc/httpd/conf.d"
			
			# Build the Redhat package for the web
			rpmbuild --define "pga_build_root ${BUILDROOT}" -bb "${BUILDROOT}/web.spec"
			
			#
			# Meta package
			#
			
			
			# Create the Redhat packaging stuffs for the meta package
			echo "Creating the meta package..."
			
			cat << EOF > "${BUILDROOT}/meta.spec"
			%global __requires_exclude_from ^/.*$
			%global __provides_exclude_from ^/.*$
			
			%undefine __brp_mangle_shebangs
			%undefine __brp_ldconfig
			
			Name:		${APP_NAME}
			Version:	${APP_LONG_VERSION}
			Release:	1%{?dist}
			BuildArch:	noarch
			Summary:	Installs all required components to run pgAdmin in desktop and web modes.
			License:	PostgreSQL
			URL:		https://www.pgadmin.org/
			Requires:	${APP_NAME}-server, ${APP_NAME}-desktop, ${APP_NAME}-web
			
			%description
			Installs all required components to run pgAdmin in desktop and web modes. pgAdmin is the most popular and feature rich Open Source administration and development platform for PostgreSQL, the most advanced Open Source database in the world.
			
			%build
			
			%install
			
			%files
			
			EOF
			
			# Build the Redhat package for the meta package
			rpmbuild --define "pga_build_root ${BUILDROOT}" -bb "${BUILDROOT}/meta.spec"
			
			#
			# mod_wsgi for CentOS 7
			#
			cp "${SOURCEDIR}/pkg/redhat/pgadmin4-python3-mod_wsgi-exports.patch" ${HOME}/rpmbuild/SOURCES
			cp "${SOURCEDIR}/pkg/redhat/pgadmin4-python3-mod_wsgi.conf" ${HOME}/rpmbuild/SOURCES
			curl -o ${HOME}/rpmbuild/SOURCES/mod_wsgi-4.7.1.tar.gz https://codeload.github.com/GrahamDumpleton/mod_wsgi/tar.gz/4.7.1
			rpmbuild -bb "${SOURCEDIR}/pkg/redhat/pgadmin4-python-mod_wsgi.spec"
			
			# Get the libpq we need
			yumdownloader --downloadonly --destdir=$DISTROOT postgresql13-libs
			
			#
			# Get the results!
			#
			cp ${HOME}/rpmbuild/RPMS/${OS_ARCH}/${APP_NAME}-*${APP_LONG_VERSION}-*.${OS_ARCH}.rpm "${DISTROOT}/"
			cp ${HOME}/rpmbuild/RPMS/noarch/${APP_NAME}-*${APP_LONG_VERSION}-*.noarch.rpm "${DISTROOT}/"
			cp ${HOME}/rpmbuild/RPMS/${OS_ARCH}/pgadmin4-python3-mod_wsgi-4.7.1-*.rpm "${DISTROOT}/"
			cp -rf /opt/local/pgsql/lib  "${DISTROOT}/" 
			
			echo "Completed. RPMs created in ${DISTROOT}."
			
* (8) 执行脚本 pkg/redhat/build.sh  

			[postgres@client-180 pgadmin]$ bash pkg/redhat/build.sh  
      
      
