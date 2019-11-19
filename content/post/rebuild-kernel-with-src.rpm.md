---
title: "Rebuild kernel package with src.rpm (Centos7)"
date: 2019-01-24T17:08:04+08:00
lastmod: 2019-01-24T17:08:04+08:00
draft: false
keywords: [Linux, Centos7, Kernel]
tags: [Linux, Centos7, Kernel]
categories: [Linux]
comment: true
toc: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

**注意，应使用非 root 用户进行 rpm 包构建。**
```bash
# add user
useradd -m -r koji
su koji
```

下载源码包，安装源码包，解压源码。
```bash
sudo yum install yum-utils rpm-build
# download source package
yumdownloader --source kernel
# install build dependencies
yum-builddep kernel
# install source package, 
rpm -i kernel-*.src.rpm
cd ~/rpmbuild
# unpacking the sources and applying any patches.
rpmbuild -bp SPECS/kernel.spec
```
<!--more-->

进入源码目录，复制一份源码，修改代码，生成 patch 放入 SOURCE 中。
```bash
cd BUILD/kernel-3.10.0-957.1.3.el7/
cp -a BUILD/kernel-3.10.0-957.1.3.el7 BUILD/kernel-3.10.0-957.1.3.el7.orig
# fix bug
diff -uprN linux-3.10.0-957.1.3.el7.x86_64.orig linux-3.10.0-957.1.3.el7.x86_64 > ../../SOURCES/remove-rmrr-check.patch
```
将 patch 头修改为如下格式：
```diff
--- a/drivers/iommu/intel-iommu.c       2019-01-22 18:24:42.622374819 +0800
+++ b/drivers/iommu/intel-iommu.c       2019-01-22 18:16:46.131348561 +0800
@@ -4834,7 +4834,7 @@
```

修改 `SPECS/kernel.spec`，应用 patch，更新版本号。
```diff
--- kernel.spec.orig	2019-01-24 10:46:53.448383625 +0800
+++ kernel.spec	2019-01-24 10:49:52.094393470 +0800
@@ -5,7 +5,7 @@ Summary: The Linux kernel
 
 %define dist .el7
 
-# % define buildid .local
+%define buildid .c1
 
 # For a kernel released for public testing, released_kernel should be 1.
 # For internal testing builds during development, it should be 0.
@@ -449,6 +449,7 @@ Patch999999: linux-kernel-test.patch
 Patch1000: debrand-single-cpu.patch
 Patch1001: debrand-rh_taint.patch
 Patch1002: debrand-rh-i686-cpu.patch
+Patch2000: remove-rmrr-check.patch
 
 BuildRoot: %{_tmppath}/kernel-%{KVRA}-root
 
@@ -781,6 +782,7 @@ ApplyOptionalPatch linux-kernel-test.pat
 ApplyOptionalPatch debrand-single-cpu.patch
 ApplyOptionalPatch debrand-rh_taint.patch
 ApplyOptionalPatch debrand-rh-i686-cpu.patch
+ApplyOptionalPatch remove-rmrr-check.patch
 
 # Any further pre-build tree manipulations happen here.
 
@@ -1757,6 +1759,9 @@ fi
 %kernel_variant_files %{with_kdump} kdump
 
 %changelog
+* Tue Jan 22 2019 CentOS Sources <bugs@centos.org> - 3.10.0-957.1.3.el7.c1
+- Remove RMRR check (caicloud)
+
 * Mon Nov 26 2018 CentOS Sources <bugs@centos.org> - 3.10.0-957.1.3.el7
 - Apply debranding changes
 - Sign with new secureboot key
```

构建源码包和二进制包：
```bash
rpmbuild -ba SPECS/kernel.spec
```