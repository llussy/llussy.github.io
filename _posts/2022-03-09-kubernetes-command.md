---
layout: post
title: "kubernetes 常用操作"
date: 2022-03-09 16:20:30 +0800
header-style: text
header-img: "img/anfeng.jpg"
catalog: ture
multilingual: false
tags:
    - kubernetes
---

### 常用命令

```bash
kubectl logs mypod --previous
kubectl delete pod -n kube-system test-q2ngc --grace-period=180
kubectl delete pod <pod> --force --grace-period=0
kubectl exec -it -n monitoring test-f7bdb9769-ctxbs -- env COLUMNS=210 LINES=60 bash
kubectl scale deployments/kubernetes-bootcamp --replicas=3
kubectl rollout status ds/<daemonset-name>  # status
kubectl rollout undo deployments/kubernetes-bootcamp #回退
kubectl  get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq
kubectl -v=8 top node
kubectl -v=8 get svc kubernetes
kubectl apply -f ds.yaml --dry-run -o go-template='{{.spec.updateStrategy.type}}{{"\\\\n"}}'

# doc
kubectl api-resources
kubectl explain statefulsets

journalctl -xef -u kubelet

# 打印使用的api
kubectl get pod -v=9
```

### 资源使用情况
```bash
kubectl get deployments.apps -n xxx -o custom-columns=:.metadata.namespace,:.metadata.name,:.spec.replicas,:.spec.template.spec.containers[0].resources.limits.cpu,:.spec.template.spec.containers[0].resources.limits.memory
```

### 运行测试 pod

```bash
kubectl run busytest --rm -it --image=busybox sh
kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
kubectl run centos --rm -it --image=centos bash
```

### 查看某个app pod分布情况

```bash
kubectl  get pods -o wide -l app="nginx-server" | awk '{print $7}'| \\
awk '{ count[$0]++  }
 END {
   printf("%-35s: %s\\n","Word","Count");
   for(ind in count){
    printf("%-35s: %d\\n",ind,count[ind]);
   }
 }'
```

### label master

```bash
# label
kubectl get node --show-labels
kubectl label node yztest-ops-k8s24v8-yz role=master
kubectl label node yztest-ops-k8s24v8-yz role-
# spec.template.spec.nodeSelector:  node label位置

# 标记为node
kubectl label node hostname node-role.kubernetes.io/node=

# master 充当node
kubectl taint node yztest-ops-k8s24v8-yz node-role.kubernetes.io/master-

# 恢复master角色
kubectl taint node yztest-ops-k8s24v8-yz node-role.kubernetes.io/master="":NoSchedule
```

### k8s自动补全

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### patch

如果一个容器已经在运行，这时需要对一些容器属性进行修改，又不想删除容器，或不方便通过replace的方式进行更新。kubernetes还提供了一种在容器运行时，直接对容器进行修改的方式，就是patch命令。 如前面创建pod的label是app=nginx-2，如果在运行过程中，需要把其label改为app=nginx-3，这patch命令如下：

```bash
kubectl patch pod rc-nginx-2-kpiqt -p '{"metadata":{"labels":{"app":"nginx-3"}}}'
```


### deployment
#### rolling-update

`rolling-update`是一个非常重要的命令，对于已经部署并且正在运行的业务，rolling-update提供了不中断业务的更新方式。rolling-update每次起一个新的pod，等新pod完全起来后删除一个旧的pod，然后再起一个新的pod替换旧的pod，直到替换掉所有的pod。

rolling-update需要确保新的版本有不同的name，Version和label，否则会报错 。

```bash
kubectl rolling-update rc-nginx-2 -f rc-nginx.yaml
```

如果在升级过程中，发现有问题还可以中途停止update，并回滚到前面版本

```bash
kubectl rolling-update rc-nginx-2 —rollback
```

#### scale

```bash
kubectl scale --replicas=3 rs/foo                                 # Scale a replicaset named 'foo' to 3
kubectl scale --replicas=3 -f foo.yaml                            # Scale a resource specified in "foo.yaml" to 3
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # If the deployment named mysql's current size is 2, scale mysql to 3
kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # Scale multiple replication controllers
```

#### rollout

```bash
kubectl rollout history deployment/frontend                      # Check the history of deployments including the revision 
kubectl rollout undo deployment/frontend                         # Rollback to the previous deployment
kubectl rollout undo deployment/frontend --to-revision=2         # Rollback to a specific revision
kubectl rollout status -w deployment/frontend                    # Watch rolling update status of "frontend" deployment until completion
kubectl rollout restart deployment/frontend                      # Rolling restart of the "frontend" deployment
```

### pod 与主机文件传输

```bash
kubectl cp foo-pod:/var/log/foo.log foo.log
kubectl cp localfile foo-pod:/etc/remotefile
```

### kubeadm

```bash
kubeadm config print init-defaults
kubeadm config view
```


### 快速查看pid的容器名字

```bash
podinfo() {
  CID=$(cat /proc/$1/cgroup | awk -F '/' '{print $5}')
  CID=$(echo ${CID:7:10})
  crictl inspect $CID | jq '.status.labels["io.kubernetes.pod.name"]'
}

增加到~/.bashrc

# podinfo 16529
"test-6cc546b5-75q9n"
```

**另一种方法**

```bash
#!/bin/bash
forid=$(ps -efl|grep log4j-core|grep -v grep|awk '{print $4}'|xargs -i pstree -sg {} |awk 'NR==1{print $0}'|grep -E -o "[0-9]{1,10}"|grep -v ^1)
for I in $forid; do
docker ps -q | xargs docker inspect --format '{{.State.Pid}}, {{.Name}}' | grep "$I"|grep -v formatMsgNoLookups;
done
```


### nsenter

快速脚本，需进入进入pod所在node,增加下面函数到~/.bashrc

```bash
function e() {
      set -eu
      ns=${2-"default"}
      pod=`kubectl -n $ns describe pod $1 | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\\/\\/\\(.*\\)$/\\1/'`
      pid=`docker inspect -f {{.State.Pid}} $pod`
      echo "entering pod netns for $ns/$1"
      cmd="nsenter -n --target $pid"
      echo $cmd
      $cmd
  }
```

进入pod所在netns

```bash
e istio-galley-58c7c7c646-m6568 istio-system
e proxy-5546768954-9rxg6 # 省略 NAMESPACE 默认为 default
tcpdump -i eth0 -w test.pcap port 80
```

原文地址： [Kubernetes 问题定位技巧：容器内抓包](https://imroc.io/posts/kubernetes/capture-packets-in-container/)

### metrics 
#### cadvisor
```bash
kubectl get --raw /api/v1/nodes/10.66.240.101/proxy/metrics/cadvisor
```

#### kube-apiserver
```bash
kubectl get --raw /metrics
```

### 参考

[kubelet常用命令](https://blog.csdn.net/xingwangc2014/article/details/51204224)