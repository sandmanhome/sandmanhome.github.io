---
title: 流量拦截
date: 2020-04-10 17:46:46
tags:
- proxy
- tproxy
- nf_conntrack
- iptables
- dnat
---

# 流量引导方案

OTT设备中的app通常需要对视频、游戏加速, 广电等运营商网络中的IPTV终端需要进行内容管控及控制带宽成本, 各类终端(笔记本、路由器)等需要科学上网首先要解决的就是拦截本机流量并转发至远程代理. 下面以 **linux** 为例探讨本地流量引导技术方案

## 应用层拦截

http 代理协议, 如 linux 系统设置全局http代理, 需要客户端系统主动设置, 可实现http流量全量引导, 应用层加速受协议限制较大

## 传输层拦截

### 引导方案

- 通过添加 `iptables -t nat ... -j DNAT/REDIRECT ...` 规则实现流量转发至本地代理(远程代理),本地代理可对流量做二次过滤将流量转发到远程代理服务器或源站

1. **DNAT** 流量转发至远程代理, 通常用于特定端口转发, 远程代理无法知道原始目的地址, 按既定的加速服务处理请求

2. **REDIRECT/DNAT** 流量转发至本地, 本地代理可以获取原始目的地址, 选择直接发送数据包至原站或发送数据包及原始目的地址到远程代理服务器加速

3. **tproxy** 配合策略路由将流量引导至本地代理, 针对 **udp**, 具体做法是在 mangle 表的 **PREROUTING** 链中为每个 udp 数据包打上 0x2333/0x2333 标签并设定需获取报文信息的本地代理地址 **127.0.0.1:1080**, 策略路由设置将具有 **0x2333/0x2333** 标志的数据包投递到本地环回设备上的 **1080** 端口

```bash
iptables -t mangle -A PREROUTING -p udp -j tproxy --tproxy-mark 0x2333/0x2333 --on-ip 127.0.0.1 --on-port 1080
ip rule add fwmark 0x2333/0x2333 pref 100 table 100
ip route add local default dev lo table 100
```
对监听 1080 端口的本地代理 socket 需设置 IP_TRANSPARENT 标志

```c
int
create_server_socket(const char *host, const char *port)
{
    ...
#ifdef MODULE_REDIR
        if (setsockopt(server_sock, SOL_IP, IP_TRANSPARENT, &opt, sizeof(opt))) {
            ERROR("[udp] setsockopt IP_TRANSPARENT");
            exit(EXIT_FAILURE);
        }
        if (rp->ai_family == AF_INET) {
            if (setsockopt(server_sock, SOL_IP, IP_RECVORIGDSTADDR, &opt, sizeof(opt))) {
                FATAL("[udp] setsockopt IP_RECVORIGDSTADDR");
            }
        } else if (rp->ai_family == AF_INET6) {
            if (setsockopt(server_sock, SOL_IPV6, IPV6_RECVORIGDSTADDR, &opt, sizeof(opt))) {
                FATAL("[udp] setsockopt IPV6_RECVORIGDSTADDR");
            }
        }
#endif
    ...
}
```

### 本地代理原始目的地址获取

- tcp协议

本地代理收到经 `iptables -t nat ... -j DNAT/REDIRECT ...` 转发后的tcp请求可采用 **getsockopt(fd, SOL_IP, SO_ORIGINAL_DST, destaddr, &socklen)** 获取原始目的地址 **destaddr**

- udp协议

事实上在 linux 中对于上述 **SO_ORIGINAL_DST** 调用最终会调用 **getorigdst**, 因此 udp 协议无法使用 **SO_ORIGINAL_DST** 获取原始目的地址

```c
/* We only do TCP and SCTP at the moment: is there a better way? */
if (tuple.dst.protonum != IPPROTO_TCP &&
    tuple.dst.protonum != IPPROTO_SCTP) {
  pr_debug("SO_ORIGINAL_DST: Not a TCP/SCTP socket\n");
  return -ENOPROTOOPT;
}
```

1. tproxy 方案 udp 原始目的地址获取

对监听 **1080** 端口的本地代理 **socket** 设置 **IP_TRANSPARENT** 标志后, 需调用 **recvmsg** 函数接收数据包,采用循环比对 **IP_RECVORIGDSTADDR/IPV6_RECVORIGDSTADDR** 获取原始目的地址

```c
static int
get_dstaddr(struct msghdr *msg, struct sockaddr_storage *dstaddr)
{
    struct cmsghdr *cmsg;

    for (cmsg = CMSG_FIRSTHDR(msg); cmsg; cmsg = CMSG_NXTHDR(msg, cmsg)) {
        if (cmsg->cmsg_level == SOL_IP && cmsg->cmsg_type == IP_RECVORIGDSTADDR) {
            memcpy(dstaddr, CMSG_DATA(cmsg), sizeof(struct sockaddr_in));
            dstaddr->ss_family = AF_INET;
            return 0;
        } else if (cmsg->cmsg_level == SOL_IPV6 && cmsg->cmsg_type == IPV6_RECVORIGDSTADDR) {
            memcpy(dstaddr, CMSG_DATA(cmsg), sizeof(struct sockaddr_in6));
            dstaddr->ss_family = AF_INET6;
            return 0;
        }
    }

    return 1;
}
```

2. REDIRECT/DNAT 方案 udp 原始目的地址获取

上述 1. 中 **tproxy** 模块需要编译内核开启支持, 实际上 linux 内核采用 **nf_conntrack** 跟踪网络连接信息(包括udp), 此表可以通过**/proc/net/nf_conntrack** 查看, 里面包含tcp、udp的连接跟踪信息
本地代理接收到 **REDIRECT/DNAT** 的udp 数据包后, 可依据本地代理地址, 客户端地址, nat端口信息查询 **nf_conntrack** 表获取原始目的地址, 可参考 [一种UDP协议加速方法和系统](http://www2.soopat.com/Patent/201610261388)

## IP层拦截