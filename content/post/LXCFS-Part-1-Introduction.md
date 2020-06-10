---
title: "LXCFS Part 1 - Intro to LXCFS"
date: 2019-08-14T21:29:44+08:00
lastmod: 2019-08-06T21:29:44+08:00
draft: false
keywords: [lxcfs, container]
tags: [lxcfs, container]
categories: [lxcfs, container]
comment: true
toc: true
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

在使用容器部署应用时，我们经常会遇到下面场景：
* 在对 Java 应用容器化部署的过程中，自己设置了容器的资源限制，但是 Java 应用容器在运行中还是会莫名奇妙地被 OOM Killer 干掉。
* 在设置了容器的资源限制的容器中通过 top，uptime，free 等查看信息，读到的是 host 上的信息，不是资源限制后的结果。

这是由于 Linux 容器的资源限制是通过 cgroup 实现的，但在容器内部依旧共享宿主机内核的 `procfs`，其中包含了如：`meminfo`，`cpuinfo`，`stat`，`uptime` 等包含宿主机资源信息的 proc 文件。一些应用或者监控工具如 free/top 还依赖 proc 下的文件内容获取资源配置和使用情况。当它们在容器中运行时，就会把宿主机的资源状态读取出来，引起了错误和不便。LXCFS 就是用来解决这类问题的。
<!--more-->

## LXCFS

### 什么是 LXCFS
为解决容器内资源状态不正确问题，社区中常见的做法是利用 LXCFS 来提供容器中的资源可见性。LXCFS 是一个小型的 FUSE 文件系统，其目的是为了使 Linux 容器更加像虚拟机。它最初是 LXC 的一个子项目，但是可以被其他任何运行时使用。

LXCFS 通过 FUSE 文件系统，为容器提供虚拟的 procfs 文件，通过这几个映射的文件，容器内应用看到的将是容器真正能够使用的资源，不再是 host 系统的全部资源。例如，通过 /proc/uptime 获取到的时间将是应用启动的真正时间，不再是系统启动的时间。LXCFS 可以提供下面几个 procfs 的文件：
* `/proc/cpuinfo`
* `/proc/diskstats`
* `/proc/meminfo`
* `/proc/stat`
* `/proc/swaps`
* `/proc/uptime`
* `/sys/devices/system/cpu/online`

LXCFS 的使用流程大致如下，把宿主机的 LXCFS 虚拟出的 procfs 文件挂载到容器对应的位置，容器中进程读取相应文件内容时，LXCFS 的 FUSE 实现会从容器对应的 cgroup 以及容器 PID 1 的进程文件中读取正确的信息，返回给容器内进程，从而使得应用获得正确的资源约束设定。

### 如何安装 LXCFS
LXCFS 提供了 ubuntu 上的 deb 包，但未提供 centos 上的 rpm 包，我们可以使用二进制部署或者使用容器部署。

#### 容器部署
参考项目 [lxcfs-initializer](https://github.com/denverdino/lxcfs-initializer) 使用 daemonset 方式部署 lxcfs：
```bash
kubectl apply -f https://raw.githubusercontent.com/denverdino/lxcfs-initializer/master/lxcfs-daemonset.yaml
```

#### 二进制部署
从 [linux containers 网站](https://linuxcontainers.org/lxcfs/downloads/)下载最新版本的源码包，编译并安装：
```bash
wget https://linuxcontainers.org/downloads/lxcfs/lxcfs-3.1.2.tar.gz
tar -xvf lxcfs-3.1.2.tar.gz
cd lxcfs-3.1.2
./configure
make && make install
```

使用 systemd 启动 lxcfs：
```bash
systemctl enable lxcfs
systemctl start lxcfs
```

### 如何使用 LXCFS

#### Docker
启动一个容器，用 lxcfs 维护的 /proc 文件替换容器中的 /proc 文件，容器内存设置为 512M，CPU 设置为 1 核：
```bash
docker run -it --rm -m 512m --cpus 1 \
    -v /var/lib/lxcfs/proc/cpuinfo:/proc/cpuinfo  \
    -v /var/lib/lxcfs/proc/diskstats:/proc/diskstats \
    -v /var/lib/lxcfs/proc/loadavg:/proc/loadavg \
    -v /var/lib/lxcfs/proc/meminfo:/proc/meminfo \
    -v /var/lib/lxcfs/proc/stat:/proc/stat \
    -v /var/lib/lxcfs/proc/swaps:/proc/swaps \
    -v /var/lib/lxcfs/proc/uptime:/proc/uptime \
    -v /var/lib/lxcfs/sys/devices/system/cpu/online:/sys/devices/system/cpu/online \
    ubuntu:16.04 /bin/bash
```

从容器中观察 Memory、CPU、启动时间等信息：
```bash
root@8fc32bd1cf47:/# free -h
              total        used        free      shared  buff/cache   available
Mem:           512M        3.6M        503M          0B        5.2M        508M
Swap:            0B          0B          0B

root@8fc32bd1cf47:/# uptime 
 08:14:13 up 0 min,  0 users,  load average: 0.64, 0.54, 0.58

root@8fc32bd1cf47:/# top
top - 08:29:56 up 3 min,  0 users,  load average: 0.97, 1.06, 0.93
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   524288 total,   520268 free,     4020 used,        0 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   520268 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND      
   17 root      20   0   36672   3080   2616 R  0.3  0.6   0:00.01 top         
    1 root      20   0   18232   3292   2820 S  0.0  0.6   0:00.16 bash
```

可以看到容器内显示的资源信息与设置的资源相同，容器内只能看到自己可以使用的资源，以及自己使用了的资源。

#### Kubernetes
对于 Kubernetes 来说，需要将对应的文件 mount 到容器中，有下面几种方式挂载文件：

**方式一**：将 lxcfs 维护的 /proc 文件挂载信息写在 Pod 模板中，要使用 lxcfs 的每个 Pod 都需要写，like:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lxcfs-pod
  namespace: default
spec:
  containers:
  - command:
    - sh
    - -c
    - echo Hello Kubernetes! && sleep 86400
    image: test.caicloudprivatetest.com/library/centos-debug:v1.0
    imagePullPolicy: IfNotPresent
    name: lxcfs-pod
    resources:
      limits:
        cpu: "1"
        memory: 1Gi
      requests:
        cpu: "0.5"
        memory: 512Mi
    volumeMounts:
      - mountPath: /proc/cpuinfo
        name: cpuinfo
      - mountPath: /sys/devices/system/cpu/online
        name: onlinecpu
      - mountPath: /proc/meminfo
        name: meminfo
      - mountPath: /proc/diskstats
        name: diskstats
      - mountPath: /proc/stat
        name: stat
      - mountPath: /proc/swaps
        name: swaps
      - mountPath: /proc/uptime
        name: uptime
  volumes:
    - name: cpuinfo
      hostPath:
        path: /var/lib/lxcfs/proc/cpuinfo
    - name: onlinecpu
      hostPath:
        path: /var/lib/lxcfs/sys/devices/system/cpu/online
    - name: meminfo
      hostPath:
        path: /var/lib/lxcfs/proc/meminfo
    - name: diskstats
      hostPath:
        path: /var/lib/lxcfs/proc/diskstats
    - name: stat
      hostPath:
        path: /var/lib/lxcfs/proc/stat
    - name: swaps
      hostPath:
        path: /var/lib/lxcfs/proc/swaps
    - name: uptime
      hostPath:
        path: /var/lib/lxcfs/proc/uptime
```

**方式二**：通过 [Admission Controller](https://github.com/Meoop/lxcfs-admission-webhook) 将需要挂载的文件自动注入需要使用 lxcfs 的 Pod 。

~~**方法三**：通过 Pod Preset 将需要挂载的文件自动注入需要使用 lxcfs 的 Pod 。~~

~~**方式四**：借鉴阿里云用 Initializers 实现的做法，将 lxcfs 维护的 /proc 文件挂载到每个容器中。~~

## 应用场景及性能分析

已经在生产环境中使用的厂商：
* 阿里
* 腾讯
* 字节跳动
* 蘑菇街
* …

### 应用场景

#### Java 
在 JDK 8u191 和 JDK 10 之后，Java 社区对 JVM 在容器中运行做了优化和增强，JVM 可以自动感知容器内部的 CPU 和内存资源限制，Java 进程可用 CPU 核数由 cpu sets, cpu shares 和 cpu quotas 等参数计算而来，默认情况下该功能已经开启。在这种情况下无需使用 lxcfs。

在 JDK 8u131 时 Java 添加了对 Cgroup 的实验性支持，在这几个版本中，需要添加下面参数开启对 Cgroup 的感知，开启后也无需使用 lxcfs。
```
java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap
```

如果使用的是老版本的 JDK，JVM 缺省的 GC、JIT 编译线程数量来自于宿主机 CPU 核数，内存分配也基于宿主机，如果我们在一个节点上运行多个 Java 应用，即使我们设置了 CPU、Memory 的限制，应用之间依然有可能因为 GC 线程抢占切换，导致应用性能收到影响，甚至由于内存计算错误导致 OOM。JVM 获取 CPU、Memeory 信息是通过 procfs 获取的，在这种场景中 lxcfs 非常有用，通过 lxcfs 虚拟的 procfs 文件，可以正确获取受限的 CPU、Memory 资源，从而提高 Java 应用的性能，避免由于内存计算错误导致的 OOM。

案例：所有使用较低版本的 Java 应用，例如跑在在旧版本的 Java 中的 Tomcat、Hadoop 等。

#### Node.js
Node.js 获取 CPU 信息是通过 procfs 文件系统获取到的，对于 node.js 应用，要获取受限的 CPU 数量信息可以使用 lxcfs。在 nodejs 12.3.1 之前的版本，获取 Memory 信息是通过 sysinfo() 系统调用完成，获取到的是 host 系统的内存系统，无法使用 lxcfs。在之后的版本中获取内存系统的函数会先尝试从 /proc/meminfo 中获取，在这几个版本中可以使用 lxcfs，使 js 获取到分配的实际内存。

案例：流水线编译 nodejs 应用。流水线编译时设置了编译线程数，使用 os.cpus() 获取，这个函数从 procfs 读取 cpu 信息，在不使用 lxcfs 时，获取到宿主机的 cpu 数量，会启动与宿主机 CPU 核数相同的的 node 进程。

#### top/free/uptime
top/free/uptime 是通过 procfs 获取信息的，使用 lxcfs，可以让这些命令在容器中获取到真实的信息。

#### Nginx
Nginx 默认线程数与 cpu 数量相同，从 /proc/cpuinfo 中获取 cpu 数量，可以通过 lxcfs 修正 cpu 数量。

#### Golang 
Golang 调度器默认的线程数以及 GC 机制等跟系统 CPU 数量有关，获取 CPU 信息是直接使用系统调用进行读取，跳过了 procfs 文件，读取到的直接是 host 系统的 CPU 数量，所以使用 lxcfs 没有意义，无法缓解由于设置了 cpu quota 导致的线程竞争问题，无法提高性能。获取系统内存信息的 api 并不在标准库中，对于内存信息的获取方式主要看库如何实现，例如 [pbnjay/memory](https://github.com/pbnjay/memory) 库使用的是 sysinfo() 系统调用，获取到的信息也不通过 procfs，也不需要 lxcfs。**对于 golang 应用，需不需要 lxcfs 还是要看情况。**

对于容器内的 golang 应用，可以借助 [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs) 项目，自动设置 `GOMAXPROCS` 环境变量来避免由于设置了 CPU quota 导致的性能问题。

## 参考文档
1. https://linuxcontainers.org/lxcfs/introduction/
2. https://github.com/lxc/lxcfs
3. https://yq.aliyun.com/articles/566208
4. https://golang.org/pkg/runtime/