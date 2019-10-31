---
title: "Linux 修改主机名"
date: 2016-03-04T10:17:15+08:00
lastmod: 2016-03-04T10:17:15+08:00
draft: false
keywords: [Linux, hostname]
tags: [Linux, hostname]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

> 原文在 [CSDN](https://blog.csdn.net/Meoop/article/details/50804308)，作部分更新整理后迁移到这里。

在 CentOS 或 RHEL 中，有三种定义的主机名:

1. 静态的（static），**静态**主机名也称为内核主机名，是系统在启动时从`/etc/hostname`自动初始化的主机名。
2. 瞬态的（transient），**瞬态**主机名是在系统运行时临时分配的主机名，例如，通过 DHCP 或 mDNS 服务器分配。静态主机名和瞬态主机名都遵从作为互联网域名同样的字符限制规则。
3. 灵活的（pretty），**灵活**主机名是允许使用自由形式（包括特殊/空白字符）的主机名，展示给终端用户的主机名。

<!--more-->
## 通过 hostname 修改
CentOS 可以使用 hostname 修改主机。但是这样修改重启就失效了。还需要在 `/etc/hostname` 文件中一并修改才可以。

```bash
$ hostname                    # 查看主机名
$ sudo hostname  <host-name>  # 临时设置主机名，重启后失效
$ sudo vim /etc/hostname      # 修改/etc/hostname文件内容
```

{{< figure src="/images/post/Linux修改主机名/1.png" class="center" title="通过 hostname 修改主机名" alt="通过 hostname 修改主机名" >}}

## 通过 hostnamectl 命令修改
在 CentOS7 中，可以通过 `hostnamectl` 命令查看或修改与主机名相关的配置。查看主机名相关的设置：

```bash
$ hostnamectl status
```

{{< figure src="/images/post/Linux修改主机名/2.png" class="center" title="通过 hostnamectl 命令查看主机名相关配置" alt="通过 hostnamectl 命令查看主机名相关配置" >}}

只查看静态、瞬态或灵活主机名，分别使用 `--static`，`--transient` 或 `--pretty`。

```bash
$ hostnamectl status [--static|--transient|--pretty]
```

{{< figure src="/images/post/Linux修改主机名/3.png" class="center" title="通过 hostnamectl 命令查看主机名相关配置" alt="通过 hostnamectl 命令查看主机名相关配置" >}}

同时修改静态、瞬态和灵活主机名：

```bash
$ sudo hostnamectl set-hostname <host-name>
```

{{< figure src="/images/post/Linux修改主机名/4.png" class="center" title="通过 hostnamectl 命令修改主机名" alt="通过 hostnamectl 命令修改主机名" >}}

在修改静态/瞬态主机名时，任何特殊字符或空白字符会被移除，而提供的参数中的任何大写字母会自动转化为小写。一旦修改了静态主机名，`/etc/hostname` 将被自动更新。然而，`/etc/hosts` 不会更新以保存所做的修改，所以你需要手动更新 `/etc/hosts`。

如果你只想修改特定的主机名（静态，瞬态或灵活），你可以使用 `--static`，`--transient` 或 `--pretty` 选项。

```bash
$ sudo hostnamectl [--static|--transient|--pretty] set-hostname <host-name>
```

例如修改静态主机名：
{{< figure src="/images/post/Linux修改主机名/5.png" class="center" title="修改静态主机名" alt="修改静态主机名" >}}

注意，不必重启机器以激活永久主机名修改。上面的命令会立即修改内核主机名。注销并重新登入后在命令行提示就可以看到新的主机名。

## 通过 nmcli 工具修改
在 Centos 7 中包含 nmcli 工具，可被用来修改主机名。

```bash
$ nmcli g hostname
$ nmcli g hostname <host-name>
```

{{< figure src="/images/post/Linux修改主机名/6.png" class="center" title="通过 nmcli 工具修改主机名" alt="通过 nmcli 工具修改主机名" >}}

## 参考文章
https://help.aliyun.com/knowledge_detail/6705956.html  
http://www.centoscn.com/CentOS/config/2014/1031/4039.html
