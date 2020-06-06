---
title: "Upgrade Glusterfs"
date: 2019-07-24T20:40:12+08:00
lastmod: 2019-07-24T20:40:12+08:00
draft: false
keywords: [glusterfs]
tags: [glusterfs]
categories: [glusterfs]
comment: true
toc: true
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

支持 3.10.x，3.12.x，4.1.x，5.x 升级到 6.x 版本。

## 升级前注意事项
* 只有复制卷和分布式复制卷可以联机升级
* 分布式卷和条纹卷不支持联机升级
* 升级过程中不得修改任何配置
* 如果使用 geo-replication ，先升级 slave 节点，再升级 master 节点
* 建议先升级服务端再升级客户端
* 建议使用相同版本的客户端和服务端
* 升级前建议阅读版本的 release note，熟悉新版本删除和添加的功能
<!--more-->

## 服务端升级
如果对卷进行单独设置过参数，建议阅读新版本 release note，查看功能删减，如果功能被删除，建议将参数设置为默认值，防止升级失败。

检查额外的配置参数，单独设置过的参数可以在 info 信息的 `Options Reconfigured` 部分看到：
```bash
gluster volume info
```

重置配置可以通过下面方式：
```bash
gluster volume reset <volname> <option>
```

### 在线升级
在线升级每次升级一台电脑，在升级过程中，卷保持 Online 状态，不影响客户端 IO 读写。在线升级假设副本集分散在不同的主机上，若副本都在同一台主机，则无法在线升级。

依次在每个节点需要执行下面的升级过程：

1、关闭所有 gluster 进程，包括正在使用 gfapi 的应用，例如：qemu，NFS-Ganesha，Samba 等。
```bash
# 停止 glusterd 服务
systemctl stop glusterd.service
# 杀死所有 gluster 相关进程，例如 fuse 客户端，brick 进程等
killall glusterfs glusterfsd glusterd
```

2、升级 gluster，下面以升级到 gluster6 为例
```bash
yum install centos-release-gluster6
yum upgrade 
```

3、确保 gluster 版本都升级到指定版本
```bash
rpm -qa |grep gluster
```

4、重新启动 glusterd 进程，brick 进程，确保所有 brick 在线
```bash
# 重新启动 glusterd 服务
systemctl start glusterd
systemctl status glusterd
# 查看 brick 状态
gluster volume status
```

5、对卷进行修复
```bash
# 修复卷
for i in `gluster volume list`; do gluster volume heal $i; done
# 查看修复状态
for i in `gluster volume list`; do gluster volume heal $i info; done
```

6、确认卷修复都完成后，重启 1 中关闭的所有 gfapi 程序，然后对下一个节点进行升级

### 离线升级
离线升级集群停止服务，在升级过程在，不允许任何客户端访问卷。

在 gluster 所有节点执行下面操作：

1、关闭所有 gluster 进程，包括正在使用 gfapi 的应用，例如：qemu，NFS-Ganesha，Samba 等。
```bash
# 停止 glusterd 服务
systemctl stop glusterd.service
# 杀死所有 gluster 相关进程，例如 fuse 客户端，brick 进程等
killall glusterfs glusterfsd glusterd
```

2、升级 gluster，下面以升级到 gluster6 为例
```bash
yum install centos-release-gluster6
yum upgrade 
```

3、确保 gluster 版本都升级到指定版本
```bash
rpm -qa |grep gluster
```

4、重新启动 glusterd 进程，brick 进程，确保所有 brick 在线
```bash
# 重新启动 glusterd 服务
systemctl start glusterd
systemctl status glusterd
# 查看 brick 状态
gluster volume status
```

5、对卷进行修复
```bash
# 修复卷
for i in `gluster volume list`; do gluster volume heal $i; done
# 查看修复状态
for i in `gluster volume list`; do gluster volume heal $i info; done
```

### 服务端升级后操作

#### 升级 op-version 
op-version 是正在运行的 Gluster 的操作版本。引入 op-version 的目的是为了确保不同版本的 gluster 运行不会出现问题，解决 gluster 向后兼容性问题。Gluster 升级后，建议更新 op-version。

获取当前使用的 op-version：
```bash
gluster volume get all cluster.op-version
```

获取当前可以使用的最高版本：
```bash
gluster volume get all cluster.max-op-version
```

例如升级到 gluster6 后，要使用 gluster6 的新特性，可以更新到最新 op-version ：
```bash
gluster volume set all cluster.op-version 60000
```

## 客户端升级

1. 卸载客户端上的所有 glusterfs 挂载点
2. 停止通过 gfapi（qemu等）访问卷的所有应用程序
3. 安装 Gluster 6.x
4. 重新挂载所有 gluster 卷
5. 启动之前在步骤 2 中停止的所有应用程序


## Troubleshooting
Glusterd 启动后，brick 未启动
1. 查看 glusterd 日志，看是否有相关错误日志，查找失败原因；
2. 如果曾经有修改过 glusterd.vol 文件，升级后需要更新 glusterd.vol 文件，部分配置不存在会导致 brick 启动失败；


## 参考文档
1. https://docs.gluster.org/en/latest/Upgrade-Guide/
2. https://docs.gluster.org/en/latest/Upgrade-Guide/op_version/

