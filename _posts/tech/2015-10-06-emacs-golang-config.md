---
layout: post
title: Emacs中的Golang配置文件
category: tech
---

# 用到的插件

主要用到了如下几个插件

  * [go-mode](https://github.com/dominikh/go-mode.el)
  * [go-flymake](https://github.com/dougm/goflymake)
  * [go-code](https://github.com/nsf/gocode)
  * [go-company](https://github.com/nsf/gocode/blob/master/emacs-company/company-go.el)
  * [yasnippet](https://github.com/capitaomorte/yasnippet)

## Go-mode

这个不用多说吧，官方标配的插件，除了语法高亮外，也与几个非常好用的官方go工具集成。提供了格式化与跳转等等一些功能，配置如下

``` elisp
(require 'go-mode)

(add-hook 'go-mode-hook '(lambda ()
  (local-set-key (kbd "C-c C-r") 'go-remove-unused-imports)))
(add-hook 'go-mode-hook '(lambda ()
  (local-set-key (kbd "C-c C-g") 'go-goto-imports)))
(add-hook 'go-mode-hook '(lambda ()
  (local-set-key (kbd "C-c C-f") 'gofmt)))
(add-hook 'before-save-hook 'gofmt-before-save)

(defvar golang-goto-stack '())

(defun golang-jump-to-definition ()
  (interactive)
  (add-to-list 'golang-goto-stack
               (list (buffer-name) (point)))
  (godef-jump (point) nil))

(defun golang-jump-back ()
  (interactive)
  (let ((p (pop golang-goto-stack)))
    (if p (progn
            (switch-to-buffer (nth 0 p))
            (goto-char (nth 1 p))))))
```

同时还需要安装几个东西到 `$GOPATH`里面去，如下：

```
go get https://github.com/rogpeppe/godef
go get code.google.com/p/go.tools/cmd/godoc
```

这样搞了后呢，你就可以使用如下的快捷键了：

* `C-c C-a`： 添加一个需要 import 的 package
* `C-u C-c C-a`： 添加一个需要 import 的 package，且通过prompt的方式输入一个别名
* `C-c C-r` ：Remove 掉 import 了，但是没使用的package
* `C-u C-c C-r` ：注释掉没用到的package
* `C-c C-g` : Jump 到 import package 的地方
* `C-c C-f` ：Format 一下，在保存后也会自动format
* `C-c C-d` ：在 mini-buffer里面显示definition
* `C-.` ：跳转到定义处
* `C-,` ：跳转回来

## Go-flymake

这个需要依赖flymake，你要先装flymake，然后装go-flymake，配置如下：

``` elisp
(add-to-list 'load-path "~/.go/src/github.com/dougm/goflymake")
(require 'go-flymake)
```

然后就可以提示 syntax 错误 on the fly 了。


## Go-code

Go语法补全的工具，后端是选择的 conpany，melpa上面有现成的装上。先要安装 gocode：

```
go get -u github.com/nsf/gocode
```

然后安装 company 和 conpany-go，`M-x package-install` 直接搞定，然后配置如下：

``` elisp
(add-hook 'go-mode-hook 'company-mode)
(add-hook 'go-mode-hook (lambda ()
  (set (make-local-variable 'company-backends) '(company-go))
  (company-mode)))
```

## Yasnippet

使用的官方的提供[Yasnippet](https://github.com/AndreaCrotti/yasnippet-snippets)资源，把 go-mode 丢到 `.emacs.d/snippets` 里面即可。



# Reference

[Writing Go in Emacs](http://dominik.honnef.co/posts/2013/03/writing_go_in_emacs/)
[Emacs for Go](http://yousefourabi.com/blog/2014/05/emacs-for-go/)
