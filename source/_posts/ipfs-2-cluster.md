---
title: ipfs cluster
date: 2020-05-13 10:56:18
tags:
- ipfs
- cluster
---

# ipfs-cluster 私有网络集群搭建

192.168.100.4 192.168.100.5 192.168.100.241 组成私有网络, 选取 192.168.100.241 为管理节点采用 
 **ipfs-cluster-ctl** 管理文件

- 管理节点部署 ipfs-cluster-service

记录如下 secret 和 cluster的管理节点信息

```bash
cd ~
mkdir ipfs-cluster
cd ipfs-cluster
wget https://dist.ipfs.io/ipfs-cluster-service/v0.12.1/ipfs-cluster-service_v0.12.1_linux-amd64.tar.gz
...

tar -xvzf ipfs-cluster-service_v0.12.1_linux-amd64.tar.gz
...

./ipfs-cluster-service/ipfs-cluster-service init
...

./ipfs-cluster-service/ipfs-cluster-service daemon &
[2] 88693
16:41:34.958  INFO    service: Initializing. For verbose output run with "-l debug". Please wait... daemon.go:46
➜  ipfs-cluster 16:41:35.449  INFO    cluster: IPFS Cluster v0.12.1 listening on:
        /ip4/127.0.0.1/tcp/9096/p2p/12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf
        /ip4/192.168.100.241/tcp/9096/p2p/12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf
        /ip4/127.0.0.1/udp/9096/quic/p2p/12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf
        /ip4/192.168.100.241/udp/9096/quic/p2p/12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf

cat ~/.ipfs-cluster/service.json|grep secret
    "secret": "8e3d96c01821b7f238e805fb28d6b873d09f279d1ff9e63aa3528848f79b73c3",
```

- 节点部署 ipfs-cluster-service

`ipfs-cluster-service daemon --bootstrap` 指定管理节点

```bash
cd ipfs-cluster
wget https://dist.ipfs.io/ipfs-cluster-service/v0.12.1/ipfs-cluster-service_v0.12.1_linux-amd64.tar.gz
...

tar -xvzf ipfs-cluster-service_v0.12.1_linux-amd64.tar.gz
...

./ipfs-cluster-service/ipfs-cluster-service init
...

sed -i 's/"secret".*/"secret": "8e3d96c01821b7f238e805fb28d6b873d09f279d1ff9e63aa3528848f79b73c3",/' ~/.ipfs-cluster/service.json

./ipfs-cluster-service/ipfs-cluster-service daemon  --bootstrap /ip4/192.168.100.241/tcp/9096/p2p/12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf &
...
09:04:53.291  INFO    cluster: ** IPFS Cluster is READY ** cluster.go:655
09:04:53.681  INFO    cluster: 12D3KooW9qFBb2K7TPRW7s654LJJM19hseGZBcvde6Hpe3sNVkWu: joined 12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf's cluster cluster.go:993
```

- 管理节点部署 ipfs-cluster-service

```bash
cd ipfs-cluster

wget https://dist.ipfs.io/ipfs-cluster-service/v0.12.1/ipfs-cluster-ctl_v0.12.1_linux-amd64.tar.gz
...

tar -xvzf ipfs-cluster-ctl_v0.12.1_linux-amd64.tar.gz
...

cp ipfs-cluster-ctl/ipfs-cluster-ctl /usr/local/bin
ipfs-cluster-ctl peers ls
17:14:13.571  INFO restapilog: 127.0.0.1 - - [13/May/2020:17:14:13 +0800] "GET /peers HTTP/1.1" 200 6641
 restapi.go:117
12D3KooW9qFBb2K7TPRW7s654LJJM19hseGZBcvde6Hpe3sNVkWu | xxxxxx | Sees 2 other peers
...
```

# ipfs-cluster pin 测试

- 管理节点添加文件

```bash
echo a > a.txt
ipfs-cluster-ctl a.txt
added Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 a.txt
17:24:10.699  INFO    cluster: pinning Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 on [12D3KooW9qFBb2K7TPRW7s654LJJM19hseGZBcvde6Hpe3sNVkWu 12D3KooWEdwm9Q7MXvZrYeMujrQ5fCzJwfJQX75gtSdTyYMpRozf 12D3KooWSKPbWzmnhjRrpi4EVz6NTCnYE6rfZLc1gzLQFdd4sHb6]: cluster.go:1400
17:24:10.700  INFO       crdt: new pin added: Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 consensus.go:209
17:24:10.700  INFO       crdt: adding new DAG head: QmZUnZo3dva57M29Ky2fTPqbEDaWnJWKnUMYMoYkxYpLFU (height: 1) heads.go:114
17:24:10.701  INFO      adder: Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 successfully added to cluster adder.go:182
17:24:10.701  INFO restapilog: 127.0.0.1 - - [13/May/2020:17:24:10 +0800] "POST /add?chunker=size-262144&cid-version=0&hash=sha2-256&hidden=false&layout=&local=false&name=&nocopy=false&progress=false&raw-leaves=false&recursive=false&replication-max=0&replication-min=0&shard=false&shard-size=104857600&stream-channels=true&user-allocations=&wrap-with-directory=false HTTP/1.1" 200 88
 restapi.go:117
➜  ipfs-cluster-ctl 17:24:10.803  INFO   ipfshttp: IPFS Pin request succeeded:  Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 ipfshttp.go:372

ipfs pin ls |grep Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9
Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 recursive
```

- 节点测试副本

192.168.100.4 及 192.168.100.5 均存在副本

```bash
ipfs pin ls |grep Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9
Qmbvkmk9LFsGneteXk3G7YLqtLVME566ho6ibaQZZVHaC9 recursive
```