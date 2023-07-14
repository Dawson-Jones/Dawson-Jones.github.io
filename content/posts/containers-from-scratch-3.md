---
title: "从零开始创建一个容器之namespace 3.PID namespaces"
date: 2023-07-13T20:18:28+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

pid namespace 隔离的是进程 id, 不同的 namespace 可以用相同的进程 id, 和 linux 系统一样, PID 为 1 的 init 进程为非常特殊, 它是 namespace 中的第一个进程, 并负责 namespace 中的某些管理任务



## 代码

```c
// clone_pidns_sleep.c

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sched.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <signal.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE); } while (0)


static int child_func(void *arg) {
    printf("child_func(): PID = %ld\n", (long) getpid());
    printf("child_func(): PPID = %ld\n", (long) getppid());

    char *mount_point = arg;

    if (mount_point != NULL) {
        mkdir(mount_point, 0555);   /* create dir for mount point*/
        if (mount("proc", mount_point, "proc", 0, NULL) == -1)
            errExit("mount");
        
        printf("Mounting procfs at %s\n", mount_point);
    }

    execlp("sleep", "sleep", "600", (char *) NULL);
    errExit("execlp");
}

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];    /* Space for child's stack */

int main(int argc, char *argv[]) {
    pid_t child_pid;
    child_pid = clone(child_func, child_stack + STACK_SIZE, CLONE_NEWPID | SIGCHLD, argv[1]);
    if (child_pid == -1)
        errExit("clone");
    
    printf("PID returned by clone(): %ld\n", (long) child_pid);
    if (waitpid(child_pid, NULL, 0) == -1)
        errExit("waitpid");

    return 0;
}
```

## 编译

`make clone_pidns_sleep`

## 输出

```bash
bytedance@ubuntu:~/project/C-program-language/namespace$ sudo ./clone_pidns_sleep /proc2
# 在默认 namespace 的 pid
PID returned by clone(): 62553
# 在新的 namespace 中的 pid
child_func(): PID = 1

# 父进程 id 为 0
# pid namespace 形成了一个层级结构
# 进程只能看到拥有相同 pid namespace 的其他进程, 或者是嵌套在该 namespace 中的子 namespaces
# 因为父进程在不同的 namespace 中, 子进程看不到父进程, 所以 getppid 为 0
child_func(): PPID = 0
Mounting procfs at /proc2
```



```bash
# 在默认的 namespace 中读取 /proc2, 会报一个错 cannot read symbolic link 'self
bytedance@ubuntu:~$ cd /proc2/
bytedance@ubuntu:/proc2$ ls
ls: cannot read symbolic link 'self': No such file or directory
ls: cannot read symbolic link 'thread-self': No such file or directory

# 进入 namespace 后, 该错误消失
bytedance@ubuntu:/proc2$ sudo nsenter --pid=/proc/62553/ns/pid
bytedance@ubuntu:/proc2$ sudo nsenter -p -t 62553
root@ubuntu:/proc2#
root@ubuntu:/home/bytedance# cd /proc2/
root@ubuntu:/proc2# ls
```



## /proc/PID 和 PID namespace

在 PID namespace 中, /proc/PID 目录仅显示 pid namespace 中的进程及其后代namespace 中的进程



ps 工具依赖 /proc, 所以将。procfs mount 到 /proc 是有必要的, 有两种方式可以实现

- 如果子进程创建时使用了CLONE_NEWNS, 子进程会在一个不同的 mount namespace, 这种情况下 mount 新的 procfs 到 /proc 不会有问题

  稍微改一下程序

  ```c
  child_pid = clone(child_func, 
                  child_stack + STACK_SIZE, 
                  CLONE_NEWNS | CLONE_NEWPID | SIGCHLD, 
                  argv[1]
              );
  ```

  ```c
  // execlp("sleep", "sleep", "600", (char *) NULL);
  pause();
  // umount 掉 /proc, 还原原来的 procfs
  umount(mount_point);
  errExit("execlp");
  ```

  输出
  ```bash
  bytedance@ubuntu:~/project/C-program-language/namespace$ sudo ./clone_pidns_sleep /proc
  PID returned by clone(): 7967
  child_func(): PID = 1
  child_func(): PPID = 0
  Mounting procfs at /proc
  
  ```

  ```bash
  # 另一个 shell
  # 现在的 /proc 是新的 procfs, 所以要指定 1
  bytedance@ubuntu:/$ sudo nsenter --pid=/proc/1/ns/pid bash
  root@ubuntu:/# ps
      PID TTY          TIME CMD
       15 pts/3    00:00:00 bash
       21 pts/3    00:00:00 ps
  # 之前 ps 看到的是父namespace 的全部进程
  root@ubuntu:/# ps -ef
  UID          PID    PPID  C STIME TTY          TIME CMD
  root           1       0  0 06:42 pts/1    00:00:00 ./clone_pidns_sleep /proc
  root          15       0  0 06:44 pts/3    00:00:00 bash
  root          22      15  0 06:44 pts/3    00:00:00 ps -ef
  ```

  结束第一个 shell, 第二个shell 会直接退出, 1号进程退出, 会导致其他进程也退出

- 子进程使用 chroot() 来改变根目录, 然后在`/proc` mount 一个 procfs

  演示: TODO



```bash
bytedance@ubuntu:~/project/C-program-language/namespace$ ps -C sleep -C clone_pidns_sleep -o "pid ppid stat cmd"
    PID    PPID STAT CMD
   8108    8107 T+   ./clone_pidns_sleep /proc2
   8109    8108 S+   sleep 600

# 两个进程在不同的 pid ns 中
bytedance@ubuntu:~/project/C-program-language/namespace$ sudo readlink /proc/8108/ns/pid
pid:[4026531836]
bytedance@ubuntu:~/project/C-program-language/namespace$ sudo readlink /proc/8109/ns/pid
pid:[4026532306]
```

