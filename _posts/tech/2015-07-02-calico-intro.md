---
layout: post
title: Project Calico的介绍
category: tech
tags: [network, docker]
---

# Calico 简介
Calico自称是一种新的虚拟网络，可以使用纯粹的三层方法与云编排系统集成，使得虚拟
机，容器，或者workload之间的安全的IP通信成为可能。

基于Internet中scalable的IP网络的概念，Calico自己搞了一个幺蛾子叫做vRouter，每个
compute node上都拥有一个vRouter，它利用了Linux内核的转发引擎，而不是vSwitch。
vRouter通过BGP协议传播各个workload之间的可达性信息（即路由）至数据中心其他结点。
小规模部署的时候可以直接部署，大规模部署的时候可以利用BGP route reflector。

Calico直接与数据中心的物理组件（L2设备和L3设备）互为peer，不需要NAT, tunnel,
overlay这些鬼。

Calico同时还支持丰富和弹性的网络策略，在每个compute node上通过使用隔离ACL来提供
tenant隔离，securtiy groups,以及一些额外的reachability的限制。

#Calico 优势

与传统的二层网络解决方案比较，calico所具备的优势有

  * 报文的传输中不需要额外的封装和解包，在二层解决方案中，跨主机的报文，通常要封装成隧道协议
  * Calico网络中，在不同的tenant's workload之间的报文的路由的行为与在单个tenant的workload中路由相同。
  * Calico更加容易理解和调试，传统的`ping`,`traceroute`可以用来测试连通性，`tcpdump`,`wireshark`可以用来监控流量，因为所有的Calico报文就是IP报文。
  * 安全策略以统一的方式指定和（使用ACLs）实现（利用iptables）使calico有更强的正确性和鲁棒性。而二层的解决方案及其复杂。

#Calico Data Path

在Calico 网络中，workload中进出的IP报文的路由和被墙是通过workload所在的host上
的路由表和iptables决定的。Calico保证无论workload怎么乱搞，都能返回下一跳MAC地址
为其所在host。当有一个发送给某workload的报文时，最后一跳是从目的workload的host
到workload。

假设一些workload的IPv4地址被分配为某数据中心的私有子网10.65/16，而他们的host们
属于IP网段172.18.203/24，这时候，你查看路由表，会看到类似下面这样的信息。

```
ubuntu@calico-ci02:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.18.203.1    0.0.0.0         UG    0      0        0 eth0
10.65.0.0       0.0.0.0         255.255.0.0     U     0      0        0 ns-db03ab89-b4
10.65.0.21      172.18.203.126  255.255.255.255 UGH   0      0        0 eth0
10.65.0.22      172.18.203.129  255.255.255.255 UGH   0      0        0 eth0
10.65.0.23      172.18.203.129  255.255.255.255 UGH   0      0        0 eth0
10.65.0.24      0.0.0.0         255.255.255.255 UH    0      0        0 tapa429fb36-04
172.18.203.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

这里面有一个workload的ip地址为10.65.0.24，能够通过一个TAP（或者veth, etc.）
interface名字为tapa429fb36-04来访问。这是因为有一个直接的路由器通过
tapa429fb36-04连接10.65.0.24。后缀为.21, .22 and .23 的workload都在
172.18.203.126 和 .129这两个host上，所以这些workload的路由要通过那些host。

直接的路由通过Calico的agent即Felix，来设置，当他请求为一个workload提供联通的时候。
然后一个BGP客户端通知和分发这些信息，到其他的host上的BGP客户端，因此非直接的路由
也就出现了。


# Calico 系统架构
Calico由以下系统组件组成
  * Felix
  * Orchestrator Plugin
  * etcd
  * BGP client
  * BGP Route Reflector

## Felix
Felix是一个运行在每一台提供endpoint的机器上deamon，Felix的主要功能有以下：

  * 接口管理：Felix把一些接口的信息编程到Kernel中去，以正确的处理各个endpoint发出的报文(其实主要就是改MAC)
  * 路由编程：将到达某个endpoint的路由信息编程到它的host的内核FIB中去，以保证到达host的报文正确转发。
  * ACL编程：编程内核中的ACLs，以提供安全策略。
  * 状态报告：Felix会报告一些关于host的网络健康信息，尤其会汇报一些配置的错误和问题，这些消息都会写到etcd中去

## Orchestrator Plugin
没有一个单独的“Orchestrator Plugin”，有的是为每个不同的云平台提供的不同的插件。这些插件的目的都是为了让Calico能够更加与云平台紧密结合

  * API translate：不同的云平台都有自己的api，所以Orchestrator很重要的一个作用就是翻译这些api到colico来
  * 反馈：如果有需要，Orchestrator还会反馈一些colico的信息给云平台

## etcd

  * 通信：Calico主要使用etcd一致性存储的特性，来提供不同组件之间的通信。
  * 数据存储：etcd以分布式，一致性，容错性的特点存储数据。


## BGP Client

## BGP Route Reflector



# Demo环境搭建

我是在两个ubuntu的虚拟机下搭建的demo,搭建的方式如下：

  1. 创建两个纯粹的Ubuntu 1404虚拟机
  2. 在虚拟机中安装最新版的docker和etcd
  3. 启动etcd可以用命令 `etcdctl ls /`命令来检测etcd是否能用
  4. 获取calico的二进制文件命令如下

    ```
    wget http://projectcalico.org/latest/calicoctl
    chmod +x calicoctl
    ```

  5. 预加载Calico Docker image

    ```
    docker pull calico/node:latest
    ```

## Demo测试

假设两台测试机alpha和beta的ip地址为172.17.8.101和172.17.8.102

##启动Calico服务

在alpha中输入

```
sudo ./calicoctl node --ip=172.17.8.101
```

在beta中输入

```
sudo ./calicoctl node --ip=172.17.8.102
```

可以输入

```
sudo docker ps 
```
以查看容器是否启动。

## Routing 通过Powerstrip

Calico使用Powerstrip在容器启动的时候自动配置网络，由于对于Docker API的调用需要通过Powerstrip代理，所以最简单的办法就是设置环境变量

```
export DOCKER_HOST=localhost:2377
```

如果你有一个容器，你想直接在里面运行命令，而不通过代理，比如`docker attach`和`docker exec`命令直接与Docker daemon交互，这时你可以这样

```
DOCKER_HOST=localhost:2375 docker attach node1
```

同样当`attach`的时候记得对此敲入回车键，然后根据提示使用`Ctrl-P, Q`而不是`exit`，来退出，否则，容器会一直运行。

##创建互联的endpoints

在alpha中输入
```
docker run -e CALICO_IP=192.168.1.1 --name workload-A -tid busybox
docker run -e CALICO_IP=192.168.1.2 --name workload-B -tid busybox
docker run -e CALICO_IP=192.168.1.3 --name workload-C -tid busybox
```

在beta中输入

```
docker run -e CALICO_IP=192.168.1.4 --name workload-D -tid busybox
docker run -e CALICO_IP=192.168.1.5 --name workload-E -tid busybox
```

添加profile，命令如下

```
./calicoctl profile add PROF_A_C_E
./calicoctl profile add PROF_B
./calicoctl profile add PROF_D
```

将相应的endpoint加入各自profile

在alpha中输入

```
./calicoctl container workload-A endpoint-id show
./calicoctl endpoint <workload-A's Endpoint-ID> profile append PROF_A_C_E

./calicoctl container workload-B endpoint-id show
./calicoctl endpoint <workload-B's Endpoint-ID> profile append PROF_B

./calicoctl container workload-C endpoint-id show
./calicoctl endpoint <workload-C's Endpoint-ID> profile append PROF_A_C_E
```

在beta中输入

```
./calicoctl container workload-D endpoint-id show
./calicoctl endpoint <workload-D's Endpoint-ID> profile append PROF_D

./calicoctl container workload-E endpoint-id show
./calicoctl endpoint <workload-E's Endpoint-ID> profile append PROF_A_C_E
```

ping测试

```
docker exec workload-A ping -c 4 192.168.1.3
docker exec workload-A ping -c 4 192.168.1.5
```

这样应该ping不通

```
docker exec workload-A ping -c 4 192.168.1.2
docker exec workload-A ping -c 4 192.168.1.4
```

自动ip生成，ip设为`auto`即可

```
docker run -e CALICO_IP=auto -e CALICO_PROFILE=PROF_A_C_E --name workload-F -tid busybox
docker exec workload-A ping -c 4 192.168.1.6
```


