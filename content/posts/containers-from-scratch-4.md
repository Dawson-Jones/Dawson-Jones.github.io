---
title: "从零开始创建一个容器之namespace 4.User namespaces"
date: 2023-07-18T15:24:26+08:00
draft: false
tags: ["tech", "clound native", "namespace"]
categories: ["namespace"]
---

User namespaces 允许进程的 user 和 group id 不同于外面的 id. 最为显著的, 一个进程可以在外面是非零的 user id, 在里面是0, 换句话说, 进程在外面是非特权的, 在里面却是有 root 权限的

从 3.8 开始, 不同于其他类型的 ns, 非特权就可以创建 user ns





## code

```c
// demo_userns.c

#define _GNU_SOURCE
#include <sys/capability.h>
#include <sys/wait.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>


#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)


static int child_func(void *arg)
{
    cap_t caps;

    for (;;) {
        printf(
            "eUID = %ld; eGID = %ld;   ",
            (long) geteuid(), (long) getegid()
        );

        caps = cap_get_proc();
        printf("capabilities: %s\n", cap_to_text(caps, NULL));

        if (arg == NULL)
            break;
        
        sleep(5);
    }

    return 0;
}

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];    /* Space for child's stack */


int main(int argc, char *argv[])
{
    pid_t pid;

    pid = clone(child_func, child_stack +  STACK_SIZE,
                CLONE_NEWUSER | SIGCHLD, argv[1]);
    if (pid == -1)
        errExit("clone");

    if (waitpid(pid, NULL, 0) == -1)
        errExit("waitpid");

    return 0;
}

```



## 编译

可能需要 `sudo apt-get install libcap-dev`

`gcc demo_userns.c -lcap -o demo_userns`



## 结果

```bash
bytedance@ubuntu:$ id -u
1000
bytedance@ubuntu:$ id -g
1000

bytedance@ubuntu:$ ./demo_userns

eUID = 65534; eGID = 65534;   capabilities: =ep
```

ep 代表拥有全部的 effective 和 permitted 能力
当 user ns 创建后, 第一个进程位于 ns 中的拥有全部的能力, 这允许进程在其他进程创建之前做一些必要的初始化

userID 和 groupID 不同于容器外, 这允许系统做一些权限检查, 当位于 user ns 中的进程了一些会影响更广泛系统的操作, 比如: 发送了一个信号到外面的 ns 中, 或者访问一个文件

- getuid 和 getgid 会返回调用进程所在的 user ns 的凭据, 如果 userID 在 ns 中没有 map, 系统会返回定义在 `/proc/sys/kernel/overflowuid` 中的值, 默认 65534, 同样 groupID, 返回 `/proc/sys/kernel/overflowgid` 中的值, 最初 user ns 没有做 mapping, 所以返回默认值

- 新的进程有全部的能力在新的 user ns 中, 但是没有在父空间的能力
- user ns 可以嵌套



# 映射 userID 和 groupID

通常, 创建新的 user ns 后的第一步, 是定义 map, 用于该 ns 中的进程的 user ID 和 group ID, 写入 map 信息到 `/proc/PID/uid_map` 和 `/proc/PID/gid_map` 这个信息由一行或多行组成

```
ID-inside-ns   ID-outside-ns   length
```

`ID-inside-ns` 和 `length` 定义了一个 ID 的范围映射到从 `ID-outside-ns`开始的相同的范围



## code

```c
// allow UID and GID mappings to be specified 
// when creating a user namespace.

#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <signal.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <limits.h>
#include <errno.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];    /* Space for child's stack */


struct child_args {
    char **argv;
    int pipe_fd[2]; // synchronize parent and child
};

static int verbose;

static void usage(char *pname)
{
    fprintf(stderr, "Usage: %s [options] cmd [arg...]\n\n", pname);
    fprintf(stderr, "Create a child process that executes a shell command in a new user namespace,\n"
                    "and possibly also other new namespace(s).\n\n");
    
    fprintf(stderr, "Options can be:\n\n");
#define fpe(str) fprintf(stderr, "      %s", str);
    fpe("-i             New IPC namespace\n");
    fpe("-m             New mount namespace\n");
    fpe("-n             New network namespace\n");
    fpe("-p             New PID namespace\n");
    fpe("-u             New UTS namespace\n");
    fpe("-U             New user namespace\n");
    fpe("-M uid_map     Specify UID map for user namespace\n");
    fpe("-G gid_map     Specify GID map for user namespcae\n");
    fpe("               if -M or -G is specified, -U ifs required\n");
    fpe("-v             Display verbose messages\n");
    fpe("\n");
    fpe("Map strings for -M and -G consist of records of the form:\n");
    fpe("\n");
    fpe("   ID-inside-ns    ID-outside-ns   len\n");
    fpe("\n");
    fpe("A map string can multiple records, separated commas;\n");
    fpe("the commas are replaced by newlines before writing to map files\n");

    exit(EXIT_FAILURE);
}


static void update_map(char *mapping, char *map_file)
{
    int fd, i;
    size_t map_len;

    map_len = strlen(mapping);
    for (i = 0; i < map_len; ++i) {
        if (mapping[i] == ',')
            mapping[i] = '\n';
    }

    fd = open(map_file, O_RDWR);
    if (fd == -1) {
        fprintf(stderr, "open %s: %s\n", map_file, strerror(errno));
        exit(EXIT_FAILURE);
    }

    if (write(fd, mapping, map_len) != map_len) {
        fprintf(stderr, "write %s: %s\n", map_file, strerror(errno));
        exit(EXIT_FAILURE);
    }
    
    close(fd);
}

static int child_func(void *arg)
{
    struct child_args *args = (struct child_args *) arg;
    char ch;

    close(args->pipe_fd[1]);
    if (read(args->pipe_fd[0], &ch, 1) != 0) {
        fprintf(stderr, "Failure in child: read from pipe returned != 0\n");
        exit(EXIT_FAILURE);
    }
    if (verbose) {
        printf("message received from parent: %x\n", ch);
    }

    execvp(args->argv[0], args->argv);
    errExit("execvp");
}


int main(int argc, char *argv[]) {
    int flags, opt;
    pid_t child_pid;
    struct child_args args;
    char *uid_map, *gid_map;
    char map_path[PATH_MAX];

    flags = 0;
    verbose = 0;
    uid_map = NULL, gid_map = NULL;

    while ((opt = getopt(argc, argv, "+imnpuUM:G:v")) != -1) {
        switch (opt) {
        case 'i': flags |= CLONE_NEWIPC;    break;
        case 'm': flags |= CLONE_NEWNS;     break;
        case 'n': flags |= CLONE_NEWNET;    break;
        case 'p': flags |= CLONE_NEWPID;    break;
        case 'u': flags |= CLONE_NEWUTS;    break;
        case 'U': flags |= CLONE_NEWUSER;   break;
        case 'M': uid_map = optarg;         break;
        case 'G': gid_map = optarg;         break;
        case 'v': verbose = 1;              break;
        default: usage(argv[0]);            break;
        }
    }
    if ((uid_map || gid_map) && !(flags & CLONE_NEWUSER)) {
        usage(argv[0]);
    }

    args.argv = &argv[optind];
    if (pipe(args.pipe_fd) == -1)
        errExit("pipe");
    
    child_pid = clone(child_func, child_stack + STACK_SIZE,
                    flags | SIGCHLD, &args);
    if (child_pid == -1)
        errExit("clone");
    
    if (verbose)
        printf("%s: PID of child created by clone() is %ld\n",
            argv[0], (long) child_pid);
    
    if (uid_map != NULL) {
        snprintf(map_path, PATH_MAX, "/proc/%ld/uid_map", (long) child_pid);
        update_map(uid_map, map_path);
    }
    if (gid_map != NULL) {
        // https://unix.stackexchange.com/questions/692177/echo-to-gid-map-fails-but-uid-map-success/692194#692194?newreg=6392de4a4bac40bda61383677511bbf9
        // Writing "deny" to the /proc/[pid]/setgroups file 
        // before writing to /proc/[pid]/gid_map 
        // will permanently disable setgroups(2) 
        // in a user namespace and allow writing to /proc/[pid]/gid_map 
        // without having the CAP_SETGID capability in the parent user namespace.
        snprintf(map_path, PATH_MAX, "/proc/%ld/setgroups", (long) child_pid);
        int fd = open(map_path, O_RDWR);    // error check
        write(fd, "deny", strlen("deny"));  // ignore
        close(fd);
        snprintf(map_path, PATH_MAX, "/proc/%ld/gid_map", (long) child_pid);
        update_map(gid_map, map_path);
    }

    close(args.pipe_fd[1]);
    if (waitpid(child_pid, NULL, 0) == -1)
        errExit("waitpid");
    
    if (verbose)
        printf("%s: terminating\n", argv[0]);

    return 0;
}
```

## 编译

`gcc userns_child_exec.c -o userns_child_exec`



## 结果

 `/proc/`*PID*`/uid_map` 文件的拥有者是创建 ns 的 user ID

```bash
# 第一个 shell
bytedance@ubuntu:$ ./userns_child_exec -v -U -M '0 1000 1' -G '0 1000 1' bash
./userns_child_exec: PID of child created by clone() is 53094
message received from parent: 0
root@ubuntu:#
```

```bash
# 拥有者是 bytedance
bytedance@ubuntu:~$ ll /proc/53094/uid_map
-rw-r--r-- 1 bytedance bytedance 0 Jul 19 07:25 /proc/53094/uid_map
```

写入进程必须拥有 CAP_SETUDID 能力



如果使用 `ns_child_exec`会遇到一个问题

```bash
bytedance@ubuntu:$ ../ns_child_exec -U bash
nobody@ubuntu:$ cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000

# 没有权限
nobody@ubuntu:$ echo '0 1000 1' > /proc/$$/uid_map
bash: echo: write error: Operation not permitted
nobody@ubuntu:$ exit
exit

bytedance@ubuntu:$ ./userns_child_exec -v -U -M '0 1000 1' -G '0 1000 1' bash
./userns_child_exec: PID of child created by clone() is 53127
message received from parent: 0
root@ubuntu:# cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh:	0000000000000000
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
```



查看用户和组映射

```bash
# 第一个 shell
bytedance@ubuntu:$ ./userns_child_exec -v -U -M '0 1000 1' -G '0 1000 1' bash
./userns_child_exec: PID of child created by clone() is 53202
message received from parent: 0
root@ubuntu:# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

# 查看当前进程的映射
root@ubuntu:# cat /proc/53202/uid_map
         0       1000          1
# 查看第二个 shell 进程的映射, 和在第二个进程看到的不一样
# 第二个 shell 的 uid 是200, 映射的是 0, 而本 user ns 的 0 映射的是 1000
root@ubuntu:# cat /proc/53195/uid_map
       200          0          1
```

```bash
# 第二个 shell
bytedance@ubuntu:$ ./userns_child_exec -v -U -M '200 1000 1' -G '200 1000 1' bash
./userns_child_exec: PID of child created by clone() is 53195
message received from parent: 0
I have no name!@ubuntu:$ id
uid=200 gid=200 groups=200,65534(nogroup)

# 查看第一个 shell 进程的映射
# 第一个 shell 的 uid 是 0, 映射的 ID-outside-ns 是 200, 因为 200 对于该 user ns 映射的 ID-outside-ns 是 1000
I have no name!@ubuntu:$ cat /proc/53202/uid_map
         0        200          1
I have no name!@ubuntu:$ cat /proc/53195/uid_map
       200       1000          1
```



每个进程都是有一个特定的 user ns, 如果是用 `fork` 或 **没有**`CLONE_NEWUSER` 的 `clone`会使用和父进程相关的 user ns, 可以使用 `setns` 来改变 user ns, 如果在目标ns 中有 `CAP_SYS_ADMIN` 能力

另一方面, `clone(CLONE_NEWUSER)` 会创建一个新的 user ns 并且将子进程放置到该 ns 中, 并且建立了一个父子关系在这两个 ns 中。使用 `unshare(CLONE_NEWUSER)`也会建立这种关系, 不同的是, unshare 会将调用者放置到 user ns 中, 这个 user ns 的父 user ns 是调用者之前的 user ns. 这种父子关系定义了进程在子 ns 中的可能具有的功能



# capabilities

每个进程具有三个相关联的能力集(sets of [capability](https://man7.org/linux/man-pages/man7/capabilities.7.html)): permitted(允许的), effective(有效的), inheritable(继承的),  这里感兴趣的是 effective 能力, 这决定了进程执行特权操作的能力

1. 一个进程在 user ns 中拥有的能力是这个 user ns 拥有的能力的子集(大概是这么理解?)
2. 一个进程在 user ns 拥有的能力, 也在子 user ns 中拥有
3. 当 ns 创建时, kernel 记录了 创建者的 user ID 作为 ns 的 owner,  user ID 的进程拥有 ns 的所有能力, 也就是说 ns 创建后, 其他进程 拥有和父 ns 相同的 user ID 也对 ns 拥有所有的能力



## 例子

**code**

```c
// userns_setns_test.c

#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <limits.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <sys/wait.h>


#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)


static void test_setns(char *pname, int fd)
{
    char path[PATH_MAX];
    ssize_t s;

    s = readlink("/proc/self/ns/user", path, PATH_MAX);
    if (s == -1)
        errExit("readlink");
    
    printf("%s readlink(\"/proc/self/ns/user\") ==> %s\n", pname, path);

    if (setns(fd, CLONE_NEWUSER) == -1)
        printf("%s setns() failed: %s\n", pname, strerror(errno));
    else 
        printf("%s setns() succeeded\n", pname);
}


static int child_func(void *arg)
{
    long fd = (long) arg;
    usleep(100000);

    test_setns("child: ", fd);

    return 0;
}

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];    /* Space for child's stack */


int main(int argc, char *argv[]) {
    pid_t child_pid;
    long fd;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s /proc/PID/ns/user\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    fd = open(argv[1], O_RDONLY);
    if (fd == -1) {
        errExit("open");
    }

    child_pid = clone(child_func, child_stack + STACK_SIZE,
                    CLONE_NEWUSER | SIGCHLD, (void *) fd);
    if (child_pid == -1) {
        errExit("clone");
    }

    test_setns("parent: ", fd);
    printf("\n");

    if (waitpid(child_pid, NULL, 0) == -1)
        errExit("waitpid");

    return 0;
}
```

**编译**

`clang userns_setns_test.c -o userns_setns_test`



**结果**

```bash
bytedance@ubuntu:$ id -u
1000
# 父 user ns
bytedance@ubuntu:$ readlink /proc/$$/ns/user
user:[4026531837]

# child pid 是 55470
bytedance@ubuntu:$ ./userns_child_exec -v -U -M '0 1000 1' -G '0 1000 1' bash
./userns_child_exec: PID of child created by clone() is 55470
message received from parent: 0

# 新的 user ns
root@ubuntu:# readlink /proc/$$/ns/user
user:[4026532318]
```



另一个 shell

```bash
# 确定该 shell 在父 user ns 中
bytedance@ubuntu:$ readlink /proc/$$/ns/user
user:[4026531837]

# 父进程成功进入, 子进程进入失败
bytedance@ubuntu:$ ./userns_setns_test /proc/55470/ns/user
parent:  readlink("/proc/self/ns/user") ==> user:[4026531837]
parent:  setns() succeeded

child:  readlink("/proc/self/ns/user") ==> user:[4026532319]
child:  setns() failed: Operation not permitted
```



![userns 层级](https://static.lwn.net/images/2013/namespaces/userns_hierarchy.png "userns 层级")



# user namespace 与其他类型的 namespaces 结合

创建其他的 namespace 需要 `CAPS_SYS_ADMIN` 能力, 而创建 user namespace 不需要能力并且第一个进程会获得所有的能力, 这意味着, **这个创建了 user ns 的进程可以创建任何其他类型的 ns**

并不需要先 `clone` 一个 `CLONE_NEWUSER` 后, 再在 `clone` 后的子进程中使用 `clone`, 可以在同一个 `clone()(或 unshare())`调用 `CLONE_NEWUSER` 的同时包含其他的 `CLONE_NEW*`, 这种情况下, 内核保证 `CLONE_NEWUSER` 第一个执行

```bash
# 单独创建 uts ns, 失败
bytedance@ubuntu:$ unshare -u bash
unshare: unshare failed: Operation not permitted
# 创建 user ns 和 uts ns, 成功
bytedance@ubuntu:$ unshare -U -u bash
nobody@ubuntu:$ exit

# 当前 hostname 
bytedance@ubuntu:$ uname -n
ubuntu

# 进入 user ns 和 uts ns (非特权进程)
bytedance@ubuntu:$ ./userns_child_exec -v -u -U -M '0 1000 1' -G '0 1000 1' bash
./userns_child_exec: PID of child created by clone() is 79954
message received from parent: 0
root@ubuntu:# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@ubuntu:# cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh:	0000000000000000
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff

# 修改 ns 的名字
root@ubuntu:# hostname dawson
root@ubuntu:# exec bash
root@dawson:# uname -n
dawson
```

```bash
# 重新开一个 shell, ns 外 hostname 不变
bytedance@ubuntu:~$ uname -n
ubuntu
```

