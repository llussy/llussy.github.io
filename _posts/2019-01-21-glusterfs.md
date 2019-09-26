---
layout: post
categories: k8s
title: "k8s使用gfs做持久化存储"
date: 2019-01-21 21:10:34 +0800
description: gfs
header-style: text
catalog: ture
multilingual: false
tags: 
    - pv
    - kubernetes
    - glusterfs
---

[TOC]

## 搭建gfs集群

10.80.137.154,10.80.137.155,10.80.137.156,10.80.137.157,10.80.137.158,10.80.137.159 这六台

下载安装：

```bash

可以用网上的安装包或使用本地已下载好的

#在线下载安装包

wget https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-3.10/glusterfs{,-server,-fuse,-geo-replication,-cli,-api,-client-xlators,-libs}-3.10.12-1.el7.x86_64.rpm --no-check-certificate

wget https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-3.10/{userspace-rcu-0.7.16-3.el7.x86_64.rpm,userspace-rcu-devel-0.7.16-3.el7.x86_64.rpm} --no-check-certificate

#使用本地包

cd /data/soft/packs
wget http://10.21.8.20/gluster3.10.12/gluster3.10.tar.gz
tar zxvf gluster3.10.tar.gz 
yum install userspace-rcu-* glusterfs-* python2-gluster* -y 
systemctl enable glusterd && systemctl start glusterd && systemctl status glusterd

```

### 客户端软件包安装

```bash

cd /data/soft/packs
mkdir glusterfs-client ;cd glusterfs-client
wget http://10.21.8.20/gluster3.10.12/glusterfs-3.10.12-1.el7.x86_64.rpm
wget http://10.21.8.20/gluster3.10.12/glusterfs-libs-3.10.12-1.el7.x86_64.rpm
wget http://10.21.8.20/gluster3.10.12/glusterfs-fuse-3.10.12-1.el7.x86_64.rpm
wget http://10.21.8.20/gluster3.10.12/glusterfs-client-xlators-3.10.12-1.el7.x86_64.rpm
yum install glusterfs-libs-3.10.12-1.el7.x86_64.rpm -y
yum install glusterfs-3.10.12-1.el7.x86_64.rpm -y 
yum install glusterfs-client-xlators-3.10.12-1.el7.x86_64.rpm -y
yum install glusterfs-fuse-3.10.12-1.el7.x86_64.rpm -y 

```

### 将主机加到存储池

```bash
gluster peer probe 10.80.137.155
gluster peer probe 10.80.137.156
gluster peer probe 10.80.137.157
gluster peer probe 10.80.137.158
gluster peer probe 10.80.137.159
```

### 查看存储池状态

```bash

# gluster peer status

Number of Peers: 5


Hostname: 10.80.137.155

Uuid: 35dca7c2-5247-4926-999b-4fa79240e48d

State: Peer in Cluster (Connected)


Hostname: 10.80.137.156

Uuid: 457c7633-fb1e-4260-9714-76a978c066d0

State: Peer in Cluster (Connected)


Hostname: 10.80.137.157

Uuid: 92efc5e0-bc95-49dc-a8ec-9b80a093896d

State: Peer in Cluster (Connected)


Hostname: 10.80.137.158

Uuid: dc431264-e7c1-4a89-aa15-b4c00e79aa00

State: Peer in Cluster (Connected)


Hostname: 10.80.137.159

Uuid: 05813c08-ed4f-4feb-9d29-dbb50d1e3448

State: Peer in Cluster (Connected)

```

### 创建k8s_vol



```

gluster volume create k8s_vol replica 2 transport tcp 10.80.137.154:/data/gluster/brick 10.80.137.155:/data/gluster/brick 10.80.137.156:/data/gluster/brick 10.80.137.157:/data/gluster/brick 10.80.137.158:/data/gluster/brick 10.80.137.159:/data/gluster/brick

```



### 启动卷



```bash

gluster volume start k8s_vol

### 查看卷信息

gluster volume info

Volume Name: k8s_vol

Type: Distributed-Replicate

Volume ID: f2ecf908-c04e-4b28-aa5a-c715677e5326

Status: Started

Snapshot Count: 0

Number of Bricks: 3 x 2 = 6

Transport-type: tcp

Bricks:

Brick1: 10.80.137.154:/data/gluster/brick

Brick2: 10.80.137.155:/data/gluster/brick

Brick3: 10.80.137.156:/data/gluster/brick

Brick4: 10.80.137.157:/data/gluster/brick

Brick5: 10.80.137.158:/data/gluster/brick

Brick6: 10.80.137.159:/data/gluster/brick

Options Reconfigured:

auth.allow: 10.80.14.142,10.80.15.142,10.80.16.142,10.80.30.144,10.80.75.146,10.80.76.146,10.80.77.146,10.80.14.175,10.80.15.175,10.80.16.175,10.80.5.176,10.80.7.176,10.80.9.176,10.80.10.176,10.80.3.179,10.80.137.154,10.80.137.155,10.80.137.156,10.80.137.157,10.80.137.158,10.80.137.159,10.80.137.160

transport.address-family: inet

nfs.disable: on

```

### 添加认证机器

```

gluster volume set k8s_vol auth.allow 10.80.14.142,10.80.15.142,10.80.16.142,10.80.30.144,10.80.75.146,10.80.76.146,10.80.77.146,10.80.14.175,10.80.15.175,10.80.16.175,10.80.5.176,10.80.7.176,10.80.9.176,10.80.10.176,10.80.3.179,10.80.137.154,10.80.137.155,10.80.137.156,10.80.137.157,10.80.137.158,10.80.137.159,10.80.137.160

#ip为k8s集群所有ip

```





### 挂载gfs

挂载前需安装glusterfs客户端

```

mount -t glusterfs 10.80.137.155:k8s_vol /share   #ip可为gfs集群任一ip

```



### Glusterfs调优

```

# 开启 指定 volume 的配额
$ gluster volume quota k8s-volume enable

# 限制 指定 volume 的配额
$ gluster volume quota k8s-volume limit-usage / 1TB

# 设置 cache 大小, 默认32MB
$ gluster volume set k8s-volume performance.cache-size 4GB

# 设置 io 线程, 太大会导致进程崩溃
$ gluster volume set k8s-volume performance.io-thread-count 16

# 设置 网络检测时间, 默认42s
$ gluster volume set k8s-volume network.ping-timeout 10

# 设置 写缓冲区的大小, 默认1M
$ gluster volume set k8s-volume performance.write-behind-window-size 1024MB
```

## k8s使用gfs

k8s集群所有机器安装glusterfs客户端

### 配置endpoints

```

# cat glusterfs-endpoints.json 
{
"kind": "Endpoints",
"apiVersion": "v1",
"metadata": {
"name": "glusterfs-cluster",
"namespace": "monitoring"    ###不写namespace时 prometheus使用pvc报错，找不到endpoints  glusterfs-cluster
},
"subsets": [
{
  "addresses": [
    {
      "ip": "10.80.137.154"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
},
{
  "addresses": [
    {
      "ip": "10.80.137.155"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
},
{
  "addresses": [
    {
      "ip": "10.80.137.156"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
},
{
  "addresses": [
    {
      "ip": "10.80.137.157"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
},
{
  "addresses": [
    {
      "ip": "10.80.137.158"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
},
{
  "addresses": [
    {
      "ip": "10.80.137.159"
    }
  ],
  "ports": [
    {
      "port": 1239
    }
  ]
}
]
}

#port可为任意端口
#应用
kubect apply -f  glusterfs-endpoints.json 

```

### 配置serivce

```

# cat glusterfs-service.json 

{
"kind": "Service",
"apiVersion": "v1",
"metadata": {
"name": "glusterfs-cluster"
},
"spec": {
"ports": [
  {"port": 1239}
]
}
}

#应用
kubectl apply -f glusterfs-service.json 

```

### k8s使用gfs测试用例

[官方文档例子](https://github.com/kubernetes/examples/blob/master/staging/volumes/glusterfs/README.md)

创建pv

```

# cat glusterfs-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: glusterfs-pv
  labels:
    type: glusterfs-nginx
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteMany
  glusterfs:
    endpoints: "glusterfs-cluster"
    path: "k8s_vol"  ##为glusterfs volume名字
    readOnly: false

#应用
kubectl apply -f glusterfs-pv.yaml

```

创建pvc

```

# cat glusterfs-pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: glusterfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 8Gi
  selector:
    matchLabels:
      type: glusterfs-nginx

#应用
kubectl apply -f glusterfs-pvc.yaml

```

### 查看pv pvc

```

# kubectl get pv

NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE

alertmanager-pv 30Gi RWX Retain Bound monitoring/prometheus-alertmanager gfs 34m

glusterfs-pv 8Gi RWX Retain Bound default/glusterfs-pvc 9s

prometheus-pv 100Gi RWX Retain Bound monitoring/prometheus-server gfs 34m

# kubectl get pvc

NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE

glusterfs-pvc Bound glusterfs-pv 8Gi RWX 6s

```

### 创建 nginx deployment 挂载 volume
```

$ vi nginx-deployment.yaml
apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80
          volumeMounts:
            - name: gluster-dev-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: gluster-dev-volume
        persistentVolumeClaim:
          claimName: glusterfs-pv

# 导入 deployment
$ kubectl apply -f nginx-deployment.yaml 

# 查看 deployment
# kubectl get pods |grep nginx-dm
nginx-dm-56d4f4fc9f-lwtx7           1/1       Running   0          5m
nginx-dm-56d4f4fc9f-trfcm           1/1       Running   0          5m
# 查看 挂载
# kubectl exec -it nginx-dm-56d4f4fc9f-lwtx7 -- df -h | grep -A2 k8s   
10.80.137.154:k8s_vol
                          2.7T      6.9G      2.7T   0% /usr/share/nginx/html
# 创建文件 测试
$ kubectl exec -it  nginx-dm-56d4f4fc9f-lwtx7 -- touch /usr/share/nginx/html/index.html

# 验证 glusterfs
# ls /share/
index.html  lock  nflog  silences  wal
```

### 参考

[使用glusterfs做持久化存储](https://jimmysong.io/kubernetes-handbook/practice/using-glusterfs-for-persistent-storage.html)

[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

[使用Glusterfs做持久化存储2](https://blog.csdn.net/cuipengchong/article/details/72152547)

[基于 GlusterFS 实现 Docker 集群的分布式存储](https://www.ibm.com/developerworks/cn/opensource/os-cn-glusterfs-docker-volume/index.html)

