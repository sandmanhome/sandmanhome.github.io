---
title: 路由器 梅林 RT-AC68U
date: 2020-04-01 18:02:30
tags:
- meilin
- RT-AC68U
- NAS
- smb
---

# 梅林固件

- RT-AC68U/RT-1750B1 默认出厂采用路由器设置界面直接上传梅林固件(第三方固件)刷机无法通过验证,可采用降级后再刷

- 固件升级在路由器管理界面中的![系统管理 - 固件升级](firmware.jpg)

1. 降级至 3.0.0.4.384.10007

[3.0.0.4.384.10007](https://pan.baidu.com/s/1Caz3GdeMhdcsbBUcuCrYcQ), 提取码 2dft

2. 升级至 380.70-0-X7.9

[380.70-0-X7.9](https://firmware.koolshare.cn/Koolshare_Merlin_Legacy_380/ASUS/RT-AC68U/X7.9/RT-AC68U_380.70_0-X7.9-koolshare.trx)

# 公网ip及ddns

- 公网ip

坐标福建厦门,电信宽带拨号,拨打10000号申请公网ip,按客服提示半小时后重启光猫及路由器即可,其中光猫客服可远程重启

- ddns

RT-AC68U自带华硕的ddns,可开启梅林 **外部网络(WAN) - DDNS** 中的 **DDNS Client**, 将自定义的 主机名称 与路由器 WAN ip绑定![ddns setting](ddns.jpg)
设置完成后梅林中的 网络地图显示如下![netmap](netmap.jpg)
外网可通过 ping 自定义前缀.asuscomm.com ( 需开启梅林中的 **防火墙 - 一般设置 中的 响应ICMP Echo(ping)要求** )

- ssh

具有公网ip后,设置梅林的 **系统管理 - 系统设置** 中的 SSH Daemon, 即可采用公网登陆梅林![ssh setting](ssh.jpg)

# NAS

由于受够了一些某网的上传慢、限速等诟病,且云盘安全性及迁移也较为麻烦.考虑到更新了电信宽带具备公网ip,因此依托梅林搭建具备 **公网访问能力的NAS服务** 供家庭使用,用于手机等客户端随时随地能下载备份照片、文件及电影等,即安全又方便.NAS服务 **以smb协议为例实现局域网及公网NAS服务**

## 局域网 NAS

- 插入外置U盘或移动硬盘

- 设置smb

开启梅林中的 **USB 相关应用程序 - 网络共享（Samba） / 云端硬盘**, 其中默认账户为路由器账户,按硬盘存储的文件夹共享,可设置账号、共享文件夹、权限![smb setting](smb.jpg)
iphone 手机连接路由wifi并设置smb(路由器ip,smb默认端口445)即可,例如 **ES文件游览器 FileExplore**

## 公网 NAS

- 由于安全问题,运营商 **封了smb服务** 的tcp端口 **445、139** 及udp端口 **137、138**, 同时封闭了 **80、8080** 等需要备案的端口

- smb服务优先运行于445端口,因此可做端口转发实现外网访问smb服务,由于 **路由器管理界面无法添加445端口的端口转发**, 因此必须 ssh 登陆路由器设置端口转发

1. 内网 NAS 设置完成后 ps |grep smbd 可以观察到 smbd 进程, netstat -ntlp|grep 445 发现其监听在 br0 及 lo 网卡上

2. 添加转发规则,其中 192.168.50.1 应改为 smbd 监听网卡ip, **本例防火墙是关闭的,若开启需要考虑防火墙对端口转发的限制,自定义防火墙设置可采用 jssf 中添加脚本守护**

```bash
iptables -t nat -I VSERVER -p tcp -m tcp --dport 30445 -j DNAT --to-destination 192.168.50.1:445
```

3. 实测发现,端口转发不起作用,原因是 **梅林** 源码 [lfpctrl](https://github.com/RMerl/asuswrt-merlin/blob/263449f32bf292fb6bc5a08cd645e61a7fb10485/release/src-rt/linux/linux-2.6/net/ipv4/netfilter/lfp.c#L99) 对 445 做了端口转发限制, 关闭限制

```bash
echo 0 > /proc/net/lfpctrl
```

由于 2. 和 3. 的操作无法固化,路由器重启后将失效,可在 **jssf 中增加启动脚本新建 cron 定时任务 check**. 其中 [jffs](https://github.com/RMerl/asuswrt-merlin.ng/wiki/JFFS) 为路由器的非易失存储,重启后数据仍保持,但是刷固件后会丢失,需要在梅林中配置开启(某些版本固件已经开启请忽略). [init-start](https://github.com/RMerl/asuswrt-merlin.ng/wiki/User-scripts#init-start) 为 jffs mount后其他服务启动前的调用脚本![jssf setting](jssf.jpg)
ssh登入路由器执行如下脚本,两分钟后 iptables -t nat -nL|grep 30445 查看是否有转发规则

```bash
cd /jffs/scripts

cat << EOF > custom-smb
#!/bin/sh
touch /tmp/000custom_smb

success_30445=`iptables -t nat -nL|grep 30445`
if [ "$success_30445" == "" ]; then
    iptables -t nat -I VSERVER -p tcp -m tcp --dport 30445 -j DNAT --to-destination 192.168.50.1:445
    echo "fix success_30445!\n" >> /tmp/000custom_smb
fi

success_lfpctrl=`cat /proc/net/lfpctrl`
if [ "$success_lfpctrl" != "0,(null),0" ]; then
        echo 0 > /proc/net/lfpctrl
        echo "fix success_lfpctrl!\n" >> /tmp/000custom_smb
fi
EOF

cat << EOF > init-start
#!/bin/sh

cru a custom-smb "*/2 * * * * /jffs/scripts/custom-smb"
EOF

chmod 755 init-start
chmod 755 custom-smb
./init-start
```

# NAS 客户端

- 由于smb服务端口设置了转发,客户端需要能 **指定非445的服务端端口** ,ios系统有 **FileExplorer** 支持多种格式,对于处理文档图片等很实用

- 关于smb中的视频, **FileExplorer** 支持的视频格式不多,但支持调用外部播放器. 其他的支持自定义端口smb服务端的NAS视频播放器有 **nPlayer** 也是非常值得推荐的