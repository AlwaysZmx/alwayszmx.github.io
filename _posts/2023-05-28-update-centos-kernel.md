---
layout: post
title: 升级CentOS7系统内核至最新版本
categories: [Linux, CentOS, kernel]
description: 本文主要介绍如何将CentOS7的内核升级至最新版本
keywords: Linux, CentOS, kernel
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
copyright: true
---

# 一、为什么要升级内核

+ 安全性：新版本的内核通常包含安全修复和增强功能，可以提高系统的安全性。
+ 支持硬件：新版的内核通常能够支持更多的硬件设备，如新出的 CPU、显卡等，从而提高系统的兼容性和性能。
+ 性能提升：新的内核版本通常会优化一些关键的子系统和模块，从而提高系统的性能和响应速度。
+ 新特性：新的内核版本通常会引入一些新的特性和功能，如容器技术、虚拟化、性能监控等，可以提高系统的可用性和管理效率。

# 二、内核升级步骤
- 升级gcc编译器版本

先查看gcc编译器版本，gcc编译器版本太低将无法编译，最低需要5.x版本
```shell
[root@iZ7xv8p8qxpbzcvlgcikvuZ ~]# gcc --version
gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44)
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
安装新版本gcc编译器，这里安装8.x版本
```shell
#可通过centos-release-scl源安装devtoolset包
yum install centos-release-scl
yum install devtoolset-8
```

devtoolset与gcc版本的对应关系：
```tex
devtoolset-3对应gcc4.x.x版本
devtoolset-4对应gcc5.x.x版本
devtoolset-6对应gcc6.x.x版本
devtoolset-7对应gcc7.x.x版本
devtoolset-8对应gcc8.x.x版本
devtoolset-9对应gcc9.x.x版本
devtoolset-10对应gcc10.x.x版本
```


激活gcc版本，使其生效
```shell
scl enable devtoolset-8 bash
#或
source /opt/rh/devtoolset-8/enable
```

- 查看当前系统内核版本

```shell
[root@iZ7xv8p8qxpbzcvlgcikvuZ ~]# uname -r
3.10.0-1160.88.1.el7.x86_64
```

- 下载linux内核源码

官方下载：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/refs/tags

这里以最新的linux-6.3正式版本为例：

```shell
#下载源码
wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/snapshot/linux-6.3.tar.gz
#创建目录
mkdir /usr/local/core
#解压源码压缩包
tar -zxvf linux-6.3.tar.gz
#将解压目录移动至/usr/local/core
mv linux-6.3/ /usr/local/core/
#切换至源码目录
cd /usr/local/core/linux-6.3
#检测程序所有安装包情况
yum grouplist
#安装工具（中间需要选择，选择Y）
yum groupinstall Development -y
#安装其它依赖工具
yum install hmaccalc zlib-develbinutils-devel elfutils-libelf-devel -y
#开始准备编译内核,删除不必要的文件和目录
make mrproper
#把当前系统旧版本内核的配置文件复制到源码目录并命名为.config，这样新编译内核就会使用原来的配置文件（指令可能会变，可以查看/boot目录下格式相同的文件，有两个类似的，使用较短的）
cp /boot/config-3.10.0-1160.88.1.el7.x86_64 .config
#安装openssl
yum install openssl -y
yum install openssl-devel -y
#编译 bzImage（中间有个选择1-6的，选择1，然后一直按着回车键，等到他开始自动编译松开，编译时间较长耐心等待）
make bzImage
#编译（时间较长，耐心等待）
make
make modules
#安装模块
make modules_install
make install 
```

经过以上步骤后，新内核以及编译成功了，现在执行以下命令查看当前系统有几个内核：
```shell
[root@iZ7xvczellkr27qwup5967Z linux-6.3]# cat /boot/grub2/grub.cfg |grep menuentry
if [ x"${feature_menuentry_id}" = xy ]; then
  menuentry_id_option="--id"
  menuentry_id_option=""
export menuentry_id_option
menuentry 'CentOS Linux (6.3.0) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.14.4.el7.x86_64-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
menuentry 'CentOS Linux (3.10.0-1160.90.1.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.14.4.el7.x86_64-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
menuentry 'CentOS Linux (3.10.0-1160.88.1.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.14.4.el7.x86_64-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
menuentry 'CentOS Linux (3.10.0-862.14.4.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.14.4.el7.x86_64-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
menuentry 'CentOS Linux (3.10.0-862.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.el7.x86_64-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
menuentry 'CentOS Linux (0-rescue-20181129113200424400422638950048) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-20181129113200424400422638950048-advanced-b98386f1-e6a8-44e3-9ce1-a50e59d9a170' {
```
可以看到已经有我们刚刚新编译的6.3版本的内核了：CentOS Linux (6.3.0) 7 (Core)
接下来，需要将6.3版本的内核设置为系统默认启动内核，执行以下命令进行设置

```shell
grub2-set-default "CentOS Linux (6.3.0) 7 (Core)"
```
验证默认内核是否设置成功
```shell
[root@iZ7xvczellkr27qwup5967Z linux-6.3]# grub2-editenv list
saved_entry=CentOS Linux (6.3.0) 7 (Core)
```
重启服务器
```shell
shutdown -r now
```

执行命令查看当前系统内核版本
```shell
[root@ydgeiosz3bpsykjo ~]# uname -r
6.3.0
```