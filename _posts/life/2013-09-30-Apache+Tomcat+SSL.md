---
layout: post
category : study
tags : [apache, tomcat, ssl]
description : Apache+Tomcat+SSL
comments : Apache+Tomcat+SSL
title : Apache+Tomcat+SSL
---
# apache+openssl

## 安装必要的软件

我用的是'apache-http-2.2.25'版，可以直接官方提供的绑定openssl的apache.
文件名是：'httpd-2.2.25-win32-x86-openssl-0.9.8y.msi'
否则单独安装windows下的openssl比较麻烦，要么找到一个第三方的编译结果，要么自己编译

## 生成服务器证书

安装好在bin目录下有一个openssl.exe文件，用来生成证书和密钥。
将openssl.exe添加至环境变量中方便直接通过命令行（CMD）进行命令操作，为了通过命令创建的证书更为清晰的划分，
创建一个单独空目录，使用CMD切换至该目录进行相关操作，以下命令执行目录都在此目录下。

1. 生成服务器用的私钥文件server.key

   执行命令行：'openssl genrsa -out server.key 1024'
注意：有文档指出使用 openssl genrsa -des3 -out server.key 1024 生成私钥文件，这样生成的私钥文件是需要口令的，需要口令该如何配置未验证。这样会出现Apache启动失败，错误提示是：Init: SSLPassPhraseDialog builtin is not supported on Win32 (key file .....)。原因是window下的apache不支持加密的私钥文件。
2). 生成未签署的server.csr
首先将apache安装目录中conf/openssl.cnf复制到命令行目录下
执行命令行：openssl req -new -key server.key -out server.csr -config openssl.cnf
提示输入一系列的参数，
  ......
  Country Name (2 letter code) [AU]:
  State or Province Name (full name) [Some-State]:
  Locality Name (eg, city) []:
  Organization Name (eg, company) [Internet Widgits Pty Ltd]:
  Organizational Unit Name (eg, section) []:
  Common Name (eg, YOUR name) []:
  Email Address []:
  .....
  注：Common Name必须和httpd.conf中ServerName必须一致，否则apache不能启动
  启动apache时错误提示为：RSA server certificate CommonName (CN) `Koda' does NOT match server name!?
3). 签署服务器证书文件server.crt
执行命令行 openssl x509 -req -days 365 -in server.csr -signkey server.key -out  server.crt
以上签署证书仅仅做测试用，真正运行的时候，应该将CSR发送到一个CA返回真正的用书.网上有些文档描述生成证书文件的过程比较繁琐，就是因为他们自己建立了一个CA中心证书，然后再签署server.csr.
用openssl x509 -noout -text -in server.crt可以查看证书的内容。证书实际上包含了Public Key.
3. 配置httpd.conf.
将通过openssl命令生成的server.key和server.crt两个文件复制到apache安装目录中的conf目录下。
在apache安装目录中的conf/extra目录下的httpd-ssl.conf文件是关于ssl的配置，是httpd.conf的一部分。
找到一个443的虚拟主机配置项，如下:
<VirtualHost _default_:443>
   SSLEngine On
   SSLCertificateFile conf/server.crt
   SSLCertificateKeyFile conf/server.key
   #SSLCertificateChainFile conf/ssl.crt/ca.crt // 暂未启用
   #......
   DocumentRoot "C:/programs/Apache2/htdocs"
   ServerName www.my.com:443
</VirtualHost>
1). 看SSLCertificateFile，SSLCertificateKeyFile两个配置项，指定对应的证书文件和证书密钥文件。
2). 看DocumentRoot，ServerName配置项，ServerName修改为任意你想要得域名，注意：前面生成.csr时输入的Common Name必须于这里的ServerName项一致。
     这样启动apache后，访问https://www.my.com将访问C:/programs/Apache2/htdocs目录下的内容。
     但是如果你想保留其他目录的访问仍然是http，那么你应该把
     <VirtualHost _default_:443> 改为 <VirtualHost www.my.com:443>
     此时，即便ServerName是任意的，系统仍然正常运行，仅仅Apache log提示"does NOT match server name"
3). 移除注释行(httpd.conf)
    LoadModule ssl_module modules/mod_ssl.so
二、apache+tomcat
整合 Apache Http Server 和 Tomcat 可以提升对静态文件的处理性能、利用 Web 服务器来做负载均衡以及容错、无缝的升级应用程序。本文介绍了三种整合 Apache 和 Tomcat 的方式。
首先我们先介绍一下为什么要让 Apache 与 Tomcat 之间进行连接。事实上 Tomcat 本身已经提供了 HTTP 服务，该服务默认的端口是 8080，装好 tomcat 后通过 8080 端口可以直接使用 Tomcat 所运行的应用程序，你也可以将该端口改为 80。既然 Tomcat 本身已经可以提供这样的服务，我们为什么还要引入 Apache 或者其他的一些专门的 HTTP 服务器呢？原因有下面几个：
1. 提升对静态文件的处理性能
2. 利用 Web 服务器来做负载均衡以及容错
3. 无缝的升级应用程序
这三点对一个 web 网站来说是非常之重要的，我们希望我们的网站不仅是速度快，而且要稳定，不能因为某个 Tomcat 宕机或者是升级程序导致用户访问不了，而能完成这几个功能的、最好的 HTTP 服务器也就只有 apache 的 http server 了，它跟 tomcat 的结合是最紧密和可靠的。
通常都是通过JK_MOD来整合Apache和Tomcat，但是Apache2.2版本以上整合Tomcat可以直接通过AJP_PROXY来完成，很方便。下面把几种方式都简单讲讲。
假设一个Apache，两个Tomcat容器，访问 a.hackang.cn 和 b.hackang.cn 分别对应 tomcata 和 tomcatb 的应用
第一种方式：JK_PROXY
安装好Apache和Tomcat，下载mod_jk-1.2.26-httpd-2.2.4.so （2.2.4对应着Apache版本）
将mod_jk-1.2.26-httpd-2.2.4.so 放到Apache安装目录的modules文件夹下。
在Apache安装目录的conf文件夹创建workers.properties配置文件，内容如下:
#下面是Tomcat实例列表
worker.list=tomcata,tomcatb
#tomcata实例配置
worker.tomcata.host=127.0.0.1
worker.tomcata.port=8009
worker.tomcata.type=ajp13
#tomcatb实例配置
worker.tomcatb.host=127.0.0.1
worker.tomcatb.port=9009
worker.tomcatb.type=ajp13
编辑apache配置文件httpd.conf，在文件末尾加上以下内容：
#以下为tomcat集成配置部分
LoadModule jk_module modules/mod_jk-1.2.26-httpd-2.2.4.so
JkWorkersFile conf/workers.properties
JkLogFile logs/mod_jk.log
JkLogLevel info
#如果机器有多个IP地址请务必使用*号
NameVirtualHost *:80

#a.hackang.cn虚拟站点
<VirtualHost *:80>
ServerName a.hackang.cn
JkMount /*.* tomcata
JkMount /* tomcata
DirectoryIndex index.jsp
</VirtualHost>
#b.hackang.cn虚拟站点
<VirtualHost *:80>
ServerName b.hackang.cn
JkMount /*.* tomcatb
DirectoryIndex index.jsp
</VirtualHost>
下面是Tomcat的配置，很重要。
tomcata可以使用默认配置，如果想访问 a.hackang.cn直接显示某应用的首页，可在tomcata的配置文件server.xml里面的host节点间加上
<Context className="org.apache.catalina.core.StandardContext" cachingAllowed="true"
charsetMapperClass="org.apache.catalina.util.CharsetMapper" cookies="true" crossContext="false" debug="0" displayName="a.hackang.cn" docBase="E:/myweb/a"
       mapperClass="org.apache.catalina.core.StandardContextMapper" path=""  privileged="false" reloadable="false" swallowOutput="false" useNaming="true"
       wrapperClass="org.apache.catalina.core.StandardWrapper">
</Context>
docBase指向的你应用所在的文件夹，不能将此应用部署到tomcata的webapps文件夹中。否则就有两个应用了，一个是根访问路径，一个是根访问路径+应用名了。
tomcatb的配置要稍加修改,修改 conf/server.xml文件
<Server port="8005" shutdown="SHUTDOWN">将此处的端口号改掉，不能与tomcata的相同，比如可以改成 9005
修改默认的8080端口为9090，修改后如下：
<Connector port="9090" maxHttpHeaderSize="8192"
             maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
             enableLookups="false" redirectPort="8443" acceptCount="100"
             connectionTimeout="20000" disableUploadTimeout="true" />

修改端口号为8009的Connector
修改前为：
<Connector port="8009" enableLookups="false" redirectPort="8443" protocol="AJP/1.3" />
修改后：
<Connector port="9009" enableLookups="false" redirectPort="8443" protocol="AJP/1.3" />
此处的9009跟workers.properties文件中tomcatb的端口号是一致的。
如果也想访问 b.hackang.cn时直接显示应用b，配置方法同a，以上已经提及，只需将docBase="E:/myweb/a" 改成 docBase="E:/myweb/b"即可
最后编辑C:/WINDOWS/system32/drivers/etc/hosts文件，在最后加上两个映射
  127.0.0.1  a.hackang.cn
  127.0.0.1  b.hackang.cn

至此，配置就结束了，可以用Apache的Test Configuration命令测试一下配置文件，如果没有问题，启动Apache，再分别启动两个Tomcat就ok了
第二种方式配置： ajp
apache2.2以上版本，无需使用jk_mod来集成tomcat，直接使用ajp，很方便。
修改apache配置文件httpd.conf
启用mod_proxy_ajp
#LoadModule proxy_module modules/mod_proxy.so
#LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
把这两行前面的#去掉即可
然后在末尾加上
<VirtualHost *:80>
ProxyPass / ajp://127.0.0.1:8009/
ProxyPassReverse / ajp://127.0.0.1:8009/
ServerName a.hackang.cn
</VirtualHost>
<VirtualHost *:80>
ProxyPass / ajp://127.0.0.1:9009/
ProxyPassReverse / ajp://127.0.0.1:9009/
ServerName b.hackang.cn
</VirtualHost>
搞定！！！方便吧

第三种方式
第三种方式其实跟第二种差不多，只不过用的是http端口
<VirtualHost *:80>
ProxyPass / http://127.0.0.1:8080/
ProxyPassReverse / http://127.0.0.1:8080/
ServerName a.hackang.cn
</VirtualHost>
<VirtualHost *:80>
ProxyPass / http://127.0.0.1:9090/
ProxyPassReverse / http://127.0.0.1:9090/
ServerName b.hackang.cn
</VirtualHost>
此处的9090跟tomcatb中配置的http端口一致

到此Apache整合Tomcat全部结束，若要加强tomcat处理静态资源的能力，可以启用APR服务

采用 proxy 的连接方式，需要在 Apache 上加载所需的模块，mod_proxy 相关的模块有 mod_proxy.so、mod_proxy_connect.so、mod_proxy_http.so、mod_proxy_ftp.so、mod_proxy_ajp.so， 其中 mod_proxy_ajp.so 只在 Apache 2.2.x 中才有。如果是采用 http_proxy 方式则需要加载 mod_proxy.so 和 mod_proxy_http.so；如果是 ajp_proxy 则需要加载 mod_proxy.so 和 mod_proxy_ajp.so这两个模块。

相对于 JK 的连接方式，后两种在配置上是比较简单的，灵活性方面也一点都不逊色。但就稳定性而言就不像 JK 这样久经考验，毕竟 Apache 2.2.3 推出的时间并不长，采用这种连接方式的网站还不多，因此，如果是应用于关键的互联网网站，还是建议采用 JK 的连接方式。

  下面是我自己用的配置,采用的就是第一种方式jk集成

将mod_jk-1.2.26-httpd-2.2.4.so 放到Apache安装目录的modules文件夹下。
在Apache安装目录的conf文件夹创建workers.properties配置文件，内容如下：

workers.tomcat_home=D:/Program Files/Tomcat 5.5  #让mod_jk模块知道Tomcat
workers.java_home=C:/Program Files/Java/jdk1.5.0_05 #让mod_jk模块知道j2sdk
ps=/
worker.list=ajp13  #模块版本,现有ajp14了,不要修改
worker.ajp13.port=8009  #工作端口,若没占用则不用修改
worker.ajp13.host=localhost  #本机,若上面的Apache主机不为localhost,作相应修改
worker.ajp13.type=ajp13  #类型
worker.ajp13.lbfactor=1  #代理数,不用修改

 修改httpd.conf，内容示下：

LoadModule   jk_module   modules/mod_jk.so
DocumentRoot "E:/website/Tomcat 5.5/webapps/ROOT"
<IfModule   jk_module>
JkWorkersFile   "D:/Program Files/Apache2.2/conf/workers.properties"
JkAutoAlias   "E:/website/Tomcat 5.5/webapps/ROOT"
JkMount /servlet/* ajp13#下面这些都是交给tomcat管理
JkMount /*.jsp ajp13
JkMount /*.do ajp13
</IfModule>

#下面这些配置源文件中已存在，只需根据自己情况作相应修改
<Directory>
 Options Indexes MultiViews
 AddOutputFilter Includes html
 AllowOverride None
 Order allow,deny
 Allow from all
#默认会是deny from all,有时访问会出现You don't have permission to access / on this server.
</Directory>
Listen localhost:81
ServerName localhost:81 #IP:端口
ServerAdmin kongdebing86@hotmail.com
<IfModule dir_module>
    DirectoryIndex index.html index.htm index.jsp default.html default.htm default.jsp#没有index.html它会去找index.jsp
</IfModule>

ok，到此apache配置完成

三、apache+tomcat+ssl

要想将apache将ssl请求转发给tomcat需要对httpd-ssl.conf进行配置，步骤2中是对http进行配置相应的转发，因此方式类似，这里采用了步骤2中的方式二进行配置。
修改conf/extra/httpd-ssl.conf 文件
在<VirtualHost _default_:443>中插入如下代码
ProxyPass / ajp://127.0.0.1:8009/
ProxyPassReverse / ajp://127.0.0.1:8009/

四.结尾
至此整个过程配置完成，首先启动tomcat然后启动apache，访问http://localhost:80/和https://localhost:443/进行测试。


