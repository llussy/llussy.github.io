---
layout: post
title: "k8s storageclass-ceph持久化存储"
date: 2019-01-26 22:08:34 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
    - ceph
---

[TOC]

> 主要参考： [使用rbd-provisioner提供rbd持久化存储](https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html)

## 部署rbd-provisioner

kube-controller-manager以容器的方式运行。这种方式下，kubernetes在创建使用ceph rbd pv/pvc时没任何问题，但使用dynamic provisioning自动管理存储生命周期时会报错。提示`"rbd: create volume failed, err: failed to create rbd image: executable file not found in $PATH:"`。

```bash
git clone https://github.com/kubernetes-incubator/external-storage.git
cd external-storage/ceph/rbd/deploy
NAMESPACE=kube-system
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/clusterrolebinding.yaml ./rbac/rolebinding.yaml
kubectl -n $NAMESPACE apply -f ./rbac
```

## 创建storageclass

部署完rbd-provisioner，还需要创建StorageClass。创建SC前，我们还需要创建相关用户的secret；

### secret

#### kube-system-ceph-secret.yaml

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
type: "kubernetes.io/rbd"  
data:
  # ceph auth get-key client.admin | base64
  key: QVFCMHlDeGNVTjA5SmhBQUx4OFJ0V3Q2cjhKdm80cnFSQVJWTXc9PQ== 

--- 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kubernetes mon 'allow r' osd 'allow rwx pool=kubernetes'
  # ceph auth get-key client.kubernetes | base64
  key: QVFBeUhFQmNRNFpDRWhBQUcrNlZOTGp4NTE4bitjSGZlRGV2V2c9PQ==
```

#### ceps-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: ceph
type: "kubernetes.io/rbd"
data:
 # ceph auth add client.kubernetes mon 'allow r' osd 'allow rwx pool=kubernetes'
  # ceph auth get-key client.kuberbetes | base64
  key: QVFBeUhFQmNRNFpDRWhBQUcrNlZOTGp4NTE4bitjSGZlRGV2V2c9PQ==
```

​		创建secret保存`client.admin`和`client.kubernetes`用户的key，`client.admin`和`client.kubernetes`用户的secret可以放在kube-system namespace，但如果其他namespace需要使用ceph rbd的dynamic provisioning功能的话，要在相应的namespace创建secret来保存client.kubernetes用户key信息。**这里在ceph ns里创建了。**

### StorageClass

**storage.yaml**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ceph.com/rbd
parameters:
    monitors: 10.21.8.60:6789,10.21.8.61:6789,10.21.8.65:6789
    adminId: admin
    adminSecretName: ceph-admin-secret
    adminSecretNamespace: kube-system
    pool: kubernetes
    userId: kubernetes
    userSecretName: ceph-secret
    fsType: ext4
    imageFormat: "2"
    imageFeatures: "layering"
```

- 其他设置和普通的ceph rbd StorageClass一致，但provisioner需要设置为`ceph.com/rbd`，不是默认的`kubernetes.io/rbd`，这样rbd的请求将由rbd-provisioner来处理；
- 考虑到兼容性，建议尽量关闭rbd image feature，并且kubelet节点的`ceph-common`版本尽量和ceph服务器端保持一致，我的环境都使用的M版本；
- 执行完查看

```bash
# kubectl get storageclasses.storage.k8s.io 
NAME                  PROVISIONER    AGE
ceph-rbd (default)    ceph.com/rbd   3h
managed-nfs-storage   test.cn/nfs    78d
```

### k8s node分发配置文件

**需要将ceph 的keyring文件分发到k8s所有node,这里是ceph.client.kubernetes.keyring**

```shell
client.kubernetes
        key: AQAyHEBcQ4ZCEhAAG+6VNLjx518n+cHfeDevWg==
        caps: [mon] allow r
        caps: [osd] allow rwx pool=kubernetes
```



## 测试ceph rbd自动分配

### 两个测试

**test-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1
spec:
  containers:
  - name: ceph-busybox
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-claim
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:  
    - ReadWriteOnce  #可读可写，但只支持被单个Pod挂载。
  resources:
    requests:
      storage: 2Gi
```

**rbd-pvc.yaml**

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-pvc1
  namespace: ceph
spec:
  storageClassName: ceph-rbd
  accessModes:
  - ReadWriteOnce  #可读可写，但只支持被单个Pod挂载。
  resources:
    requests:
      storage: 1Gi
```

### 测试结果

```bash
[root@yztest-ops-k8s24v8-yz test-ceph]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                STORAGECLASS   REASON    AGE
altermanager-pv                            10Gi       RWO            Recycle          Bound       monitoring/prometheus-alertmanager   nfs                      12d
etcd-pv                                    8Gi        RWO            Recycle          Bound       default/cron-nas                     nfs                      103d
naftis-pv                                  10Gi       RWO            Retain           Released    naftis/naftis-mysql                  manual                   87d
prometheus-pv-pro                          8Gi        RWO            Recycle          Bound       monitoring/prometheus-server         nfs                      12d
pvc-12faa893-1e0e-11e9-a900-005056bd0b19   2Gi        RWO            Delete           Bound       ceph/ceph-claim                      ceph-rbd                 3h
pvc-1d9d66c9-1e25-11e9-b6b8-005056bd7916   1Gi        RWO            Delete           Bound       ceph/rbd-pvc1                        ceph-rbd                 25m
[root@yztest-ops-k8s24v8-yz test-ceph]# kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-claim   Bound     pvc-12faa893-1e0e-11e9-a900-005056bd0b19   2Gi        RWO            ceph-rbd       3h
rbd-pvc1     Bound     pvc-1d9d66c9-1e25-11e9-b6b8-005056bd7916   1Gi        RWO            ceph-rbd       25m
[root@yztest-ops-k8s24v8-yz test-ceph]# rbd -p kubernetes list
kubernetes-dynamic-pvc-17703950-1e0e-11e9-a8ab-b65250c94f1a
kubernetes-dynamic-pvc-1da1909f-1e25-11e9-a8ab-b65250c94f1a
```

**请注意：删除测试 rbd 设备会被删除**

可以在StorageClass中设置删除策略`reclaimPolicy: Retain`

## 遇到的问题

### map failed exit status 5, rbd output: rbd: sysfs write failed

```shell
 MountVolume.WaitForAttach failed for volume "pvc-12faa893-1e0e-11e9-a900-005056bd0b19" : rbd: map failed exit status 5, rbd output: rbd: sysfs write failed
In some cases useful info is found in syslog - try "dmesg | tail".
```

**dmesg会看到**

```bash
libceph: mon0 10.103.2.24:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
libceph: mon0 10.103.2.24:6789 missing required protocol features
```

**目前解决办法**(不确定生产环境执行一下命令会遇到什么问题，慎重)

```bash
$ ceph osd crush tunables legacy
$ ceph osd crush reweight-all
```

>> 其實也可以將 kernel 升級到 4.5 以上解決此問題

使用这种方式ceph会出现报错**crush map has straw_calc_version=0**

```bash
ceph -s 
  cluster:
    id:     e9ffcfbe-976d-450f-91df-7072fc03367a
    health: HEALTH_WARN
            crush map has straw_calc_version=0
 
  services:
    mon: 3 daemons, quorum yztestopshadoop60v8yz,yztestopshadoop61v8yz,yztestopshadoop65v8yz
    mgr: yztestopshadoop60v8yz(active), standbys: yztestopshadoop65v8yz, yztestopshadoop61v8yz
    mds: cephfs-1/1/1 up  {0=yztestopshadoop65v8yz=up:active}
    osd: 5 osds: 5 up, 5 in
    rgw: 2 daemons active
 
  data:
    pools:   12 pools, 400 pgs
    objects: 2.86 k objects, 7.7 GiB
    usage:   28 GiB used, 172 GiB / 200 GiB avail
    pgs:     400 active+clean
```

可通过`ceph osd crush tunables optimal`解决。

## pv 和 ceph rbd 对应关系查看

**kubectl get pv xxxx -o yaml**  spec.rbd.image的值
```bash
[root@yztest-ops-k8s24v8-yz ~]# kubectl get pv pvc-a647ce74-72cf-11e9-acde-005056bd5fee -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: ceph.com/rbd
    rbdProvisionerIdentity: ceph.com/rbd
  creationTimestamp: 2019-05-10T02:59:38Z
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-a647ce74-72cf-11e9-acde-005056bd5fee
  resourceVersion: "75359563"
  selfLink: /api/v1/persistentvolumes/pvc-a647ce74-72cf-11e9-acde-005056bd5fee
  uid: a65872d7-72cf-11e9-acde-005056bd5fee
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 50Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: prometheus-prometheus-operator-prometheus-db-prometheus-prometheus-operator-prometheus-0
    namespace: monitoring
    resourceVersion: "75359542"
    uid: a647ce74-72cf-11e9-acde-005056bd5fee
  persistentVolumeReclaimPolicy: Delete
  rbd:
    fsType: ext4
    image: kubernetes-dynamic-pvc-a64bbbb8-72cf-11e9-b847-86ee373712c2
    keyring: /etc/ceph/keyring
    monitors:
    - 10.21.8.57:6789
    - 10.21.8.59:6789
    - 10.21.8.60:6789
    pool: kubernetes
    secretRef:
      name: ceph-secret
    user: kubernetes
  storageClassName: ceph-rbd
status:
  phase: Bound
```

## 参考

[使用rbd-provisioner提供rbd持久化存储](https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html)

[Kubernetes設定 StorageClass 以 Ceph RBD 為例](https://godleon.github.io/blog/Kubernetes/k8s-Config-StorageClass-with-Ceph-RBD/)

[Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

[Ceph - Crush Map has Legacy Tunables](http://indrapr.blogspot.com/2016/06/ceph-crush-map-has-legacy-tunables.html)

