---
title: "容器实现技术技术及其原理：Namespace"
date: 2017-12-24T11:25:47+08:00
lastmod: 2018-06-24T11:25:47+08:00
draft: false
keywords: [linux, namespace, container]
tags: [Linux, container]
categories: [Linux, container]
comment: true
toc: true
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

容器是当前的热门技术，容器使用到 Linux 的 `namespace` 和 `cgroup` 功能，`namespace` 用于资源隔离，`cgroup` 用于资源限制。除了这两个外，容器还使用到了 selinux/apparmor 增强容器安全，veth pair/bridge/ovs 等技术提供容器网络，aufs/overlayfs/lvm 等技术构建容器的 rootfs。这篇文章主要对 namespace 进行介绍，了解它功能已经使用方式，后续文章再对其他技术进行介绍。

## Linux Namespace

`Namespace` 是 Linux 内核的一项功能，用于对资源进行隔离，`namespace` 以一种抽象的方式包装特定的资源，使得在这个 `namespace` 中的进程实例看起来它们具有自己的受到隔离的全局资源，不同 `namespace` 的进程能看到的资源是不同的。
<!--more-->

当前 Linux Namespace 的类型有 7 种，提供了对 `UTS`、`IPC`、`Mount`、`PID`、`Network`、`User`、`Cgroup` 等资源的隔离机制。

|namespace|Constant|Isolates|
|----|----|----|
|Cgroup  |CLONE_NEWCGROUP |Cgroup root directory (since Linux 4.6)|
|IPC     |CLONE_NEWIPC    |System V IPC, POSIX message queues (since Linux 2.6.19)|
|Network |CLONE_NEWNET    |Network devices, stacks, ports, etc. (since Linux 2.6.24)|
|Mount   |CLONE_NEWNS     |Mount points (since Linux 2.4.19)|
|PID     |CLONE_NEWPID    |Process IDs (since Linux 2.6.24)|
|User    |CLONE_NEWUSER   |User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)|
|UTS     |CLONE_NEWUTS    |Hostname and NIS domain name (since Linux 2.6.19)|

**`Cgroup namespace`** 提供了对 cgroup 资源的隔离，对从 `/proc/[pid]/cgroup` 和 `/proc/[pid]/mountinfo` 获取到的 cgroup 目录提供了虚拟的视图。每个 cgroup namespace 都有自己的 cgroup 根目录被记录在 `/proc/[pid]/cgroup` 中，创建新的 cgroup namespace 时，当前的 cgroup 目录将会作为新的 cgroup namespace 的根目录，cgourp namespace 可以避免保存在 cgroup 中的进程信息被泄露。cgroup namespace 比较新，功能也还尚未稳定，docker 在启动容器中也未使用 cgroup namespace。

**`IPC namespace`** 隔离 IPC 资源，包括 System V IPC 以及 POSIX 消息队列，每个 IPC namespace 都有其自己的 System V IPC 标识符集和自己的 POSIX 消息队列文件系统。在当前 IPC namespace 中的进程可以看到当前 IPC namespace 中的所有 IPC 对象，无法看到其他 IPC namespace 中的 IPC 对象，当 IPC namespace 被销毁时，IPC namespace 中的所有 IPC 对象会被自动销毁。

**`Network namespace`** 提供了与网络相关的资源的隔离，每个 Network namespace 都有其自己的网络设备，IPv4 和 IPv6 地址栈，路由表，防火墙规则表，/proc/net 目录，端口号等。在不同 Network namespace 中的应用可以使用相同的端口号。一个物理网络设备只可以被放入到一个 Network namespace 中，当 Network namespace 释放时，物理设备会被移到原先的 Network namespace 中。veth pair 的实现类与于管道，可以用于连接两个 Network namespace，当 Network namespace 被删除时，在该 namespace 中创建的 veth pair 也会被删除。

**`Mount namespace`** 实现了对文件系统挂载点进行隔离，不同 Mount namespace 中的进程可以具有不同视图的文件系统结构（可以通过 `/proc/[pid]/mounts`, `/proc/[pid]/mountinfo`,和 `/proc/[pid]/mountstats` 看到）。通过 `clone()` 调用创建 Mount namespace 时，会把当前进程的文件系统结构复制给新的 namespace；使用 `unshare()` 方式创建 Mount namespace 时，会将调用者先前的文件系统结构复制给新的 namespace。默认情况下新 namespace 中的所有 mount 操作都只影响自身的文件系统结构，而对外界不会产生任何影响。为解决挂载共享问题，内核中中引入的挂载传播（mount propagation），挂载传播定义了挂载点之间的关系，系统用这些关系决定挂载点中的挂载/卸载操作如何传播到其他挂载点中。关于挂载传播的的更多细节参考 [Mount namespace 文档](http://man7.org/linux/man-pages/man7/mount_namespaces.7.html)。

**`PID namespace`** 对进程 ID 号进行隔离，不同 PID namespace 中的进程可以具有相同的 PID。在 PID namespace 中创建的第一个进程作为 PID 1 的进程，作为该 namespace 的 init 进程，用于管理各种系统初始化任务，并对孤儿进程进行回收。当 PID  namespace 中的 1 号进程消亡时，PID namespace 将会被内核回收，namespace 中的所有进程会收到 SIGKILL 的信号。PID namespace 可以嵌套，在 PID namespace 中的进程只能看到自己 PID namespace 以及嵌套在该 PID namespace 下的 namespace 中的进程。需要注意的是`unshare()` 以及 `setns()` 系统调用创建新的 PID namespace，调用者并不进入新的 PID namespace，接下来创建的子进程才会进入新的 namespace，这个子进程也就随之成为新 namespace 中的 init 进程。这是因为 pid 是在程序创建的时候就确立的，一旦程序进程创建以后，那么它的 PID namespace 的关系（父子进程关系）就被确定下来了，进程不能改变他们对应的 PID namespace，否则会引起歧义。

**`User namespace`** 隔离跟安全相关的用户标识符和属性，例如用户 ID、组 ID、根目录、key 等，在 User namespace 内和外的 UID 和 GID 可以不同。进程可以在 User namespace 之外具有正常的非特权用户 ID，而在 User namespace 内部具有 0 的用户 ID，意味着该进程可以在 User namespace 内的操作具有完全的 root 权限，但对于该 namespace 外则没有 root 权限。从 Linux 3.8 开始，无特权的进程也可以创建 User namespace。

**`UTS namespace`** 隔离由 `uname()` 系统调用返回的两个系统标识符--节点名和 NIS 域名。使用 `sethostname()` 和 `setdomainname()` 系统调用可以设置节点名和节点域名。新创建的 UTS namespace 的 hostname 和 domain 复制自调用者 UTS namespace。在 container 中，UTS namespace 功能允许每个 container 拥有自己的主机名和 NIS 域名。

## Namespace API
与 namespace 相关的主要有 3 个系统调用，分别是 `clone()`、`setns()`、`unshare()`。

### clone()
[clone()](http://man7.org/linux/man-pages/man2/clone.2.html) 系统调用用于创建一个新的进程，如果 `clone()` 系统调用包含 `CLONE_NEW*` 相关的 flag 时，会根据 flag 创建新的 namespace，将子进程作为新 namespace 的成员。`clone()` 系统调用是创建新进程比较常用的函数，也包含了许多跟 namespace 无关的功能。`clone()` 系统调用的定义如下：

``` c
/* Prototype for the glibc wrapper function */

#define _GNU_SOURCE
#include <sched.h>

int clone(int (*fn)(void *), void *child_stack,
                 int flags, void *arg, ...
                 /* pid_t *ptid, void *newtls, pid_t *ctid */ );

```

利用 `clone()` 系统调用创建新的 UTS namespace:

``` c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    sethostname("container",10);         // set hostname
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | SIGCHLD, NULL);          // new UTS Namespace
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

### setns()

[setns()](http://man7.org/linux/man-pages/man2/setns.2.html) 用于将调用者进程加入到指定的命名空间，要加入的 namespace 可以通过 `/proc/[PID]/ns` 看到。

``` c
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sched.h>

int setns(int fd, int nstype);
```

加入到已存在的 namespace 中:

``` c
// usage: ./ns_exec /proc/[PID]/ns/uts /bin/bash
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                       } while (0)

int main(int argc, char *argv[])
{
   int fd;

   if (argc < 3) {
       fprintf(stderr, "%s /proc/PID/ns/FILE cmd args...\n", argv[0]);
       exit(EXIT_FAILURE);
   }

   fd = open(argv[1], O_RDONLY); /* Get file descriptor for namespace */
   if (fd == -1)
       errExit("open");

    if (setns(fd, 0) == -1)       /* Join that namespace */
        errExit("setns");

    execvp(argv[2], &argv[2]);    /* Execute a command in namespace */
    errExit("execvp");
}
```


### unshare()
[unshare()](http://man7.org/linux/man-pages/man2/unshare.2.html) 系统调用将调用者移动到新的 namespace 中。`unshare()` 调用根据包含的 `CLONE_NEW*` 相关 flag，创建对应的 namespace。

``` c
#define _GNU_SOURCE
#include <sched.h>

int unshare(int flags);
```

`unshare` 命令简单实现:

``` c
/* unshare.c

   A simple implementation of the unshare(1) command: unshare
   namespaces and execute a command.

   See https://lwn.net/Articles/531381/
*/
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <sys/wait.h>

#ifndef CLONE_NEWCGROUP         /* Added in Linux 4.6 */
#define CLONE_NEWCGROUP         0x02000000
#endif

/* A simple error-handling function: print an error message based
   on the value in 'errno' and terminate the calling process */

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)
static void usage(char *pname)
{
    fprintf(stderr, "Usage: %s [options] cmd [arg...]\n", pname);
    fprintf(stderr, "Options can be:\n");
    fprintf(stderr, "    -f   fork() before executing cmd "
            "(useful when unsharing PID namespace)\n");
    fprintf(stderr, "    -C   unshare cgroup namespace\n");
    fprintf(stderr, "    -i   unshare IPC namespace\n");
    fprintf(stderr, "    -m   unshare mount namespace\n");
    fprintf(stderr, "    -n   unshare network namespace\n");
    fprintf(stderr, "    -p   unshare PID namespace\n");
    fprintf(stderr, "    -u   unshare UTS namespace\n");
    fprintf(stderr, "    -U   unshare user namespace\n");
    exit(EXIT_FAILURE);
}
int main(int argc, char *argv[])
{
    int flags, do_fork, opt;

    flags = 0;
    do_fork = 0;
    while ((opt = getopt(argc, argv, "CfimnpuU")) != -1) {
        switch (opt) {
        case 'f': do_fork = 1;                  break;
        case 'C': flags |= CLONE_NEWCGROUP;     break;
        case 'i': flags |= CLONE_NEWIPC;        break;
        case 'm': flags |= CLONE_NEWNS;         break;
        case 'n': flags |= CLONE_NEWNET;        break;
        case 'p': flags |= CLONE_NEWPID;        break;
        case 'u': flags |= CLONE_NEWUTS;        break;
        case 'U': flags |= CLONE_NEWUSER;       break;
        default:  usage(argv[0]);
        }
    }

    if (optind >= argc)
        usage(argv[0]);

    if (unshare(flags) == -1)
        errExit("unshare");

    /* If we are unsharing the PID namespace, then the caller is *not*
       moved into the new namespace. Instead, only the children are moved
       into the namespace. Therefore, we support an option that causes
       the program to call fork() before executing the specified program,
       in order to create a new child that will be created in a new PID
       namespace. */

    if (do_fork) {
        if (fork()) {
            wait(NULL);         /* Parent waits for child to complete */
            exit(EXIT_SUCCESS);
        }

        /* Child falls through to execute command */
    }

    execvp(argv[optind], &argv[optind]);
    errExit("execvp");
}
```

## /proc/[pid]/ns
`/proc/[pid]/ns` 下保存有进程所在的 namespace 信息，用户可以通过 `/proc/[pid]/ns` 下看到进程所在的 namespace。如果两个进程指向的 namespace 的 inode 号相同，就说明这两个进程在同一个 namespace 下。`/proc/[pid]/ns` 下的这些文件只要被打开，其 fd 被占用着，那么就算所属的所有进程都已经结束，该 namespace 也会一直存在，可以通过 `setns()` 系统调用进入所维持的 namespace。此外，通过 bind mount 这些文件到其他地方，也可以维持 namespace 在进程退出时 namespace 被回收。

``` bash
$ ls -l /proc/$$/ns         <<-- $$ 表示应用的PID
total 0
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 net -> 'net:[4026532008]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 pid -> 'pid:[4026531836]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 user -> 'user:[4026531837]'
lrwxrwxrwx. 1 meoop meoop 0  3月 28 19:12 uts -> 'uts:[4026531838]'
```

## nsenter and unshare
Linux 系统提供了 `nsenter` 和 `unshare` 两个工具可以对 namespace 进行操作。

`nsenter` 用于进入指定进程的 namespace 运行命令。
``` bash
nsenter -t $$ --uts --ipc --net --pid
```

`unshare` 命令从当前父进程的 namespace 脱离出来，根据参数进入新的 namespace。
``` bash
# 创建新的pid空间，并让其作为pid为1的进程
unshare --fork --pid --mount-proc readlink /proc/self
```

## 参考文档
- [overview of Linux namespaces](http://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/)