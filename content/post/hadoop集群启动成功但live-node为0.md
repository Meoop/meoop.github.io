---
title: "Hadoop 集群启动成功但 live Node 为 0"
date: 2016-03-05T10:17:04+08:00
lastmod: 2016-03-05T10:17:04+08:00
draft: false
keywords: [Linux, Hadoop]
tags: [Linux, Hadoop]
categories: [Hadoop]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: false
---

> 一次 Hadoop 集群启动成功但 live node 为 0 的问题解决过程。
> 
> 原文在 [CSDN](https://blog.csdn.net/Meoop/article/details/50806724)，现在迁移到这里。
> 

## 问题描述
这两天用虚拟机搭建了 hadoop 集群环境，创建了三台虚拟机 hadoop01，hadoop02，hadoop03，hadoop01 代表 Master，hadoop02 以及 hadoop03 作为 Slave 节点。三台虚拟机都是 centos7 最小化安装。hadoop 版本用的最新的版本 2.7.2。
开始一切都很顺利，启动 hadoop 集群也成功启动，运行 jps，三个节点对应的进程都成功启动，但是检查集群状态时发现 live node 为 0（如图）。

{{< figure src="/images/post/hadoop集群启动成功但live-node为0/20160304233347273.png" class="center" title="集群状态" alt="集群状态" >}}

<!--more-->

## 问题跟踪

这个就让人很郁闷了，检查了下 Slave 节点日志，发现一些异常的地方：

> 2016-03-05 07:34:23,972 WARN org.apache.hadoop.hdfs.server.datanode.DataNode: Problem connecting to server: hadoop01/192.168.116.101:9000
2016-03-05 07:34:29,980 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 0 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:30,985 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 1 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:31,989 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 2 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:32,991 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 3 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:33,995 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 4 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:35,000 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 5 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:36,002 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 6 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:37,003 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 7 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:38,006 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 8 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2016-03-05 07:34:39,009 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: hadoop01/192.168.116.101:9000. Already tried 9 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)

hadoop01 节点的 9000 端口**无法访问**，在 hadoop01 上运行 netstat，发现 hadoop01 上没有监听 9000 端口：

{{< figure src="/images/post/hadoop集群启动成功但live-node为0/20160304234330058.png" class="center" title="集端口状态" alt="集端口状态" >}}

这个就让人很郁闷了，到底什么原因呢，防火墙早就关闭了啊，不应该会出现端口被拒绝。检查了半天也没检查到，最后终于发现罪魁祸首了，原来是 hosts 文件的问题。

在开始时我就将 hadoop01 节点上的 `/etc/hosts` 文件改成了下面的样子：
{{< figure src="/images/post/hadoop集群启动成功但live-node为0/20160304234956138.png" class="center" title="hosts 文件" alt="hosts 文件" >}}

问题就出现在这两个红框出，我在设置时在 127.0.0.1 后面添加了 hadoop01，这样 hadoop 在启动的时候，根据配置文件监听的时候监听的是 hadoop01 的 9000 端口，而这个hadoop01 被解析成了 127.0.0.1，这样 hadoop01 节点就不会监听 192.168.116.101 的 9000 端口，来自 hadoop02 和 hadoop03 的信息不会被 hadoop01 节点接收到，也就会出现 hadoop02 和 hadoop03 节点日志里面的内容，live node 一直为 0。

## 问题解决
方法一：修改hosts文件，删除上边红框的内容。  
方法二：修改hadoop集群配置文件中的主机名，把所有主机名全换成对应的ip地址。

修改后重新启动 hadoop 集群：
{{< figure src="/images/post/hadoop集群启动成功但live-node为0/20160305002038858.png" class="center" title="集群状态" alt="集群状态" >}}
