---
title: "fedora23 GRUB2 更换主题"
date: 2016-01-12T10:16:42+08:00
lastmod: 2016-01-12T10:16:42+08:00
draft: false
keywords: [Linux, GRUB2, fedora]
tags: [Linux, GRUB2, fedora]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

>原文在[CSDN](https://blog.csdn.net/Meoop/article/details/50505609)，现在迁移到这里。

刚刚装了 win10 和 fedora23 双系统，启动时黑框只占屏幕左上角的 1/4，感觉十分的难看，于是就寻思这怎么让其好看点，于是翻了翻资料，看见很多地方都可以换 GRUB2 的主题，就试着动了下手，美化了下。

**效果图**（主题作者给的效果图），下载地址：https://pan.baidu.com/s/1c2mgBCG
{{< figure src="/images/post/fedora23-GRUB2更换主题/20160112180610905.png" class="center" title="主题样式" >}}

<!--more-->

## 修改方法

### 下载主题
将下载下的主题放到 `/boot/grub2/theme` 文件夹中。这个主题文件夹里应该有个 theme.txt 的文件。 

### 更该 grub 配置
更改 `/etc/default/grub`

```bash
sudo gedit /etc/default/grub
```

在文件中添加下面内容：

```bash
GRUB_THEME="theme.txt所在的路径/theme.txt"
```

并且注释掉 `GRUB_TERMINAL_OUTPUT="console"`（这个控制是以哪种方式显示，不注释的话会以 console 窗口显示启动列表）

```bash
#GRUB_TERMINAL_OUTPUT="console"
```

### 更新 grub2
如果是 `gpt` 分区，以 `uefi` 启动方式启动的话，启动时读取的文件是 `/boot/efi/EFI/fedora/grub.cfg` ，这个时候更新时需更新这个文件，命令如下：

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg 
```

如果是一般的启动方式，启动时读取的文件是 `/boot/grub2/grub.cfg`，这个时候更新时需更新这个文件，命令如下：

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

好了，现在启动下看看，就可以看看美美的启动界面了。
