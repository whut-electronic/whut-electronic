---
title: 手把手教你用hexo搭建个人博客
date: 2019-07-14 11:23:26
tags: hexo 
top: 1  
---
### 什么是 Hexo？
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
比如我们电子科技协会的官方博客就是用hexo搭建的

#### 话不多说，这就开始我们的建站教程

### 一、安装Git和Node.js
1.安装Git版本工具
Git是目前世界上较为流行的分布式版本控制系统，使用Git可以帮助我们把本地的网页和文章等内容提交到Gihub上，实现同步。 
下载地址：https://git-scm.com/downloads 

2.搭建Node.js环境
我们了解到Hexo基于Node.js的，那么我们搭建博客网站首先需要安装Node.js环境。 Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境，可以在非浏览器环境下，解释运行 JS 代码。 
下载地址：http://nodejs.cn/download 

<!--more-->
### 二、安装hexo
在一个你可以找到的地方，新建一个文件夹，路径中不要包含中文，文件夹名就叫myblog吧

然后鼠标右键点击git bash (！！！之后的命令都是在git bash里输入的)

```
npm install -g hexo-cli    //安装hexo
```



### 三、开始了开始了

```
hexo init 初始化文件夹
npm install 安装插件
```



可能要等上一会儿，安装完，你会发现，你的空文件里多了一些东西
>.
├── _config.yml  配置信息  
├── package.json 应用程序信息  
├── scaffolds    模板信息  
├── source       资源文件夹  
|   ├── _drafts  
|   └── _posts  
└── themes       主题文件夹  
>


### 四、配置开始
在 _config.yml 中修改大部分的配置

参数	    描述
title	    网站标题
subtitle	网站副标题
description	网站描述
author	    您的名字
language	网站使用的语言
timezone	网站时区。Hexo 默认使用您电脑的时区。


这是我这部分的配置
![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/hexo/%E9%85%8D%E7%BD%AE.png)

### 五、要开始出效果了
```
hexo clean 删除公共文件
hexo g     生成静态文章
hexo s     开始服务
```


![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/hexo/first.png)

在浏览器中输入 http://localhost:4000 即可看到你的博客

当然这样肯定使不够的，因为你只要一按Ctrl+C,这个端口就访问不了了，你也就看不了你的博客了

### 六、GitHub（要发挥gayhub的作用啊）
GitHub创建个人仓库
首先，你先要有一个GitHub账户，去注册一个吧。

注册完登录后，在GitHub.com中看到一个New repository，新建仓库


创建一个和你用户名相同的仓库，后面加.github.io，只有这样，将来要部署到GitHub page的时候，才会被识别，也就是xxxx.github.io，其中xxx就是你注册GitHub的用户名。我这里是已经建过了。


点击create repository。

#### 生成SSH添加到GitHub
回到你的git bash中

```
git config --global user.name "yourname"
git config --global user.email "youremail"
```



这里的yourname输入你的GitHub用户名，youremail输入你GitHub的邮箱。这样GitHub才能知道你是不是对应它的账户。

可以用以下两条，检查一下你有没有输对

```
git config user.name
git config user.email
```


然后创建SSH,一路回车

```
ssh-keygen -t rsa -C "youremail"
```


这个时候它会告诉你已经生成了.ssh的文件夹。在你的电脑中找到这个文件夹。(不在C盘，就在你博客文件的这个文件夹里)

ssh，就是一个秘钥，其中，`id_rsa`是你这台电脑的私人秘钥，不能给别人看的，`id_rsa.pub`是公共秘钥，可以随便给别人看。把这个公钥放在GitHub上，这样当你链接GitHub自己的账户时，它就会根据公钥匹配你的私钥，当能够相互匹配时，才能够顺利的通过git上传你的文件到GitHub上。

而后在GitHub的setting中，找到SSH keys的设置选项，点击New SSH key
把你的id_rsa.pub里面的信息复制进去。

![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/hexo/SSH.png)



### 七、将hexo部署到GitHub
这一步，我们就可以将hexo和GitHub关联起来，也就是将hexo生成的文章部署到GitHub上，打开站点配置文件 _config.yml，翻到最后，修改为
YourgithubName就是你的GitHub账户

>deploy:  
type: git  
repo: https://github.com/YourgithubName/YourgithubName.github.io.git  
   branch: master  

这个时候需要先安装deploy-git ，也就是部署的命令,这样你才能用命令部署到GitHub。


```
npm install hexo-deployer-git --save
```



然后

```
hexo clean
hexo g
hexo d  //deploy部署文章
```





注意deploy时可能要你输入username和password。


过一会儿就可以在http://yourname.github.io 这个网站看到你的博客了！！

# 感谢阅读

![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/whut-electronic.jpg)

  
---  
<div align=right>author:kcqnly</div>