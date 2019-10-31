---
title: "使用 debootstrap 建立 Ubuntu 系统"
date: 2017-03-27T10:15:58+08:00
lastmod: 2017-03-27T10:15:58+08:00
draft: false
keywords: [Linux,Ubuntu,Debian]
description: ""
tags: [Linux,Ubuntu,Debian]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

摘要：debootstrap 能够快速建立一套 Debian 或 Ubuntu 的 rootfs，并且支持多种 CPU 架构。
<!--more-->

## debootstrap 介绍
debootstrap 是基于 Debian 以及其衍生系统（包括 Ubuntu）下的一个工具，用于构建一个基本系统。我们可以将 debootstrap 看作是一个特殊的安装工具，其只需要软件源的支持，不需要使用 dpkg 或者 apt，因此可以在任意 Linux 发行版上运行 debootstarp 构建基本系统。

debootstarp 实际上是一系列的脚本，它的依赖非常少，只是强制依赖了 wget。debootstrap 的工作原理实际上是将包解压到指定目录，然后可以 chroot 进入这个系统，进行相应的定制。另外 debootstarp 也可以构建其他架构的系统，例如在 AMD64 上构建 ARM64 的基本系统。debootstrap 有个大抵相当的工具 cdebootstrap，使用 `c` 语言写的，比debootstarp 体积更加小。

## 安装 debootstarp
debootstrap 在任何 Debian 系统及其衍生版本的 Linux 中都可以直接安装得到，但要注意如果要新版本的系统，需要下载比较新的 debootstrap 包。

* Debian / Ubuntu
    ```
    $ sudo apt-get install debootstrap
    ```

## debootstarp 命令的使用
一般使用 debootstrap 命令的格式如下：

```bash
$ sudo debootstrap --arch=<ARCH> --include=<PACKAGES> <VERSION> <DIRECTORY> <MIRROR>
```

* ARCH：设置目标系统的 CPU 架构，常用的有 i386、amd64、armhf、arm64 等。
* PACKAGES：需要额外安装的包包名，如果多个包名需要用逗号隔开。
* VERSION：需要安装的系统的版本代号，例如 Debian的sid、stretch、jessie、wheezy，或者 Ubuntu 的 xenial、vivid 等。
    + Debian 的发行版代号：<https://en.wikipedia.org/wiki/Debian_version_history>
    + Ubuntu 的发行版代号：<https://en.wikipedia.org/wiki/Ubuntu_version_history>
* DIRECTORY：安装到的目录，根据自己的需求设定即可。
* MIRROR：软件源，例如 Ubuntu 源：<http://archive.ubuntu.com/ubuntu>。

例如，利用 debootstrap 制作一个 Ubuntu 16.04 的 arm64 基础系统，并且包含 vim 包：

```bash
$ sudo debootstrap --arch=arm64 --include=vim xenial ./xenial-arm-rootfs http://ports.ubuntu.com/ubuntu-ports
```

更多使用方式参考 [debootstrap man page](http://manpages.ubuntu.com/manpages/xenial/man8/debootstrap.8.html#contenttoc2)。

_注意：debootstrap 仅能从一个软件仓库获取软件包，假如你需要从多个软件仓库安装或合并软件包用以建立 rootfs，或者你希望自动定制 rootfs，那么可以使用 multistrap。_


##  Additional Information
使用 debootstrap 构建基本系统一般是我们工作的第一步，后面我们可以 chroot 到这个基本系统中，安装一些包，进行一些系统配置。
使用 debootstrap 安装一个完整的系统可以参考[这篇文章](https://help.ubuntu.com/community/Installation/FromLinux)。
使用 debootstrap 制作 docker 镜像可以参考 [docker 文档](https://docs.docker.com/engine/userguide/eng-image/baseimages/#create-a-full-image-using-tar)。

## 参考文章：
[1]. [Debian 上 Debootstrap 的 wiki 文档](https://wiki.debian.org/zh_CN/Debootstrap);  
[2]. [Ubuntu 系统 debootstrap 的使用](http://www.latelee.org/using-gnu-linux/ubuntu-debootstrap.html);  
[3]. [使用 debootstrap 建立完整的 Debian 系統](https://github.com/KingBing/blog-src/blob/master/%E4%BD%BF%E7%94%A8%20debootstrap%20%E5%BB%BA%E7%AB%8B%E5%AE%8C%E6%95%B4%E7%9A%84%20Debian%20%E7%B3%BB%E7%B5%B1.org);

