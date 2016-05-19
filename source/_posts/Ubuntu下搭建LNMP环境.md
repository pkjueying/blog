---
title: Ubuntu下搭建LNMP环境
toc: true
comment: true
date: 2014-08-19 00:05:23
categories: [old blog]
tags:
description: Ubuntu 搭建LNMP环境---及过程中所遇见的问题的相关解决方法

---

### 安装MYSQL

下载：
```bash 
sudo wget http://downloads.MySQL.com/archives/mysql-5.0/mysql-5.0.45.tar.gz
```

解压：
```bash 
flzhang@flzhang:~/Downloads/software$ tar zxvf mysql-5.0.45.tar.gz 
```
配置：
```bash         
   #进入源码目录：
   cd mysql-5.0.45
   #键入：
   ./configure --prefix=/usr/local/server/mysql/ --enable-assembler --with-extra-charsets=complex --enable-thread-safe-client --with-big-tables --with-embedded-server --enable-local-infile --with-plugins=innobase
```
在我ubuntu12.0.4中出现了如下错误：
checking for termcap functions library... configure: error: No curses/termcap library found

明显是缺少curses库，因此我们需要安装此库：

```bash
root@flzhang:/home/flzhang/Downloads/mysql-5.0.45$ sudo apt-get install libncurses5-dev
```
库安装成功后，重新执行
```bash
./configure --prefix=/usr/local/server/mysql/ --enable-assembler --with-extra-charsets=complex --enable-thread-safe-client --with-big-tables --with-embedded-server --enable-local-infile --with-plugins=innobase
```
配置成功后如图所示：
![](/img/lnmp/mysql_01.png)

因在上面的配置中，我们将环境全部装在“/usr/local/server”下，所以需要先建立目录。

输入“sudo mkdir /usr/local/server”回车创建目录。并将目录所属者设置成“share2your.info”，
执行命令“sudo chown -R flzhang:flzhang /usr/local/server

接下来就是编译MySQL，在mysql-5.0.45目录下键入“sudo make && make install clean”：

```bash 
flzhang@flzhang:~/Downloads/mysql-5.0.45$ make && make install clean  
```

成功后如图示：
![](/img/lnmp/mysql_02.png)


成功后为了方便就是配置相关的命令了：
首先进入到安装目录：flzhang@flzhang:~/Downloads/mysql-5.0.45$ cd /usr/local/server/mysql
创建配置文件，键入“cp ./share/mysql/my-medium.cnf ./my.cnf”回车。
然后安装默认数据库文件，键入“./bin/mysql_install_db”回车。成功如下所示：

```html
flzhang@flzhang:/usr/local/server/mysql$ ./bin/mysql_install_db   
Installing MySQL system tables...  
OK  
Filling help tables...  
OK  
  
To start mysqld at boot time you have to copy  
support-files/mysql.server to the right place for your system  
  
PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !  
To do so, start the server, then issue the following commands:  
/usr/local/server/mysql//bin/mysqladmin -u root password 'new-password'  
/usr/local/server/mysql//bin/mysqladmin -u root -h flzhang password 'new-password'  
See the manual for more instructions.  
You can start the MySQL daemon with:  
cd /usr/local/server/mysql/ ; /usr/local/server/mysql//bin/mysqld_safe &  
  
You can test the MySQL daemon with mysql-test-run.pl  
cd mysql-test ; perl mysql-test-run.pl  
  
Please report any problems with the /usr/local/server/mysql//bin/mysqlbug script!  
  
The latest information about MySQL is available on the web at  
http://www.mysql.com  
Support MySQL by buying support/licenses at http://shop.mysql.com  
flzhang@flzhang:/usr/local/server/mysql$   
```

对后我们设置服务启动脚本，执行“sudo cp ./share/mysql/mysql.server /etc/init.d/mysql”回车。
再执行“sudo chmod +x /etc/init.d/mysql”回车。
这样我们就可以使用“/etc/init.d/mysql start”及“/etc/init.d/mysql stop”运行和结束mysql服务了。

最后设置数据库root密码，这步要在数据库运行的情况下执行，首先“/etc/init.d/mysql start”启动数据库，然后再执行“./bin/mysqladmin -u root password zfl123”（zfl123是密码，根据实际情况自行设置）回车....
这样，MYSQL就算是安装完成了。



### 安装Nginx服务器
下载：nginx-1.4.0.tar.gz
解压：```bash 
flzhang@flzhang:~/Downloads$ tar zxvf nginx-1.4.0.tar.gz 
```
配置：
```bash
 flzhang@flzhang:~/Downloads/nginx-1.4.0$  ./configure --prefix=/usr/local/server/nginx --with-http_stub_status_module 
 ```

在我的当前环境下出现如下错误：

```html
checking for PCRE library in /opt/local/ ... not found  
  
./configure: error: the HTTP rewrite module requires the PCRE library.  
You can either disable the module by using --without-http_rewrite_module  
option, or install the PCRE library into the system, or build the PCRE library  
statically from the source with nginx by using --with-pcre=<path> option.  
  
flzhang@flzhang:~/Downloads/nginx-1.4.0$ 
```

解决方法：

```bash 
flzhang@flzhang:~/Downloads/nginx-1.4.0$ sudo apt-get install pcre-devel ```


再次执行 
```bash
./configure --prefix=/usr/local/server/nginx --with-http_stub_status_module 
```

又出现下面错误：

```html
./configure: error: the HTTP rewrite module requires the PCRE library.  
You can either disable the module by using --without-http_rewrite_module  
option, or install the PCRE library into the system, or build the PCRE library  
statically from the source with nginx by using --with-pcre=<path> option.  
  
flzhang@flzhang:~/Downloads/nginx-1.4.0$
```

解决方法：
```bash 
flzhang@flzhang:~/Downloads/nginx-1.4.0$ sudo apt-get install libpcre3-dev 
```

再次执行 

```bash
./configure --prefix=/usr/local/server/nginx --with-http_stub_status_module 
```

又出现下面错误：

```html

checking for OpenSSL md5 crypto library ... not found  
  
./configure: error: the HTTP cache module requires md5 functions  
from OpenSSL library.  You can either disable the module by using  
--without-http-cache option, or install the OpenSSL library into the system,  
or build the OpenSSL library statically from the source with nginx by using  
--with-http_ssl_module --with-openssl=<path> options.  
  
flzhang@flzhang:~/Downloadsnginx-1.4.0
```

解决方法：

```bash
flzhang@flzhang:~/Downloads/nginx-1.4.0$ sudo apt-get install libssl-dev libperl-dev 
```

再次执行 

```bash
./configure --prefix=/usr/local/server/nginx --with-http_stub_status_module 
```

呵呵，最终还是成功了：成功如下所示：

```html
checking for struct dirent.d_type ... found  
  
Configuration summary  
  + using system PCRE library  
  + OpenSSL library is not used  
  + md5: using system crypto library  
  + sha1 library is not used  
  + using system zlib library  
  
  nginx path prefix: "/usr/local/server/nginx"  
  nginx binary file: "/usr/local/server/nginx/sbin/nginx"  
  nginx configuration prefix: "/usr/local/server/nginx/conf"  
  nginx configuration file: "/usr/local/server/nginx/conf/nginx.conf"  
  nginx pid file: "/usr/local/server/nginx/logs/nginx.pid"  
  nginx error log file: "/usr/local/server/nginx/logs/error.log"  
  nginx http access log file: "/usr/local/server/nginx/logs/access.log"  
  nginx http client requ  est body temporary files: "client_body_temp"  
  nginx http proxy temporary files: "proxy_temp"  
  nginx http fastcgi temporary files: "fastcgi_temp"  
  
flzhang@flzhang:~/Downloads/nginx-1.4.0
```

编译：flzhang@flzhang:~/Downloads/nginx-1.4.0$ make && make install  哈哈，一步就OK 啦！打印的信息如下：

```html

nginx.old
cp objs/nginx '/usr/local/server/nginx/sbin/nginx'  
test -d '/usr/local/server/nginx/conf'      || mkdir -p '/usr/local/server/nginx/conf'  
cp conf/koi-win '/usr/local/server/nginx/conf'  
cp conf/koi-utf '/usr/local/server/nginx/conf'  
cp conf/win-utf '/usr/local/server/nginx/conf'  
test -f '/usr/local/server/nginx/conf/mime.types'       || cp conf/mime.types '/usr/local/server/nginx/conf'  
cp conf/mime.types '/usr/local/server/nginx/conf/mime.types.default'  
test -f '/usr/local/server/nginx/conf/fastcgi_params'       || cp conf/fastcgi_params '/usr/local/server/nginx/conf'  
cp conf/fastcgi_params      '/usr/local/server/nginx/conf/fastcgi_params.default'  
test -f '/usr/local/server/nginx/conf/fastcgi.conf'         || cp conf/fastcgi.conf '/usr/local/server/nginx/conf'  
cp conf/fastcgi.conf '/usr/local/server/nginx/conf/fastcgi.conf.default'  
test -f '/usr/local/server/nginx/conf/uwsgi_params'         || cp conf/uwsgi_params '/usr/local/server/nginx/conf'  
cp conf/uwsgi_params        '/usr/local/server/nginx/conf/uwsgi_params.default'  
test -f '/usr/local/server/nginx/conf/scgi_params'      || cp conf/scgi_params '/usr/local/server/nginx/conf'  
cp conf/scgi_params         '/usr/local/server/nginx/conf/scgi_params.default'  
test -f '/usr/local/server/nginx/conf/nginx.conf'       || cp conf/nginx.conf '/usr/local/server/nginx/conf/nginx.conf'  
cp conf/nginx.conf '/usr/local/server/nginx/conf/nginx.conf.default'  
test -d '/usr/local/server/nginx/logs'      || mkdir -p '/usr/local/server/nginx/logs'  
test -d '/usr/local/server/nginx/logs' ||       mkdir -p '/usr/local/server/nginx/logs'  
test -d '/usr/local/server/nginx/html'      || cp -R html '/usr/local/server/nginx'  
test -d '/usr/local/server/nginx/logs' ||       mkdir -p '/usr/local/server/nginx/logs'  
make[1]: Leaving directory `/home/flzhang/Downloads/nginx-1.4.0'  
flzhang@flzhang:~/Downloads/nginx-1.4.0$  

```


成功后新建nginx脚本，用来管理nginx服务：
``` bash

#!/bin/bash  
#  
# chkconfig: - 85 15  
# description: Nginx is a World Wide Web server.  
# processname: nginx  
nginx=/usr/local/server/nginx/sbin/nginx  
conf=/usr/local/server/nginx/conf/nginx.conf  
case $1 in  
start)  
echo -n "Starting Nginx"  
$nginx -c $conf  
echo " done"  
;;  
stop)  
echo -n "Stopping Nginx"  
killall -9 nginx  
echo " done"  
;;  
test)  
$nginx -t -c $conf  
;;  
reload)  
echo -n "Reloading Nginx"  
ps auxww | grep nginx | grep master | awk '{print $2}' | xargs kill -HUP  
echo " done"  
;;  
restart)  
$0 stop  
$0 start  
;;  
show)  
ps -aux|grep nginx  
;;  
*)  
echo -n "Usage: $0 {start|restart|reload|stop|test|show}"  
;;  
esac                                                                                                                                                    
~                                                                                                                                                       
:se nonumber  
```


将其放置/etc/init.d/nginx 然后修改其权限，我们就可以用/etc/init.c/nginx start / stop 来开启或关闭nginx了。

PS: 修改端口：conf下的nginx.conf文件中的listen:80  将80改为你所需要的端口就OK 了......



### 安装PHP
下载：```bash 
wget http://cn2.php.net/distributions/php-5.3.8.tar.gz 
```
解压：
```bash 
tar zxvf php-5.3.8.tar.gz 
```
配置： 

```bash 
./configure --prefix=/usr/local/server/php --with-config-file-path=/usr/local/server/php --enable-mbstring --enable-ftp --with-gd --with-jpeg-dir=/usr --with-png-dir=/usr --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-pear --enable-sockets --with-freetype-dir=/usr --enable-gd-native-ttf --with-zlib --with-libxml-dir=/usr --with-xmlrpc --enable-zip --enable-fpm --enable-fpm --enable-xml --enable-sockets --with-gd --with-zlib --with-iconv --enable-zip --with-freetype-dir=/usr/lib/ --enable-soap --enable-pcntl --enable-cli
```

途中出现如下错误：
```html
checking libxml2 install dir... /usr  
checking for xml2-config path...   
configure: error: xml2-config not found. Please check your libxml2 installation.  
flzhang@flzhang:~/Downloads/php-5.3.8$  
```

解决方法：
```bash 
flzhang@flzhang:~/Downloads/php-5.3.8$ sudo apt-get install libxml2-dev ```

重新执行：

```bash
./configure --prefix=/usr/local/server/php --with-config-file-path=/usr/local/server/php --enable-mbstring --enable-ftp --with-gd --with-jpeg-dir=/usr --with-png-dir=/usr --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-pear --enable-sockets --with-freetype-dir=/usr --enable-gd-native-ttf --with-zlib --with-libxml-dir=/usr --with-xmlrpc --enable-zip --enable-fpm --enable-fpm --enable-xml --enable-sockets --with-gd --with-zlib --with-iconv --enable-zip --with-freetype-dir=/usr/lib/ --enable-soap --enable-pcntl --enable-cli
```

成功如下所示：
```html

creating sapi/cli/php.1  
creating sapi/fpm/php-fpm.conf  
creating sapi/fpm/init.d.php-fpm  
creating sapi/fpm/php-fpm.8  
creating main/php_config.h  
creating main/internal_functions.c  
creating main/internal_functions_cli.c  
+--------------------------------------------------------------------+  
| License:                                                           |  
| This software is subject to the PHP License, available in this     |  
| distribution in the file LICENSE.  By continuing this installation |  
| process, you are bound by the terms of this license agreement.     |  
| If you do not agree with the terms of this license, you must abort |  
| the installation process at this point.                            |  
+--------------------------------------------------------------------+  
  
Thank you for using PHP.  
  
flzhang@flzhang:~/Downloads/php-5.3.8$ 
```

安装:
```bash 
flzhang@flzhang:~/Downloads/php-5.3.8$ make && make install
```

直接OK:
```html
Installing helper programs:       /usr/local/server/php/bin/  
  program: phpize  
  program: php-config  
Installing man pages:             /usr/local/server/php/man/man1/  
  page: phpize.1  
  page: php-config.1  
Installing PEAR environment:      /usr/local/server/php/lib/php/  
[PEAR] Archive_Tar    - installed: 1.3.7  
[PEAR] Console_Getopt - installed: 1.3.0  
[PEAR] Structures_Graph- installed: 1.0.4  
[PEAR] XML_Util       - installed: 1.2.1  
[PEAR] PEAR           - installed: 1.9.4  
Wrote PEAR system config file at: /usr/local/server/php/etc/pear.conf  
You may want to add: /usr/local/server/php/lib/php to your php.ini include_path  
/home/flzhang/Downloads/php-5.3.8/build/shtool install -c ext/phar/phar.phar /usr/local/server/php/bin  
ln -s -f /usr/local/server/php/bin/phar.phar /usr/local/server/php/bin/phar  
Installing PDO headers:          /usr/local/server/php/include/php/ext/pdo/  
flzhang@flzhang:~/Downloads/php-5.3.8$   
```

### 配置nginx支持PHP

首先建立存放网页文件的目录，执行“mkdri /usr/local/server/www”。  然后进入到该目录中，“cd /usr/local/server/www”。

建立一index.php
```php
<?php  
phpinfo();
```

修改nginx.conf:如下所示：
```d

flzhang@flzhang:/usr/local/server/nginx$ git diff conf/nginx.conf  
diff --git a/conf/nginx.conf b/conf/nginx.conf  
index f73cd49..4f8c9fd 100644  
--- a/conf/nginx.conf  
+++ b/conf/nginx.conf  
@@ -40,10 +40,13 @@ http {  
   
         #access_log  logs/host.access.log  main;  
   
-        location / {  
-            root   html;  
-            index  index.html index.htm;  
-        }  
+#        location / {  
+#            root   html;  
+#            index  index.html index.htm;  
+#        }  
+#  
+        root /usr/local/server/www;  
+       index index.html index.htm index.php  
   
         #error_page  404              /404.html;  
   
@@ -62,13 +65,12 @@ http {  
   
         # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000  
         #  
-        #location ~ \.php$ {  
-        #    root           html;  
-        #    fastcgi_pass   127.0.0.1:9000;  
-        #    fastcgi_index  index.php;  
-        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;  
-        #    include        fastcgi_params;  
-        #}  
+        location ~ \.php$ {  
+            fastcgi_pass   127.0.0.1:9000;  
+            fastcgi_index  index.php;  
+            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;  
+            include        fastcgi_params;  
:  

```

重启nginx，执行“sudo /etc/init.d/nginx reload”回车，并开启PHP。打开浏览器，输入“http://127.0.0.1：8080”，如下图所示：
![](/img/lnmp/lnmp.png)

好了，至此，关于LNMP的搭建已算完成了..................................