---
layout: post
title: Docker第三方网络插件
category: tech
tags: [network, docker, libnetwork]
---

Docker的libnetwork是支持动态的集成第三方的网络插件的，此功能由drivers.remote包提
供，它并不是某个网络driver具体实现，而是提供了一个代理供第三方网络drivers到
libnetwork的注册与通信。

### Libnetwork与docker plugin

当LibNetwork 利用Init()函数初始化drivers.remote 的时候，他会传递一个
DriverCallback作为参数，DriverCallback中实现了RegisterDriver()这个接口。远程的
Driver就利用这个接口然后通过plugins.Handle这个回调函数来注册driver到libnetwork的
NetworkController中。

当driver通过plugins.Get方法载入的时候就会调用这个回调函数。

### 实现

这些第三方的driver的实现都利用plugin.Client来与远程driver通信，
driverapi.Driver在plugin driver之上实现被作为rpc。而这些rpc大部分是直接与json通
信。

### Usage

所有的第三法网络插件与内置的网络插件都遵守同样的规则，拥有同样的接口，同时也提供
了特殊的`options`，和用户提供的`labels`来影响网络driver的行为。

### Protocol

代理发送HTTP请求，网络driver需要回应一个相对应的respond。

### Errors

如果driver无法decode，或者发现了http请求或者错误有问题，则要回复一个HTTP error
status (4xx或者5xx), 回复的内容中也必须包含如下内容json数据

```
{
    "Err": String
}
```

### Handshake

Driver载入的时候将会接收到一个没有payload的POST请求到地址/Plugin.Activate，此时
必须回复一个下面格式的manifest。

```
{
    "implements": ["NetworkDriver"]
}
```

其他的选项也被允许，NetworkDriver表示，这个插件将以driver的形式注册到libnetwork
中去。

### Set capability

当handshake之后，driver将会收到另外一个POST请求到地址
`/NetworkDriver.GetCapabilities`，driver要回复如下格式的response。

```
{
    "Scope": "local"
}
```

scope的值可以是“local”或者“global”，否则就会出错。

### Create Network

当代理被请求create一个新的网络的时候，driver将会接收一个到
`NetworkDriver.CreateNetwork`的POST请求。格式如下

```
{
    "NetworkID": string,
    "Options": {
       ...
    }
}
```

`NetworkID`值是由libnetwork生成的。`Options`值是一个任意的map，有libnetwork传递
个proxy的。

成功的的话，则返回一个空json

```
{}
```

### Delete Network

当driver的网络被删除后，将会收到POST请求到`/NetworkDriver.DeleteNetwork`

```
{
    "NetworkID": string
}
```

成功则返回空

```
{}
```

### Create Endpoint

method: post

url: `/NetworkDriver.CreateEndpoint`

form:

```Go
{
    "NetworkID": string,
    "EndpointID": string,
    "Options": {
        ...
    },
    "Interface": {
        "Address": string,
        "AddressIPv6": string,
        "MacAddress": string
    }
}
```

其中，`NetworkID`是生成的network的标识符，`EndpointID`用来表示一个Endpoint，
options是一个proxy提供的一个map。`Interface`值可以为空，如果提供了，那么
`Address`是一个ipv4地址，与一个子网的CIDR表示法，如果提供了`AddressIPv6`那么则是
一个ipv6的CIDR表示法，`MacAddress`则是Mac地址。

成功返回的格式如下:

```
{
    "Interface": {
        "Address": string,
        "AddressIPv6": string,
        "MacAddress": string
    }
}
```

如果远程driver已经提供了一个非空的Interface那么必须回复一个空的Interface值，因为
如果Libnetwork提供一个非空的值，接收了一个非空值将会被视为错误。

### Endpoint operational info

Proxy 可能会被请求“operational info”，则
nethod: post
url: `/NetworkDriver.EndpointOperInfo`
form:

```
{
    "NetworkID": string,
    "EndpointID": string
}
```

返回格式如下

```
{
    "Value": { ... }
}
```

其中Value可以是任意的map

### Delete endpoint

method: post
url: `NetworkDriver.DeleteEndpoint`
form:

```
{
    "NetworkID": string,
    "EndpointID": string
}
```

成功则返回空

```
{}
```

### Join

当一个sandbox要join一个一个endpoint的时候
method: post
url: `NetowrkDriver.Join`
form:
```
{
    "NetworkID": string,
    "EndpointID": string,
    "SandboxKey": string,
    "Options": { ... }
}
```

其中`SandboxKey`用来标识sandbox，返回的结果要以下格式：

```
{
    "InterfaceName": {
        SrcName: string,
        DstPrefix: string
    },
    "Gateway": string,
    "GatewayIPv6": string,
    "StaticRoutes": [{
        "Destination": string,
        "RouteType": int,
        "NextHop": string,
    }, ...]
}
```

其中`Gateway`和`GatewayIPv6`是可选的，如果提供的话，值为一个ip地址，
`InterfaceName`是一个OS级的interface，会被Libnetwork移动到sandbox中去，其中
`srcName`是driver创建的OS级的interface的name,`DstPrefix`是这个接口移动到sandbox
中去之后所应该拥有的前缀（libnetwork会在后面添加一个index，防止冲突）。
`StaticRoutes`表示一旦interface加入sandbox中去的时候需要添加的路由。

路由可以给`RouteType`一个`0`，然后然后给`NextHop`一个值，或者给`RouteType`一个
`1`，然后不给`NextHop`赋值。


# 具体的实现

### 创建一个network

首先libnetwork的client会接收相应的命令
然后调用`CmdNetworkCreate()`函数，
然后通过获取的网络名字name参数，调`networkCreate()`，
其中会调用controller的`loadDriver`函数。
如果此网络的driver以前没有创建，那么则会调用`discovery.go`中的`Plugin`方法，他会创建相应的的插件的socks文件来初始化plugin，
当此过程完成后，则会调用`plugins.go`中的`activateWithLock`这个函数，他将会发送一个post请求到`/Plugin.Activate`这个url，而插件那边则会监听这个socks，然后接收到这个请求，完成插件端的初始化。
