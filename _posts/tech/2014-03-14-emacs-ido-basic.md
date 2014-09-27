---
layout: post
title: Emacs的Ido插件的基本用法
category: tech
tags: [emacs, id]
---

# IDO是啥？
ido是emacs下的一个用来切换缓冲区以及查找文件的神器，从emacs22开始就是emacs的一部分了

# 激活ido
在`*.emacs*`文件中添加下面的代码

```elisp
(require 'ido)
(ido-mode t)
```

然后可以配置一下

```elisp
M-x customize-group RET ido RET
```

# 使用IDO
## 在buffer之间各种跳转，输入`C-x b`，然后

* 输入buffer名的字符，`RET`后就可以浏览排在第一位的buffer了
* 可以使用`C-s`和`C-r`在list中间前后移动
* [TAB]可以补全buffer名
* 使用`C-f`可以转换为发现文件(带ido-mode)，`C-b`返回到switch-to-buffer(不带ido-mode)

## 查找文件，输入`C-x C-f`

* 输入字符，按下回车就选中list中的第一个文件或者文件夹
* `C-s`和`C-r`可以在整个list中来回选择
* `[TAB]`用来补全在list中所有可能的结果
* `backspace`进入上一级目录
* `//`进入root目录
* `~/`进入home目录
* `C-f`回到没有ido-mode的查找文件功能，`C-b`回到带有ido-mode切换buffer功能
* `C-d`打开当前的文件夹
* `C-j`创建一个新的文件

## 最近访问的文件夹

* 输入`M-p`和`M-n`来选择历史记录中的文件夹
* `M-s`查找匹配文件的输入
* 从历史记录中删除当前的文件夹
