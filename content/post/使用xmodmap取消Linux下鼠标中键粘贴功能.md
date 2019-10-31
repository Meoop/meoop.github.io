---
title: "使用 xmodmap 取消 Linux 下鼠标中键粘贴功能"
date: 2018-03-07T10:16:23+08:00
lastmod: 2018-03-07T10:16:23+08:00
draft: false
keywords: [Linux, xmodmap]
description: ""
tags: [Linux, xmodmap]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: false
---

在 Linux 下选中东西后，按住鼠标的滚轮中键进行粘贴，是 Linux 下非常好用的功能。但是不巧，鼠标的滚轮中键太灵敏，在滚动滚轮时候非常容易按下中键，这时候编辑好的文档突然就插入了不知道从哪复制来的东西。

xmodmap 是用于修改X中键盘映射和指针按钮映射的实用程序，我们可以用 xmodmap 修改鼠标中键的映射，进而关掉鼠标中键的复制功能，解决误触鼠标中键问题。
<!--more-->

**方法一：**在终端下执行 xmodmap 命令进行修改

```shell
$ xmodmap -e "pointer = 1 25 3 4 5 6 7 8 9 10"
```

**方法二：**编辑~/.Xmodmap文件进行修改

```shell
$ echo "pointer = 1 25 3 4 5 6 7 8 9 10" > ~/.Xmodmap
```


**参考文章：**  
[1]. [xmodmap - utility for modifying keymaps and pointer button mappings in X](https://www.x.org/archive/X11R6.8.1/doc/xmodmap.1.html)  
[2]. [How do I disable middle mouse button click paste?](https://askubuntu.com/questions/4507/how-do-i-disable-middle-mouse-button-click-paste)
