---
layout: post
title: Guard与rspec的安装与配置
category: tech
tags: [rails, test, rspec, model]
---

# Guard与RSpec

**Guard** 可以用来监控文件的变化，由文件的变化可以触发一系列自己设置好的动作。

**RSpec** 可以是一个灰常好用的ruby的Test框架

这两个货混到一起会有啥效果呢？没错了，他们的配合可以很好的完成自动化单元测试这项艰巨的任务。下面来看如何安装和配置他们吧。

## RSpec 安装

灰常简单，输入命令即可

```bash
gem install rspec
```

你可以测试一下，下面是官网的例子。
1. 新建一个`game_spec.rb`的文件，里面内容为

```ruby
require './game'
describe Game do
  describe "#score" do
    it "returns 0 for an all gutter game" do
      game = Game.new
      20.times {game.roll(0)}
      expect(game.score).to eq(0)
    end
  end
end
```
2. 现在在终端里面输入 `rspec`，当然，测试是无法通过的。因为你还没有开始写代码。
3. 现在来创建一个新的文件`game.rb`，将代码写进去吧。

```ruby
class Game
  def roll(pins)
  end

  def score
    0
  end
end
```
4. 好了，现在再运行`rspec`就可以发现测试通过了。

## Guard的安装
1. 首先创建一个`Gemfile`，然后里加上这些内容

```ruby
source 'https://rubygems.org'

group :development do
  gem 'guard'
  gem 'guard-rspec', require: false
end
```

2. 然后输入`bundle`命令安装
3. 命令行输入下面命令生成`guardfile`

```bash
bundle exec guard init
```

## Guard与RSpec合体
其实现就可以使用了，但是怎么让他们方便的合体呢？有一个Gem，Guard-Rspec可以帮助你。

1. 在Gemfile的develop组添加 `gem 'guard-rspec', require: false`
2. 命令行中输入下面命令，在`guardfile`添加一些配置内容
```ruby
guard init rspec
```
3. 好了，高完后，就可以直接输入`guard`运行啦～
