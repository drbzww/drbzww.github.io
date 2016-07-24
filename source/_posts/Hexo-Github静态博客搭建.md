---
title: Hexo-Github静态博客搭建
date: 2016-07-01 17:02:36
tags:

---
# 环境配置
1. 前言
Hexo是一个基于Node.js的静态博客程序，可以方便的生成静态网页托管在github和Heroku上。作者是来自台湾的[tommy351](https://github.com/hexojs/hexo)。

2. 安装Node.js
[Node.js下载地址](https://nodejs.org/download/)
[Node.js安装流程](http://www.w3cschool.cc/nodejs/nodejs-install-setup.html)
安装完成后，执行 node --version和npm --version 命令，如果输出如下图所示结果，就代表安装成功了。
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-80d7438a35b6e66a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
git的安装和github账户的注册和配置我在这里就不在赘叙了。

# 使用GitHub Pages建立博客
GitHub账号建立好之后，就可以方便的使用它提供的Pages服务，GitHub Pages分两种，一种是你的GitHub用户名建立的username.github.io这样的用户&组织页（站），另一种是依附项目的pages。想建立个人博客用的是第一种，形如cytmxk.github.io这样的可访问的站，每个用户名下面只能建立一个。
1. github上建立仓库
登录后系统，在github首页，点击页面右下角「New Repository」
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-e8ed2631cf59673a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 填写项目信息：
**project name：**cytmxk.github.io
注：Github Pages的Repository名字是特定的，比如我Github账号是cytmxk，那么我Github Pages Repository名字就是cytmxk.github.io。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-639555efa70dcc75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击「Create Repository」 完成创建。

# 安装和配置Hexo
1. 执行如下命令安装hexo
        $ npm install -g hexo
2. 搭建本地博客
        $ hexo init blog
上面的命令会在当前目录下创建一个blog目录，用来存放构建静态网页的资源，目录结构如下所示：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-b7e22a86ef2ff936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中source文件夹就是用来存放MarkDown文件的，默认会存放一个hello-world.md文件。
执行如下命令将source中的MarkDown文件生成相应的html文件：
        $ hexo generate
执行上面的命令后，多了一个public目录，该目录就是用来存放生成的html文件，目录结构如下图示：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-1976b2614426314a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在部署到github上之前，需要将资源库cytmxk.github.io配置到_config.yml文件中，具体配置如下所示：
        deploy:
          type: git
          repo: git@github.com:cytmxk/cytmxk.github.io.git
          branch: master
执行如下命令将public文件夹中存放的html文件部署到资源库cytmxk.github.io中：
        $ hexo deploy
执行完上面的命令后会生成一个.deploy_git的目录，该目录实际上存放的实一个git资源库，对应的远程资源库就是_config.yml文件中配置的资源库，目录结构如下图所示：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-ff3901fd3e5e9d1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
打开浏览器输入https://cytmxk.github.io/
就可以看到自己的博客首页了，至此一切好像大功告成了，但是还有一个问题没有解决，就是多台电脑之间博客同步的问题。

# 多台电脑之间博客同步
我们通过 hexo deploy会将public目录（html文件）自动push到远程仓库的master分支。但这个对多终端博客同步没有任何意义，因为我们每次hexo generate都会根据source目录下的markdown源文件重新生成html文件，所以解决问题的关键是怎么同步source目录下的源文件，不仅如此，还有配置文件_config.yml，依赖包记录文件package.json，scanffolds目录，themes目录。

1. 首先我们进入到博客系统的根目录，比如blog目录，这里边有个.gitignore文件（如果该文件不存在，自己创建一个），里边默认已经把该忽略的目录文件都写好了，里边内容如下：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-0211fb04f82bc283.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 然后在blog目录初始化仓库，切换到source分支，关联远程仓库，并push到远程仓库的source分支：
        $ cd blog
        $ git init
        $ git checkout -b source
        $ git add .
        $ git commit -m "first commit"
        $ git remote add origin git@github.com:cytmxk/cytmxk.github.io.git
        $ git push origin source
3. 操作完成后，在另外一台电脑上，首先按照上面的进行环境配置，然后创建
注意不要再执行：
         $ hexo init blog
取而代之的是
         $ git clone -b source git@github.com:cytmxk/cytmxk.github.io.git
         $ npm install //根据package.json来下载依赖包
执行上面的命令就把远程仓库的source分支克隆下来并且安装依赖包。接下来我们就可以继续写博客了
        $ hexo new "about hexo sync"
        $ hexo generate$ hexo deploy
        $ git add .$ git commit -m "add blog"
        $ git push origin source
这样就完成了多终端的博客同步。

# 遇到的问题
1. 执行 hexo deploy 报错，如下图所示：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2171639-69c9d99139a9de76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面的报错是由于依赖包hexo-deployer-git没有安装，所以执行如下命令安装即可：
       $ npm install hexo-deployer-git --save
