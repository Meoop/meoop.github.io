---
title: "在 Kubernetes 上使用 KubeVirt 管理虚拟机负载"
date: 2019-11-04T21:29:18+08:00
lastmod: 2019-11-21T21:29:18+08:00
draft: false
keywords: [kubernetes, kubevirt]
tags: [kubernetes, kubevirt]
categories: [kubernetes, kubevirt]
comment: true
toc: true
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

近几年的时间里，kubernetes 不断发展壮大，各个功能也逐渐完善，越来越多的项目开始基于 kubernetes 构建，企业内部也大多搭建了自己的 kubernetes 平台。同时企业也存在大量的旧的项目，大多都是基于虚拟机构建，迁移到 kubernetes 存在着较高的成本，风险也比较大。随着 kubernetes 的功能完善，很多新业务已经上了 kubernetes，企业内部需要同时维护着 k8s 和虚拟机两套平台，同时维护两套平台给运维带来了巨大的负担，因此急切的需求一种方案解决该问题。在 kubernetes 可以通过 KubeVirt 管理虚拟机，正好用于解决上述管理难题。

<!--more-->
## KubeVirt 架构设计
kubevirt 是 Redhat 开源的以容器方式运行虚拟机的项目，利用 kubernetes CRD 机制增加了新的资源类型 `VirtualMachineInstance（VMI）`， VMI 是对虚拟机实例的抽象，一个 VMI 对象在其生命周期内始终与某个 Pod 关联。 CRD 的方式使得 kubevirt 对虚拟机的管理不再局限于 Pod 管理接口，但是也无法使用现有的 Pod 的 ReplicaSet，StatefulSet，Deployment 等的管理能力，也意味着 kubevirt 如果想要这些管理能力，就要自己去实现，目前 kubevirt 实现了类似 RS 的功能。

Kubevirt 主要实现了下面几种资源，实现了对虚拟机的管理：
* `VirtualMachineInstance（VMI）`：类似于 kubernetes Pod，是虚拟机管理的最小资源。一个 `VirtualMachineInstance` 对象即表示一台正在运行的虚拟机实例，包含一个虚拟机所需要的各种配置。
* `VirtualMachine`：为群集内的 `VirtualMachineInstance` 提供管理功能，例如开机/关机/重启虚拟机，确保虚拟机实例的启动状态，与虚拟机实例是 1:1 的关系，类似与 spec.replica 为 1 的 StatefulSet。
* `VirtualMachineInstanceReplicaSet`：类似 `ReplicaSet`，可以启动指定数量的 `VirtualMachineInstance`，并且保证指定数量的 `VirtualMachineInstance` 运行，可以配置 HPA。
* `DataVolume`： 在 CDI 中实现，`DataVolume` 是对 PVC 之上的抽象，用于自动创建和填充带有数据的 PVC，提供了在虚拟机启动流程中自动将虚拟机磁盘导入 PVC 的方法。

{{< figure src="/images/post/Use-KubeVirt-to-manage-virtualization-workloads-on-Kubernetes/kubevirt-components.png" class="center" title="KubeVirt Architecture" alt="" >}}

上图描述了 KubeVirt 的整体架构，其中包含了主要的四个组件：
* `virt-api-server`，HTTP API 服务器，是所有虚拟化相关流程的入口点。
* `virt-controller`，管理和监控 VMI 对象及其关联的 Pod，对其状态进行更新。
* `virt-hander`，以 DaemonSet 运行在每一个节点上，监听 VMI 的状态向上汇报，管理 VMI 的生命周期。
* `virt-launcher`，以 Pod 方式运行，每个 VMI Object 都会对应一个 virt-launcher Pod，容器内有单独的 libvirtd，用于启动和管理虚拟机。

## 虚拟机实例
### VMI 创建流程
```
Client                     K8s API     VMI CRD  Virt Controller         VMI Handler
-------------------------- ----------- ------- ----------------------- ----------

                           listen <----------- WATCH /virtualmachines
                           listen <----------------------------------- WATCH /virtualmachines
                                                  |                       |
POST /virtualmachines ---> validate               |                       |
                           create ---> VMI ---> observe --------------> observe
                             |          |         v                       v
                           validate <--------- POST /pods              defineVMI
                           create       |         |                       |
                             |          |         |                       |
                           schedPod ---------> observe                    |
                             |          |         v                       |
                           validate <--------- PUT /virtualmachines       |
                           update ---> VMI ---------------------------> observe
                             |          |         |                    launchVMI
                             |          |         |                       |
                             :          :         :                       :
                             |          |         |                       |
DELETE /virtualmachines -> validate     |         |                       |
                           delete ----> * ---------------------------> observe
                             |                    |                    shutdownVMI
                             |                    |                       |
                             :                    :                       :
```

上面的通信图大致描述了一个 VMI 通信流程，要启动一个 VMI，其大致步骤为：
1. client 发送 VMI Spec 给 k8s api server
2. k8s api server 验证 VMI Spec 并且创建自定义资源
3. virt-controller watch 到新 VMI Object 的创建，创建对应的 Pod
4. kubernetes 调度 Pod
5. virt-controller 监测到 Pod 的启动，更新 VMI Obeject 的 nodeName，然后对应节点上的 virt-handler 做进一步的处理
6. virt-handler 将 VMI Object 发送给 Pod 中的 virt-launcher，通知其启动对应的 VM

### 磁盘和卷
虚拟机镜像（磁盘）是启动虚拟机必不可少的部分，KubeVirt 中提供多种方式的虚拟机磁盘，虚拟机镜像（磁盘）使用方式非常灵活。

* `cloudInitNoCloud/cloudInitConfigDrive`，用于提供 cloud-init 初始化所需要的 user-data，可以使用 k8s secret/configmap 作为数据源。
* `PersistentVolumeClaim`，使用 PVC 做为后端存储，适用于数据持久化，即在虚拟机重启或者重建后数据依旧存在。使用的 PV 类型可以是 block 和 filesystem，使用 filesystem 时，会使用 PVC 上的 /disk.img，格式为 RAW 格式的文件作为硬盘。block 模式时，使用 block volume 直接作为原始块设备提供给虚拟机。
* `ephemeral`，基于后端存储在本地做一个写时复制（COW）镜像层，所有的写入都在本地存储的镜像中，VM 实例停止时写入层就被删除，后端存储上的镜像不变化。
* `containerDisk`，基于 scratch 构建的一个 docker image，镜像中包含虚拟机启动所需要的虚拟机镜像，可以将该 docker image push 到 registry，使用时从 registry 拉取镜像，直接使用 containerDisk 作为 VMI 磁盘，数据是无法持久化的。
* `hostDisk`，使用节点上磁盘镜像，类似于 hostpath，也可以在初始化时创建空的镜像。
* `dataVolume`，提供在虚拟机启动流程中自动将虚拟机磁盘导入 pvc 的功能，在不使用 DataVolume 的情况下，用户必须先准备带有磁盘映像的 pvc，然后再将其分配给 VM 或 VMI。dataVolume 拉取镜像的来源可以时 http，对象存储，另一块 PVC 等。

### 网络
KubeVirt 是建立在 kubernetes 之上的，提供了容器和虚拟机的混部方案，其网络应该与 kubernetes 集成的，与 Pod 网络是互通的，KubeVirt 的 VMI 使用的应当是 Pod 的网络。为了实现该目标，KubeVirt 的对网络做了特殊实现。虚拟机具体的网络如下图所示， virt-launcher Pod 网络的网卡不再挂有 Pod IP，而是作为虚拟机的虚拟网卡的与外部网络通信的交接物理网卡。

{{< figure src="/images/post/Use-KubeVirt-to-manage-virtualization-workloads-on-Kubernetes/vm-networking.png" class="center" title="" alt="" >}}

虚拟机网络的启动需要以下几个步骤：
1. 通过 CNI 插件配置 libvirt 所在的 Pod 的网络
2. virt-launcher 记录 CNI 插件配置的接口信息（IP，routes，gateway，MAC），用于后续 DHCP 的配置
3. 将 Pod 的 eth0 网卡 ip 删除，设置接口状态为 down，修改 eth0 的 MAC 地址后重新启动 eth0
4. 创建后端为 bridge 的 libvirt 网络:
   + eth0 直接加入 bridge
   + 虚拟机网卡对应的 tap 口 也加入 bridge
   + bridge 配置 xx.xx.xx.xx 作为 DHCP server IP
   + virt-laucher ，启用 DHCP 功能，设置虚拟机网卡 MAC 地址和上面记录的 IP 地址的映射，使得虚拟机在启动时通过 DHCP 获取到 IP 地址
5. 虚拟机需要启用 dhclient，以自动获取 IP 地址

## In Kubernetes
### Virtual Machine
KubeVirt 实现了 `VirtualMachine` 类型资源，与 VMI 的是一对一的关系，类似于 replica 为 1 的 StatefulSet。VirtualMachine 相对于 VMI 提供了更多的管理功能，其中包括：
* ABI 稳定性
* 在控制层面提供开机/关机/重启功能
* 离线更新 VMI 配置功能
* 确保 VMI 在应该运行的时候运行

当需要上述 `VirtualMachine` 扩展的功能时，应当使用 `VirtualMachine` 类型管理 VMI，例如下面两种应用场景：
* 绑定硬件 licenses 的应用，虚拟机重启后需要保证硬件信息不变动
* 需要更新虚拟机的硬件资源

下面是一个 `VirtualMachine` 的例子：

```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-cirros
  name: vm-cirros
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-cirros
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
        machine:
          type: ""
        resources:
          requests:
            memory: 64M
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: kubevirt/cirros-container-disk-demo:latest
      - cloudInitNoCloud:
          userDataBase64: IyEvYmluL3NoCgplY2hvICdwcmludGVkIGZyb20gY2xvdWQtaW5pdCB1c2VyZGF0YScK
        name: cloudinitdisk
```

### Virtual Machine Replica Set
KubeVirt 实现了 `VirtualMachineInstanceReplicaSet` 控制器，类似于 kubernetes ReplicaSet，可以保证集群中指定数量的 VMI 运行。为保证多副本的一致性，使用多副本方式运行的虚拟机内部不应该有持久化的数据，可以使用只读文件系统，数据保存在 tmpfs 中或者使用相同的后端存储。

下面例子创建了一个 3 副本的 VMI，使用的 `ContainerDisks` 这种类型的临时存储：

```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceReplicaSet
metadata:
  name: testreplicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      myvmi: myvmi
  template:
    metadata:
      name: test
      labels:
        myvmi: myvmi
    spec:
      domain:
        devices:
          disks:
          - disk:
            name: containerdisk
        resources:
          requests:
            memory: 64M
      volumes:
      - name: containerdisk
        containerDisk:
          image: kubevirt/cirros-container-disk-demo:latest

VirtualMachineInstanceReplicaSet 同样支持 Service，与 VMI 的 Service 创建方式一样。此外，VirtualMachineInstanceReplicaSet 也支持 HPA，例如：
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: myhpa
spec:
  scaleTargetRef:
    kind: VirtualMachineInstanceReplicaSet
    name: vmi-replicaset-cirros
    apiVersion: kubevirt.io/v1alpha3
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

### Node Placement
KubeVirt 的虚拟机是建立在 Pod 内的，这个 Pod 是被 kubernetes 调度的，也就是可以直接使用 kubernetes 的机制进行节点选择，VMI 支持下面三种设置方式，设置的内容都会透传到 VMI 所在的 Pod 上：
* nodeSelector
* affinity and anti-affinity
* taints and tolerations

例如，设置 `spec.nodeSelector` 来指定运行的节点：
```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  name: testvmi-ephemeral
spec:
  nodeSelector:
    cpu: slow
    storage: fast
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

### Service
在 kubernetes 中，一般通过 service 访问 Pod 提供的服务，同样，VMI 启动后，需要连接到虚拟机，也可以通过 service 进行访问。KubeVirt 的虚拟机是建立在 Pod 内的，可以直接使用 kubernetes 原生地 service 暴露服务。 在创建 VirtualMachineInstance Object 时，可以指定 metadata.label，配置会透传到 VMI 所在的 Pod 上的，创建服务只需要指定这个特殊的 label 即可。当前 VMI 支持 `ClusterIP`，`NodePort` 和 `LoadBalancer` 三种类型的服务。

例如，下面配置可以创建 `ClusterIP` 类型的服务来暴露虚拟机的 22（ssh）端口，VMI Object需要指定 `testvmi-special: test` 标签。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: testvmiservice
spec:
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    testvmi-special: test
  type: ClusterIP
```

## 部署及管理
### KubeVirt 的部署
KubeVirt 部署在 k8s 之上，这里假设已经存在处于 Ready 状态的 k8s 集群（部署好 CNI 插件），再此基础上部署 KubeVirt。

1、部署最新版本的 KubeVirt
```bash
# The last version of KubeVirt
export KUBEVIRT_VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt/releases/latest | jq -r .tag_name)

# kubevirt
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config
  namespace: kubevirt
  labels:
    kubevirt.io: ""
data:
  feature-gates: "DataVolumes"
EOF
```

2、Containerized Data Importer（CDI）项目提供了用于使 PVC 作为 KubeVirt VM 磁盘的功能。建议同时部署 CDI 
```bash
# cdi
export CDI_VERSION=$(curl -s https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -o "v[0-9]\.[0-9]*\.[0-9]*")
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-cr.yaml
```

### 虚拟机镜像准备
#### 手动上传到 PVC
KubeVirt 可以使用 PVC 作为后端磁盘，使用 `filesystem` 类型的 PVC 时，默认使用的时 `/disk.img` 这个镜像，用户可以将镜像上传到 PVC，在创建 VMI 时使用此 PVC。使用这种方式需要注意下面几点：
* 一个 PVC 只允许存在一个镜像，只允许一个 VMI 使用，要创建多个 VMI，需要上传多次
* `/disk.img` 的格式必须是 RAW 格式

#### 通过 CDI 准备虚拟机镜像
CDI 提供了使用使用 PVC 作为虚拟机磁盘的方案，在虚拟机启动前通过下面方式填充 PVC：
* 通过 URL 倒入虚拟机镜像到 PVC，URL 可以是 http 链接，s3 链接
* Clone 一个已经存在的 PVC
* 通过 container registry 倒入虚拟机磁盘到 PVC，需要结合 ContainerDisk 使用
* 通过客户端上传本地镜像到 PVC

KubeVirt 提供了一个命令行 virtctl，结合 CDI 项目，可以上传本地镜像到 PVC 上，支持的镜像格式有：
* .img
* .qcow2
* .iso
* 压缩为 .tar，.gz，.xz 格式的上述镜像

例如，上传本地 cirros 镜像到 cirros-vm-disk 中，指定的 PVC 会自动创建：
```bash
./virtctl image-upload                          \
    --pvc-name=cirros-vm-disk                   \
    --pvc-size=500Mi                            \
    --storage-class=heketi-storageclass         \
    --image-path=cirros-0.4.0-x86_64-disk.img   \
    --uploadproxy-url=https://cdi-uploadproxy:31001
```
#### ContainerDisk
KubeVirt 可以使用 `ContainerDisk` 类型的磁盘，`ContainerDisk` 提供了一种以 registry 存储和分发虚拟机镜像的方案，可以使用这种方式制作上传镜像。

创建虚拟机镜像，dockerfile 如下：
```Dockerfile
FROM scratch
ADD fedora28.qcow2 /disk/
```

使用 docker 命令构建容器镜像，上传容器镜像到 registry：
```bash
docker build -t test.cargo.io/vmidisks/fedora28:latest .
docker push test.cargo.io/vmidisks/fedora28:latest 
```

使用 `containerDisk` 创建 VMI：
```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  name: testvmi-containerdisk
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: containerdisk
        disk: {}
  volumes:
    - name: containerdisk
      containerDisk:
        image: test.cargo.io/vmidisks/fedora28:latest
```

`containerDisk` 作为 VMI 磁盘是一种临时存储，此外可以结合 CDI，自动将 `containerDisk` 中的镜像倒入到 PVC 中，作为持久存储，例如，创建源为 `containerDisk` 的 `DataVolume`：
```yaml
apiVersion: cdi.kubevirt.io/v1alpha1
kind: DataVolume
metadata:
  name: registry-image-datavolume
spec:
  source:
    registry:
      url: "docker://test.cargo.io/vmidisks/fedora28:latest"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
```

VMI 使用 DataVolume：
```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-alpine-datavolume
  name: vmi-alpine-datavolume
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: registry-image-datavolume
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: registry-image-datavolume
    dataVolume:
      name: registry-image-datavolume
```

### 连接到虚拟机
连接到虚拟机内部可以使用下面方式：

1、使用 virtctl console 命令通过串口连接到虚拟机，虚拟机要允许串口登陆
```bash
./virtctl console vm-cirros
```

2、虚拟机开启 ssh 服务，通过 service 暴露 ssh 端口


## Use Cases
### Multiple Workloads
借助 KubeVirt ，旧的无法迁移的应用与新的容器化应用同时跑在 kubernetes 上，开发 VM 也可以建立在 kubernetes 上，减少同时维护 VM 和 kubernetes 两个平台的成本。同时 VM 生命周期和 Pod 生命周期一致，VM 同容器化应用一样编排，借助 kubernetes 灵活的网络，使用相同的工具（kubectl），可以更加高效的部署，高效测试。

{{< figure src="/images/post/Use-KubeVirt-to-manage-virtualization-workloads-on-Kubernetes/multiple-workloads.png" class="center" title="Multiple Workloads" alt="" >}}

### Kubernetes on Kubernetes
当用户用管理多台服务器时，一般会部署 openstack 作私有云，新的 kubernetes 平台会部署在 openstack 集群上，openstack 十分复杂，管理 openstack 需要非常高技能要求。KubeVirt 提供了一个新的方案，Kubernetes + KubeVirt  部署在服务器上，作为基础设施，借助 KubeVirtcloud-provider 快速构建 kubernetes 集群。使用这种方式有下面优点：
* 架构简单，平台统一，所有的全部运行在 kubernetes 上
* 更少的技能需求，all k8s
* 更少的资源占用

{{< figure src="/images/post/Use-KubeVirt-to-manage-virtualization-workloads-on-Kubernetes/k8s-in-k8s.png" class="center" title="Kubernetes on Kubernetes" alt="" >}}

目前使用这种方式的多为多云厂商，例如：
* [kubermatic](https://www.kubermatic.com/)
* [platform9](https://platform9.com/)
* [SAP Gardener](https://github.com/gardener/gardener)

## 参考
1. https://kubevirt.io/user-guide/docs/latest/welcome/index.html
2. https://github.com/kubevirt/kubevirt/tree/master/docs
3. https://kubevirt.io/2019/How-To-Import-VM-into-Kubevirt.html
4. https://kubevirt.io/2018/KubeVirt-Network-Deep-Dive.html
5. https://github.com/kubevirt/containerized-data-importer/tree/master/doc

