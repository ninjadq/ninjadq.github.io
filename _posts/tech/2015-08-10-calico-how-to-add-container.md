---
layout: post
title: 当把一个容器加入 Calico 网络的时候，Calico 在做什么
category: tech
tags: [network, docker, calico]
---

本文是介绍calico中如何将container加入到calico网络中

calicoctl中添加container时,首先调用了`container_add(container_name, ip, interface)`, 其中第一个参数是容器的名字, 第二个参数是赋予容器的ip地址, 第三个参数是容器内部绑定此ip的device的名字。函数做的事情如下：

  1. 函数首先查询 `/calico/v1/host/%(hostname)s/workload/%(orchestrator_id)s/%(workload_id)s/` 看这个container是否已经存在
  2. 然后确定容器是在运行
  3. 检查network mode不为host.
  4. 确认分配的ip地址在pool中.
  5. 在从etcd中通过hostname获取下一跳的地址.
  6. 然后再在pool中将将ip赋值,并跟新到etcd中.
  7. 根据提供的信息,创建一个endpoint,生成一个endpoint的id并将其信息存放到etcd中.

Host部分步骤如下：

  1. 运行, 创建一个虚拟网卡

    * `ip link add :endpoint_name type veth peer name :endpoint_temp_iface_name`
    * `ip link set :endpoint_name up `

  2. namespace， 创建一个网络命名空间，把网卡加到特定的命名空间里去

    * `os.mkdirs("/var/run/netns")`
    * `os.symlink("/proc/%s/ns/net" % cpid, "/var/run/netns/%s" % ns_name)`, cpid是容器的pid,name是随机生成的uuid
    * `ip link set :endpoint_temp_iface_name netns ns_name`
    * `ip netns exec ip link set dev :endpoint_temp_iface_name name :interface`
    * `ip netns exec ip link set :interface up`
    * `os.unlink("/var/run/netns/%s" % ns_name)`

  3. 给命名空间的虚拟网卡加上ip

    * `os.mkdirs("/var/run/netns")`
    * `os.symlink("/proc/%s/ns/net" % cpid, "/var/run/netns/%s" % ns_name)`, cpid是容器的pid,name是随机生成的uuid
    * `ip netns exec ip -4 addr addr :ip/:length dev :interface `
    * `os.unlink("/var/run/netns/%s" % ns_name)`

  4. 给网络命名空间加上路由

    * `os.mkdirs("/var/run/netns")`
    * `os.symlink("/proc/%s/ns/net" % cpid, "/var/run/netns/%s" % ns_name)`, cpid是容器的pid,name是随机生成的uuid
    * `ip netns exec ip -4 route replace :next_hop dev interface`
    * `ip netns exec ip -4 route replace default via :next_hop dev :interface`
