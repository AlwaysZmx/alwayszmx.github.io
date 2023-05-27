---
layout: post
title: 一文教你本地部署ChatGLM-6B开源双语对话语言模型
categories: [Java]
description: 本文主要介绍如何基于CentOS7搭建一个Java开发环境，以及常用软件的安装及卸载。
keywords: Java, JDK, Maven, MySQL, Nginx, Docker
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
copyright: true
---

当你拿到一台全新的CentOS服务器时，你应该如何搭建一个Java开发环境？本文主要介绍如何基于CentOS7搭建一个Java开发环境，以及常用软件的安装及卸载。

# 一、替换软件源

- 备份原来的软件源
```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

- 下载阿里源
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```
*如果提示没有wget，就先执行以下命令安装wget*

```shell
yum install wget -y
```

- 生成缓存
```shell
yum makecache
```

- 清除不必要的提示信息
```shell
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

- 更新软件源
```shell
yum update -y
```

- 安装常用软件
```shell
yum install -y telnet telnet-server net-tools
```

# 二、安装JDK

- 下载JDK
官方下载：https://www.oracle.com/java/technologies/downloads/#java8

- 安装JDK
上传JDK安装包至服务器home目录，执行解压安装包命令
```shell
tar -zxvf jdk-8u261-linux-x64.tar.gz
```

移动解压目录至/usr/local下
```shell
mv jdk1.8.0_261/ /usr/local/
```

- 配置环境变量
  编辑/etc/profile配置JDK环境变量
```shell
vi /etc/profile
```
编辑/etc/profile，并将以下内容粘贴到/etc/profile文件的尾部：
```shell
export JAVA_HOME=/usr/local/jdk1.8.0_261
export CLASSPATH=$:CLASSPATH:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin
```
执行以下命令，使环境变量立即生效
```shell
source /etc/profile
```

- 验证
执行java -version命令，判断环境变量配置是否生效
```shell
[root@ydgeiosz3bpsykjo local]# java -version
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```

# 三、安装Maven
- 下载Maven
```shell
wget https://dlcdn.apache.org/maven/maven-3/3.9.2/binaries/apache-maven-3.9.2-bin.tar.gz
```

- 安装Maven
解压安装包
```shell
tar -zxvf apache-maven-3.9.2-bin.tar.gz
```
将解压目录移动至/usr/local
```shell
mv apache-maven-3.9.2/ /usr/local/
```

- 配置环境变量
编辑/etc/profile，并将以下内容粘贴到/etc/profile文件的尾部：
```shell
MAVEN_HOME=/usr/local/apache-maven-3.8.6
PATH=$MAVEN_HOME/bin:$PATH
export MAVEN_HOME PATH
```
执行以下命令，使环境变量立即生效
```shell
source /etc/profile
```

- 验证
执行mvn -v命令，判断环境变量配置是否生效
```shell
[root@ydgeiosz3bpsykjo local]# mvn -v
Apache Maven 3.8.6 (84538c9988a25aec085021c365c560670ad80f63)
Maven home: /usr/local/apache-maven-3.8.6
Java version: 1.8.0_261, vendor: Oracle Corporation, runtime: /usr/local/jdk1.8.0_261/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-1160.88.1.el7.x86_64", arch: "amd64", family: "unix"
```

# 四、安装MySQL

- 下载MySQL安装包源
```shell
wget http://repo.mysql.com/mysql57-community-release-el7-10.noarch.rpm
```

- 安装MySQL源
```shell
rpm -Uvh mysql57-community-release-el7-10.noarch.rpm
```

- 安装MySQL服务端
```shell
yum install -y mysql-community-server
```

- 启动MySQL
```shell
systemctl start mysqld.service
```

- 检查是否启动成功
```shell
systemctl status mysqld.service
```

- 获取root用户临时密码
```shell
grep 'temporary password' /var/log/mysqld.log
```

- 通过临时密码登录MySQL
```shell
mysql -uroot -p
```
*使用临时密码登录后，不能进行其他的操作，否则会报错，接下来我们进行修改密码操作*

- 修改MySQL密码

因为MySQL的密码规则需要很复杂，我们一般自己设置的不会设置成这样，所以我们全局修改一下
```sql
mysql> set global validate_password_policy=0;
mysql> set global validate_password_length=1;
```
这时候我们就可以自己设置想要的密码了
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'yourpassword';
```

- 开启远程登录
```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'yourpassword' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

- 设置开机自启动
先退出mysql命令行，然后输入以下命令
```shell
systemctl enable mysqld
systemctl daemon-reload
```

- 设置MySQL的字符集为UTF-8
```shell
vim /etc/my.cnf
```
改成如下,然后保存
```shell
[mysql]
default-character-set=utf8

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
default-storage-engine=INNODB
character_set_server=utf8

symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

- 重启一下MySQL,令配置生效
```shell
service mysqld restart
```

- 防火墙开放3306端口
```shell
firewall-cmd --state
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
```

- 卸载MySQL
```shell
rpm -qa | grep mysql
yum -y remove mysql57-community-release-el7-10.noarch
```

- 相关操作命令

查看mysql是否启动：
```shell
service mysqld status
```
启动mysql：
```shell
service mysqld start
```
停止mysql：
```shell
service mysqld stop
```
重启mysql：
```shell
service mysqld restart
```
查看临时密码：
```shell
grep password /var/log/mysqld.log
```

# 五、安装Nginx
- 下载Nginx
```shell
wget http://nginx.org/download/nginx-1.22.1.tar.gz
```

- 安装Nginx

解压安装包
```shell
tar -zxvf nginx-1.22.1.tar.gz
```
安装gcc编译器
```shell
yum install gcc-c++
```
安装nginx依赖库pcre pcre-devel
```shell
yum install -y pcre pcre-devel
```
安装nginx依赖库zlib
```shell
yum install -y zlib zlib-devel
```
安装OpenSSL
```shell
yum install -y openssl openssl-devel
```
配置Nginx
```shell
cd /usr/local/nginx-1.22.1
./configure
```
编译、安装
```shell
make
make install
```
查找安装路径
```shell
[root@ydgeiosz3bpsykjo ~]# whereis nginx
nginx: /usr/local/nginx
```
修改配置
```shell
vi /usr/local/nginx/conf/nginx.conf
```

- 开放访问端口

```shell
#--permanent永久生效，没有此参数重启后失效
firewall-cmd --zone=public --add-port=9000/tcp --permanent
#重新载入配置
firewall-cmd --reload
#查看已经开启的端口
firewall-cmd --zone=public --list-ports
```

- 设置开机自启

```shell
vi /lib/systemd/system/nginx.service
```
nginx.service内添加以下内容：
```shell
Description=nginx - high performance web server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
[Install]
WantedBy=multi-user.target
```
使配置生效
```shell
systemctl daemon-reload
```
设置开机启动
```shell
systemctl enable nginx.service
```

- 相关操作命令

启动
```shell
/usr/local/nginx/sbin/nginx
```
停止nginx
```shell
/usr/local/nginx/sbin/nginx -s quit
```
强制停止nginx
```shell
/usr/local/nginx/sbin/nginx -s stop
```
重新加载配置文件
```shell
/usr/local/nginx/sbin/nginx -s reload
```
重启Nginx
```shell
/usr/local/nginx/sbin/nginx -s quit
/usr/local/nginx/sbin/nginx 
```
查询nginx进程
```shell
ps -ef | grep nginx
```
访问Nginx
```shell
curl http://127.0.0.1/
```

# 六、安装Docker

- 卸载旧版本安装包

查询docker安装包
```shell
yum list installed | grep docker
```
删除安装包
```shell
yum remove docker*
```
删除镜像/容器等
删除之前需要查看docker的存储路径，查看存储路径
```shell
[root@ydgeiosz3bpsykjo ~]# docker info|grep "Docker Root"
  WARNING: You're not using the default seccomp profile
Docker Root Dir: /var/lib/docker
```
删除镜像和容器数据，我这里存储的路径是默认路径/var/lib/docker，所以直接删除这个路径的数据
```shell
rm -rf /var/lib/docker
```

- 安装docker

Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker ，通过 uname -r 命令查看你当前的内核版本。如果内核版本较低可以尝试升级内核
```shell
[root@ydgeiosz3bpsykjo ~]# uname -r
3.10.0-1160.88.1.el7.x86_64
```
确保 yum 包更新到最新
```shell
yum update -y
yum clean all
yum makecache
```
设置yum源，解决安装慢的问题
```shell
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
查看仓库中所有docker版本
```shell
yum list docker-ce --showduplicates | sort -r
```
安装默认docker版本
```shell
yum install docker-ce -y
```
也可以安装指定版本：
`yum install docker-ce-17.12.0.ce` 或者 `yum install docker-ce-19.03.9`

```shell
#启动
systemctl start docker
#设置开机自启动
systemctl enable docker
#查看启动状态
systemctl status docker
```
- 验证

```shell
[root@iZ7xv8p8qxpbzcvlgcikvuZ ~]# docker version
Client: Docker Engine - Community
 Version:           23.0.6
 API version:       1.42
 Go version:        go1.19.9
 Git commit:        ef23cbc
 Built:             Fri May  5 21:21:29 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          23.0.6
  API version:      1.42 (minimum version 1.12)
  Go version:       go1.19.9
  Git commit:       9dbdbd4
  Built:            Fri May  5 21:20:38 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.21
  GitCommit:        3dce8eb055cbb6872793272b4f20ed16117344f8
 runc:
  Version:          1.1.7
  GitCommit:        v1.1.7-0-g860f061
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

