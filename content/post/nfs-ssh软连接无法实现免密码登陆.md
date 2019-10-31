---
title: "nfs+ssh 软连接无法实现免密码登陆"
date: 2016-03-14T10:17:26+08:00
lastmod: 2016-03-14T10:17:26+08:00
draft: false
keywords: [Linux, Hadoop, nfs, ssh]
description: ""
tags: [Linux, Hadoop, nfs, ssh]
categories: [Hadoop]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

> 一次 Hadoop 集群安装过程中 nfs+ssh 软连接无法实现免密码登陆的问题解决过程
> 
> 原文在 [CSDN](https://blog.csdn.net/Meoop/article/details/50886535)，现在迁移到这里。
> 

## nfs+ssh 软连接无法实现免密码登陆

1. 开始以为自己文件名写错，仔细检查未发现文件名有错。
2. 检查文件的权限，发现 authorized_keys 文件的权限是 600，修改成 644 后，问题依旧，无法实现免密码登陆。
3. 以为是 nfs_share 目录的权限问题。因为当时建立时使用的是 root 用户，查看后发现 nfs_share 目录权限无问题。
4. 查看建立的软连接，软连接建立之后所生成的文件权限为 777，尝试各种修改后，完全没有什么用处，依旧无法免密码连接。
<!--more-->

各种谷歌百度，最后找到原因：自己没有关闭 SELinux。
在配置 hadoop 集群时应当提前关闭**防火墙**和**SELinux**（水平比较渣，大神可以配置防火墙及 SELinux）。


## 关闭方法
### 关闭SELinux
1）、永久有效：修改 `/etc/selinux/config` 文件中的 `SELINUX=""` 为 `disabled`，然后重启。  
2）、即时生效：`setenforce 0`

### 关闭防火墙
在 CentOS 6.x 中，可以通过如下命令关闭防火墙：

```shell
$ sudo service iptables stop   # 关闭防火墙服务
$ sudo chkconfig iptables off  # 禁止防火墙开机自启，就不用手动关闭了
```

若用是 CentOS 7，需通过如下命令关闭（防火墙服务改成了 firewall）：

```shell
$ sudo systemctl stop firewalld.service    # 关闭firewall
$ sudo systemctl disable firewalld.service # 禁止firewall开机启动
```


