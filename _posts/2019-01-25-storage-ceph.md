---
layout: post
title: "k8s使用ceph存储"
date: 2019-01-25 20:23:34 +0800
catalog: ture  
header-img: "img/WX20190126-093114.png" 
multilingual: false
tags: 
    - kubernetes
    - ceph
---


[TOC]

> kubernetes使用ceph存储

## cephFS挂载到pod

### ceph管理节点操作

#### 创建元数据服务器 MDS

```bash
ceph-deploy mds create yztestopshadoop61v8yz 
ceph mds stat
```

#### 创建cephfs

```bash
$ ceph osd pool create cephfs_data 128
$ ceph osd pool create cephfs_metadata 128
$ ceph fs new cephfs cephfs_metadata cephfs_data
$ ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```



### k8s操作

#### 认证文件方式

**k8s节点创建认证文件 admin.secret**

在ceph中查看 

```bash
cat /etc/ceph/ceph.client.admin.keyring
[client.admin]
        key = AQAf9ORbCibyORAA+atbXYtlVVo+nktj4ybhLQ==
        caps mds = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

k8s节点：/etc/ceph/admin.secret要同步到各个节点

```bash
vim /etc/ceph/admin.secret
AQAf9ORbCibyORAA+atbXYtlVVo+nktj4ybhLQ==
```

**挂载测试**

```bash
#一个mon
mount -t ceph 10.21.8.56:6789:/ /mnt/cephfs -o name=admin,secretfile=/etc/ceph/admin.secret
#多个mon
mount -t ceph 10.21.8.57:6789,10.21.8.59:6789,10.21.8.60:6789:/ /mnt/cephfs -o name=admin,secretfile=/etc/ceph/admin.secret
```

**k8s pod测试**

```yaml
# cat cephfs.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: cephfs1
spec:
  containers:
  - name: cephfs-rw
    image: docker.ifeng.com/base/centos:7.2.1511
    command: ["top","-b"]
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 10.21.8.56:6789
      user: admin
      secretFile: "/etc/ceph/admin.secret"
      readOnly: true
```

进入pod查看

```yaml
[root@yztest-ops-k8s24v8-yz ceph]# kubectl exec -it cephfs1 /bin/bash
[root@cephfs1 /]# df -h
Filesystem         Size  Used Avail Use% Mounted on
overlay             64G  2.8G   61G   5% /
tmpfs               64M     0   64M   0% /dev
tmpfs              3.9G     0  3.9G   0% /sys/fs/cgroup
10.21.8.56:6789:/  160G   21G  140G  13% /mnt/cephfs
/dev/sda3           20G  3.5G   17G  18% /etc/hosts
/dev/sda9           64G  2.8G   61G   5% /etc/hostname
shm                 64M     0   64M   0% /dev/shm
tmpfs              3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs              3.9G     0  3.9G   0% /proc/scsi
tmpfs              3.9G     0  3.9G   0% /sys/firmware
```



#### secret方式

**推荐**

ceph中加密KEY

```yaml
$ ceph auth get-key client.admin |base64
QVFBZjlPUmJDaWJ5T1JBQSthdGJYWXRsVlZvK25rdGo0eWJoTFE9PQ==
```

ceph-secret.yaml 

```yaml
[root@yztest-ops-k8s24v8-yz ceph]# cat ceph-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFBZjlPUmJDaWJ5T1JBQSthdGJYWXRsVlZvK25rdGo0eWJoTFE9PQ== 
  
```

cephfs-with-secret.yaml

```yaml
[root@yztest-ops-k8s24v8-yz ceph]# cat cephfs-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs2
spec:
  containers:
  - name: cephfs-secret
    image: docker.ifeng.com/base/centos:7.2.1511
    command: ["top","-b"]
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 10.21.8.56:6789
      user: admin
      secretRef:
        name: ceph-secret
      readOnly: true
```





## ceph-rbd挂载到pod

#### 认证文件方式

**ceph 节点操作**

创建ceph image

```bash
[cephd@yztestopshadoop64v8yz my-cluster]$ rbd list
[cephd@yztestopshadoop64v8yz my-cluster]$ rbd create k8s --size 1024   
[cephd@yztestopshadoop64v8yz my-cluster]$ rbd feature disable k8s exclusive-lock object-map fast-diff deep-flatten
[cephd@yztestopshadoop64v8yz my-cluster]$ rbd list
k8s
# 把 k8s images 映射到内核
[cephd@yztestopshadoop64v8yz my-cluster]$ sudo rbd map k8s 
/dev/rbd1
[cephd@yztestopshadoop64v8yz my-cluster]$ rbd showmapped
id pool image       snap device    
0  kube pv001-image -    /dev/rbd0 
1  rbd  k8s         -    /dev/rbd1 
# 将 k8s image 格式化为 ext4 格式的文件系统，注意这里也可以不执行，后边创建 Pod 时也会自动完成
$ sudo mkfs.ext4 -m0 /dev/rbd1
```



centos-ceph-rbd.yaml

```yaml
# cat centos-ceph-rbd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: centos-w
spec:
  containers:
    - image: docker.ifeng.com/base/centos:7.2.1511
      command: ["top","-b"]
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  nodeSelector:
    ceph: "true"
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.21.8.56:6789'
        - '10.21.8.60:6789'
        pool: rbd
        image: k8s
        fsType: ext4
        readOnly: false
        user: admin
        keyring: /etc/ceph/keyring
```

 k8s 节点上生成 keyring 密钥文件，否则是连接不上 Ceph 存储集群的

```yaml
$ cp /etc/ceph/ceph.client.admin.keyring /etc/ceph/keyring 
```

**pod 上查看**

/dev/rbd1       976M  2.6M  907M   1% /mnt/rbd

```bash
[root@yztest-ops-k8s24v8-yz ceph]# kubectl exec -it centos-w /bin/bash
[root@centos-w /]# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          64G  2.8G   61G   5% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda3        20G  3.6G   17G  18% /etc/hosts
/dev/rbd1       976M  2.6M  907M   1% /mnt/rbd
/dev/sda9        64G  2.8G   61G   5% /etc/hostname
shm              64M     0   64M   0% /dev/shm
tmpfs           3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           3.9G     0  3.9G   0% /proc/scsi
tmpfs           3.9G     0  3.9G   0% /sys/firmware
```



#### secret 方式

**ceph 节点操作**

```bash
[cephd@yztestopshadoop64v8yz my-cluster]$ ceph auth get-key client.admin |base64
QVFBZjlPUmJDaWJ5T1JBQSthdGJYWXRsVlZvK25rdGo0eWJoTFE9PQ==

rbd create ifeng-test --size 2048
rbd feature disable ifeng-test exclusive-lock, object-map, fast-diff, deep-flatten
```

secret-rbd.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-rbd
type: "kubernetes.io/rbd"
data:
  key: QVFBZjlPUmJDaWJ5T1JBQSthdGJYWXRsVlZvK25rdGo0eWJoTFE9PQ==
```

rbd-with-secret.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: docker.ifeng.com/base/centos:7.2.1511
      command: ["top","-b"]
      name: rbd-test
      volumeMounts:
      - name: rbd-secret
        mountPath: /mnt/test
  volumes:
    - name: rbd-secret
      rbd:
        monitors:
        - '10.21.8.56:6789'
        - '10.21.8.60:6789'
        pool: rbd
        image: ifeng-test
        fsType: ext4
        readOnly: false
        user: admin
        secretRef:
          name: ceph-secret-rbd
```



## Kubernetes PV & PVC 方式使用 Ceph RBD

ceph 创建images

```
rbd create ceph-rbd-pv-test --size 2048
rbd feature disable ceph-rbd-pv-test exclusive-lock, object-map, fast-diff, deep-flatten
```

rbd-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-rbd-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - 10.21.8.56:6789
    pool: rbd
    image: ceph-rbd-pv-test
    user: admin
    secretRef:
      name: ceph-secret-rbd
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```

rbd-pv-claim.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-rbd-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

rbd-pvc-pod1.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: rbd-pvc-pod
  name: ceph-rbd-pv-pod1
spec:
  containers:
  - name: ceph-rbd-pv-busybox
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-rbd-vol1
      mountPath: /mnt/ceph-rbd-pvc/busybox
      readOnly: false
  volumes:
  - name: ceph-rbd-vol1
    persistentVolumeClaim:
      claimName: ceph-rbd-pv-claim
```

pod查看

```bash
kubectl exec -it ceph-rbd-pv-pod1 /bin/sh
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  63.5G      3.0G     60.5G   5% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     3.9G         0      3.9G   0% /sys/fs/cgroup
/dev/sda3                20.0G      2.8G     17.2G  14% /dev/termination-log
/dev/sda9                63.5G      3.0G     60.5G   5% /etc/resolv.conf
/dev/sda9                63.5G      3.0G     60.5G   5% /etc/hostname
/dev/sda3                20.0G      2.8G     17.2G  14% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
/dev/rbd1                 1.9G      6.0M      1.8G   0% /mnt/ceph-rbd-pvc/busybox
```

## ceph客户端安装

```bash
wget http://10.21.8.20/ceph.repo
yum install ceph-common ceph-fuse -y


sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*

```



### 参考:

 [初试 Kubernetes 集群使用 Ceph RBD 块存储](https://blog.csdn.net/aixiaoyang168/article/details/78999851)

[使用Ceph RBD为Kubernetes集群提供存储卷](https://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)

