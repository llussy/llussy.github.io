---
layout: post
title: "使用Kubernetes RBAC创建kubeconfig"
date: 2019-01-31 10:08:34 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
---


### 介绍

​        RBAC基于角色的访问控制使用（Role-Based Access Control） rbac.authorization.k8s.io  API 组来实现权限控制，RBAC 允许管理员通过 Kubernetes API 动态的配置权限策略。在 1.6 版本中 RBAC 还处于 Beat 阶段，如果想要开启 RBAC 授权模式需要在 apiserver 组件中指定 --authorization-mode=RBAC 选项。

### 四个重要概念

在 RBAC API 的四个重要概念：
Role：是一系列的权限的集合，例如一个角色可以包含读取 Pod 的权限和列出 Pod 的权限
ClusterRole： 跟 Role 类似，但是可以在集群中到处使用（ Role 是 namespace 一级的）
RoloBinding：把角色映射到用户，从而让这些用户继承角色在 namespace 中的权限。
ClusterRoleBinding： 让用户继承 ClusterRole 在整个集群中的权限。

简单点说RBAC实现了在k8s集群中对api-server的鉴权，更多的RBAC知识点请查阅官方文档：<https://kubernetes.io/docs/admin/authorization/rbac/>



### 创建一个kubectl只读权限

#### 创建用户

**签发证书的json**  readonly.json

```json
{
  "CN": "readonly",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "develop:readonly",
      "OU": "develop"
    }
  ]
}
```

**ca-config-readonly.json**

```yaml
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "87600h"
            }
        }
    }
}
```

**基于以 Kubernetes CA 证书创建只读用户的证书**

```bash
cfssl gencert --ca /etc/kubernetes/pki/ca.pem \
              --ca-key /etc/kubernetes/pki/ca-key.pem \
              --config ca-config-readonly.json \
              --profile kubernetes readonly.json | \
              cfssljson --bare readonly
```

以上命令会生成 `readonly-key.pem`、`readonly.pem` 两个证书文件以及一个 csr 请求文件

#### 创建kubeconfig

有了用于证明身份的证书以后，接下来创建一个 kubeconfig 文件方便 kubectl 使用

**kubeconfig.sh**

```bash
kubectl config set-cluster llussy --server="https://10.21.8.220:6443" \
    --certificate-authority=/etc/kubernetes/pki/ca.pem \
    --embed-certs=true \
    --kubeconfig=readonly.kubeconfig

kubectl config set-credentials llussy-readonly \
    --certificate-authority=/etc/kubernetes/pki/ca.pem \
    --embed-certs=true \
    --client-key=readonly-key.pem \
    --client-certificate=readonly.pem \
    --kubeconfig=readonly.kubeconfig

kubectl config set-context llussy-system --cluster=llussy \
    --user=llussy-readonly \
    --kubeconfig=readonly.kubeconfig

kubectl config use-context llussy-system --kubeconfig=readonly.kubeconfig
```



#### 创建ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cluster-readonly
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - replicationcontrollers
  - replicationcontrollers/scale
  - secrets
  - serviceaccounts
  - services
  - services/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - deployments
  - deployments/rollback
  - deployments/scale
  - statefulsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  - scheduledjobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - ingresses
  - replicasets
  verbs:
  - get
  - list
  - watch
```



#### clusterroleing

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: llussy-readonly
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: llussy-readonly
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: develop:readonly
```



### 创建cluster-admin权限用户

**kubectl admin权限有内置的clusterrole**  admin 和 cluster-admin ,这里使用cluster-admin

```bash
kubectl get clusterrole | grep admin
admin                                                                  229d
cluster-admin                                                          229d

#可以使用下面命令查看具体权限信息
kubectl get clusterrole admin -o yaml
kubectl get clusterrole cluster-admin -o yaml
```



```yaml
# cat cluster-admin-lisai.json 
{
  "CN": "cluster-admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "cluster-admin",
      "OU": "cluster-admin"
    }
  ]
}


# cat ca-config-cluster-admin.json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "87600h"
            }
        }
    }
}
```

**创建证书**

```bash
cfssl gencert --ca /etc/kubernetes/pki/ca.pem \
              --ca-key /etc/kubernetes/pki/ca-key.pem \
              --config ca-config-cluster-admin.json \
              --profile kubernetes cluster-admin-lisai.json | \
              cfssljson --bare cluter-admin-lisai
```

**kubeconfig文件**

```shell
kubectl config set-cluster cluster-admin-lisai --server="https://10.21.8.220:6443"     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --kubeconfig=cluster-admin-lisai.kubeconfig

kubectl config set-credentials cluster-admin-lisai-admin     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --client-key=cluster-admin-lisai-key.pem     --client-certificate=cluster-admin-lisai.pem     --kubeconfig=cluster-admin-lisai.kubeconfig

kubectl config set-context cluster-admin-lisai-context --cluster=cluster-admin-lisai     --user=cluster-admin-lisai-admin     --kubeconfig=cluster-admin-lisai.kubeconfig

kubectl config use-context cluster-admin-lisai-context --kubeconfig=cluster-admin-lisai.kubeconfig
```



**clusterroleing**

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-lisai
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin      # 要绑定的权限
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: cluster-admin   #和cluster-admin-lisai.json的O对应。 
```

### 参考

[使用 RBAC 控制 kubectl 权限](https://mritd.me/2018/03/20/use-rbac-to-control-kubectl-permissions/)
[Kubernetes RBAC详解](https://www.qikqiak.com/post/use-rbac-in-k8s/)


