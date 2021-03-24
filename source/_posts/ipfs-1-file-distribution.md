---
title: ipfs 文件分发
date: 2020-05-08 18:49:23
tags:
- ipfs
- repo gc
- pin
---

![ipfs](ipfs.png?170*130)
初识ipfs, 认为其能保证文件高可用.听某人聊起在A节点添加文件,过了许久后,A节点离线(退出 ipfs daemon), B节点已无法(添加后的第一次)访问A节点添加的文件, 即 文件并不会主动分发 .由此做一些测试并做如下总结.

# ipfs文件add及访问测试

## ipfs单节点的文件存储

ipfs add xxx **默认启用pin**, 即将文件复制一份副本存储在本地

- 添加文件

```bash
echo 'pin.no.daemon' > pin.no.daemon.txt

ipfs add pin.no.daemon.txt
added bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o pin.no.daemon.txt
 14 B / 14 B [=======================================================================================================] 100.00%

ipfs pin ls |grep bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o recursive

ipfs cat /ipfs/bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
pin.no.daemon
```

添加文件后使用 ipfs pin ls 命令可查询文件到已 **pin** 到本地

- 删除本地文件测试

```bash
rm pin.no.daemon.txt

ipfs pin ls |grep bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o recursive

ipfs cat /ipfs/bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
pin.no.daemon
```

删除源文件及仍可访问, 推测原因是ipfs add 命令 **pin** 了副本

- 删除pin文件测试

```bash
ipfs pin rm bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
unpinned bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o

ipfs pin ls |grep bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o

ipfs cat /ipfs/bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
pin.no.daemon
```
"删除" pin文件发现仍可访问，推测’删除’并未立刻生效,联想到是否有类似 **GC** 机制

- GC后测试

```bash
ipfs repo gc
...
removed bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o

ipfs cat /ipfs/bafk43jqbedag3yc4ixfr73frvbvz5fs6afx4rogxiiaipmdcce7jgcrdlwa5o
Error: merkledag: not found
```
GC才会真正回收 **unpined** 文件

- GC 机制

ipfs config show 查看 Datastore 配置

```json
{
  "Datastore": {
      "StorageMax": "10GB",           // 最大存储空间, 实际上, 存储量超过该值仍可以继续存储
      "StorageGCWatermark": 90,       // 存储空间警戒线, 只有 已使用存储空间/最大存储空间 超过该值, 定时自动 GC 才会生效, 否则不会 GC
      "GCPeriod": "1h",               // 每过 1h, 检查是否需要 GC
      
      ...
  },
}
```
**默认定时GC不开启**, 使用`ipfs daemon --enable-gc`开启, 使用`ipfs repo gc`手动触发 **GC**

## 多节点情况下测试

A、B 两节点组成的私有网络，都使用 ipfs daemon 启动

- A节点添加文件

```bash
ipfs repo gc
echo 'pin.with.daemon' > pin.with.daemon.txt

ipfs add pin.with.daemon.txt 
added bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i pin.with.daemon.txt
 16 B / 16 B [==========================================================================================] 100.00%
 
ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i recursive
```
文件pin.with.daemon.txt **pin** 在 A 节点

- B节点访问

```bash
ipfs repo gc

ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i

ipfs cat /ipfs/bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
pin.with.daemon

ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i

ipfs get /ipfs/bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
Saving file(s) to bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
 16 B / 16 B [=======================================================================================] 100.00% 0s

ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
```
B 节点可访问,但并无 pin 副本

- A节点删除文件

```bash
rm pin.with.daemon.txt 

ipfs pin rm bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
unpinned bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i

ipfs repo gc
...
removed bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
```

- B节点访问

```bash
ipfs cat /ipfs/bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
pin.with.daemon

ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
```
此时B节点仍旧可以访问,但无 pin 副本

- B节点GC后访问

```bash
ipfs repo gc
removed bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i

ipfs pin ls |grep bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i

ipfs cat /ipfs/bafk43jqbedzjup3mmlj5vuq65nqpkoob3ayjnnsmv4sq75zy7bmu4h3zws43i
```
B节点缓存失效, ipfs cat 因私有网络已无副本挂起

## ipfs文件add及访问机制

- ipfs add并不会主动分发文件到其他节点

- 访问文件**并不会pin到本地节点**

- 无论是访问操作还是 **pin** 操作均会在数据库中备份(大于分片的情况下可能不是全量)

- 触发GC会删除unpined数据

## ipfs如何保证可靠分布式存储

参考 [Replication on IPFS](https://github.com/ipfs-inactive/faq/issues/47)

- ipfs 协议用于保证节点之间的数据快速共享访问, 并不保证可靠存储

- 若要实现可靠存储，应基于 ipfs 之上构建实现可靠存储的新协议，如 **ipfs-cluster**
