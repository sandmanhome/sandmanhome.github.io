---
title: ipfs autonat
date: 2020-09-02 11:03:16
tags:
- ipfs
- autonat
- go-libp2p-autonat
- hostAddr
---

# hostAddr机制

## hostAddr查询与运用

``` bash
ipfs id
{
        "ID": "bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
        "PublicKey": "CAQSIQMXKhfeoAepfrHLesbR+6QXV5EbBDoz/e1+g9DvUw7ABw==",
        "Addresses": [
                "/ip4/127.0.0.1/tcp/4001/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
                "/ip4/192.168.100.242/tcp/4001/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
                "/ip6/::1/tcp/4001/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
                "/ip6/fe80::7adc:71f6:25a4:40e0/tcp/4001/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
                "/dns4/xxx.baasze.com/tcp/4001/p2p/bafzm3jqbec7ulhfmm7s7ydt2mf32nbsjy4237mvzj5skzbkxrfxz7axghsyum/p2p-circuit/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg",
                "/ip4/xx.xx.xx.xx/tcp/4001/p2p/bafzm3jqbec7ulhfmm7s7ydt2mf32nbsjy4237mvzj5skzbkxrfxz7axghsyum/p2p-circuit/p2p/bafzm3jqbeczaadu3v52hvjplcll7zsydxibzipg5yaxij3hmdhskysc52sdxg"
        ],
        "AgentVersion": "go-ipfs/0.3.4/183988a",
        "ProtocolVersion": "ipfs/0.1.0"
}
```

`ipfs id` 命令可显示ipfs节点信息，其中Addresses为服务地址即hostAddr, ipfs 会将hostAddr广播出去, 其余节点会利用hostAddr的地址进行连接, 因此通过hostAddr能否连接节点是 **节点互联的关键**.

## hostAddr的初始化

ipfs 启动后默认监听 “/ip4/0.0.0.0/tcp/0” “/ip6/::/tcp/0”, 此时 ipfs 查询本机各网卡地址并进行监听, 复杂网络环境下有如下几种情况:

- 云主机通常采用经典网络模式分配的ECS固定公网IP, 在本机网卡上可见公网ip, 因此hostAddr的可查询到此公网ip

- 云主机通常采用弹性公网ip **EIP（Elastic IP Address**）访问, 在主机上网卡上不可见公网ip,因此hostAddr的 **无法查询到此公网ip**

## hostAddr的更新

对于部署在无外网网卡ip主机的ipfs, 若依靠广播初始化后的hostAddr, 无法被其余节点主动连接, 即 Inbound 无法走通, 因此 libp2p 设计了autonat、autorelay等机制

注: **fix host can be dialed by autonat public addr, but lost the public addr to announce** **此pr修复了hostAddr更新缺失EIP的情况,已被合并**.

# autonat机制

## autonat基本原理

**go-libp2p-autonat** 运用remotePeer（对端节点）**DialBack(回拨)机制** 来获取自身服务的外网IP及端口, 其过程如下:

1. 全网中开启autonat service的某节点A, 该节点在p2p网络中支持 **autonat协议（”/libp2p/autonat/1.0.0”）**

2. 节点B主动连接上支持 autonat service的节点, 此连接为后续协议交互的连接, 包括 **identify协议** 和 **autonat协议**

3. 节点B在与A完成身份认证后判断节点A是否支持autonat协议, 若支持则由节点B的 autonat client 发起公网地址拨号请求(DialBack)

4. 节点A的autonat service 收到DialBack请求后对节点B进行DialBack **(回拨的地址为节点B的hostAddr以及已有连接的节点B地址)**, 并将结果observations（包含Multiaddr及Reachability）发送给节点B

5. 节点B根据DialBack的结果判断自身是 ReachabilityPublic 还是 ReachabilityPrivate, 并补充更新 hostAddr

## autonat优化

除了上述基本功能外, autonat还做了一些额外的优化

1. 节点B启动动态定时器主动对Peerstore的节点进行autonat协议探测以便获取自身最新的nat状态

2. 对于网络状态的翻转（flipping)设定一个confidence, confidence的增加代表对当前状态的自信心（[0,3]）, 根据是否翻转以及confidence的值决定是否触发网络状态的改变, 修改hostAddr

3. confidence小于最大值时, 1.的探测定时器为固定时长(15 min), 否则对探测定时器做如下调整

4. 当为外网可访问状态时，若最新的外网节点主动连接（Inbound Conn)成功且比最新的探测时间晚, 则延长1.的探测定时器时长, 段时间虚无继续探测

5.当为外网不可访问状态时，若最新的外网节点主动连接（Inbound Conn)成功且比最新的探测时间晚, 减小1.的探测定时器时长以变获取最新的外网地址

总结: 目前ipfs中默认开启了autonat service及autonat探测