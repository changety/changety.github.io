---
layout: post
title: "使用github page与octopress搭建个人blog"
subtitle : "Blog based on Github page & Octopress"
date: 2014-10-12 03:00:00 +0800
comments: true
categories: [octopress]
---

#前言
前段时间云音乐Android小组打算自己搭建一个blog，用于总结跟记录工作中遇到的技术问题。之前自己在其他地方记录过几篇blog，也一直想搭个blog的玩玩，借此机会正好操练。
出于对Github极大的好感，自然就想到利用Github Page服务搭建一个blog，网络上已经存在大量关于利用Github与Octopress搭建blog的文章，但是质量参次不齐，实际我在搭建的过程中就遇到了不少问题，这里记录一下其他文章中忽略的一些细节，梳理一下流程，同时帮碰到相同问题的同学节省时间。可能少有人会跟我一样折腾，还打算在一台mac上维护两个octopress blog、其中一个octopress blog 还需要在多台设备上一起更新（android 小组），因此这篇blog先介绍创建一个常规octopress blog的步骤，之后再介绍下一台电脑上维护两个octopress blog 以及 一个octopress blog 在多个地方(电脑)上更新的方法。

---------

#配置github信息
默认你已经创建了github帐号，如果还没有，先到[github](https://github.com/)上注册一个，记得起一个酷炫或者有特殊意义的`username`，这会是个稍候blog域名的prefix。并且默认你已经完成本机上完成github帐号信息的配置，如果还没有，请查看[github help](https://help.github.com/articles/set-up-git/)

---------


#节省漫长的十多分钟等待时间
首先到[github](https://github.com/new)创建一个`username.github.io`的repo，`username.github.io`以后就是blog的域名（当然他支持使用自定义的域名）， username就是你在github上的username。之所以先完成这一步，是因为github一般需要10分钟左右来同步缓存数据，一般创建完repo后，直接访问，浏览器会告诉你这个：

{% img /images/why_this_first.png %} 

---------


#准备安装Octopress所需的环境
[这里](http://octopress.org/docs/setup/)是Octopress的官方指南，里面很精简的描述了安装步骤，我这里再啰嗦一次（主要是想讲坑

安装octopress时，首先需要git跟ruby环境，这两样工具，一般mac都已经自带了，可以在terminal里分别

```
$ git --version
```
git的版本号对安装octopress影响不大，这里不用关心，只要有git工具就可以了。

```
$ ruby --version
```
如果打印出来的ruby版本信息是1.9.3，那接下来的安装会省却不少麻烦。ruby的向下兼容不好，因此基于Ruby所做的框架大多要求特定版本，octpress也是，[这里](http://octopress.org/docs/setup/)就提到需要ruby 1.9.3这个版本，我当时想试试mac自带的2.1.1这个ruby版本上的安装，结果是一坑接一坑，会缺各种各样依赖，所以不想折腾的朋友老实安装1.9.3这个版本好了。ruby多版本管理可以用RVM和rbenv。在mac下，推荐使用[Homebrew](http://brew.sh/)来安装rbenv（ruby社区普通推荐rbenv），如果你没有Homebrew，打开终端，先安装完Homebrew，根据终端一步一步安装完就好，基本不会遇到什么问题。

```
$ ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"
```

有了Homebrew就可以安装rbenv了

```
$ brew update 
$ brew install rbenv
$ brew install ruby-build
```
使用rbenv安装1.9.3版本的ruby，ruby1.9.3下还有多次发行版，稳妥期间，推荐直接安装1.9.3-p125，参考过其他blog的经验

```
$ rbenv install 1.9.3-p125
$ rbenv local 1.9.3-p125 #Sets a local application-specific Ruby version
$ rbenv rehash
$ ruby --version #ruby 1.9.3p125 (2012-02-16 revision 34643) [x86_64-darwin13.1.0]
```
---------

安装完成后可以用ruby --version进行验证，那么问题就来了，虽然上面敲了rbenv local设置在当前的ruby版本环境，终端很有可能显示的还是mac自带的版本，原因是因为还没有在
```
cd ~/.bash_profile
```
配置rbenv的环境变量，具体步骤如下： （更详尽的配置说明参考[这里](https://github.com/sstephenson/rbenv)

Add ~/.rbenv/bin to your $PATH for access to the rbenv command-line utility.

```
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
```
Ubuntu Desktop note: Modify your ~/.bashrc instead of ~/.bash_profile.
Zsh note: Modify your ~/.zshrc file instead of ~/.bash_profile.

Add rbenv init to your shell to enable shims and autocompletion.

```
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
```
Same as in previous step, use ~/.bashrc on Ubuntu, or ~/.zshrc for Zsh.
```
$ source ~/.bash_profile
```

然后再切到~/octopress下，
```
$ ruby --version #此时应该已是1.9.3p125 (2012-02-16 revision 34643)
```
---------

#安装Octopress
准备完毕octopress所需的环境后，就可以按照官方指南安装Octpress了

###clone octopress

```
$ git clone git://github.com/imathis/octopress.git octopress
```

###安装依赖

```
$ cd octopress
$ gem install bundler
$ rbenv rehash
$ bundle install
```

###安装octopress默认主题

```
$ rake install
```

接下去会专门写一篇关于octopress主题的blog

---------

运行以下命令，仔细看提示完成github和Octopress的关联（就是第一步创建的这个repo https://github.com/username/username.github.io

```
$ rake setup_github_pages
```
---------


#创建博客

###生成博客

```
$ rake generate 
```

###上传代码

```
$ git add .
$ git commit -m 'create first blog'
$ git push origin source
```

###部署博客

```
$ rake deploy
```

这里`origin`代表与octopress关联的repo
此时如果顺利完成后就能直接访问`http://username.github.io`看到自己的博客了， 由于首先创建了repo，就不需要等待10分钟了。

###修改配置
配置文件路径为`~/octopress/_config.yml`

```
url:                # For rewriting urls for RSS, etc
title:              # Used in the header and title tags
subtitle:           # A description used in the header
author:             # Your name, for RSS, Copyright, Metadata
simple_search:      # Search engine for simple site search
description:        # A default meta description for your site
date_format:        # Format dates using Ruby's date strftime syntax
subscribe_rss:      # Url for your blog's feed, defauts to /atom.xml
subscribe_email:    # Url to subscribe by email (service required)
category_feeds:     # Enable per category RSS feeds (defaults to false in 2.1)
email:              # Email address for the RSS feed if you want it.
```

编辑完成后

```
$ rake generate

$ git add .
$ git commit -m "settings" 
$ git push origin source

$ rake deploy
```

---------

#写博客

到此为此博客已经成功搭建，赶紧测试一篇`hello world`的blog压压惊

###创建博文

```
$ rake new_post['hello world']
```

生成的blog文件在`~/octopress/source/_posts`目录下，接下来可以用你所熟悉的markdown工具写博客了。

###使用markdown写博文
如果一开始对markdown的语法不熟悉，这里推荐一个[在线学习markdown语法的网站](http://dillinger.io/))，所见即所得；或者可以直接其他blog上的用markdown写的文章，fork下来，参考样式与语法的比对，这样上手还是很快的。
写完后使用以下指令，并在浏览中输入`localhost:4000`, 查看blog的效果

```
$ rake preview #localhost:4000
```

调试完后，生成blog并部署到github page上


```
$ rake generate

$ git add .
$ git commit -m "comment" 
$ git push origin source

$ rake deploy
```

以上便是创建一个octopress blog的全部过程了，下一篇会继续说明如果在一台电脑上管理多个octpress blog 以及 一个octopress在多台电脑上共同维护的方法。

---------

#参考资料

* http://octopress.org/
* http://msching.github.io/blog/2014/04/11/starting/
* http://stackoverflow.com/questions/10940736/rbenv-not-changing-ruby-version
