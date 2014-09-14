---
layout: post
title: 重新装机后ubuntu的一些基本配置
category: tech
tags: [ubuntu, linux]
---

# step1 安装Chrome
# step2 安装字体noto字体 http://www.google.com/get/noto/#/
* 修改文件`/etc/fonts/conf.d/49-sansserif.conf`的第四项为 `Noto Sans S Chinese`
* 安装monaco

  ```bash
  curl -kL https://raw.github.com/cstrap/monaco-font/master/install-font-ubuntu.sh | bash
  ```
* 安装 sourcecode pro
# step3  安装virtualbox
# step4  安装win8.1在vbox中
# step5  安装vlc
# step6  安装 Ubuntu restricted extras
# step7  安装7zip
# step8  安装git
# step9  安装emacs
# step10 安装sublime
 1. http://askubuntu.com/questions/172698/how-do-i-install-sublime-text-2-3

```bash
sudo 
sudo apt-get 
sudo apt-get install sublime-text
```
 2. install sulime package controll
  * https://sublime.wbond.net/installation#st2
 3. install Sass syntax highlight
     Preference > Package Control. Select `Install Package` and select Sass package 
 4. install Rails Turorial snippets
 5. Install Sublime Text Alternative Auto-completion
 6. Install SublimeERB
 7. Install RubyTest
# step11 安装搜狗输入法
 1. 先使用apt-get安装fcitx

```bash
 sudo apt-get install fcitx
```
 2. 官网下载deb包，然后安装就可以了
 3. 如果显示没有搜狗输入法，可以在配置里面找到，并添加

# step12 skype安装
skype是唯一一个跨所有平台的可以语音，视频，发送文件的im,安装很简单

1. 在源里面添加third party的源，然后输入

```bash
  sudo apt-get install skype
```

2. 如果要设置skype的字体和大小，可以安装一个qt-config配置

```bash
sudo apt-get install qt4-qtconfig
```
  然后在里面设置就可以了

