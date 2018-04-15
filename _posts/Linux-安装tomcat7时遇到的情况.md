---
title: Linux-安装tomcat7时遇到的情况
date: 2017-12-11 15:26:23
tags: [linux,tomcat]
---

首先我们需要在linux上安装jdk，[linux-centOS7下安装jdk1.8](http://guojian.fun/2017/12/11/linux-%E5%AE%89%E8%A3%85jdk1-8/)。

解压文件很简单，这里主要是我分享一下我遇到的问题

**1、下载apache-tomcat-8.0.47.tar.gz包并且解压，命令为tar-zxvf  apache-tomcat-8.0.47.tar.gz**

**2、在解压的文件下，新建目录logs(与bin目录平级)，这个目录是用来记录日志的。不建的话，tomcat会提示你的。**

**3、因为我用的是centos7，因为防火墙的原因是不会在外网访问的。所以我们要讲8080端口写入到防火墙了。**

注意：防火墙操作需要root用户，所以不是root用户先切换到root用户。

firewall-cmd --zone=public(作用域) --add-port=80/tcp(端口和访问类型) --permanent(永久生效)

添加端口：firewall-cmd --zone=public --add-port=8080/tcp --permanent

重启防火墙：firewall-cmd --reload

检查8080端口是否添加到：firewall-cmd --list-ports

如果以后想删除,命令为firewall-cmd --zone= public --remove-port=8080/tcp --permanent（这句话现在就不要执行了）。

**4、将bin目录下的权限都改一下，否则无法执行。**

在bin的目录下 chmod 775 * 

**5、在logs目录下查看日志catalina.out**

Error: Could not find or load main class org.apache.catalina.startup.Bootstrap

我在官网上重新下了不同版本的包，均是这个错误。然后在资料上查询是jvm的版本低于tomcat支持的版本，即tomcat的版本过高，所以我尝试用了版本较低的tomcat版本，错误消失。

**6、进行访问192.168.14.172:8080 访问成功，我们linux上安装tomcat就结束了。**