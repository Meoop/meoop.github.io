---
title: "LXCFS Part 2 - Implementation Details"
date: 2019-08-22T21:58:32+08:00
lastmod: 2019-08-22T21:58:32+08:00
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

[上一篇文章我们主要了解了 LXCFS 的使用场景以及在 docker/kubernetes 中如何使用 LXCFS](https://blog.meoop.me/post/lxcfs-part-1-introduction/)，这篇文章我们从源码入手，了解下 LXCFS 是如何实现上面功能的。LXCFS 是基于 fuse 实现的虚拟文件系统，理解 LXCFS，我们需要先了解 fuse，然后再对 LXCFS 的源码进行解读。下面代码解读是在 lxcfs-3.1.2 的代码基础上进行分析的。
<!--more-->

## FUSE
FUSE（filesystem in userspace)，用户空间文件系统，是将文件系统实现在用户空间的一套框架。Linux 的文件系统都工作在内核态，要写一个完整的文件系统就需要在内核中添加代码，内核代码是比较复杂的，这给新文件系统的实现和调试带了极大的困难。FUSE 正是解决该问题的，它将内核态的文件系统实现放在用户空间去实现。FUSE 有很多应用，例如 glusterfs client 端的实现，[sshfs](https://github.com/libfuse/sshfs) 系统实现等。

FUSE 主要分为两部分：FUSE 内核模块（在内核代码中维护）和 libfuse 用户空间库。libfuse 提供了与 FUSE 内核模块通信的参考实现。FUSE 文件系统为基于 libfuse 的用户空间程序实现，libfuse 提供了文件系统的 mount/unmount，从内核读取请求以及发送响应的函数。FUSE 文件系统的流程非常简单，如下图所示，内核截获到文件调用，交给对应的用户空间函数去执行，然后返回对应结果。实现 FUSE 文件系统主要工作是要实现对应的文件接口（[fuse_operations](https://github.com/libfuse/libfuse/blob/82e10e576e6c11eeaa13ad45421578401b14f180/include/fuse.h#L277-L779)）。

{{< figure src="/images/post/LXCFS-Part-1-Implementation/FUSE_structure.png" class="center" title="" >}}

## LXCFS
熟悉了 FUSE，接下来我们再来看 lxcfs。从 procfs 文件系统中获取信息，归结到文件调用就两个：`open` 和 `read`。

```c
// lxcfs.c: 919
const struct fuse_operations lxcfs_ops = {
	// ...code...
	.open = lxcfs_open,
	.read = lxcfs_read,
	// ..code....
}
```

### open
我们先来看 `open`，`open` 指向的是 `lxcfs_open()` 函数，`lxcfs_open()` 函数根据路径执行不同的函数，参数 `path` 表示访问的文件路径，`fuse_file_info` 结构体保存了文件信息。

```c
// lxcfs.c: 742
static int lxcfs_open(const char *path, struct fuse_file_info *fi)
{
	int ret;
	if (strncmp(path, "/cgroup", 7) == 0) {
		up_users();
		ret = do_cg_open(path, fi);
		down_users();
		return ret;
	}
	if (strncmp(path, "/proc", 5) == 0) {
		up_users();
		ret = do_proc_open(path, fi);
		down_users();
		return ret;
	}
	if (strncmp(path, "/sys", 4) == 0) {
		up_users();
		ret = do_sys_open(path, fi);
		down_users();
		return ret;
	}
	return -EACCES;
}
```

路径为 `/proc` 调用的函数是 `do_proc_open()`，`do_proc_open()` 最终调用的函数是 `proc_open()`，`proc_open()` 根据访问路径，分配对应的内存，这段内存在 `read` 时用来保存文件内容，到这里 `open` 函数就结束了。

```c
// bindings.c: 5877
int proc_open(const char *path, struct fuse_file_info *fi)
{
	int type = -1;
	struct file_info *info;

	if (strcmp(path, "/proc/meminfo") == 0)
		type = LXC_TYPE_PROC_MEMINFO;
	else if (strcmp(path, "/proc/cpuinfo") == 0)
		type = LXC_TYPE_PROC_CPUINFO;
	else if (strcmp(path, "/proc/uptime") == 0)
		type = LXC_TYPE_PROC_UPTIME;
	else if (strcmp(path, "/proc/stat") == 0)
		type = LXC_TYPE_PROC_STAT;
	else if (strcmp(path, "/proc/diskstats") == 0)
		type = LXC_TYPE_PROC_DISKSTATS;
	else if (strcmp(path, "/proc/swaps") == 0)
		type = LXC_TYPE_PROC_SWAPS;
	else if (strcmp(path, "/proc/loadavg") == 0)
		type = LXC_TYPE_PROC_LOADAVG;
	if (type == -1)
		return -ENOENT;

	info = malloc(sizeof(*info));
	if (!info)
		return -ENOMEM;

	memset(info, 0, sizeof(*info));
	info->type = type;

	info->buflen = get_procfile_size(path) + BUF_RESERVE_SIZE;
	do {
		info->buf = malloc(info->buflen);
	} while (!info->buf);
	memset(info->buf, 0, info->buflen);
	/* set actual size to buffer size */
	info->size = info->buflen;

	fi->fh = (unsigned long)info;
	return 0;
}
```

### read
`read` 调用 `lxcfs_read()`，根据对应路径执行不同的函数，相对于 `open` 函数多了 `buf` 相关的信息，这个 `buf` 是用来保存文件内容的。

```c
// lxcfs.c: 768
static int lxcfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi)
{
	int ret;
	if (strncmp(path, "/cgroup", 7) == 0) {
		up_users();
		ret = do_cg_read(path, buf, size, offset, fi);
		down_users();
		return ret;
	}
	if (strncmp(path, "/proc", 5) == 0) {
		up_users();
		ret = do_proc_read(path, buf, size, offset, fi);
		down_users();
		return ret;
	}
	if (strncmp(path, "/sys", 4) == 0) {
		up_users();
		ret = do_sys_read(path, buf, size, offset, fi);
		down_users();
		return ret;
	}
	return -EINVAL;
}
```

同样我们关注 `/proc` 下的文件，`do_proc_read()` 最终调用的函数是 `proc_read()`，`proc_read()` 根据访问路径，分别执行不同的函数，用来填充 `buf`，各个文件获取信息的方式不同。

```c
// bindings.c: 5935
int proc_read(const char *path, char *buf, size_t size, off_t offset,
		struct fuse_file_info *fi)
{
	struct file_info *f = (struct file_info *) fi->fh;

	switch (f->type) {
	case LXC_TYPE_PROC_MEMINFO:
		return proc_meminfo_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_CPUINFO:
		return proc_cpuinfo_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_UPTIME:
		return proc_uptime_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_STAT:
		return proc_stat_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_DISKSTATS:
		return proc_diskstats_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_SWAPS:
		return proc_swaps_read(buf, size, offset, fi);
	case LXC_TYPE_PROC_LOADAVG:
		return proc_loadavg_read(buf, size, offset, fi);
	default:
		return -EINVAL;
	}
}
```

要获取启动信息等信息时无法避免需要知道容器内进程对应到 host 上对应的 PID，lxcfs 使用了一个巧妙的办法获取到容器内 PID 为 1 的进程对应到 host 上的进程 PID：
* 调用 `fuse_get_context()` 获得正在发起读请求的进程的 `current_pid`，`current_pid` 是 lxcfs 所在 namespace 中的进程号，不是容器中看到的进程号。
* 调用 `lookup_initpid_in_store()`，根据上面获取到的进程号查询容器内进程 PID=1 的进程的 PID，如果缓存中没有，则通过下面方式获取：
  * 创建一个 `socketpair`（一对 `socket`），`sock[0]` 和 `sock[1]`；
  * fork 一个子进程，父进程准备从 `sock[1]` 中读数据，子进程准备向 `sock[0]` 写数据；
  * 子进程通过发起请求进程的 `/proc/%d/ns/pid` 查询到 pid namespaces（即容器的 pid namespaces），`clone()` 一个新进程并将其放到容器的 pid namespaces 中；
  * 孙子进程使用指定的 `socket` 控制头（控制头中发送者的进程 pid=1）发送发送信息到 `sock[0]` 中；
  * 父进程读取 `sock[1]` 的控制头，父进程在容器空间之外，所以读到的是容器内 PID=1 的进程在容器外的 PID。

### /proc/uptime
`/proc/uptime` 文件中包含两个数值标识系统启动时间信息，使用单位为秒，第一个数字表示系统运行了多长时间，第二个数字表示系统空闲了多长时间，例如：

```bash
root@f3e726b2ff83:/# cat /proc/uptime 
177685.78 177650.54
```

计算 `/proc/uptime` 的值涉及到 cgroup 和 `/proc/$PID/` 下的文件，计算方式如下：
* 通过容器外的进程号获取到  cpuacct cgroup 子系统，读取 cpuacct.usage 文件（实际使用到的 CPU 时间，时间 ns），计算 cpu 使用时间（busytime）；
* 通过容器 PID=1 的进程 `/proc/$PID/stat` 文件，获取到进程启动时间，作为为系统启动运行的时间（reaperage），作为 `/proc/uptime` 的第一个数字；
* `idletime = reaperage - busytime`，idletime 作为 `/proc/uptime` 的第二个数字；

### /proc/meminfo
`/proc/meminfo` 包含了有关系统内存使用情况的所有统计信息，结构如下所示。具体字段的含义参考 [PROC(5)](https://man7.org/linux/man-pages/man5/proc.5.html)。

```bash
root@2fa248c8b660:/# cat /proc/meminfo 
MemTotal:         524288 kB    <-- 总内存
MemFree:          519848 kB    <-- LowFree+HighFree
MemAvailable:     521300 kB    <-- 可以用于新程序启动的内存，不包含 swap
Buffers:               0 kB    
Cached:             1452 kB    
SwapCached:            0 kB
Active:              528 kB
Inactive:           1452 kB
Active(anon):        528 kB
Inactive(anon):        0 kB
Active(file):          0 kB
Inactive(file):     1452 kB
Unevictable:           0 kB
Mlocked:              48 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:               580 kB
Writeback:             0 kB
AnonPages:       3879568 kB
Mapped:          1282988 kB
Shmem:                 0 kB
KReclaimable:     178736 kB
Slab:               0 kB
SReclaimable:          0 kB
SUnreclaim:            0 kB
KernelStack:       16128 kB
PageTables:        54060 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     8044584 kB
Committed_AS:   15352260 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:             3584 kB
HardwareCorrupted:     0 kB
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      338852 kB
DirectMap2M:    10860544 kB
DirectMap1G:     5242880 kB
```

`/proc/meminfo `的信息主要从 [memory cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) 中获取，主要读取下面几个文件：
```
memory.limit_in_bytes        --> 容器可用内存
memory.usage_in_bytes        --> 容器已使用内存
memory.stat                  --> 容器内存状态信息
memory.memsw.limit_in_bytes  --> 容器可以使用的缓存
memory.memsw.usage_in_bytes  --> 容器已经使用的缓存
```

从这几个文件读取到信息后填充虚拟的 `/proc/meminfo`，例如：
* 从 memory.limit_in_bytes 获取到容器内存，作为 MemTotal
* 从 memory.usage_in_bytes 获取到已使用内存，`MemFree = MemTotal - memory.usage_in_bytes`
* 其他字段通过 stat 文件等信息进行对应计算

### /proc/cpuinfo
`/proc/cpuinfo` 文件包含了 cpu 相关的信息，结构如下所示。具体字段的含义参考 [PROC(5)](https://man7.org/linux/man-pages/man5/proc.5.html)。

```bash
root@2fa248c8b660:/# cat /proc/cpuinfo 
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 142
model name	: Intel(R) Core(TM) i7-8565U CPU @ 1.80GHz
stepping	: 11
microcode	: 0xb8
cpu MHz		: 800.028
cache size	: 8192 KB
physical id	: 0
siblings	: 4
core id		: 0
cpu cores	: 4
apicid		: 0
initial apicid	: 0
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb invpcid_single ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp md_clear flush_l1d arch_capabilities
bugs		: spectre_v1 spectre_v2 spec_store_bypass mds swapgs
bogomips	: 3984.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 39 bits physical, 48 bits virtual
power management:
```

lxcfs 实现的 `/proc/cpuinfo` 与 host 上的极为相似，主要区别在于 cpu 的数量，其他的内容由 host 上的 `/proc/cpuinfo` 填充，这里也主要关注 cpu 核数的计算：
* 读取 cpuset cgroup 子系统的 cpuset.cpus 文件，表示可以使用的 cpu
* 如果 cpu,cpuacct cgroup 子系统存在，则应用 cpu 视图，根据 cpu quota 计算 cpu 数量
* 获取 cpu.cfs_quota_us
* 获取 cpu.cfs_period_us
* cpu num 为 `cfs_quota/cfs_period` 向上取整

### /proc/swaps
`/proc/swaps` 文件包含了交换分区相关的信息，文件格式如下：

```bash
root@2fa248c8b660:/# cat /proc/swaps 
Filename				Type		Size	Used	Priority
none                          virtual	524288	0	0
```

在 lxcfs 中，写死了 Filename 为 none，Type 为 virtual，Priority 为 0。其他的信息主要从 [memory cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) 中获取，主要读取下面几个文件：

```
memory.limit_in_bytes 
memory.usage_in_bytes 
memory.memsw.limit_in_bytes
memory.memsw.usage_in_bytes
```

计算方式为：
* swap_total = (memswlimit - memlimit) / 1024;
* swap_free = (memswusage - memusage) / 1024;

### /proc/loadavg
`/proc/loadavg` 包含了系统的负载信息，结构如下：

```bash
root@2fa248c8b660:/# cat /proc/loadavg 
0.29 0.36 0.40 2/1001 24713
```

该文件中的前三个字段是 CPU 或者 I/O 的负载平均数，分别取平均时间为 1 分钟，5 分钟和 15 分钟的值。 第四个字段由两个用斜杠（/）分隔的数字组成，第一个是当前可运行的内核调度实体（进程，线程）的数量，斜杠后面的值是系统上当前存在的内核调度实体的数量。 第五个字段是最近在系统上创建的进程的 PID 。

loadavg 的修正有两种方式，一种时后台启动 load_daemon 线程，定时计算负载信息，再通过修正函数修正，另外一种时直接通过固定函数计算。load 信息会保存在 hash 表中，如果第一次读取需要初始化 hash（bindings.c: 5719） 表，后续读取时会从 hash 表中读取上一次的信息，修正下一次的结果。

修正函数：
```c
#define FSHIFT		11		/* nr of bits of precision */
#define FIXED_1		(1<<FSHIFT)	/* 1.0 as fixed-point */
#define LOAD_INT(x) ((x) >> FSHIFT)
#define LOAD_FRAC(x) LOAD_INT(((x) & (FIXED_1-1)) * 100)
```

```c
// bingdings.c: 5746
// n->avenrun[0]: former load (1min)
// n->avenrun[1]: former load (5min)
// n->avenrun[2]: former load (15min)
	a = n->avenrun[0] + (FIXED_1/200);  
	b = n->avenrun[1] + (FIXED_1/200);
	c = n->avenrun[2] + (FIXED_1/200);
	total_len = snprintf(d->buf, d->buflen, "%lu.%02lu %lu.%02lu %lu.%02lu %d/%d %d\n",
		LOAD_INT(a), LOAD_FRAC(a),
		LOAD_INT(b), LOAD_FRAC(b),
		LOAD_INT(c), LOAD_FRAC(c),
		n->run_pid, n->total_pid, n->last_pid);
```

如果启动时添加 -l 参数，则会运行单独的 `load_daemon()` 线程定时去计算 load，计算方式如下：
* 根据 cpu cgroups 的 cgroup.procs 文件统计容器内的进程，统计该 cgroups 下的子结点的进程
* 根据上面统计到的进程 PID，到 `/proc/$PID/task` 下找到所有的 task
* 所有的任务数，作为 total_pid
* 统计 “S” 和 “D” 状态 task 数，为 run_pid
* 找到 PID 最大的进程，作为 last_pid
* 修正 1 分钟，5 分钟，15 分钟的负载值
* 修正公式： `load1 = load0 * exp + active * (1 - exp)`

### /proc/stat
`/proc/stat` 文件的内容表示的是内核和系统的状态信息，文件格式如下所示：
```bash
[root@localhost cpu]# cat /proc/stat
cpu  328886 1000 91078 3348444 21129 31272 7488 0 0 0
cpu0 78248 237 19859 835397 5186 16737 1278 0 0 0
cpu1 73197 240 22745 848707 5181 5810 1105 0 0 0
cpu2 85215 276 23129 836216 5126 3942 3754 0 0 0
cpu3 92225 245 25345 828122 5634 4782 1350 0 0 0
intr 33888944 23 8115 0 0 0 0 0 0 0 119423 0 0 224 0 0 0 556 13735274 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 371 0 0 0 0 0 0 0 0 0 0 0 287540 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 878998 0 20 22 1822499 43624 39378 50873 38019 37 407 407 0 0 0 0 0 0 0 0 0 0 0 0 0 0 279782 911 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ctxt 27699895
btime 1566523572
processes 11349
procs_running 1
procs_blocked 0
softirq 20884042 1371835 12001595 13 377657 27 0 11088 3466661 388 3654778
```

`/proc/stat` 文件的具体每个字段的含义参考 [PROC(5)](https://man7.org/linux/man-pages/man5/proc.5.html) 文档，lxcfs 修正时只关心 cpu 相关的行，根据 cpu 视图信息，重新矫正各个字段。

```bash
cpu  328886 1000 91078 3348444 21129 31272 7488 0 0 0
cpu0 78248 237 19859 835397 5186 16737 1278 0 0 0
cpu1 73197 240 22745 848707 5181 5810 1105 0 0 0
cpu2 85215 276 23129 836216 5126 3942 3754 0 0 0
cpu3 92225 245 25345 828122 5634 4782 1350 0 0 0
```

cpu 后面每个字段的含义为：
* `user` -- Time spent in user mode.
* `nice` -- Time spent in user mode with low priority (nice)
* `system` -- Time spent in system mode.
* `idle` -- Time spent in the idle task.  This value  should be USER_HZ times the second entry in the /proc/uptime pseudo-file.
* `iowait` -- Time waiting for I/O to complete.
* `irq` --Time servicing interrupts.
* `softirq` -- Time servicing softirqs.
* `steal` -- Stolen time, which is the time spent in other operating systems when running in a virtualized environment
* `guest` -- Time spent running a virtual CPU for guest operating systems under the control of the Linux  kernel.
* `guest_nice` --Time spent running a niced guest (virtual CPU for guest operating systems under the control of the Linux kernel).

修正方式为：
* 修正 cpu 数量，与 `/proc/cpuinfo` 修正方式一样
* 读取 cpu cgroup controller 中的文件，修正各个字段
  * 读取 cpuacct.usage_all，获取在各个 cpu 上执行的 `system/user` 时间
  * 读取 cpu.stat，获取 cpu 使用状态信息
  * 只呈现 `user`、`system`、`idle` 时间，其它都为 0
  * 只设置 cpu quota 没有设置 cpuset，需要将其他 cpu 上的时间重新分到可用 cpu 上，调用方法 `cpuview_proc_stat()`
  * 计算 idle 时间
    * 从系统 `/proc/stat` 获取总使用时间
    * `all_used = user + nice + system + iowait + irq + softirq + steal + guest + guest_nice`
    * 从 cpuacct.usage_all 获取在各个 cpu 的使用时间
    * `cg_used = cg_cpu_usage[curcpu].user + cg_cpu_usage[curcpu].system`;
    * `cg_cpu_usage[curcpu].idle = idle + (all_used - cg_used)`;

## 参考资料
1. https://linuxcontainers.org/lxcfs/introduction/
2. https://github.com/lxc/lxcfs
3. https://github.com/libfuse/libfuse
4. [Lxcfs根据cpu-share、cpu-quota等cgroup信息生成容器内的/proc文件（上）](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2019/02/11/lxcfs-support-cpu-share-and-cpu-quota-1.html)
5. [Lxcfs根据cpu-share、cpu-quota等cgroup信息生成容器内的/proc文件（中）](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2019/02/11/lxcfs-support-cpu-share-and-cpu-quota-2.html)
6. [Lxcfs根据cpu-share、cpu-quota等cgroup信息生成容器内的/proc文件（下）](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2019/02/11/lxcfs-support-cpu-share-and-cpu-quota-3.html)



