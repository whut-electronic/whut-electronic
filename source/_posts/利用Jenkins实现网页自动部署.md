---
title: 利用Jenkins实现网页自动部署 
date: 2019-06-16 11:22:03
tags: linux运维 
---
在上一篇文章中，我们在Ubuntu下搭建了Jenkins自动部署环境，这篇文章将利用Jenkins实现网页的自动部署。
## 初始化Jenkins
在浏览器输入服务器地址:端口号进入Jenkins的控制台，如127.0.0.1:8081。第一次启动时，要求输入默认管理员密码。默认管理员密码存放在/var/lib/jenkins/secrets/initialAdminPassword文件中。使用vim打开此文件，复制管理员密码并填入相应位置，点击继续。
接着进入插件安装的页面，选择安装推荐插件，等待插件安装完成。
接着按照指引，完成后续部分。
<!--more-->
## 安装Gitee插件
由于Github在国内访问较慢，我们选择国内优秀的代码托管平台Gitee构建代码仓库。
进入Jenkins工作台后，点击左侧的系统管理，然后找到插件管理，点击可选插件，在右上角过滤条件中输入"Gitee"，勾选Gitee插件前面的复选框，点击页面最下端的"直接安装"。
## 注册Gitee账号
如果你已经有Gitee账号，则跳过这一步
Gitee官网：https://gitee.com/
## 配置Gitee for Jenkins
进入Jenkins控制台，点击系统管理->系统设置，找到"Gitee 配置"。
- 连接名：Gitee
- Gitee域名Url：https://gitee.com
- 证书令牌：点击"添加"->"Jenkins"，在弹出的窗口中，类型选择"Gitee令牌"。接着进入 https://gitee.com/profile/personal_access_tokens ,点击"生成新令牌",描述填写"Jinkins"，点击提交，复制生成的令牌。然后回到Jenkins控制台，将复制的令牌粘贴到"Gitee APIV5 私人令牌"一项中，点击添加，窗口关闭。接着点击证书令牌后的下拉框，选择"Gitee API令牌"，再点击"测试连接"，若测试成功，则点击页面下方的保存。
## 在Gitee创建一个项目
操作类似Github。点击"创建仓库"，输入仓库名称，"是否开源"选择"公开"，"分支模型"选择"生产/开发模型",然后点击创建。
## 在Jenkins创建一个任务
点击Jenkins控制台左侧的"新建任务",输入任务名，如"Pages",选择"构建一个自由风格的软件项目"，确定
找到源码管理，选择"Git"，"Repository URL"填写你的项目地址.git，再找到构建触发器，选择"Gitee webhook"，只勾选"推送代码"，在"允许触发构建的分支"中选择"根据分支名过滤"，"包括"一项中填写"master"。在"Gitee WebHook密码"一项，点击"生成"，并复制生成的密码。
接着找到"构建"，点击"增加构建步骤"，选择"执行shell"，接着在命令中填写以下内容
``` bash
echo 开始执行自动部署
sudo sh /home/pull.sh
echo 自动部署结束
```
最后点击保存。
## 配置Gitee WebHook
在Gitee中打开你的项目，点击"管理"->"Web Hooks"，点击添加
- Url: http://服务器地址:Jenkins端口/project/Jenkins中的任务名 (如:http://azhrzho.com:8081/project/Pages)
- 密码: 上一步生成的WebHook密码
- 选择事件: 只勾选Push
然后保存，点击"测试",若请求结果类似以下结果，则配置成功。
``` text
push_hooks ref = refs/heads/master commit sha = xxxxxxxxx has been accepted.
```
## 配置服务器部署操作
使用SSH登录服务器，输入以下命令
``` bash
$ cd /home
$ git clone 你的Gitee项目地址.git
# 如： git clone https://gitee.com/xxx/Pages.git
$ ln -s /home/你的项目名字 /var/www/html/pages
# 如： ln -s /home/Pages /var/www/html/pages
```
接着编写一个脚本
``` bash
$ vim pull.sh
# 在文本编辑器中输入以下内容
```
``` sh
#!/bin/bash
cd /home/你的项目名字
git pull
```
保存文件并退出编辑器。此时，自动部署已经配置完毕。你可以在任意一台电脑上对你的项目代码进行修改并推送，每次推送后，新的网页将被即时部署在 http://服务器地址/pages 当中。
## 添加部署后邮件提醒
虽然自动部署已经配置完成，但每次部署后我们无法接收到是否部署成功的消息。因此我们可以给任务添加邮件提醒。
首先，进入Jenkins控制台，点击"系统管理"->"系统设置"，找到"系统管理员邮件地址",填写你的QQ邮箱。继续向下，找到"Extended E-mail Notification",并点击"高级"。
- SMTP server: smtp.qq.com
- Default user E-mail suffix: @qq.com
- Use SMTP Authentication: 勾选
- User Name: 你的QQ账号
- Password：一会再说
- Use SSL：勾选
- SMPT port：465
- Default Content Type：HTML
- Default Recipients：你的QQ邮箱
- Default Subject：填写以下内容
``` text
$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!
```
- Default Content：填写以下内容
``` html
<!DOCTYPE html>    
<html>    
<head>    
<meta charset="UTF-8">    
<title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>    
</head>    
    
<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"    
    offset="0">    
    <table width="95%" cellpadding="0" cellspacing="0"  style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">    
        <tr>    
            本邮件由系统自动发出，无需回复！<br/>            
            以下为${PROJECT_NAME }项目构建信息</br> 
            <td><font color="#CC0000">构建结果 - ${BUILD_STATUS}</font></td>   
        </tr>    
        <tr>    
            <td><br />    
            <b><font color="#0B610B">构建信息</font></b>    
            <hr size="2" width="100%" align="center" /></td>    
        </tr>    
        <tr>    
            <td>    
                <ul>    
                    <li>项目名称 ： ${PROJECT_NAME}</li>    
                    <li>构建编号 ： 第${BUILD_NUMBER}次构建</li>    
                    <li>触发原因： ${CAUSE}</li>    
                    <li>构建状态： ${BUILD_STATUS}</li>    
                    <li>构建日志： <a href="${BUILD_URL}console">${BUILD_URL}console</a></li>    
                    <li>构建  Url ： <a href="${BUILD_URL}">${BUILD_URL}</a></li>    
                    <li>工作目录 ： <a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>    
                    <li>项目  Url ： <a href="${PROJECT_URL}">${PROJECT_URL}</a></li>    
                </ul>    

<h4><font color="#0B610B">失败用例</font></h4>
<hr size="2" width="100%" />
$FAILED_TESTS<br/>

<h4><font color="#0B610B">最近提交(#$SVN_REVISION)</font></h4>
<hr size="2" width="100%" />
<ul>
${CHANGES_SINCE_LAST_SUCCESS, reverse=true, format="%c", changesFormat="<li>%d [%a] %m</li>"}
</ul>
详细提交: <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a><br/>

            </td>    
        </tr>    
    </table>    
</body>    
</html>
```
继续向下，找到"邮件通知"，点击"高级"，这里的配置与上文一样，不再赘述。
接着，填写两处的密码。这里的密码不是QQ密码，而是QQ邮箱的授权码。下面介绍QQ邮箱获取授权码的方法。
## 获取QQ邮箱授权码
第一步，进入QQ邮箱，点击"设置"->"账户"
第二步，找到"POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务"，点击"生成授权码"
第三步，根据指引生成授权码，并复制
最后，将复制的授权码填写到Jenkins的两处邮箱密码配置中，点击保存。
## 为任务添加邮件提醒
返回Jenkins控制台主页，点击你创建好的任务，再点击"配置"，找到"构建后操作"，点击"增加构建后操作步骤",选择"Editable Email Notification"。"Content Type"选择"HTML"，再点击"Advanced Settings"，删除所有默认的Triggers，再点击"Add Triggers"，选择"Always"，在"Send To"配置中添加"Developers"和"Recipient List"，最后点击保存。
至此，通知邮件配置完毕。你可以尝试推送一次你的项目，然后你会接收到一封邮件，说明部署成功或失败，并附带本次部署的详细信息。


# 感谢阅读

![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/whut-electronic.jpg)
  
---  

<div align=right>author:AZ</div>
