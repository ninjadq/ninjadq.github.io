---
layout: post
title: calicoctl详解
category: tech
tags: [network, docker]
---

本文是介绍calicoctl的命令,calicoctl所有的命令都是以calicoctl开头

### node

* 启动node

    ```
    calicoctl node --ip=<IP> [--node-image=<DOCKER_IMAGE_NAME>] [--ip6=<IP6>] [--as=<AS_NUM>]
    ```

* 停止node

    ```
    calicoctl node stop [--force]
    ```
 
* 手动添加一个bgp peer

    ```
    calicoctl node bgppeer add <PEER_IP> as <AS_NUM>
    ```
    
* 删除一个bgp peer

    ```
    calicoctl node bgppeer remove <PEER_IP>
    ```
   
* 展示所有的bgp peer

    ```
    calicoctl node bgppeer show [--ipv4 | --ipv6]
    ```

### profile

Profile 主要是来做acl的

* 显示所有的profile

    ```
    calicoctl profile show [--detailed]
    ```

* 增删一个profile
    
    ```
    calicoctl profile (add|remove) <PROFILE>
    ```
    
* 显示某个profile的tags

    ```
    calicoctl profile <PROFILE> tag show
    ```
    
* 给某个profile增删profile

    ```
    calicoctl profile <PROFILE> tag (add|remove) <TAG>
    ```
    
* 给某个profile 增加acl规则

    ```
    calicoctl profile <PROFILE> rule add (inbound|outbound) [--at=<POSITION>]
    %(rule_spec)s
    ```

* 给某个profile 删除acl规则

    ```
    calicoctl profile <PROFILE> rule remove (inbound|outbound) (--at=<POSITION>|
    %(rule_spec)s)
    ```

* 显示profile的所有规则

    ```
    calicoctl profile <PROFILE> rule show
    ```

* json 
    ```
    calicoctl profile <PROFILE> rule json
    ```
    
* 更新 profile的rule
    ```
    calicoctl profile <PROFILE> rule update
    ```

### pool

命令如下

* 增加一个ip池
   ```
   calicoctl pool (add|remove) <CIDR> [--ipip] [--nat-outgoing]
   ```
 
* 展示所有的ip池
   ```
   calicoctl pool show [--ipv4 | --ipv6]
   ```

### bgp

相关命令

```
calicoctl default-node-as [<AS_NUM>]
calicoctl bgppeer add <PEER_IP> as <AS_NUM>
calicoctl bgppeer remove <PEER_IP>
calicoctl bgppeer show [--ipv4 | --ipv6]
calicoctl bgp-node-mesh [on|off]
```

### container

    ```
    calicoctl container <CONTAINER> ip (add|remove) <IP> [--interface=<INTERFACE>]
    calicoctl container <CONTAINER> endpoint-id show
    calicoctl container add <CONTAINER> <IP> [--interface=<INTERFACE>]
    calicoctl container remove <CONTAINER> [--force]
    ```
    
    * `calicoctl container add <CONTAINER> <IP> [--interface=<INTERFACE>]`
        1. 保证要有root权限
        2. 通过inspect_container,从容器名称获取容器信息,并获取容器id
        3. 检验etcd中`/calico/v1/host/%(hostname)s/workload/docker/%(container_id)s/endpoint/`键的所有叶子,然后通过第一个叶子的中的值的取endpoint_id值,以确定它是否加入calico网络,未加入才能继续
        4. 通过etcd中`/calico/v1/ipam/%(version)s/pool`键,获取ip_pools,如果ip在这个ip_pool中,则选中此pool
        5. 通过etcd中`/calico/v1/host/%(hostname)s/bird_ip`,获取下一跳ip地址(如果有v6则还将最后改为bird6_ip,再获取v6的地址)
        6. 在etcd中键`/calico/v1/ipam/%(version)s/assignment/%(pool)s/%(address)s`写入空字符串,如果之前已经存在,则报错,已确认地址未分配
        7. 通过容器info["State"]["Pid"]获取pid
        8. 若没有endpoint_id则生成一个uuid给他
        9. interface的名字为`cali`+ uuid 的后面11位, tmp_interface名字为`tmp` + uuid后11位
    
    * `calicoctl container <CONTAINER> ip (add|remove) <IP> [--interface=<INTERFACE>]`
    
    * `calicoctl container <CONTAINER> endpoint-id show`
        1. 以容器的名字为参数,调用docker\_client的inspect_container,获取容器信息
        2. 通过容器信息的id选项获取容器id
        3. 通过hostname,orchestrator_id,和上一步获取的容器id,在etcd中查找`/calico/v1/host/%(hostname)s/workload/%(orchestrator_id)s/%(workload_id)s/endpoint/`中存储的值,并返回

### endpoint
endpoint 指calico网络模型中的一个结点

```
calicoctl endpoint show [--host=<HOSTNAME>] [--orchestrator=<ORCHESTRATOR_ID>] [--workload=<WORKLOAD_ID>] [--endpoint=<ENDPOINT_ID>] [--detailed]
calicoctl endpoint <ENDPOINT_ID> profile (append|remove|set) [--host=<HOSTNAME>] [--orchestrator=<ORCHESTRATOR_ID>] [--workload=<WORKLOAD_ID>]  [<PROFILES>...]
calicoctl endpoint <ENDPOINT_ID> profile show [--host=<HOSTNAME>] [--orchestrator=<ORCHESTRATOR_ID>] [--workload=<WORKLOAD_ID>]
```

### other

```
calicoctl diags [--upload]
calicoctl checksystem [--fix]
calicoctl restart-docker-with-alternative-unix-socket
calicoctl restart-docker-without-alternative-unix-socket
```
