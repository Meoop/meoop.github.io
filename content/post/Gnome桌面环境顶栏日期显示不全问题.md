---
title: "Gnome 桌面环境顶栏日期显示不全问题"
date: 2016-07-23T10:16:50+08:00
lastmod: 2016-08-01T10:16:50+08:00
draft: false
keywords: [Linux,GNOME]
tags: [Linux,GNOME]
categories: [Linux]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

>2016.08.01 日更新
>该问题不止在 Fedora 中出现，在 ubuntu GNOME 桌面版版也出现这个问题，是 GNOME 桌面环境的问题。修改方法与 Fedora 相同。

Fedora 23，Fedora 24，ububtu GNOME 桌面版顶栏中文日期及星期显示不全的问题，Fedora 23 时找到修改的办法，未做记录，更新到 Fedora 24 时又遇到这个问题，于是又翻找资料解决问题，为避免下次麻烦，将该问题以及问题解决办法记录下来，以备后用。

<!--more-->

## 问题描述
顶栏中文日期显示 `XX月XX`，后面 `日` 字不显示，以及星期几没有 `星期`。
{{< figure src="/images/post/Gnome桌面环境顶栏日期显示不全问题/1.png" class="center" title="问题描述" alt="问题描述" >}}

## 解决办法
1、在 `/usr/share/locale/zh_CN/LC_MESSAGES` 文件夹有 `gnome-desktop-3.0.mo` 的文件，先将 `gnome-desktop-3.0.mo` 进行备份，然后用命令 `msgunfmt` 将其还原成 `.po` 文件。

```bash
sudo msgunfmt gnome-desktop-3.0.mo -o gnome-desktop-3.0.po
```

2、编辑 `gnome-desktop-3.0.po` 文件，在文件中 `msgstr` 部分的 `%e` 后面加个 `日` 就可以显示出正常的日期（XX月XX日）。

3、把 `%a` 改成 `%A` 就可以将星期正常显示。

4、将改好的 `.po` 文件用 `msgfmt` 变成 `.mo` 文件，覆盖 `gnome-desktop-3.0.mo` 文件。

```bash
sudo msgunfmt gnome-desktop-3.0.po -o gnome-desktop-3.0.mo
```

5、输入 `alt+f2`，键入命令 `r `并回车，重启 `gnome` 即可。
{{< figure src="/images/post/Gnome桌面环境顶栏日期显示不全问题/2.png" class="center" title="显示正常" alt="显示正常" >}}

## 修改后的文件部分内容

```
msgid "%R:%S"
msgstr "%R:%S"

msgid "%a %R"
msgstr "%A %R"

msgid "%a %R:%S"
msgstr "%A %R:%S"

msgid "%a %b %e, %R"
msgstr "%b %e日 %A, %R"

msgid "%a %b %e, %R:%S"
msgstr "%b %e日 %A, %R:%S"

msgid "%a %b %e, %l:%M %p"
msgstr "%b %e日 %A, %p %l:%M"

msgid "%a %b %e, %l:%M:%S %p"
msgstr "%b %e日 %A, %p %l:%M:%S"

msgid "%a %l:%M %p"
msgstr "%A %p %l:%M"

msgid "%a %l:%M:%S %p"
msgstr "%A %p %l:%M:%S"

msgid "%l:%M %p"
msgstr "%p %l:%M"

msgid "%l:%M:%S %p"
msgstr "%p %l:%M:%S"
```
