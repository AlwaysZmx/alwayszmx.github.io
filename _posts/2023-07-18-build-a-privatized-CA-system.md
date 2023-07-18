---
layout: post
title: 基于ejbca搭建私有化CA系统
categories: [SSL]
description: 基于ejbca搭建私有化CA系统
keywords: SSL, CA, 数字证书
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
copyright: false
---

# 一、前言
让我们从http与https说起。http是超文本传输协议（HyperText Transfer Protocol）的缩写，也是互联网的基石，互联网绝大部分数据传输都是基于http协议，但是，http本身有个致命的缺点，一个早期并不引人注目，如今不得不重视的缺点，就是明文传输。

当你访问一个http站点，例如http://www.login.com/

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNzAxLzc5NzkzMC0yMDE3MDEyMzExMzc0NjIwNi03NjAxNTQ1NTcucG5n.jpg)

输入用户名与密码，例如admin/123456，点击 Sign in 按钮登录，数据将被发往服务器，期间登录的个人数据 admin/123456 仍以明文形式在网络上传输，如果有人抓到这个数据包，你的账号密码就有泄漏的风险。今天看到一则新闻，google chrome 56将把http站点要求输入账户密码的情况标记为不安全，相信未来http将成为历史。

https很好的解决了这个问题，以www.baidu.com为例，早期使用http://www.biadu.com，如今已更换为https://www.baidu.com，访问https://www.baidu.com可以一把绿色的锁，点击就出现了如下信息，此时，与服务器之间的通信连接是加密的，即使被他人截获数据包，也很难知道传输的内容

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNzAxLzc5NzkzMC0yMDE3MDEyNDE1NTU1NDg3OC0xNDE5NTg0MzYxLnBuZw==.jpg)

https与http有啥区别？除了多一个 s 其实没啥区别（开玩笑），这个 s 至关重要，使用https必须在服务器上安装数字证书，上图中点击 查看证书 就可以看到服务器证书信息

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNzAxLzc5NzkzMC0yMDE3MDEyNDE2MDczMzcyMi0xODEzMTE5NzQ2LnBuZw==.jpg)

有了数字证书，https还能实现另一个非常重要的功能：身份验证。还是以www.baidu.com为例，早期的http://www.biadu.com没有数字证书，所以可能会出现这样的情况，你的网关被人劫持，所有你发往www.baidu.com的信息都被人截获，同时，他对你的请求作出响应，以此冒充真正的www.baidu.com，也就是说，你访问的www.baidu.com并不是真正的www.baidu.com，而是一个代理网站，它的行为表现与www.baidu.com很像。

https://www.baidu.com就不会出现这种情况了，因为它需要数字证书，而数字证书在全球范围内都是唯一并不可伪造的

# 二、密码学基础

理解数字证书需要先了解对称加密与非对称加密，我们从普通加密通信模型开始

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE2MjI1OTQ1OS0xMDMzMzc4MDA1LnBuZw==.jpg)
我向朋友发送一条消息 有时间一起吃饭吗 ，实际在网络中传输的是加密后的数据，不是原文，只是加密和解密过程对我和朋友是透明的

- 加密算法

上述通信过程中，如果加密与解密使用相同的密钥，称为对称加密，如果使用不同的密钥，称为非对称加密。

对称加密的特点是速度快，可加密的数据量大；非对称加密的特点是速度慢，因此只用来加密少量数据，但极难破解。

非对称加密过程中使用的两个不同的密钥，一个称为 私钥 ，一个称为 公钥 ，它们一一对应，称为 密钥对 ， 公钥 可以给任何人使用，但 私钥 必须自己保持，一旦 私钥 泄露，这个密钥对就应该被抛弃。

密钥对有两个非常重要的特点：
        1、 私钥 可以导出 公钥 ，但 公钥 无法导出 私钥 ；
        2、经 私钥 加密的内容只能由 公钥 解密，经 公钥 加密的内容也只能由 私钥 解密。

这两个特点有两个重要用途：
        1、 私钥 持有者用 私钥 加密内容，发送给 公钥 持有者解密，验证 私钥 持有者的身份。因为 公钥 能解密的内容，只能是由 私钥 加密的；
        2、 公钥 持有者用 公钥 加密内容，发送给 私钥 持有者解密，保证内容安全。因为只有 私钥 能解密，即使内容被截获，截获者也无法知道内容是什么


- 摘要与签名算法

数字证书是一个文件，包含了使用者的身份信息、以及权威机构（CA）的数字签名，就向我们的居民身份证一样，数字证书是 网络身份证 ，用于验证互联网上证书持有者的身份

颁发数字证书的机构叫CA，全世界只有少数权威的CA，因为颁发出去的证书它们是要负法律责任的，所以向它们申请证书也要缴费，我们自己搭建的ejbca颁发的证书只适合在企业内部使用，这个CA是没有权利向互联网上其他商业公司颁发证书的

有了数字证书，只要配置服务开启https就可以使用数字证书了

https对于数字证书的认证包括 单向认证 与 双向认证 ， 单向认证 只由客户端验证服务器， 双向认证 则两者相互验证，只要验证不通过，通信就自动中断，下面介绍通信的流程以及如何在tomcat中配置

# 三、数字证书

那么，数字证书从何而来？数字证书由专门的机构颁发，这种机构称为CA（Certificate Authority），ejbca则是CA的一种java实现。

# 四、https原理
## https单向认证

单向认证的两个实体：

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE2NDMxODE3Ny04MTUyNjYzOTAucG5n.jpg)

单向认证的流程如下图：

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE2NDIyMTI3MS0xNjM2MDU0MDM1LnBuZw==.jpg)

简要说明如下：

+ 客户端访问服务器
+ 服务器响应客户端，发送服务器证书给客户端
+ 客户端查询 受信任根证书颁发机构 ，验证服务器证书
+ 客户端验证完服务器证书，生成 密钥对 及会话密钥，与服务器协商会话密钥
+ 会话密钥协商完成，开始安全加密通信

tomcat配置https单向认证

```xml-dtd
 Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
 maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
 clientAuth="false" sslProtocol="TLS"
 keystoreFile="D:\tomcat.jks" keystorePass="123456" / 
```

nginx配置https单向认证

```nginx
server {
    listen 443 ssl;
    server_name localhost;
    ssl_certificate cert.pem; #服务器证书公钥部分
    ssl_certificate_key cert.key; #服务器证书私钥
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 5m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    location / {
        proxy_pass http://127.0.0.1:4000; #代理tomcat的http服务
    }
}
```

## https双向认证

双向认证的两个实体：

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE2NDUyNzUyMS00MDQ4MTMxMDQucG5n.jpg)

双向认证的流程如下图：

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE2NDYwMzI0MC04NDc3MDgxNTIucG5n.jpg)

简要说明如下：

+ 客户端访问服务器
+ 服务器响应客户端，发送服务器证书给客户端
+ 客户端查询 受信任根证书颁发机构 ，验证服务器证书
+ 验证完服务器证书，客户端发送客户端证书给服务器
+ 服务器查询 信任库 或通过证书链，验证客户端证书
+ 客户端与服务器协商会话密钥加密方案
+ 客户端与服务器协商会话密钥
+ 会话密钥协商完成，开始安全加密通信

tomcat配置https双向认证

```xml-dtd
 Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
 maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
 clientAuth="true" sslProtocol="TLS"
 keystoreFile="D:\tomcat.jks" keystorePass="123456"
 truststoreFile="D:\tomcat.jks" truststorePass="123456"/ 
```

nginx配置https双向认证

```nginx
server {
     listen 443 ssl;
     server_name localhost;
     ssl_certificate cert.pem; #服务器证书公钥部分
     ssl_certificate_key cert.key; #服务器证书私钥
     ssl_verify_client on; #开启浏览器认证
     ssl_client_certificate ca.pem; #CA根证书
     ssl_session_cache shared:SSL:1m;
     ssl_session_timeout 5m;
     ssl_ciphers HIGH:!aNULL:!MD5;
     ssl_prefer_server_ciphers on;
     location / {
     	proxy_pass http://127.0.0.1:4000; #代理tomcat的http服务
     	proxy_set_header client-cert $ssl_client_cert; #将浏览器证书传递给tomcat，tomcat通过header拿到证书
     }
}
```

# 五、linux环境下安装ejbca
#### 1、安装jdk

检查已安装jdk，如果有，先删除

```shell
rpm -qa|grep java
rpm -e --nodeps filename
```

从下载jdk安装包：jdk-8u111-linux-x64.rpm，放在/usr/file目录

```shell
rpm -ivh jdk-8u111-linux-x64.rpm
```

默认安装在/usr/java目录，安装完后配置环境变量

```shell
vi /etc/profile
```

追加如下内容

```shell
#java conf
export JAVA_HOME=/usr/java/jdk1.8.0_111
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=:$JAVA_HOME/lib
```

使配置立即生效，然后检查安装结果

```shell
source /etc/profile
java -version
```

#### 2、安装ant

从下载ant安装包：apache-ant-1.9.7-bin.tar.gz，解压

```shell
tar xvf apache-ant-1.9.7-bin.tar.gz -C /usr/java/
```

配置环境变量

```shell
vi /etc/profile
```

追加如下内容

```shell
#ant conf
export ANT_HOME=/usr/java/apache-ant-1.9.7
export PATH=$PATH:$ANT_HOME/bin
```

使配置立即生效，然后检查安装结果

```shell
source /etc/profile
ant -version
```

#### 3、安装mysql

安装前查看已安装的mysql或mysql库，并删除它们

```shell
rpm -qa|grep -i mysql
rpm -e --nodeps filename
```

如果重装mysql，查找安装mysql产生的文件，并删除它们

```shell
find / -name mysql
rm -fr filename
```

安装mysql所需的perl依赖

```shell
yum install -y perl-Module-Install.noarch
```

从下载Red Hat Enterprise Linux 6 / Oracle Linux 6 (x86, 64-bit)对应5.6版本的安装包：MySQL-client-5.6.33-1.el6.x86_64.rpm、MySQL-server-5.6.33-1.el6.x86_64.rpm，依次安装

```shell
rpm -ivh MySQL-server-5.6.33-1.el6.x86_64.rpm
rpm -ivh MySQL-client-5.6.33-1.el6.x86_64.rpm
```

安装完成，启动mysql服务，并设置为开机启动

```shell
service mysql start
chkconfig mysql on
```

mysql安装完成后为root账户生成随机密码，位于/root目录下的.mysql_secret文件中，用这个密码登录root

```shell
cat /root/.mysql_secret
mysql -uroot -pLTHVNyKmmg7jwiLk
```

修改root密码并配置远程访问

```shell
set password=password('root');
grant all privileges on *.* to 'root'@'%' identified by 'root';
```

#### 4、安装jboss

从下载jboss安装包：jboss-as-7.1.1.Final.tar.gz，解压并配置环境变量

```shell
tar xvf jboss-as-7.1.1.Final.tar.gz -C /usr/java
vi /etc/profile
```

追加内容

```shell
#jboss conf
export JBOSS_HOME=/usr/java/jboss-as-7.1.1.Final
```

使配置立即生效

```shell
source /etc/profile
```

启动jboss，注意最后的 符号，待启动完成再运行 exit

```shell
sh /usr/java/jboss-as-7.1.1.Final/bin/standalone.sh 
```

这样jboss就运行在后台了，以下命令查看jboss进程并关闭

```shell
ps -ef|grep jboss
kill -9 进程号
```

#### 5、配置jboss的mysql数据源

创建目录，然后在该目录下创建module.xml

```shell
mkdir -p /usr/java/jboss-as-7.1.1.Final/modules/com/mysql/main
cd /usr/java/jboss-as-7.1.1.Final/modules/com/mysql/main
vi module.xml
```

module.xml内容如下

```xml-dtd
 ?xml version="1.0" encoding="UTF-8"? 
 module xmlns="urn:jboss:module:1.0" name="com.mysql" 
 resources 
 resource-root path="mysql-connector-java-5.1.27.jar"/ 
 /resources 
 dependencies 
 module name="javax.api"/ 
 module name="javax.transaction.api"/ 
 /dependencies 
 /module 
```

下载mysql的驱动包，放在/usr/file目录，然后拷贝到当前目录

```shell
cp /usr/file/mysql-connector-java-5.1.27.jar ./
```

打开新的shell窗口，运行

```shell
sh /usr/java/jboss-as-7.1.1.Final/bin/jboss-cli.sh -c
```

如果是 disconnect 状态，先输入 connect ，多回车几次后，运行下面命令

```shell
/subsystem=datasources/jdbc-driver=com.mysql.jdbc.Driver:add(driver-name=com.mysql.jdbc.Driver,driver-class-name=com.mysql.jdbc.Driver,driver-module-name=com.mysql,driver-xa-datasource-class-name=com.mysql.jdbc.jdbc.jdbc2.optional.MysqlXADataSource)
:reload
```

#### 6、配置ejbca

从下载ejbca安装包：ejbca_ce_6_3_1_1.zip，放在/usr/file目录，解压，准备修改配置

```shell
unzip /usr/file/ejbca_ce_6_3_1_1.zip -d /usr/java
cd /usr/java
mv ejbca_ce_6_3_1_1 ejbca-ce-6.3.1.1
cd /usr/java/ejbca-ce-6.3.1.1/conf/
```

1、修改ejbca.properties

```shell
mv ejbca.properties.sample ejbca.properties
vi ejbca.properties
```

修改如下内容

```properties
appserver.home=/usr/java/jboss-as-7.1.1.Final
appserver.type=jboss
```

2、修改database.properties

```shell
mv database.properties.sample database.properties
vi database.properties
```

修改如下内容

```properties
# dataSource
datasource.jndi-name=jboss/datasources/MySqlDS
# mysql info
database.name=mysql
database.url=jdbc:mysql://127.0.0.1:3306/ejbca?characterEncoding=UTF-8
database.driver=com.mysql.jdbc.Driver
database.username=root
database.password=root
```

3、修改install.properties

```shell
mv install.properties.sample install.properties
vi install.properties
```

修改如下内容

```properties
#设置ca名称
ca.name=test
#设置ca信息
ca.dn=CN=test,O=test,C=cn
```

4、修改cesecore.properties、jaxws.properties，不需要修改内容

```shell
mv cesecore.properties.sample cesecore.properties
mv jaxws.properties.sample jaxws.properties
```

5、修改web.properties

```shell
mv web.properties.sample web.properties
vi web.properties
```

修改如下内容

```properties
superadmin.password=ejbca
superadmin.cn=superadmin
httpsserver.hostname=ca.test.com
httpsserver.dn=CN=${httpsserver.hostname},O=test,C=cn
```

#### 7、部署ejbca到jboss

首先，在配置的mysql中创建 ejbca 数据库，编码 utf-8 ，然后正式用ant构建ejbca并安装到jboss

```shell
cd /usr/java/ejbca-ce-6.3.1.1
ant clean deploy
ant install
ant deploy-keystore
```

deploy用ant部署，install生成证书，deploy-keystore将证书部署到jboss，前两步所需时间较长，过程中如需输入，请直接回车

#### 8、配置jboss的https

打开新的shell窗口，运行

```shell
sh /usr/java/jboss-as-7.1.1.Final/bin/jboss-cli.sh -c
```

如果是 disconnect 状态，运行 connect ，多回车几次，准备运行下面4部分配置

第一部分（配置任意主机可访问）

```shell
/interface=http:add(inet-address="0.0.0.0")
/interface=httpspub:add(inet-address="0.0.0.0")
/interface=httpspriv:add(inet-address="0.0.0.0")
/socket-binding-group=standard-sockets/socket-binding=http:add(port="8080",interface="http")
/subsystem=undertow/server=default-server/http-listener=http:add(socket-binding=http)
/subsystem=undertow/server=default-server/http-listener=http:write-attribute(name=redirect-socket, value="httpspriv")
:reload
```

第二部分（配置证书）

```shell
/core-service=management/security-realm=SSLRealm:add()
/core-service=management/security-realm=SSLRealm/server-identity=ssl:add(keystore-path="${jboss.server.config.dir}/keystore/keystore.jks", keystore-password="serverpwd", alias="prod-ica1")
/core-service=management/security-realm=SSLRealm/authentication=truststore:add(keystore-path="${jboss.server.config.dir}/keystore/truststore.jks", keystore-password="changeit")
/socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")
/socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442", interface="httpspub")
:reload
```

第三部分（配置ssl）

```shell
/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding=httpspriv, security-realm="SSLRealm", verify-client=REQUIRED)
/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding=httpspub, security-realm="SSLRealm")
:reload
```

第四部分（配置web service）

```shell
/system-property=org.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH:add(value=true)
/system-property=org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH:add(value=true)
/system-property=org.apache.catalina.connector.URI_ENCODING:add(value="UTF-8")
/system-property=org.apache.catalina.connector.USE_BODY_ENCODING_FOR_QUERY_STRING:add(value=true)
/subsystem=webservices:write-attribute(name=wsdl-host, value=jbossws.undefined.host)
/subsystem=webservices:write-attribute(name=modify-wsdl-address, value=true)
:reload
```

#### 9、浏览器访问

拷贝ejbca根目录下的p12目录下的superadmin.p12文件到windows，推荐xftp工具，或者访问

```
http://172.17.210.124:8080/ejbca/doc/superadmin.p12
```

下载该文件到windows，双击开始安装，密码 ejbca ，安装完，访问

```
https://172.17.210.124:8443/ejbca/adminweb
```

也就是ejbca的管理后台

# 六、使用ejbca管理数字证书

假设安装ejbca系统的服务器地址为

```
172.17.210.124
```

编辑windows的 C:\Windows\System32\drivers\etc 下的hosts文件，加入一行

```
172.17.210.124 ca.xmyself.com
```

ejbca提供了两个操作界面

管理员界面（需要superadmin证书）

```
https://ca.xmyself.com:8443/ejbca/adminweb/
```

用户界面（不需要证书）

```
http://ca.xmyself.com:8080/ejbca/
```

1、注册用户

ejbca管理员界面，打开 RA Functions Add End Entity 菜单，填写以下 Required 列打勾的项。

用户模板选择 EMPTY

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTUwNDAzNy0xMjkxODg3ODQ4LnBuZw==.jpg)

输入用户名与密码

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTUyMDcyNC0yMDk2MjIwMzk4LnBuZw==.jpg)

Common name，如果是服务器用证书，这里请填写域名

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTU0MTk5MC0xNjA2NDUzNDI4LnBuZw==.jpg)

填写证书信息，证书模板选择 ENDUSER ，CA选择 dev ，Token选择 P12 file

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTU1ODkyNy0xMzMxNTQ1NDQyLnBuZw==.jpg)

点击 Add 注册

2、下载证书

ejbca用户界面，打开 Enroll Create Browser Certificate 菜单

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTcxMjk0My0xMDg1NTExNjEzLnBuZw==.jpg)

输入用户名和密码，点击 OK 按钮

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTcyNjU2OC0xODU1ODgzMjk4LnBuZw==.jpg)

Key length 选择 2048 bits ； Certificate profile 选择 ENDUSER ，点击 Enroll 按钮下载证书

3、吊销证书

ejbca管理员界面，打开 RA Functions Search End Entities 菜单。 Search end entities with status 处下拉框选择 All ，点击右边的 Search 按钮查看用户信息（下图省略其他列）

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTg0NzgzNC0xOTgzMDg2NDAyLnBuZw==.jpg)

勾选需要吊销的用户，点击表格下方的 Revoke Selected 按钮

4、更新证书

ejbca管理员界面，打开 RA Functions Search End Entities 菜单。 Search end entities with status 处下拉框选择 All ，点击右边的 Search 按钮查看用户信息（下图省略其他列）

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDEzNTk0OTI3MS0yMTEzMzkyMzM3LnBuZw==.jpg)

点击需要更新证书用户的最右边列中的 Edit End Entity 超链接，编辑用户

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE0MDAwNDMzNC0yMTQ2OTQ5NTgwLnBuZw==.jpg)

设置 Status 为 New ，点击右边的 Save 按钮。然后输入新密码，其他项保持不变，点击页面最下方的 Save 按钮保存设置

5、根证书

ejbca用户界面，打开 Retrieve Fetch CA Certificates 菜单，可以下载不同格式的根证书

6、tomcat服务器证书

用户注册时，证书模板选择 SERVER ，CA选择 dev ，Token选择 JKS file ，其他项不变

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE0MDM0NjU2OC0yMDI3Mjk0ODEyLnBuZw==.jpg)

下载证书时，在ejbca用户界面中，打开 Enroll Create Keystore 菜单，输入用户名与密码，进入下面的页面

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE0MDQwMzQ1OS0xNjc3NzIyNS5wbmc=.jpg)

Key length 选择 2048 bits ； Certificate profile 选择 SERVER ，点击 Enroll 按钮下载证书

# 七、调用ejbca接口实现私有CA

ejbca系统装好了，可以管理数字证书了，但是，所有操作都在ejbca界面中执行，先不说全是英文，单单它里面配置项就多的人眼花缭乱，很多配置项要么是固定的，要么是不需要的，因此，最合理的做法是在ejbca之上构建中间层，用户访问中间层提供的服务，中间层调用ejbca的web service interface

1、superadmin.jks证书

ejbca提供的web service需要证书认证，证书格式：jks。对superadmin用户执行更新操作，修改 Token 值为 JKS file

![img](http://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltYWdlczIwMTUuY25ibG9ncy5jb20vYmxvZy83OTc5MzAvMjAxNjEyLzc5NzkzMC0yMDE2MTIwNDE0MTMzODA1Mi04ODIzNjQzNjEucG5n.jpg)

保存，然后按照下载普通用户证书的步骤下载superadmin的证书

2、初始化web service连接

将web service所需的jar包添加到工程中，这些jar包是下面两个目录下的所有jar

```shell
/usr/java/ejbca-ce-6.3.1.1/dist/ejbca-ws-cli/lib
/usr/java/ejbca-ce-6.3.1.1/dist/ejbca-ws-cli
```

初始化web service

```java
public void init() {
     if (!new File(certPath).exists()) return;
     CryptoProviderTools.installBCProvider();
     System.setProperty("javax.net.ssl.trustStore", "d:/superadmin.jks");
     System.setProperty("javax.net.ssl.trustStorePassword", "123456");
     System.setProperty("javax.net.ssl.keyStore", "d:/superadmin.jks");
     System.setProperty("javax.net.ssl.keyStorePassword", "123456");
     QName qname = new QName("http://ws.protocol.core.ejbca.org/", "EjbcaWSService");
     try {
         EjbcaWSService service = new EjbcaWSService(new URL("https://ca.xmyself.com:8443/ejbca/ejbcaws/ejbcaws?wsdl"), qname);
         EjbcaWS ejbcaWS = service.getEjbcaWSPort();
     } catch (Exception e) {
         
     }
}
```

注意：连接地址只能是域名，因此，连接ejbca的web service的机器要配置hosts

```
172.17.210.124 ca.xmyself.com
```

3、数字证书管理

查看用户是否已经注册

```java
private boolean isExist(String username) throws Exception {
     UserMatch usermatch = new UserMatch();
     usermatch.setMatchwith(UserMatch.MATCH_WITH_USERNAME);
     usermatch.setMatchtype(UserMatch.MATCH_TYPE_EQUALS);
     usermatch.setMatchvalue(username);
     try {
         List UserDataVOWS users = ejbcaWS.findUser(usermatch);
         if (users != null && users.size() > 0) {
         	return true;
         } else {
            return false;
         }
     } catch (Exception e) {
     	throw new Exception("检查用户 " + username() + " 是否存在时出错：" + e.getMessage());
     }
}
```

用户注册与更新，用的都是editUser()方法，因此要先判断是否存在

```java
public void editUser() throws Exception {
     UserDataVOWS userData = new UserDataVOWS();
     userData.setUsername("testname");//用户名
     userData.setPassword("123456");//密码
     userData.setClearPwd(false);//默认
     userData.setSubjectDN("CN=" + "testname"
     + ",OU=" + "testou"
     + ",O=" + "testo"
     + ",C=cn"
     + ",telephoneNumber=" + "1234567890"
     );//设置唯一甄别名
     String pattern = "yyyy-MM-dd HH:mm:ssZZ"; // ISO 8601标准时间格式
     userData.setStartTime(DateFormatUtils.format(new Date(),pattern));//证书有效起始日期
     userData.setEndTime(DateFormatUtils.format(DateUtils.addDays(new Date(), 100), pattern));//结束日期
     userData.setCaName("test");//ca名称，ejbca的名称
     userData.setSubjectAltName(null);
     userData.setEmail("test@test.com");//邮件地址
     userData.setStatus(UserDataVOWS.STATUS_NEW);//状态为new
     userData.setTokenType(UserDataVOWS.TOKEN_TYPE_P12);//设置p12格式证书
     userData.setEndEntityProfileName("user");//终端实体模板
     userData.setCertificateProfileName("user");//证书模板
     try {
     	ejbcaWS.editUser(userData);
     } catch (Exception e) {
     	throw new Exception(e.getMessage());
     }
}
```

代码中有几处值得注意的，终端实体模板 user 和证书模板 user 需要在ejbca管理员界面中配置，并且终端实体模板 user 中要配置开启 SubjectDN 的属性如CN、OU、O、C、telephoneNumber等，还要允许修改startTime和endTime

吊销证书

```java
public void revoke(String username) throws ServiceException {
    try {
        ejbcaWS.revokeUser(username, RevokedCertInfo.REVOCATION_REASON_UNSPECIFIED, false);
    } catch (Exception e) {
        
    }
}
```

创建证书

```java
private void createCert(String username, String password, String path) throws Exception {
     FileOutputStream fileOutputStream = null;
     try {
         // 创建证书文件
         KeyStore ksenv = ejbcaWS.pkcs12Req(username, password, null, "2048", AlgorithmConstants.KEYALGORITHM_RSA);
         java.security.KeyStore ks = KeyStoreHelper.getKeyStore(ksenv.getKeystoreData(), "PKCS12", password);
         fileOutputStream = new FileOutputStream(path + File.separator + username + ".p12");
         ks.store(fileOutputStream, password.toCharArray());
         // 创建密码文件
         File pwdFile = new File(path + File.separator + username + ".pwd");
         pwdFile.createNewFile();
         BufferedWriter out = new BufferedWriter(new FileWriter(pwdFile));
         out.write(password);
         out.flush();
         out.close();
     } catch (Exception e) {
     	throw new Exception("用户 " + username + " 证书创建失败：" + e.getMessage());
     } finally {
         if (fileOutputStream != null) {
             try {
                 fileOutputStream.close();
             } catch (IOException e) {

             }
     	  }
 	 }
}
```

证书创建在服务器上，用户调用下载证书的接口服务，返回下载地址，因此，这里需要一个下载服务器，下面介绍将nginx配置为下载服务器，文件存放的目录是/var/tmp + /download/

```nginx
location ^~ /download/{
     root /var/tmp;
     if ($request_filename ~* ^.*?\.(txt|doc|pdf|rar|gz|zip|docx|exe|xlsx|ppt|pptx)$) {
     	add_header Content-Disposition: 'attachment;';
     }
}
```