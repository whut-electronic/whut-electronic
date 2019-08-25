---
title: 寝室微型网络服务器的配置（Ubuntu+LAMP) 
date: 2019-05-17 10:11:40
tags: linux运维 
---
## 第一步 安装Ubuntu
Ubuntu下载地址：https://www.ubuntu.com/download/desktop

使用软碟通工具将光盘镜像文件写入到U盘，服务器从U盘启动安装Ubuntu系统。建议安装时断网，防止安装过程中下载更新。整个过程大概需要40分钟。
## 第二步 配置SSH  
<!-- more -->
### 安装SSH
``` bash
# 更新软件源
$ sudo apt-get update
# 安装ssh
$ sudo apt-get install openssh-server openssh-client
```
### 配置SSH
``` bash
# 打开ssh配置文件
$ sudo vim /etc/ssh/sshd_config
```
修改以下配置项
``` ini
# 允许root账号登录
PermitRootLogin yes
# 自定义SSH端口号
Port 2222
# 允许密码登录
PasswordAuthentication yes
```
保存文件，输入以下命令
``` bash
# 设置开机自动启动ssh服务
$ sudo update-rc.d ssh defaults
# 启动ssh服务
$ sudo service ssh start
```
修改root账号的密码
``` bash
# 更改root账户默认密码
$ sudo passwd
> 输入新的UNIX密码：
> 重新输入新的UNIX密码：
> passwd：已成功更新密码
```
查看本机IP
```
$ ifconfig -a
> enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.101  netmask 255.255.255.0  broadcast 192.168.1.255
        ...
        ...

```
此时即可通过局域网内另一台电脑SSH登录到此服务器上（推荐使用XShell工具），本博文剩余部分全部使用root账户SSH登录到服务器上进行操作。

## 第三步 内网穿透或DDNS
如果你的寝室网络为内网IP，那么你需要使用内网穿透工具，将服务器穿透到公网。如果你的寝室网络为公网IP，但这个IP会变化，那么你需要使用动态域名解析（DDNS）。
> 如何判断我的IP是内网IP还是公网IP？

百度搜索"IP"，你就会看到自己的外网IP地址。然后Ping这个地址，若有相应，那么恭喜你，你的IP是公网IP；若Ping无响应，那么你的IP是内网IP
### 内网IP：使用FRP工具进行内网穿透
FRP是一个免费的内网穿透软件。使用FRP进行内网穿透，你需要拥有一台可以连接到公网的服务器
#### 服务器端配置
我的公网服务器是固定IP，系统是WindowsServer。访问 http://diannaobos.iok.la:81/frp/frp-v0.20.0/ 下载frp_0.20.0_windows_amd64.zip ，解压，在cmd中运行frps.exe

Linux系统服务端的配置与Windows系统类似，具体可参考客服端配置
#### 客户端配置
使用wet命令下载frp:
``` bash
# 创建一个文件夹用来放frp
$ mkdir /home/frp
$ cd /home/frp
# 下载frp
$ wget http://diannaobos.iok.la:81/frp/frp-v0.20.0/frp_0.20.0_linux_amd64.tar.gz
# 解压
$ tar -xzf frp_0.20.0_linux_amd64.tar.gz
# 进入解压后的文件夹
$ cd frp_0.20.0_linux_amd64
# 编辑配置文件
$ vim frpc.ini
```
配置文件样例：
``` ini
[common]
# 公网服务端的地址或IP
server_addr = xxx.xxx.xxx.xxx
# frp服务器端口，默认7000，与服务端一致
server_port = 7000

# 添加一个名为ssh的端口映射
[ssh]
type = tcp
local_ip = 127.0.0.1
# 本地端口2222，将来配置SSH时使用的端口应和这个一致
local_port = 2222
# 映射到公网服务器的端口
remote_port = 2222

[web]
type = tcp
local_ip = 127.0.0.1
# 本地web服务器80端口
local_port = 80、、
# 映射到公网服务器的8080端口
remote_port = 8080            
```
保存文件并退出vim编辑器，使用以下命令测试运行frpc:
``` bash
$ frpc -c frpc.ini
```
若无错误，即可关闭当前终端，重新启动一个新的终端，输入以下命令后台运行frpc:
``` bash
$ cd /home/frp/frp_0.20.0_linux_amd64
$ nohup ./frpc -c ./frpc.ini > /dev/null 2>&1 &
```
至此，内网穿透配置完毕。你可以使用 '公网IP或公网地址:2222' 来SSH登录到你的服务器，或使用 http://公网IP或公网地址:8080 来访问你的服务器上的网页资源。
### 公网IP：利用GoDaddy API实现DDNS
GoDaddy是全球最大的域名注册商，它提供了一些API，可以实现动态域名解析
#### 在GoDaddy注册账号并购买一个域名
首先在GoDaddy购买一个域名，很多域名都是有优惠的，10-30元就可以注册一个域名一年的使用权。如果你已经在GoDaddy拥有域名，那么可以跳过这一步。
#### 注册GoDaddyAPI
打开GoDaddy API网站：https://developer.godaddy.com/ ，点击“API Keys”，登录自己的账号，点击“Create New API Key”，复制生成的密钥并妥善保存。
#### 编写Python脚本
``` python
import urllib.request
import http.client
import json
import time

def log(info):
    print(time.strftime("[%Y-%m-%d %H:%M:%S]", time.localtime())+info)

def update_NS(domain, ip_addr):
    api_url = 'https://api.godaddy.com/v1/domains/' + domain + '/records'
    # 此处替换为你的secretID
    secret_id = '*************************'
    # 此处替换为你的secretKey
    secret_key = '************************'
    head = {}
    head['Accept'] = 'application/json'
    head['Content-Type'] = 'application/json'
    head['Authorization'] = 'sso-key ' + secret_id + ':' + secret_key
    records_a = {
        "data": ip_addr,
        "name": "@",
        "ttl": 600,
        "type": 'A',
    }
    #必须包含的Record
    records_NS01 = {
        "data": "ns07.domaincontrol.com",
        "name": "@",
        "ttl": 3600,
        "type": "NS",
    }
    records_NS02 = {
        "data": "ns08.domaincontrol.com",
        "name": "@",
        "ttl": 3600,
        "type": "NS",
    }
    put_data = [records_a, records_NS01, records_NS02]
    try:
        req = urllib.request.Request(api_url,
                                     headers=head,
                                     data=json.dumps(put_data).encode(),
                                     method="PUT")
        rsp = urllib.request.urlopen(req)
        code = rsp.getcode()
        if code == 200:
            log('域名解析更新成功:' + domain + ' -> ' + ip_addr)
        else:
            log('域名解析更新失败.')
    except:
        log('update_NS中发生了错误.')
        log('域名解析更新失败.')


def get_IP():
    try:
        url_ip = 'http://2019.ip138.com/ic.asp'
        req = urllib.request.Request(url_ip, method="GET")
        rsp = urllib.request.urlopen(req)
        data = rsp.read()
        data_str = str(data)
        left = data_str.index('[') + 1
        right = data_str.index(']')
        ip = data_str[left:right]
        return ip
    except:
        log('get_IP中发生了错误.')
        return ""

# 此处替换为你的域名
domain = 'xxxxx.xxx'
last_ip = ""
while True:
    crt_ip=get_IP()
    if crt_ip!="" and crt_ip!=last_ip:
        log('监测到本机IP更新:' + crt_ip)
        last_ip=crt_ip
        update_NS(domain, last_ip)
    time.sleep(5)
```
该脚本每隔5秒检测一次本机IP变化，并通过接口更新域名的A记录，实现DDNS的效果。

在任意位置创建一个py脚本文件，输入以上脚本，注意脚本中部分变量要替换为你的数据。输入以下命令来测试脚本：
``` bash
# 切换到脚本所在的目录
$ cd /home/ddns
$ python3 ddns.py
```
若脚本工作正常，按下Ctrl + C 终止脚本运行，然后输入以下命令后台运行脚本：
``` bash
$ nohup ./ddns.py > ddns.log 2>&1 &
```
脚本的输出会重定向到ddns.log这个文件，你可以打开这个文件来查看脚本输出是否正常。

一切没有问题的话，你就可以通过域名来访问你的服务器了。
## 第四步 安装Apache
```
$ sudo apt-get install apache2 -y
```
安装完成后Apache会自动启动。你也可以通过以下命令手动启动Apache：
```
$ sudo service apache2 start
```
使用浏览器访问你的IP或域名，若你能看到Ubuntu Apache2的欢迎页，证明你的Apache已经正常运行。
## 第五步 安装Mysql
``` bash
# 安装MySQL
$ sudo apt install mysql-server mysql-client
# 启动MySQL服务
$ sudo service mysql start
```
若你使用root账号ssh登录到服务器执行命令，则安装过程中无需设置任何密码。安装完成后在终端输入指令 'mysql' ,若能够进入mysql终端，则MySQL服务正常运行。
## 第六步 安装PHP
``` bash
# 安装PHP及相关组件
$ sudo apt install php-mysql php-curl php-json php-cgi php libapache2-mod-php
# 重启Apache服务
$ sudo service apache2 restart
```
验证PHP安装，在www/html目录下新建一个php脚本文件
``` bash
$ vim /var/www/html/phpinfo.php
```
输入以下代码：
``` php
<?php
    echo phpinfo();
?>
```
保存并退出Vim编辑器，在浏览器中打开 http://你的地址或域名/phpinfo.php ,若你能够看到PHP的欢迎页面，说明你的PHP安装成功。
## 第七步 安装PHPMyAdmin
PHPMyAdmin是一个基于PHP的MySQL数据库管理系统，能够使你方便地对数据库进行管理。
``` bash
$ sudo apt install php-mbstring php7.0-mbstring php-gettext
$ sudo systemctl restart apache2.service
$ sudo apt install phpmyadmin
```
安装过程中一切选项保持默认即可。下一步，将PHPMyAdmin的安装目录连接到网页目录
``` bash
# 设置软连接
$ sudo ln -s /usr/share/phpmyadmin /var/www/html/pma
# 重启Apache
$ sudo /etc/init.d/apache2 restart
```
此时访问 http://你的地址或域名/pma ,你将进入到phpmyadmin的登录页面。但使用root账号无法登录。下一步我们将配置一个用于登录phpmyadmin的mysql账号：
``` bash
# 进入MySQL
$ mysql
# 切换数据库
mysql> use mysql
# 创建一个用户,用户名为test,密码为1234
mysql> insert into mysql.user(Host,User,Password) values("%","test",password("1234"));
# 为该用户授予所有权限，注意密码要一致
mysql> grant all privileges on *.* to test@% identified by '1234';
# 刷新权限
mysql> flush privileges;
```
此时再打开PhpMyAdmin的页面，你就可以通过刚才创建的用户登录并管理数据库了。

至此，小型寝室网络服务器的基本环境搭建完成。

# 感谢阅读

![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/whut-electronic.jpg)

  
---  

<div align=right>author:AZ</div>
