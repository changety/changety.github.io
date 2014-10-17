---
layout: post
title: "octopress blog由多台设备维护"
date: 2014-10-15 00:11:52 +0800
comments: true
categories: octopress
---


#前言
[之前这篇](http://changety.github.io/blog/2014/10/12/setup-octopress-github-blog/)介绍了如何在mac下部署octopress，[这篇](http://changety.github.io/blog/2014/10/13/muti-octopress-blog-on-one-pc:mac/)还介绍了一台mac上维护同时维护多个octopress的方法，本篇再介绍一下如何再多台设备上，同时维护一个octopress blog上的方法。

---------

#octopress blog的repo分支说明

一个octopress blog的repo由source、master两个分支组成。


source分支位于 `cd ~/octopress` 中

```
cd ~/octopress
git branch # *source
```

这个分支用于存放markdown源文件，theme，plugin等用于生成blog所需要的所有文件的，因此之前我们写博客的时候，提交的内容，一般都到在source上：

```
git push origin source #即将本地内容push到远端git repo 的source分支上。
```

master分支位于 `cd ~/octopress/_deploy`中，这个分支用于存放blog站点本身

```
cd ~/octopress/_deploy
git branch # *master
```
---------

#在新的设备上部署已有的octopress

首先像构建全新octopress 一样， 准备好octopress的部署环境

```
$ git --version
```
git的版本号对安装octopress影响不大，这里不用关心，只要有git工具就可以了。

```
$ ruby --version #shold be 1.9.3#p125 or other version can be set up an octopress
```
配置环境的方法参考[这里](http://changety.github.io/blog/2014/10/12/setup-octopress-github-blog/)


理解octopress blog的工作原理以及分支组成后，新的设备复制已有的octopress blog 只要把source/master clone到本地，并安装完成类似构建新octopress blog一样的配置及工具即可。以下是具体步骤：
###将source分支 clone到 octopress目录下
```
git clone -b source git@github.com:username/username.github.com.git octopress 

```
###将master分支 clone到 octopress/_deploy目录下
```
$ cd octopress #进入到octopress目录下，看看_deploy是否存在，存在的话且不为空，先清空
$ git clone git@github.com:username/username.github.com.git _deploy  #clone master分支
```


###安装配置工作

```
$ gem install bundler
$ rbenv rehash
$ bundle install
```

注意：由于远端的blog可能已经配置主题、所以省去安装默认主题的这一步： `rake install`

###关联blog

```
rake setup_github_pages
```
提示你输入关联的repo的 git url

```
Enter the read/write url for your repository

```
输入需要关联的repo的git url即可， 比如 git@github.com:username/username.github.com  

关联完成后，检查一下：

```
git remote -v

octopress	git://github.com/imathis/octopress.git (fetch)
octopress	git://github.com/imathis/octopress.git (push)
origin	https://github.com/username/username.github.io (fetch)  #关联blog 的远程 repo
origin	https://github.com/username/username.github.io (push)
```
至此，如果没有错误的话，本地已经完成了一个已有octopress blog复制跟关联。


#在两台设备上写blog

接下来其实就与git 上共同开发项目一样，首先需要pull下来最新的远端repo的内容，本地做出修改后，也要将内容push到远端。每次同步与提交，都要把source/master两个分支的同步。

比如新部署完成的 `A电脑`上写一篇文章：

```
rake new_post['new post on new computer']

git add .
git commit -m 'create a new post in new compter'
git push origin source #push source内容到repo上

rake deploy 发布blog
```

切到另一台`B电脑`上：

```
cd octopress
git pull origin source  # 同步source分支
cd octopress/_deploy
git pull origin master  # 同步master分支， 因为之前rake dedploy会修改master的内容

```
此后，在这台`B电脑`上修改并且 `rake deploy`后，在`A电脑`上也需要先pull 完两个分支的内容。

###总结:
####1.pull source 分支、pull master分支
####2.写文章、改主题...... (git commit)
####3.git push origin source
####4.rake deploy（相当于push到远端master分支）

---------
