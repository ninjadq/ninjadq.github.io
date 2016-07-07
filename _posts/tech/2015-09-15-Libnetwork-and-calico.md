---
layout: post
title: Docker 容器网络模型 CNM 简介和其与 Calico 的结合
category: tech
tags: [network, docker, calico]
---

## Container Network Modal

libnetwork 提出了一个很妖的模型叫做草泥马（CNM），如图所示：

![CNM](https://blog.docker.com/media/2015/04/cnm-model.jpg)

图中有三个重要的幺蛾子

1. Sandbox
  这是一个隔离的环境，所有的docker容器相关的网络配置都存放在这里
2. Endpoint
  Endpoint是某个Network的接口，在一个特定的网络中就通过Endpoint来进行通信，一个Endpoint只能加入一个网络，但是一个Sandbox可以拥有多个Endpoint
3. Network
  网络本质上就是一组特定的Endpoint的标识符，这组内的Endpoint可以相互通信。

网络和和容器有如下约定：
* 同一个网络中的容器可以相互通信
* 分流容器网络流量主要利用多网络的放肆实现，所以所有的Driver都要支持多网络
* 一个容器可以通过多Endpoint的方式，而属于多个网络
* 一个Endpoint加入了一个SandBox之后，才能获得网络连通性

## Docker Command For Network

Docker 将会支持network相关的命令，这些命令用由libnetwork来实现，命令用法如下

* docker network

```bash
Usage: docker network [OPTIONS] COMMAND [OPTIONS] [arg...]

    Commands:
        create                   Create a network
        rm                       Remove a network
        ls                       List all networks
        info                     Display information of a network

    Run 'docker network COMMAND --help' for more information on a command.

      --help=false       Print usage
```

* docker service

```bash
    Usage: docker service COMMAND [OPTIONS] [arg...]

    Commands:
        publish   Publish a service
        unpublish Remove a service
        attach    Attach a backend (container) to the service
        detach    Detach the backend from the service
        ls        Lists all services
        info      Display information about a service

    Run 'docker service COMMAND --help' for more information on a command.

      --help=false       Print usage
```

其中network类似于cnm的network，而service是对endpoint的一层封装。在docker network下，你要将一个容器加入网络需要先创建一个网络，然后publish一个service，最后attach一个容器到service里面，就创建好了，ep：

```bash
docker network create -d brige foo
docker service publish my-service.foo
docker service attach <container-id> my-service.foo
```

或者在创建容器的时候一次性到位

 ```bash
 docker run -itd --publish-service db.foo.brige busybox
 ```

## Calico plug-in
libnetwork提供了一套driver的接口，只要实现了这一套接口，那么就可以作为一个插件的方式来提供网络服务，最新版本的calico中则提供了libnetwork功能的插件，用法如下

```bash
## 首先需要启动calico node并加上 libnetwork参数
sudo calicoctl node --libnetwork

## 然后在启动容器的时候指定参数
docker run --publish-service srvA.net1.calico --name workload-A -tid busybox
```

很方便很简单 ～

calico libnetwork插件的原理实际上就是把网络的配置信息存在etcd中，当加入容器的时候，利用netns创建一个veth，然后把容器相关的信息存ETCD，与普通的calico创建差不多，但是简化了过程，而且，与docker更好的集成
