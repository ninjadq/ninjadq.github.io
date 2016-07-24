---
layout: post
title: Profiling Python 服务
category: tech
tags: [python]
---

# 生产环境中调优python

上一篇博文翻译了Nalys是如何再生产环境下进行profiling和调优的，这篇文章是在上文的基础上，利用Nylas开源的代码，教你如何线上Profiling。

## 环境配置

首先到github上将Nylas性能监控工具的代码克隆到本地

```bash
git clone https://github.com/nylas/nylas-perftools
```

为了测试这些功能，将其跑起来这些我们需要3个组件：可视化的组件、数据采集的组件、以及被监控的服务。由于他们是采用侵入式的方式对程序运行时的堆栈进行采样，所以需要略微的修改自己的服务代码，在代码中加入以下语句：

```python

import stacksampler
gevent.spawn(stacksampler.run_profiler)

```

这里其实就是开了一个协程（gevent中叫做greenlet）运行采样的代码，`stacksampler.run_profiler` 中每隔`0.005`秒对当前运行时堆栈进行采样并且格式化。数据存在内存中，同时起一个wsgi的服务，默认监听16384端口，使用http请求访问时会吐出数据。测试的demo程序如下：

```python

from __future__ import print_function

import time
import stacksampler
import gevent


def main():
    while  True:
        print('hello')
        gevent.sleep(1)

if __name__ == '__main__':
    gevent.spawn(stacksampler.run_profiler)
    main()

```

这样之后就能通过默认端口访问采用数据了

![sample](http://7o50i4.com1.z0.glb.clouddn.com/profiling-python%2Fsample_data.png)

接下来是采集和可视化的组件。因为采集的时间序列数据需要利用dbm这个库存储在本地，也有一些依赖，所以要先创建数据文件夹，安装相应的依赖：

```bash

# create a directory for data files
sudo mkdir -p /var/lib/stackcollector
sudo chmod a+rw /var/lib/stackcollector

virtualenv .
source bin/activate
python setup.py install

```

接着就能运行采集的脚本了

```python
python -m stackcollector.collector --host localhost --ports 16384 --interval 60
```

![采集脚本](http://7o50i4.com1.z0.glb.clouddn.com/profiling-python%2Fcollector.png)

此脚本会每隔60秒，调用对应的接口，采集数据并且存入本地的时间序列库。

最后是启动可视化的组件：

```python

python -m stackcollector.visualizer --port 5555

```

这个会从时序库中读出数据并且在浏览器中画出火焰图。http请求类似`http://localhost:5555?from=-15minutes`这样。

![火焰图](http://7o50i4.com1.z0.glb.clouddn.com/profiling-python%2Fflame_graph.png)


## 碰到的问题

这其中碰到了几个问题

1. 刚开始是想对一个flask服务进行监控的，那个flask服务是用gunicorn启动多个worker，绑定了一个端口，相当于就是开了多个进程，每个进程开两个端口一个提供服务，一个给采集器吐采样数据。当时这样总是启动失败，无法采样。原因不明。猜想一个是端口冲突，因为每个worker都坚听了同一个端口。另一个猜想是gunicorn的端口是在配置文件里面写的，这样的话，不知道如何映射两个端口。这个问题没有深究，后面就想先写个最简单的demo跑通再说。

2. 启动demo服务后，就被阻塞了。后面查明原因是gevent是协程库，程序自己来调度，当其他的线程阻塞后如果不显示让出使用权，当前就会一直等下去，解决办法也很简单，全部交给gevent去调度就可以了，在demo中将sleep改为gevent.sleep，在`stacksampler`中，加上 `import gevent.monkey; gevent.monkey.patch_all()`。


## 一些想法

1. 这个库有这么一个问题，每个进程都需要暴露一个端口给采集器。采集器来采集并且聚合。这样一来，你需要给每个进程实例都提供一个端口，数目少好说，但是数目一多，或者不固定的话，就很难采集数据了。因为需要涉及到动态的分配端口。所以是否可以修改代码，在采样的时候，直接将数据推送到数据聚合的服务端，并且打上自己信息的标签以区分不同服务。

2. 这一套思路应该延续到其他的带有runtime的环境，可否实现一个Go语言版本。
