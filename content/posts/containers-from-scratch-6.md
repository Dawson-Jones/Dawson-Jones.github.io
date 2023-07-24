---
title: "从零开始创建一个容器之namespace 6.Mount namespaces"
date: 2023-07-24T22:05:13+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

Mount namespace 是第一个添加到 linux(2002 2.4.19) namespace 类型, 他隔离了进程中能够看到的 mount point 列表



# Shared subtrees

mount namespace 完成后, 遇到一个问题, mount namespace 提供了太多的 ns 间的隔离, 例如加载一个新的光盘💿驱动时, 只有将磁盘 mount 到每个 ns 中, 才能使这个磁盘在所有的 ns 中可见

shared subtree 允许在 ns 之间自动、受控地传播 mount 和 umount 事件。例如, mount 光盘到一个 ns 中, 可以触发 mount 到所有的其他的 ns

subtrees 有几种传播类型, 用来判断当 mount point 创建和销毁的时候是否传播到其他的 mount point

- MS_SHARED: 传播和接收 mount 和 umount 事件
- MS_PRIVATE: 和 MS_SHARED 相反, 不传播和接收任何事件
- MS_SLAVE: 介于 shared 和 private 之间, 有一个 master, 可以传播 mount 和 umount 事件到 slave mount, slave 自己不传播事件到 master
- MS_UNBINDABLE: 该 mount point 不可绑定, 不传播和接收任何事件, 且该 mount point 不能当作 bind mount 的源

## Peer groups

```bash
bytedance@ubuntu:~/project/C-program-language/a_n_so$ sudo mount --make-private /

# 没有没关系, 大致是这么个意思, 可以看图解
bytedance@ubuntu:~/project/C-program-language/a_n_so$ sudo mkdir /X
bytedance@ubuntu:~/project/C-program-language/a_n_so$ sudo mount --make-shared /dev/sda2 /X
bytedance@ubuntu:/X$ sudo mkdir /Y
bytedance@ubuntu:/X$ sudo mount --make-shared /dev/sda1 /Y

# 等待第二个 shell 执行结束
# 
bytedance@ubuntu:/X$ sudo mkdir /Z
bytedance@ubuntu:/X$ sudo mount --bind /X /Z
bytedance@ubuntu:/X$ mount
/dev/sda2 on /X type ext4 (rw,relatime)
/dev/sda2 on /Y type ext4 (rw,relatime)
/dev/sda2 on /Z type ext4 (rw,relatime)
# 查看现在第二个 shell 只有 /X 和 /Y, 因为被隔离了
```

```bash
# 复制了 init ns 的 mount
bytedance@ubuntu:~$ sudo unshare -m --propagation unchanged sh
# 回到第一个
# 
# mount
/dev/sda2 on /X type ext4 (rw,relatime)
/dev/sda2 on /Y type ext4 (rw,relatime)
```

![](https://static.lwn.net/images/2016/mountns_peer_groups.svg)

此时有两个 peer groups

- 第一个: 包含了 mount point ***X, X'***, X' 是 X 的复制品, 在 ns 创建的时候创建, Z 创建自源 mount point X 在 init ns 中
- 第二个: 包含了 mount point ***Y, Y'***

> Z 创建在 init ns 中, 在第二个 ns 创建之后创建, 没有复制到第二个 ns 中, 因为父 mount(/) 被标记为 *private*



## peer groups 经 /proc/pid/mountinfo

该文件会展示一些可选信息, 关于每个 mount 的传播类型和 peer group

对于 shared mount 可选域记录了 一个tag share:N, N 在同一 peer group 中的成员是相同的

```bash
bytedance@ubuntu:/X$ cat /proc/self/mountinfo | sed 's/ - .*//'

30 1 253:0 / / rw,relatime
45 30 8:2 / /X rw,relatime shared:34
46 30 8:2 / /Y rw,relatime shared:67
641 30 8:2 / /Z rw,relatime shared:34
```
- 第一列是唯一 ID

- 第二列是父 mount, /X, /Y, /Z 都是 root mount 的子 mount, ID 30

- 根目录的 mount point 是 private 的, 因为缺少 tag

- /X 和 /Z 是在同一个 peer group 中的 shared mount points, ID 34 (mount 事件可以传播)

- /Y 是在另一个 peer group 中的 shared mount points, ID 67(mount 事件不会传播到 group 34)

  
```bash
# 第二个 shell
# cat /proc/self/mountinfo | sed 's/ - .*//'
48 47 253:0 / / rw,relatime
614 48 8:2 / /X rw,relatime shared:34
640 48 8:2 / /Y rw,relatime shared:67
```

- root mount point 是 private
- /X 是 shared mount 在 peer group 34
- /Y 是 shared mount 在 peer group 67
- mount points 是复制的, 因为唯一 ID 和 init ns 中相应 mounts 不一样

## 默认的传播类型

新设备的 mount 创建遵循以下

- mount point 有父 mount point 且父的传播类型是 MS_SHARED, 新 mount 的传播类型也是 MS_SHARED
- 其他情况, 新的 mount 的传播类型是 MS_PRIVATE

根据规则, root mount 是 MS_PRIVATE, 所以所有的后代都默认是 MS_PRIVATE, 但是 MS_SHARED 是更好的默认, 更通用, 所以 systend 将所有的 mount point 设置为 MS_SHARED, 因此在绝大多数的 Linux 发行版, 默认传播类型都是 MS_SHARED. 

当创建一个新的 mount ns 时, unshare 假定用户需要一个完全隔离的 ns, 于是让所有的 mount points private, 通过与以下命令等效的操作

```
# 递归的 mark 所有的根目录下的 mounts 变为 private
mount --make-rprivate /
```

阻止该操作, 可以使用

```bash
unshare -m --propagation unchanged <cmd>
```

