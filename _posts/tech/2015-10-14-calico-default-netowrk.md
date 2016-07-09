---
layout: post
title: Calico 网络启动相关代码
category: tech
tags: [docker, calico]
---

# Calico 三种工作模式

Calico在docker环境下，现在有三中工作模式即，powerstrip、docker default network、libnetwork。

曾今calico大量的依赖powerstrip，但是由于powerstrip很多不方便，所以基本上已经废弃，calico也打算放弃powerstrip，这个[issue](https://github.com/projectcalico/calico-docker/issues/546)里面提到，打算在1.9的 Docker 发布后，就删除关于Powerstrip的文档，估计离删除Powerstrip的代码也远了。

Libnetwork是docker网络未来发展的方向，calico也支持Libnetwork，但是这个特性一直在experimental中，从稳定性的方面考虑，所以现在也不推荐使用。

本文的重点是讨论Calico在Docker default network中的使用。此模式需要你手动的添加容器到calico网络中，所有1.6以后的docker版本都支持此功能。

## Default Network 的使用方法

1. 当配置好出初始环境，并且启动了calico node 后，首先就是要启动一个新的容器，不要给予任何网络配置，如下：

    ```bash
    docker run --net=none --name workload -tid busybox
    ```

2. 然后使用calico给其添加网络,如下：

    ```bash
    sudo calicoctl container add workload 192.168.0.1
    ```

3. 然后创建一个profile

    ```bash
    calicoctl container workload profile append PROFILE
    ```

4. 最后把容器加入一个profile中去

    ```bash
    calicoctl container workload profile append PROFILE
    ```


## Docker default network 启动源码分析

### Add container
当运行 `sudo calicoctl container add workload 192.168.0.1`时。调用的函数如下：

    ```python
    def container_add(container_id, ip, interface):
        """
        Add a container (on this host) to Calico networking with the given IP.
        :param container_id: The namespace path or the docker name/ID of the container.
        :param ip: An IPAddress object with the desired IP to assign.
        :param interface: The name of the interface in the container.
        """
        # The netns manipulations must be done as root.
        enforce_root()

        # TODO: This section is redundant in container_add_ip and elsewhere
        if container_id.startswith("/") and os.path.exists(container_id):
            # The ID is a path. Don't do any docker lookups
            workload_id = escape_etcd(container_id)
            orchestrator_id = NAMESPACE_ORCHESTRATOR_ID
            namespace = netns.Namespace(container_id)
        else:
            info = get_container_info_or_exit(container_id)
            workload_id = info["Id"]
            orchestrator_id = DOCKER_ORCHESTRATOR_ID
            namespace = netns.PidNamespace(info["State"]["Pid"])

            # Check the container is actually running.
            if not info["State"]["Running"]:
                print "%s is not currently running." % container_id
                sys.exit(1)

            # We can't set up Calico if the container shares the host namespace.
            if info["HostConfig"]["NetworkMode"] == "host":
                print "Can't add %s to Calico because it is " \
                      "running NetworkMode = host." % container_id
                sys.exit(1)

        # Check if the container already exists
        try:
            _ = client.get_endpoint(hostname=hostname,
                                    orchestrator_id=orchestrator_id,
                                    workload_id=workload_id)
        except KeyError:
            # Calico doesn't know about this container.  Continue.
            pass
        else:
            # Calico already set up networking for this container.  Since we got
            # called with an IP address, we shouldn't just silently exit, since
            # that would confuse the user: the container would not be reachable on
            # that IP address.
            print "%s has already been configured with Calico Networking." % \
                  container_id
            sys.exit(1)

        ip, pool = get_ip_and_pool(ip)

        # The next hop IPs for this host are stored in etcd.
        next_hops = client.get_default_next_hops(hostname)
        try:
            next_hops[ip.version]
        except KeyError:
            print "This node is not configured for IPv%d." % ip.version
            unallocated_ips = client.release_ips({ip})
            if unallocated_ips:
                print ("Error during cleanup. {0} was already unallocated."
                      ).format(unallocated_ips)
            sys.exit(1)

        # Get the next hop for the IP address.
        next_hop = next_hops[ip.version]

        network = IPNetwork(IPAddress(ip))
        ep = Endpoint(hostname=hostname,
                      orchestrator_id=DOCKER_ORCHESTRATOR_ID,
                      workload_id=workload_id,
                      endpoint_id=uuid.uuid1().hex,
                      state="active",
                      mac=None)
        if network.version == 4:
            ep.ipv4_nets.add(network)
            ep.ipv4_gateway = next_hop
        else:
            ep.ipv6_nets.add(network)
            ep.ipv6_gateway = next_hop

        # Create the veth, move into the container namespace, add the IP and
        # set up the default routes.
        netns.create_veth(ep.name, ep.temp_interface_name)
        netns.move_veth_into_ns(namespace, ep.temp_interface_name, interface)
        netns.add_ip_to_ns_veth(namespace, ip, interface)
        netns.add_ns_default_route(namespace, next_hop, interface)

        # Grab the MAC assigned to the veth in the namespace.
        ep.mac = netns.get_ns_veth_mac(namespace, interface)

        # Register the endpoint with Felix.
        client.set_endpoint(ep)

        # Let the caller know what endpoint was created.
        print "IP %s added to %s" % (str(ip), container_id)
        return ep
    ```

可以看到，首先判断容器是否是在运行，且网络是否不为host模式，否则则退出。然后查询endpoint是否已经存在，存在则退出。再到etcd中获取下一跳的地址。一切顺利的话，则根据传进的参数，创建一个endpoint，然后设置下一跳地址，接着利用netns在容器的网络namespace下创建网络设备，最后把相关的信息存储到etcd中去。

### Add profile

当运行 `calicoctl profile add PROFILE`的时候，运行的代码如下：

    ```python
    def profile_add(profile_name):
        """
        Create a policy profile with the given name.
        :param profile_name: The name for the profile.
        :return: None.
        """
        # Check if the profile exists.
        if client.profile_exists(profile_name):
            print "Profile %s already exists." % profile_name
        else:
            # Create the profile.
            client.create_profile(profile_name)
            print "Created profile %s" % profile_name
    ```

首先先判断profile是否已经存在，如果不存在，则创建的profile


### Append workload to profile

运行 `calicoctl container Workload profile append PROFILE` 的时候，运行的代码如下：

    ```python
    def endpoint_profile_append(hostname, orchestrator_id, workload_id,
                                endpoint_id, profile_names):
        """
        Append a list of profiles to the container endpoint profile list.
        The hostname, orchestrator_id, workload_id, and endpoint_id are all
        optional parameters used to determine which endpoint is being targeted.
        The more parameters used, the faster the endpoint query will be. The
        query must be specific enough to match a single endpoint or it will fail.
        The profile list may not contain duplicate entries, invalid profile names,
        or profiles that are already in the containers list.
        :param hostname: The host that the targeted endpoint resides on.
        :param orchestrator_id: The orchestrator that created the targeted endpoint.
        :param workload_id: The ID of workload which created the targeted endpoint.
        :param endpoint_id: The endpoint ID of the targeted endpoint.
        :param profile_names: The list of profile names to add to the targeted
                            endpoint.
        :return: None
        """
        # Validate the profile list.
        validate_profile_list(profile_names)
        try:
            client.append_profiles_to_endpoint(profile_names,
                                               hostname=hostname,
                                               orchestrator_id=orchestrator_id,
                                               workload_id=workload_id,
                                               endpoint_id=endpoint_id)
            print_paragraph("Profiles %s appended" %
                            (", ".join(profile_names)))
        except KeyError:
            print "Failed to append profiles to endpoint.\n"
            print_paragraph("Endpoint %s is unknown to Calico.\n" % endpoint_id)
            sys.exit(1)
        except ProfileAlreadyInEndpoint, e:
            print_paragraph("Profile %s is already in endpoint "
                            "profile list" % e.profile_name)
        except MultipleEndpointsMatch:


                            "search.")
            sys.exit(1)
    ```

可以看到，本函数只有调用了一个函数来将一个profile加入一个endpoint。如果profile已经存在在endpoint中了，或者匹配到了多个endpoint都会出错且退出。
