---
layout: post
title: "一台mac如何维护两个octopress blog"
date: 2014-10-13 22:03:06 +0800
comments: true
categories: octopress
---

#前言
上篇介绍了如何在mac下部署octopress，可能少有人会跟我一样折腾，还打算在一台mac上维护两个octopress blog，这里介绍一下具体办法，一言以蔽之就是在其他目录下再创建一个octopress，以后在目录下维护第二个blog，以下是具体流程。
---------

#创建第二个octopress blog的repo
github不支持同一个账户创建多个两个github page的repo， 因此再去申请一个github帐号就可以了，然后依然是首先到[github](https://github.com/new)创建一个`username.github.io`的repo，`username.github.io`以后就是blog的域名（当然他支持使用自定义的域名）， username就是你在github上的username。
---------


#准备安装Octopress所需的环境
由于有了第一个octopress blog的安装基础， 因此默认认为mac上已经有了octopress的环境：

```
$ git --version
```
git的版本号对安装octopress影响不大，这里不用关心，只要有git工具就可以了。

```
$ ruby --version #此时应该已是1.9.3p125 (2012-02-16 revision 34643)，或者其他你成功安装了octopress的ruby版本
```

#安装第二个Octopress
一般第一个octopress一般都装在 `cd ~/octopress` 下，这里假设把第二个octopress安装在 `~/secondBlog/octopress`下

```
$ cd ~
$ mkdir secondBlog #作为第二个octopress所在目录
$ cd secondBlog
$ git clone git://github.com/imathis/octopress.git octopress 
```

###同样安装完依赖跟默认主题

首先查看一下`~/secondBlog/octopress`下的ruby版本：

```
$ruby --version #如果跟第一个octopress所在目录下version版本一致即可，不一样的话，参考第一篇中介绍rbenv 环境变量的设置
```

```
$ gem install bundler
$ rbenv rehash
$ bundle install

$ rake install #默认主题

```

---------

运行以下命令，仔细看提示完成github和Octopress的关联（就是第一步创建的第二个博客的repo https://github.com/username/username.github.io

```
$ rake setup_github_pages
```
关联成功后可以看下：

```
$ git remote -v #关联的远程repo信息

```
正确关联的话就会如下显示：
```
octopress	git://github.com/imathis/octopress.git (fetch)
octopress	git://github.com/imathis/octopress.git (push)
origin	https://github.com/secondUsername/secondUsername.github.io (fetch)
origin	https://github.com/secondUsername/secondUsername.github.io (push)
```

###接下来生成blog,本地preview一下第二个blig吧

```
$ rake generate

$ rake preview #http://localhost：4000
```

打开[http://localhost：4000](http://localhost：4000)，就能看到第二个octopress 也建起来了。

至此第二个octopress blog 就搭建完了，就可以在一台mac下同时维护两个blog了，之后写blog、装插件、换配置就都一样了，够折腾吧- -!
---------

















以上便是创建一个github的全部过程了，下一篇会继续说明如果在一台电脑上管理多个octpress blog 以及 一个octopress在多台电脑上共同维护的方法。

---------
