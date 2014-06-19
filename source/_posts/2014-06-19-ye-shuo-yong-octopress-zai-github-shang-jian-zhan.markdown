---
layout: post
title: "也说用 Octopress 在 Github 上建站"
date: 2014-06-19 14:15:15 +0800
comments: true
categories: [web]
---
谈用 Octopress 在 Github 上建站的教程在网上非常非常多，比如下面这两篇写的就很好：

[《记录：Octopress建站之旅》](http://jackycode.github.io/blog/2014/02/23/record/)

[《Octopress Build Blog on Github 个人建站实录》](http://sumnous.github.io/blog/2014/05/11/octopress-build-blog-on-github/)

我基本上就是看着上面这两篇文章把自己的 Blog 给建立起来的。所以在这里我不想“重复造轮子”，我是想谈谈我在操作过程中遇到的“坑”和我是如何解决的。

###0、我的环境

我是在 Mac 环境下操作的，系统版本是 10.9，其他环境是否也会遇到我下面所说的问题我不太确定。

###1、Ruby 的版本

Mac OS X 10.9 默认安装的 Ruby 版本是 2.0.0 完全符合要求。

```
ruby --version
```

这里我想说的是一开始我是尝试在 CentOS 下面进行操作的，CentOS 下 Ruby 的版本是 1.8.4 ，而 Octopress 需求的 Ruby版本号是 1.9.6 以上，否则会在执行

```
rake generate
```

这步时报错。所以一定要提前注意一下自己环境的 Ruby 版本。

###2、Gem 源

Ruby Gem 默认的源是：

```
https://rubygems.org/
```

由于一些“你懂的”的国内网络原因导致从这个源获取东西的时候会失败。特别是你在操作到如下这部的时候：

```
gem install bundler
```

如果遇到这样的网络问题，你可以考虑更换国内淘宝提供的源，具体的更换方法可直接参考淘宝源上写的[教程](https://ruby.taobao.org)。或者直接按照下面的步骤操作：

```
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
*** CURRENT SOURCES ***

https://ruby.taobao.org
# 请确保只有 ruby.taobao.org
```

###3、Rake 的版本

在 Mac OS X 10.9 环境下查看 rake 的版本

```
rake --version
rake, version 0.9.6
```

发现版本是 0.9.6 但是在 octopress 目录下打开 Gemfile 文件

```
vim Gemfile
```

发现里面配置的内容是这个样子的

```
source "https://rubygems.org"

group :development do
  gem 'rake', '~> 0.9'
  gem 'jekyll', '~> 0.12'
  gem 'rdiscount', '~> 2.0.7'
  gem 'pygments.rb', '~> 0.3.4'
  gem 'RedCloth', '~> 4.2.9'
  gem 'haml', '~> 3.1.7'
  gem 'compass', '~> 0.12.2'
  gem 'sass', '~> 3.2'
  gem 'sass-globbing', '~> 1.0.0'
  gem 'rubypants', '~> 0.2.0'
  gem 'rb-fsevent', '~> 0.9'
  gem 'stringex', '~> 1.4.0'
  gem 'liquid', '~> 2.3.0'
  gem 'directory_watcher', '1.4.1'
end

gem 'sinatra', '~> 1.4.2'
```

这导致在执行 rake 命令的时候会报错

```
Gem::LoadError: You have already activated rake 0.9.6, but your Gemfile requires rake 0.9......
```

于是我把 Gemfile 文件中的 gem 'rake', '~> 0.9' 修改成了 gem 'rake', '~> 0.9.6' 就没有问题了。

###4、zsh

我的命令行环境是 zsh 在安装站点主题的时候执行下面这个命令：

```
rake install['whitelake']
```

会报错 zsh： no matches found: install['whitelake'] 这个解决起来到容易，把命令写成下面这样：

```
rake 'install["whitelake"]'
```

就不会有问题，但是在发布新 Blog 的时候会有个“坑”，因为发布新 Blog 时候用到的命令：

```
rake new_post["title"]
```

在 zsh 下也会报同样的错，所以也要把命令修改成：

```
rake 'new_post["title"]'
```

这个时候你需要特别注意了，在执行上面的命令成功生成的 markdown 文件中 title 里的内容是错的，因为我们在输入命令时多用了一对引号，所以会导致 markdown 文件中的 title 变成错误的样子：

```
title: ""title""
```

这时如果执行 rake generate 命令时由于 markdown 文件的 title 格式错误会导致命令执行失败，所以需要手动将 markdown 文件中的 title 的多余引号去掉。如果你不是用 zsh 或者敲命令是没有带那对多余的引号就不会碰上这个问题。

###5、其他

上面说的都是我在操作过程中遇到的一些主要问题。剩下还有两点提醒：

#####1.代码部署到 GitHub 之后请耐心等几分钟

代码部署到 GitHub 之后，马上去访问 xxx.github.io 可能看到的是 404 页面，这时候不要着急，不要瞎折腾（血的教训），耐心登上几分钟再刷新一下页面就可以看到自己的站点了。

#####2.部署完毕后的多台设备同时更新网站问题

你可能也会遇到这个问题。例如像我在公司的电脑上部署好了 Octopress 站点环境，甚至还迫不及待的发了新的 Blog 或是网站内容上去，这时候你会需要在家里的台式机、自己的笔记本上也可以部署好同样的环境，然后在不同的设备上顺利的更新自己站点的内容，要如何做呢？去读读下面这个网站上的内容，这个问题会迎刃而解的。

[《Clone Your Octopress to Blog From Two Places》](http://blog.zerosharp.com/clone-your-octopress-to-blog-from-two-places/)

最后祝大家玩的愉快！