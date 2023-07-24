---
title: "从零开始创建一个容器之namespace 5.Network namespaces"
date: 2023-07-21T16:55:13+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

Network namespace 分割了网络的使用: 设备、地址、端口、路由、firewall 规则等

network namespace 可以使用 clone() 系统调用加上 `CLONE_NEWNET` 这个 flag, 也可以使用 ip 这个网络配置工具设置和使用 net ns



## 基本 net ns 管理

```
ip netns add netns1
ip netns delete netns1
```
使用 ip 命令添加一个名为 netns1 的 network ns, 当 ip 命令创建了一个 network ns, 同时会创建一个 bind moutn 在 `/var/run/netns` 下, 这可以使 ns 持续存在, 即使没有进程在里面, network ns 在 ready前需要做很多配置

```
ip netns exec netns1 ip link list
```

在 netns1 这个 network ns 中执行 `ip link list`



## NetWork namespace 配置

新的 net ns 有环回(lo)设备, 其他设备(虚拟网卡, bridge等)也可以存在, 物理网卡只能在 root ns 中, 其他设备(virtual ethernet/veth) 可以被创建在 net ns 中, 虚拟设备允许进程在 ns 中通信, route 决定了和谁通信

**让当前 ns 和 root ns 通信**

```
# 创建一个 veth pair, 他们是联通的, 从 veth0 发送的包会从 veth1 接收到, 反之亦然
ip link add veth0 type veth peer name veth1
# 将 veth1 分配给 veth1
ip link set veth1 netns netns1

ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up`
ifconfig veth0 10.1.1.2/24 up

root@ubuntu:/run/netns# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.056 ms
root@ubuntu:/home/bytedance# ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=1.85 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.068 ms
```

如果希望连接到互联网

- 根据来自 netns1 中的 veth 设备在 root ns 中创建bridge
- 在 root ns 中配置 ip_forward 和相应的 nat



## ip netns 的实现

```bash
# root net ns 的 inode
root@ubuntu:/run/netns# readlink /proc/$$/ns/net
net:[4026531840]

# 进入一个新的 net ns
root@ubuntu:/run/netns# unshare -n bash
root@ubuntu:/run/netns# echo $$
90883
# 等待第二个 shell 前两步执行完
# 查看现在的 inode 和 root net ns 不一样
root@ubuntu:/run/netns# ls -i ns1
4026532302 ns1
root@ubuntu:/run/netns# exit
exit
# mount bind 后, inode 一直存在
root@ubuntu:/run/netns# ls -i ns1
4026532302 ns1

# 可以重新进入
root@ubuntu:/run/netns# nsenter --net=./ns1 bash
```

```bash
# 另一个 shell
root@ubuntu:/run/netns# touch /var/run/netns/netns1
# mount bind
root@ubuntu:/run/netns# mount --bind /proc/90883/ns/net /var/run/netns/netns1

# 查看 mount, 已经 mount
root@ubuntu:/run/netns# mount
nsfs on /run/netns/ns1 type nsfs (rw)
nsfs on /run/netns/ns1 type nsfs (rw)

# 删除 bind mount
root@ubuntu:/run/netns# umount ./netns1
```

