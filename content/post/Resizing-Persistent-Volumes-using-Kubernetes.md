---
title: "Resizing Persistent Volumes Using Kubernetes"
date: 2019-01-21T17:38:53+08:00
lastmod: 2019-08-23T17:38:53+08:00
draft: false
keywords: [Kubernetes, PVC, resize]
tags: [Kubernetes, PVC, resize]
categories: [Kubernetes]
comment: true
toc: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
---

**Feature gates:**  

* enable `ExpandPersistentVolumes` feature gate (1.11:arrow_up: default:ture).
* enable `PersistentVolumeClaimResize` admission controller (1.11:arrow_up: default:ture).

**StorageClass:**

* Storage class’s `allowVolumeExpansion` field should set to true.

<!--more-->
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-heketi
  resourceVersion: "2557237"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/gluster-heketi
  uid: 0cab6ccf-09c9-11e9-bb03-525400c73db8
parameters:
  resturl: http://10.104.202.182:8080
  restuser: admin
  restuserkey: My Secret Life
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true  # <===============
```

To request a larger volume for a PVC, edit the PVC object and specify a larger size. This triggers expansion of the volume that backs the underlying PersistentVolume. A new PersistentVolume is never created to satisfy the claim. Instead, an existing volume is resized.

-> `kubectl edit pvc glusterfs-data-01 -n play-by-yl`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    storage.resource.caicloud.io/app-ids: ""
    storage.resource.caicloud.io/app-names: ""
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/glusterfs
  creationTimestamp: 2019-01-29T10:55:26Z
  finalizers:
  - kubernetes.io/pvc-protection
  name: glusterfs-data-01
  namespace: play-by-yl
  resourceVersion: "130561"
  selfLink: /api/v1/namespaces/play-by-yl/persistentvolumeclaims/glusterfs-data-01
  uid: 62509f38-23b4-11e9-8dc1-525400f31786
spec:
  accessModes:
  - ReadWriteMany
  dataSource: null
  resources:
    requests:
      storage: 30Gi   # <== update this field to resize the PVC
  storageClassName: glusterfs-20190129164702-d41e1f
  volumeName: pvc-62509f38-23b4-11e9-8dc1-525400f31786
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 30Gi
  phase: Bound
```
* Support following types of volumes:
  * gcePersistentDisk
  * awsElasticBlockStore
  * Cinder
  * glusterfs
  * rbd
  * Azure File
  * Azure Disk
  * Portworx
  * FlexVolumes
  * CSI
* Resizing a volume containing a file system
  * file system is XFS, Ext3, or Ext4.
  * Pod is started using the `PersistentVolumeClaim` in ReadWrite mode.
  * if volume is using and you want to expand it, you need to delete or recreate the pod after the volume has been expanded by the cloud provider in the controller-manager.
  * check the status of resize operation by running the `kubectl describe pvc` command, if the `PersistentVolumeClaim` has the status `FileSystemResizePending`, it is safe to recreate the pod using the PersistentVolumeClaim.
  * gcePersistentDisk, awsElasticBlockStore, Azure Disk, Cinder, Ceph RBD
* Resizing an in-use PersistentVolumeClaim
  * Expanding in-use PVCs is an alpha feature. To use it, enable the `ExpandInUsePersistentVolumes` feature gate (1.15:arrow_up: default:ture).
  * Supported by the in-tree volume plugins 



**Reference:**  

* [Expanding Persistent Volumes Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
* [Resizing Persistent Volumes using Kubernetes](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)
