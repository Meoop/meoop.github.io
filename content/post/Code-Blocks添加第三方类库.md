---
title: "Code Blocks 添加第三方类库"
date: 2015-04-13T10:24:24+08:00
lastmod: 2015-04-13T10:24:24+08:00
draft: false
keywords: [Code::Blocks]
description: ""
tags: [Code::Blocks]
categories: [类库]
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

第一步：编译第三方库，得到头文件和库，例如路径关系：
在 include 中放头文件，在 lib 中放置库文件。

> D:\MyLib\include 
> 
> D:\MyLib\lib

![编译后的第三方库](/images/post/Code-Blocks添加第三方类库/20150413230733748.png)

<!--more-->
第二步：创建全局变量，菜单：Settings > Global variables, New 一个新的，选名字，例如 MYLIB

> base: D:\MyLib
> 
> lib: $(base)\lib
> 
> include: $(base)\include

{{< figure src="/images/post/Code-Blocks添加第三方类库/20150413225928478.png" class="center" >}}

第三步：设置工程搜索路径，菜单：Project > build options > Search directories

>  Compiler: $(#MYLIB.include)
> 
>  Linker: $(#MYLIB.lib)

这样第三方库即可使用。

如果不想添加全局变量，则可以直接进行第三步，使用绝对路径，来分别设置头文件和库文件的查找路径（我的配置完后使用时还出现个问题，编译报错显示 `undefined reference to 'dgesv_'`，解决办法是在 Project > build options >Linker settings 中添加库文件，比较懒就全扔进去了~_~）。

![](/images/post/Code-Blocks添加第三方类库/20150413233612827.png)
