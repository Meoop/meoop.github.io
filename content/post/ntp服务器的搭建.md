---
title: "NTP 服务器的搭建"
date: 2016-08-08T10:17:32+08:00
lastmod: 2016-08-08T10:17:32+08:00
draft: false
keywords: [Linux, ntp]
description: ""
tags: [Linux, ntp]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

由于制造工艺多种多样，所有的(非原子)时钟并不按照完全一致的速度行走。有一些时钟走的比较快而有一些走的比较慢。因此经过很长一段时间以后，一个时钟的时间慢慢的和其它的发生偏移，这就是常说的 “时钟漂移” 或 “时间漂移”。时钟的一致对一些应用有着至关重要的作用，为了将时钟漂移的影响降低到最小，需要将时钟进行同步，这就需要 ntp 服务。
网络时间协议 (NTP) 用来同步网络上不同主机的系统时间。你管理的所有主机都可以和一个指定的被称为 ntp 服务器的时间服务器同步它们的时间。而另一方面，一个 ntp 服务器会将它的时间和任意公共 ntp 服务器，或者你选定的服务器同步。由 ntp 管理的所有系统时钟都会同步精确到毫秒级。
<!--more-->

## 安装 ntp 服务器
### 使用包管理器安装

- Ubuntu/Debian
```
sudo apt-get install ntp              # 安装ntp
sudo systemctl enable ntp.service     # 设置ntp开机自启动
sudo systemctl restart ntp.service    # 启动ntp服务
```

- Centos7/Redhat7
```
sudo dnf install ntp                  # 安装ntp
sudo systemctl enable ntpd.service    # 设置ntp开机自启动
sudo systemctl start ntpd.service     # 启动ntp服务
```

__注意：__ntp 服务器启动后，服务器需要需要与其 server 进行时间同步，这需要一个时间段，这个过程可能达到 5 分钟，在这个时间段内，客户机与服务器进行同步将可能出现`no server suitable for synchronization found`错误。

### 通过 ntp 源码包安装
发行版的源里面的 ntp 包不一定是最新的，有时候我们需要使用最新的 ntp 版本，这时候就需要通过 ntp 官网发布的 ntp 源码进行安装 ntp 服务了，下面是具体的安装方式。
```
# 下载ntp源码
wget https://www.eecis.udel.edu/~ntp/ntp_spool/ntp4/ntp-4.2/ntp-4.2.8p8.tar.gz   
tar -xvf ntp-4.2.8p8.tar.gz           # 解压源码
cd ntp-4.2.8p8.tar.gz                
./configure
make                                  # 编译源码
sudo make install                     # 安装
```

### 查询 ntp 服务状态
命令`ntpq -p`可以打印出该 ntp 服务器已知的节点列表和它们的状态概要信息。`ntpq -p`输出如下这样一个表：

```
    remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
*ns.buptnet.edu. 10.3.8.150       5 u    7   64    1    7.607   -8.767   1.006
```

|  |  |
|:--|:--|
|remote |源在 ntp.conf 中定义。`*` 表示当前使用的，也是最好的源；`+` 表示这些源可作为 NTP 源；`-` 标记的源是不可用的。|
|refid	|用于和本地时钟同步的远程服务器的 IP 地址。|
|st	|Stratum（阶层）|
|t	|类型。 `u` 表示单播 (unicast)。其它值包括本地 (local)、多播 (multicast)、广播 (broadcast)。|
|when|自从上次和服务器交互后经过的时间(以秒数计)。|
|poll	|和服务器的轮询间隔，以秒数计。|
|reach	|表示和服务器交互是否有任何错误的八进制数。值 337 表示 100% 成功（即十进制的 255）。|
|delay	|服务器和远程服务器来回的时间。|
|offset	|我们服务器和远程服务器的时间差异，以毫秒数计。|
|jitter	|两次取样之间平均时差，以毫秒数计。|

关于`ntpq -p`的输出细节可以参考[这篇文章](https://linux.cn/article-4664-1.html)，这里不多加叙述。


## ntp 服务器的配置
ntp 服务的配置文件在`/etc/ntp.conf`   
ntp 配置官方文档：http://support.ntp.org/bin/view/Support/ConfiguringNTP  

### 设置上层服务器
ntp 上层服务器，Ubuntu 上默认配置为 ubuntu 提供的 ntp 服务器，通过上游 Ubuntu ntp 服务器来更新本地时间，可以更改这几行，添加其他 ntp 服务器或禁止与 Ubuntu 提供的 ntp 服务器对时。

```
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.

# 禁用Ubuntu提供的ntp服务器
#pool 0.ubuntu.pool.ntp.org iburst
#pool 1.ubuntu.pool.ntp.org iburst
#pool 2.ubuntu.pool.ntp.org iburst
#pool 3.ubuntu.pool.ntp.org iburst
# Use Ubuntu's ntp server as a fallback.
#pool ntp.ubuntu.com

# 设置自己的的ntp上游服务器
server 192.168.1.1 
```

### 访问控制
ntp 服务器默认情况下对所有机器都提供 ntp 服务，这可能导致一些安全隐患，可以通过设置 ntp 服务的访问策略，进行一些服务的限制，保证 ntp 服务器的安全。

```
# By default, exchange time with everybody, but don't allow configuration. 
restrict -4 default kod notrap nomodify nopeer noquery limited
restrict -6 default kod notrap nomodify nopeer noquery limited

# Add the following line to allow a subnet to receive time service and query server
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap nopeer
restrict 2001:838:0:1:: mask ffff:ffff:ffff:ffff:: nomodify notrap nopeer
```

### 以本地时间作为对时时间
设置外部时间服务器不可用，以本地时间作为时间服务

```
server 127.127.1.0 #local clock
fudge 127.127.1.0 stratum 10
```

## 客户机配置
客户机可以通过两种方式进行对时:

- ntpdate 进行对时
```
sudo apt-get install ntpdate   # 安装ntpdate命令
sudo ntpdate 192.168.1.2       # 与192.168.1.2进行对时
```
使用 ntpdate 对时只会时间更新为当前服务器的时间，若后续出现始终漂移，还需要重新执行 ntpdate 进行对时。如果保持时间一直同步，可以使用第二种方法。

- 通过 ntp 服务对时
可以同样在客户机安装 ntp 服务，注释掉其他上游 ntp 服务器，设置对时服务器为前面部署的 ntp 服务器为上游服务器进行对时即可。
```
server 192.168.1.1 
```

## 参考文章
[1]. [Configuring NTP](http://support.ntp.org/bin/view/Support/ConfiguringNTP)  
[2]. [网络时间的那些事及ntpq详解](https://linux.cn/article-4664-1.html)  
[3]. [如何在CentOS中设置NTP服务器](https://linux.cn/article-5581-1-rel.html)  