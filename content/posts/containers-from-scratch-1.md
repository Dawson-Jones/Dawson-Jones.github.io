---
title: "从零开始创建一个容器之namespace 1.overview"
date: 2023-07-05T01:10:36+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

容器化需要 linux 提供的三个能力

- namespace
- cgroup
- chroot

**namespace**     用来隔离与宿主机环境

**cgroup** 			 用来限制每个进程的资源

**chroot** 			 用来切换根目录



# namespace 概览

### 当前 linux 支持的 namespace

```
Namespace Flag              Isolates
Cgroup    CLONE_NEWCGROUP		Cgroup root directory
IPC       CLONE_NEWIPC      System V IPC,
       											POSIX message queues
Network   CLONE_NEWNET     	Network devices,
       											stacks, ports, etc.
Mount     CLONE_NEWNS       Mount points
PID       CLONE_NEWPID      Process IDs
Time      CLONE_NEWTIME			Boot and monotonic clocks
User      CLONE_NEWUSER     User and group IDs
UTS       CLONE_NEWUTS      Hostname and NIS domain name
```



### 查看进程所属的 namespace

每个进程的 namespace 信息都可以在`/proc/<pid>/ns/`看到, 文件描述符, 可以当作 `setns` 函数的参数

- mount 该目录的文件到其他地方会使当前pid 所在的 namespace 存活, 即使所有 namespace 的进程都终止(TODO: 还不太明白)
- 打开目录下的某一文件, 会返回相应的 namespace, 也会使当前pid 所在的 namespace 存活, 即使所有 namespace 的进程都终止
- 如果两个进程在同一个 namespace, 软连接将会是同一个, (TODO: 可以通过调用 stat 检查 stat.st_dev 和 stat.st+ino )
- 数字是 inode number

```bash
# 查看当前进程的 namespace
ubuntu@ubuntu:~$ ll /proc/self/ns/
total 0
dr-x--x--x 2 ubuntu ubuntu 0 Jul  5 01:40 ./
dr-xr-xr-x 9 ubuntu ubuntu 0 Jul  5 01:40 ../
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 net -> 'net:[4026531840]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 time -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 user -> 'user:[4026531837]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul  5 01:40 uts -> 'uts:[4026531838]'
ubuntu@ubuntu:~$ readlink /proc/self/ns/ipc
ipc:[4026531839]
```

### 各种 namespace 创建限制

```bash
ubuntu@ubuntu:~$ ll /proc/sys/user/
total 0
dr-xr-xr-x 1 root root 0 Jul  5 02:54 ./
dr-xr-xr-x 1 root root 0 Jun 29 03:44 ../
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_cgroup_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_fanotify_groups
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_fanotify_marks
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_inotify_instances
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_inotify_watches
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_ipc_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_mnt_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_net_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_pid_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_time_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_user_namespaces
-rw-r--r-- 1 root root 0 Jul  5 02:54 max_uts_namespaces
```



### namespace 系统调用API

#### clone: 创建并加入 namespace

```c
int clone(int (*fn)(void *), void *stack, int flags, void*arg, .../* pid_t *parent_tid, void *tls, pid_t *child_tid */ );
```

和 `fork` 类似创建一个子进程, 如果 flag 指定了 **CLONE_NEW\***, 则会根据要求的 flag 创建新的 namespace, 子进程运行在新的 namespace 中

#### setns: 加入已有 namespace

```c
int setns(int fd, int nstype);
```

将一个进程加入已有的 namespace 中, 通过一个指定的 fd, 比如: `/proc/pid/ns`下面的

#### unshare: 将进程移动到新的 namespace 中

```c
 int unshare(int flags);
```

如果 flag 指定了 **CLONE_NEW\***, 新的 namespace 将会为每个 flag 创建新空间, 进程将成为这些 namespace 的成员

#### ioctl

```c
int ioctl(int fd, unsigned long request, ...);
```

感觉哪里都有这货, 用来

- 发现 namespace 关系
- 发现 namespace 类型 (since Linux 4.11)
- 发现 user namespace 的 owner (since Linux 4.11)



### namespace 生命周期

正常情况下, **namespace 会在该 namespace 中的最后一个进程终结的时候销毁**

特殊情况: (暂时不需要看)

- `/proc/pid/ns/*` 文件被打开或 mount 存在, 比如: `mount --bind /proc/1000/ns/ipc /other/file`
- 有子 namespace
- user namespace, 有多个 nonuser namespace
- PID namespace，并且有一个进程通过 `/proc/<pid>/ns/pid_for_children` 符号链接引用该 namespace (不太明白)
- time namespace, 有进程通过 `/proc/pid/ns/time_for_children` 引用了该 namespace
- IPC namespace, mount 的 mqueue 引用了该 namespace
- PID namespace, proc 文件系统引用了该 namespace

