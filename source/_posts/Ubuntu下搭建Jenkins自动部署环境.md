---
title: Ubuntu下搭建Jenkins自动部署环境 
date: 2019-06-15 19:36:14
tags: linux运维 
---
## 安装JDK
``` bash
# 添加java的ppa
$ sudo add-apt-repository ppa:webupd8team/java
# 更新软件源
$ sudo apt-get update
# 安装java8
$ sudo apt-get install oracle-java8-installer
```
<!-- more -->
## 安装tomcat8
``` bash
$ sudo apt-get install tomcat8 tomcat8-docs tomcat8-examples tomcat8-admin -y
```
配置权限
``` bash
$ sudo vim /var/lib/tomcat8/conf/tomcat-users.xml
```
在文件尾添加以下内容
``` xml
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<user username="root" password="123456" roles="manager-gui,admin-gui"/>
```
重启服务
``` bash
$ sudo service tomcat8 restart
```

## 安装Jenkins
使用Jenkins官方软件仓库，添加软件仓库密钥
``` bash
$ wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
```
将官方提供的软件仓库地址加入到本地的apt软件源中，本地用于存放软件源的文件在/etc/apt/sources.list
``` text
deb https://pkg.jenkins.io/debian-stable binary/
```
保存文件并退出，更新软件源并安装Jenkins
``` bash
$ sudo apt-get update
$ sudo apt-get install jenkins
```
下载速度比较慢，可耐心等待。安装结束后启动Jenkins
``` bash
$ sudo /etc/init.d/jenkins start
```
若遇到启动失败的情况，可能是端口冲突，通过以下方法更改端口
``` bash
$ sudo vim /etc/default/jenkins

#修改如下内容
HTTP_PORT=8081

#启动jenkins服务
$ sudo /etc/init.d/jenkins start
```
此时即可通过你的IP:端口号访问Jenkins了，如127.0.0.1:8081
Ubuntu下的Jenkins自动部署环境搭建完成。

# 感谢阅读

![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/whut-electronic.jpg)

  
---  

<div align=right>author:AZ</div>

