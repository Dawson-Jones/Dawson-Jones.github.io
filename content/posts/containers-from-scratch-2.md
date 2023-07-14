---
title: "从零开始创建一个容器之namespace 2.Namaspace API"
date: 2023-07-12T16:19:57+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

namespace 包装全局系统资源到一个抽象中, 使 namespace 中的进程拥有隔离的资源

每个进程都有一个 `/proc/PID/ns` 目录, 每个类型的 namespace 都有一个文件, 这些文件是一个特殊的符号链接(symbolic link), 可以在进程关联的 namespace 上做一些操作

```bash
ubuntu@bytedance:~/project/my_project/C/C-program-language/namespace$ ls -l /proc/$$/ns/
total 0
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 net -> 'net:[4026531840]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 14:51 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 time -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 14:51 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 user -> 'user:[4026531837]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jul 13 01:54 uts -> 'uts:[4026531838]'
```

这些符号链接可以用来:  

- 判断两个进程是否在同一个 namespace 中, 如果 inode 编号相同, 表示在同一个 namespace 中, 可以使用 `stat()` 来获取 inode 编号 (st_ino 字段). 例子: [stat_get_inode](https://github.com/Dawson-Jones/C-program-language/blob/master/stat_dir/stat_test.c)

- 如果打开了某个文件, 那么这个 namespace 将持续存在, 即便所有的 namespace 的进程都结束了. 有同样作用的还有 bind mount 一个符号链接到另一个本地的文件系统

  ```
  # touch ~/uts                            # Create mount point
  # mount --bind /proc/<pid>/ns/uts ~/uts
  ```

  



# UTS namespace 



## clone: 创建并加入 namespace

### code

```c
// clone_uts_manural.c
// 创建一个新的进程并把他放到新的namespace中

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stddef.h>
#include <stdint.h>
#include <unistd.h>
#include <sched.h>
#include <sys/wait.h>
#include <sys/mman.h>
#include <sys/utsname.h>

#define errExit(msg) do {perror(msg); exit(EXIT_FAILURE);} while (0)

#define STACK_SIZE (1024 * 1024)    /* Stack size for cloned child 1M*/


static int child_func(void *hostname)
{
    struct utsname uts;

    // 子进程的 hostname
    if (sethostname(hostname, strlen(hostname)) == -1)
        errExit("sethostname");

    // get name and information about current kernel
    if (uname(&uts) == -1)   
        errExit("uname");
    printf("uts.nodename in child:  %s\n", uts.nodename);

    sleep(120);
    return 0;
}


int main(int argc, char *argv[]) {
    pid_t child_pid;
    char *stack;
    char *stackTop;
    struct utsname uts;
    

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <child-hostname>\n", argv[0]);
        return -1;
    }

    // 申请一块内存, 用来当作新的进程的栈空间
    stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED)
        errExit("mmap");
    
    // 栈空间从高地址到低地址
    stackTop = stack + STACK_SIZE;

    child_pid = clone(child_func,   // 执行函数
                        stackTop,   // 栈地址
                        CLONE_NEWUTS | SIGCHLD, // UTS ns | 子进程退出返回给父进程的信息
                        argv[1]     // 函数参数
                );
    if (child_pid == -1)    // 父进程继续执行
        errExit("clone");
    printf("clone() returned %jd\n", (intmax_t) child_pid);
    sleep(1);

    if (uname(&uts) == -1)
        errExit("uname");
    printf("uts.nodename in parent: %s\n", uts.nodename);

    if (waitpid(child_pid, NULL, 0) == -1)
        errExit("waitpid");
    printf("child has terninated\n");

    return 0;
}
```

### 编译

`make clone_uts_manural`

### 输出

```bash
ubuntu@bytedance:~/project/my_project/C/C-program-language/namespace$ sudo ./clone_uts_manural dawson
clone() returned 19800	# 子进程 pid
uts.nodename in child:  dawson	# 子进程 hostname
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# uts.nodename in parent: bytedance	# 父进程 hostname

root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# hostname
dawson

# pstree 查看进程关系, 能够看到所有的进程关系, 也说明了 pid 没有被隔离
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# pstree -pl
systemd(1)───sshd(712)───sshd(16181)───sshd(16259)───bash(16260)───sudo(19797)───sudo(19798)───clone_uts_manur(19799)───bash(19800)───pstree(19808)

# 当前 bash pid
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# echo $$
19800

# 当前 bash 的 uts namespace
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# readlink /proc/self/ns/uts
uts:[4026532204]

# 运行 clone_uts_manur 的 uts
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# readlink /proc/19799/ns/uts
uts:[4026531838]

# 当我打开一个新的 bash 的时候, uts namespace 和 clone_uts_manur 的 一样
ubuntu@bytedance:~$ readlink /proc/self/ns/uts
uts:[4026531838]

# 在 dawson 这个 namespace 中运行 clone_uts_manural 程序, 会创建一个新的 uts namespace
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# ./clone_uts_manural nera
clone() returned 19912
uts.nodename in child:  nera
root@nera:/home/ubuntu/project/my_project/C/C-program-language/namespace# uts.nodename in parent: dawson

root@nera:/home/ubuntu/project/my_project/C/C-program-language/namespace# pstree -pl
systemd(1)───sshd(712)───sshd(16181)───sshd(16259)───bash(16260)───sudo(19797)───sudo(19798)───clone_uts_manur(19799)───bash(19800)───clone_uts_manur(19911)───bash(19912)───pstree(19918)

# 退出 nera, 回到了 dawson
root@nera:/home/ubuntu/project/my_project/C/C-program-language/namespace# exit
exit
child has terninated
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# hostname
dawson
```



## setns: 将当前进程加入指定 namespace

当没有进程时, 保持 namespace 打开, 我们希望稍后添加进程到 namespace 中, setns 可以将当前进程加入一个已经存在的 namespace 中

我猜测这个 setns 的原理就是找到当前 `struct task_struct` 然后找到对应的 type 的 ns, 然后设置为打开的 fd 的 ns

伪代码: 

```c
SYSCALL_DEFINE2(setns, int, fd, int, nstype) {
    struct ns_common *ns;
    struct nsproxy *nsproxy;

    // 根据给定的文件描述符 fd 获取对应的命名空间结构体
    ns = get_ns_common(fd, nstype);
    // 获取当前进程的命名空间描述符
    nsproxy = current->nsproxy;

    // 根据命名空间类型 nstype 进行切换
    switch (nstype) {
        case CLONE_NEWNS:
            nsproxy->mnt_ns = ns;
            break;
        case CLONE_NEWUTS:
            nsproxy->uts_ns = ns;
            break;
        case CLONE_NEWIPC:
            nsproxy->ipc_ns = ns;
            break;
        case CLONE_NEWPID:
            nsproxy->pid_ns_for_children = ns;
            break;
        case CLONE_NEWUSER:
            nsproxy->user_ns = ns;
            break;
        case CLONE_NEWNET:
            nsproxy->net_ns = ns;
            break;
        default:
            ret = -EINVAL;
            goto out;
    }
}
```

源码在: [kernel/nsproxy.c](https://elixir.bootlin.com/linux/latest/source/kernel/nsproxy.c#L546)



### code

```c
// setns_manural.c
// 将当前进程加入到已有的namespace中

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>
#include <fcntl.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); } while (0)


int main(int argc, char const *argv[]) {
    int fd;
    if (argc < 3) {
        fprintf(stderr, "%s /proc/PID/ns/FILE cmd args...\n", argv[0]);
        exit(EXIT_FAILURE);
    }

  	// 打开指定的 ns 文件
    fd = open(argv[1], O_RDONLY | O_CLOEXEC);
    if (fd == -1)
        errExit("open");

  	// 将当前进程加入指定 ns, 0 表示自动检测 fd 属于那种 ns
    if (setns(fd, 0) == -1)
        errExit("setns");
    
    execvp(argv[2], &argv[2]);
    errExit("evecvp");

    return 0;
}
```

### 编译

`make setns_manural`

### 输出

```bash
ubuntu@bytedance:~/project/my_project/C/C-program-language/namespace$ sudo ./setns_manural /proc/19800/ns/uts bash
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# hostname
dawson

# 和上面 clone 的 bash 在同一个 uts namespace
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# readlink /proc/self/ns/uts
uts:[4026532204]
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# echo $$
20056

root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# pstree -pl
├─sshd(16181)───sshd(16259)───bash(16260)───sudo(19797)───sudo(19798)───clone_uts_manur(19799)───bash(19800)
├─sshd(19575)───sshd(19622)───bash(19623)───bash(19628)			└─sshd(19924)───sshd(19980)───bash(19981)───sudo(20054)───sudo(20055)───bash(20056)───pstree(20062)
```



> TODO: setns 可以做很多事情, 比如 setns 到某一个 netns 中, 然后create socket / create tun/tap device, 然后 setns 回到原始的 ns 空间中, 这样子就可以监听 namespace 中的网络事件了



## 退出当前进程并新创建的 namespace

### code 

```c
// unshare_manural.c
// 使当前进程退出指定类型的namespace，并加入到新创建的namespace（相当于创建并加入新的namespace）
// 使当前进程加入新的namespace

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>
#include <fcntl.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); } while (0)


static void usage(char *pname)
{
    fprintf(stderr, "Usage: %s [options] program [arg...]\n", pname);
    fprintf(stderr, "Options can be: \n");
    fprintf(stderr, "    -C unshare cgroup namespace\n");
    fprintf(stderr, "    -i unshare IPC namespace\n");
    fprintf(stderr, "    -m unshare mount namespace\n");
    fprintf(stderr, "    -n unshare network namespace\n");
    fprintf(stderr, "    -p unshare PID namespace\n");
    fprintf(stderr, "    -t unshare time namespace\n");
    fprintf(stderr, "    -u unshare UTS namespace\n");
    fprintf(stderr, "    -U unshare user namespace\n");
    
    exit(0);
}

int main(int argc, char *argv[]) {
    int flags = 0;
    int opt;
    pid_t pid =  getpid();
    printf("pid: %jd\n", (intmax_t) pid);

    while ((opt = getopt(argc, argv, "CimnptuU")) != -1) {
        switch (opt) {
        case 'C': flags |= CLONE_NEWCGROUP;  break;
        case 'i': flags |= CLONE_NEWIPC;     break;
        case 'm': flags |= CLONE_NEWNS;      break;
        case 'n': flags |= CLONE_NEWNET;     break;
        case 'p': flags |= CLONE_NEWPID;     break;
        case 'u': flags |= CLONE_NEWUTS;     break;
        case 'U': flags |= CLONE_NEWUSER;    break;
        default: usage(argv[0]);
        }
    }
    if (optind >= argc)
        usage(argv[0]);
    
    if (unshare(flags) == -1)
        errExit("unshare");
    
    execvp(argv[optind], &argv[optind]);
    errExit("evecvp");

    return 0;
}


/*
    $ readlink /proc/$$/ns/mnt
    mnt:[4026531840]
    $ sudo ./unshare -m /bin/bash
    # readlink /proc/$$/ns/mnt
    mnt:[4026532325]
*/
```

### 编译

`make unshare_manural`

### 输出

```bash
# 当前 uts
ubuntu@bytedance:~/project/my_project/C/C-program-language/namespace$ readlink /proc/$$/ns/uts
uts:[4026531838]

ubuntu@bytedance:~/project/my_project/C/C-program-language/namespace$ sudo ./unshare_manural -u bash
pid: 20138

# 当前 bash 还是之前的进程
root@bytedance:/home/ubuntu/project/my_project/C/C-program-language/namespace# echo $$
20138

# 设置 hostname
root@bytedance:/home/ubuntu/project/my_project/C/C-program-language/namespace# hostname dawson
root@bytedance:/home/ubuntu/project/my_project/C/C-program-language/namespace# exec bash
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# echo $$
20138

# 读取当前进程的 uts namespace 的 inode, 与之前的不同
root@dawson:/home/ubuntu/project/my_project/C/C-program-language/namespace# readlink /proc/self/ns/uts
uts:[4026532205]
```



## 内核实现

每个进程都有一个 `struct thread_info` , 该结构体有一个字段 `struct task_struct`

所以可以理解为每个进程都有对应的 namespace, namespace 是基于进程的, 本质是进程隔离机制

```c
struct task_struct {
  ...
  /* namespaces */
  struct nsproxy *nsproxy;
  ...
}
```

```c
static inline struct new_utsname *utsname(void)
{
	return &current->nsproxy->uts_ns->name;
}
```



>  socket 创建的时候先要获取当前进程的 netns, 然后再创建, 这样就达到了 socket 在 netns 中的隔离

```c
int sock_create(int family, int type, int protocol, struct socket **res)
{
	return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
```

