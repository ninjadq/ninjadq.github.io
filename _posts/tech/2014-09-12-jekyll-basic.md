---
layout: post
title: Jekyll 的很弱智的东西 
date: 2014-09-12
category: tech
tags: [jekyll]
---

# Jekyll的安装
Jekyll是一个博客框架，可以很容的就生成一个博客系统，而且能够部署到github page上，这样就可以很方便的得到一个功能强大而且免费的个人博客系统了。而且博文用markdown编写，这样你就可以用自己最喜欢的编译器写博文啦！当然咯，安装jekyll之前需要安装一些它依赖的环境，其中包括*`Ruby`*、*`RubyGems`*、*`NodeJS`*，安装都很容易。
# Jekyll入门
Jekyll是一个Ruby Gem，所以可以用gem安装，根据下列步骤，很容易就可以自动生成一个最基本的博客系统啦
{%  highlight  bash  %}
~ $ gem install jekyll
~ $ jekyll new myblog
~ $ cd myblog
~/myblog $ jekyll serve
# => Now browse to http://localhost:4000
{%  endhighlight %}

此时可以通过浏览器的`localhost:4000`来访问了

# 部署到Github Page上
接着上面的步骤
{%  highlight  bash  %}
~ $ git init
~ $ git add .
~ $ git commit -am "init commit"
~ $ git push master https://github.com/ninjadq/ninjadq.github.io.git
{%  endhighlight %}

网站改成你自己的github page的地址，也就是新创建一个repo然后取名字为username.github.io就可以了，其中username改成你自己的名字。这样就可以通过ninjadq.github.io来访问了。

# 绑定域名
如果想绑定自己的域名的话也很方便

1. 在域名托管网站新加一个A记录，然后IP地址填写`192.30.252.153`
2. 在你的repo中添加一个文件名为CNAME的文件，里面的内容为你的域名，比如`ninjadq.com`。
