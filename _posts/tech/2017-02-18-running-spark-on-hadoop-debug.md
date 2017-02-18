---
layout: post
title: 记录一次Spark在Hadoop集群上运行时虚拟内存超出限制
category: tech
tags: [distributed system, yarn, spark]
---

在内网搭建了一套Hadoop，接着想在yarn上运行spark-shell，之前单机运行spark-shell没有任何问题，但是当跑到yarn集群的时候问题就来了。
运行时候的报错总是莫名其妙，Google错误后，发现都不是我的情况，所以一直卡了两天都没有头绪，后来看到so上说，可以查每个节点上面的错误信息，查了yarn-nodemanager的日志，
发现了问题的所在，日志里面有这么一条

```
Container [xxx] is running beyond virtual memory limits. Current usage: 324 MB of 1 GB physical memory used; 2.3 GB of 2.1 GB virtual memory used. Killing container. 
```

原来是超出了内存的限制，所以改配置文件就可以了，我就顺手改了配置，将 `yarn.nodemanager.resource.memory-mb`的值改为了8192，结果还是报错

这时候发现原来这个只是改变了每台机器划分来给yarn的调度的内存的大小而不是其给每个容器分配的内存大小，后来继续Google后才知道了容器资源分配相关的一些参数，如下：

| 参数 | 默认值 | 描述 |
|---|---|---|
| yarn.scheduler.minimum-allocation-mb | 1024 | 每个container请求的最低jvm配置，单位m。如果请求的内存小于该值，那么会重新设置为该值 |
| yarn.scheduler.maximum-allocation-mb | 8192 | 每个container请求的最高jvm配置，单位m。如果大于该值，会被重新设置 |
| yarn.nodemanager.resource.memory-mb | 8192 | 每个nodemanager节点准备最高内存配置，单位m |
| yarn.nodemanager.vmem-pmem-ratio | 2.1 | 虚拟内存和物理内存之间的比率，如果提示virtual memory limits的时候，可以将该值调大 |
| yarn.nodemanager.pmem-check-enabled | true | 是否进行物理内存限制比较，设置为false，不会进行大小的比较 |
| yarn.nodemanager.vmem-check-enabled | false | 是否进行虚拟内存限制比较。|

所以这个异常可以通过一下方式解决

1. 修改物理或者虚拟的内存值
2. 改大虚拟内存和物理内存的的比例
3. 关闭虚拟内存检查

总结：发生问题不要慌，显示的错误不一定是真正的错误，先找日志，可能日志在本机，也可能不在