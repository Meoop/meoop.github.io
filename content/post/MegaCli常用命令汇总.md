---
title: "MegaCli 常用命令汇总"
date: 2017-05-09T10:17:21+08:00
lastmod: 2017-05-09T10:17:21+08:00
draft: false
keywords: [Linux, MegaCli]
tags: [Linux, MegaCli]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

MegaCli 是 LSI 公司提供的工具, 可以实现对 RAID 卡进行配置和状态监控。下面是一些比较常用的 MegaCli 命令：

<!--more-->

## 状态信息读取
1.1 显示所有逻辑硬盘组信息
```bash
MegaCli -LDInfo -LALL -aAll
```
1.2 显示所有物理硬盘信息
```bash
MegaCli -PDList -aAll
```
1.3 显示 raid 卡信息
```bash
MegaCli -AdpAllInfo -aALL
```
1.4 查询 raid 级别，磁盘数量，容量，条带大小
```bash
MegaCli -ldinfo -lall -aall
```
1.5 查询 cache 策略（Cache Policy 后面跟的是磁盘的缓存策略）
```bash
MegaCli -cfgdsply -aALL |grep Policy
```

## Raid 设置相关
2.1 清除当前 raid 配置(L1 表示编号为 1 的 raid 组；a0 表示 0 号适配器)
```bash
MegaCli -CfgLdDel -L1 -force -a0
```
2.2 清除所有的 raid 组的配置
```bash
MegaCli -cfgforeign -clear -a0
```
2.3 增加 raid0 (r0 表示 raid0；[E0:S0] 表示 `Enclosure Device ID` 为 0，`Slot Number` 是 0 的硬盘；`WB NORA Cached CachedBadBBU` 表示缓存策略)
```bash
MegaCli -CfgLdAdd -r0[E0:S0] WB NORA Cached CachedBadBBU -a0
```
2.4 增加 raid1
```bash
MegaCli -CfgLdAdd -r1[E0:S0,E1:S1] WB NORA Cached CachedBadBBU -a0
```
2.5 增加 raid5
```bash
MegaCli -CfgLdAdd -r5[E0:S0,E1:S1,E2:S2] WB NORA Cached CachedBadBBU -a0
```
2.6 阵列创建完后，会有一个初始化同步块的过程，可以看看其进度。
```bash
MegaCli -LDInit -ProgDsply -LALL -aALL
```

## 电池设置相关
3.1 查看电池信息
```
MegaCli -AdpBbuCmd -aAll
```
3.2 查看电池容量
```
MegaCli -AdpBbuCmd -GetBbuCapacityInfo –aALL
```
3.3 查看充电状态
```
MegaCli -AdpBbuCmd -GetBbuStatus -aALL
```
3.4 查看电池设计参数
```
MegaCli -AdpBbuCmd -GetBbuDesignInfo –aALL
```
3.5 设置电池为学习模式为循环模式
```
MegaCli -AdpBbuCmd -BbuLearn -aN|-a0,1,2|-aALL
```
3.6 设置即使电池坏了还是保持wb功能
```
MegaCli -LDSetProp CachedBadBBU -L0 -a0
```

## 缓存设置相关

### 缓存策略
4.1.1 WriteBack 和 WriteThrough

* WriteBack：进行写操作时，将数据写入 RAID 卡缓存，并直接返回，RAID 卡控制器将在系统负载低或者 Cache 满了的情况下把数据写入硬盘。该设置会大大提升 RAID 卡写性能，绝大多数的情况下会降低系统 IO 负载。数据的可靠性由 BBU(Battery Backup Unit) 电池进行保证。大多数情况下，我们都使用这种策略。
* WriteThrough: 数据写操作不使用缓存，数据直接写入磁盘。RAID 卡写性能下降，在大多数情况下该设置会造成系统 IO 负载上升;

4.1.2 ReadAheadNone, ReadAdaptive, ReadAhead

* ReadAheadNone: 不开启预读。默认的设置；
* ReadAhead: 在读操作时，预先把后面顺序的数据加载入 Cache，在顺序读取时，能提高性能，相反会降低随机读的性能。
* ReadAdaptive: 自适应预读，当 Cache memory 和 IO 空闲时，采取顺序预读，平衡了连续读性能及随机读的性能，需要消耗一定的计算能力。

4.1.3 Cached,Direct

* Direct: Direct IO 模式，读操作不缓存到 Cache memory 中，数据将同时传输到 cache 中和应用，如果接下来要读取相同的数据块，则直接从 Cache memory 中获取. 这是默认的设置。
* Cached: Cached IO 模式，所有读操作都会缓存到 Cache memory 中。

4.1.4 Write Cache OK if Bad BBU, No Write Cacheif Bad BBU

* WriteCache OK if Bad BBU: 在 BBU 有问题时(如电池失效), 依旧使用 WriteCache, 有一定的数据丢失风险。
* NoWrite Cache if Bad BBU: 在 BBU 有问题时, 不使用 Write Cache。

### 缓存模式设置
4.2.1 设置 WriteBack 功能
```bash
MegaCli -LDSetProp WB -L0 -a0
```
4.2.2 设置 WriteThrough 功能
```bash
MegaCli -LDSetProp -WT -L0 -a0
```
4.2.3 设置 No Write Cache if Bad BBU 功能
```bash
MegaCli -LDSetProp -NoCachedBadBBU -Lall -aAll
```
4.2.3 设置 Write Cache OK if Bad BBU 功能
```bash
MegaCli -LDSetProp -CachedBadBBU -Lall -aAll
```

