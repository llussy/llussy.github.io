---
layout: post
title: "基于 Kubernetes RBAC 创建 kubeconfig"
date: 202-03-17 18:08:34 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
---

## 介绍

RBAC基于角色的访问控制使用（Role-Based Access Control） rbac.authorization.k8s.io  API 组来实现权限控制，RBAC 允许管理员通过 Kubernetes API 动态的配置权限策略。

## 四个重要概念

在 RBAC API 的四个重要概念：
Role：是一系列的权限的集合，例如一个角色可以包含读取 Pod 的权限和列出 Pod 的权限
ClusterRole： 跟 Role 类似，但是可以在集群中到处使用（ Role 是 namespace 一级的）
RoloBinding：把角色映射到用户，从而让这些用户继承角色在 namespace 中的权限。
ClusterRoleBinding： 让用户继承 ClusterRole 在整个集群中的权限。

简单点说RBAC实现了在k8s集群中对api-server的鉴权，更多的RBAC知识点请查阅官方文档：<https://kubernetes.io/docs/admin/authorization/rbac/>


## 通过kubectl创建kubeconfig
###  Create ServiceAccount

```bash
kubectl -n kube-system create serviceaccount llussy-ro
```

### ClusterRoleBinding
此处使用的 `ClusterRole` 为 `tke:ro` （tencent tke只读权限）,如果想拥有最高权限可以使用`cluster-admin`。
```bash
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: llussy-ro
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tke:ro
subjects:
- kind: ServiceAccount
  name: llussy-ro
  namespace: kube-system
EOF
```
### Generate kubeconfig 

使用生成新 kubeconfig 文件所需的访问数据设置以下环境变量。
```bash
export USER_TOKEN_NAME=$(kubectl -n kube-system get serviceaccount llussy-ro -o=jsonpath='{.secrets[0].name}')
export USER_TOKEN_VALUE=$(kubectl -n kube-system get secret/${USER_TOKEN_NAME} -o=go-template='{{.data.token}}' | base64 --decode)
export CURRENT_CONTEXT=$(kubectl config current-context)
export CURRENT_CLUSTER=$(kubectl config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CURRENT_CONTEXT}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
export CLUSTER_CA=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}"{{with index .cluster "certificate-authority-data" }}{{.}}{{end}}"{{ end }}{{ end }}')
export CLUSTER_SERVER=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')


# 生成 kubeconfig 文件

cat << EOF > llussy-ro-config
apiVersion: v1
kind: Config
current-context: ${CURRENT_CONTEXT}
contexts:
- name: ${CURRENT_CONTEXT}
  context:
    cluster: ${CURRENT_CONTEXT}
    user: llussy-ro
    namespace: kube-system
clusters:
- name: ${CURRENT_CONTEXT}
  cluster:
    certificate-authority-data: ${CLUSTER_CA}
    server: ${CLUSTER_SERVER}
users:
- name: llussy-ro
  user:
    token: ${USER_TOKEN_VALUE}
EOF
```

#### test
```bash
🐳 15:48:01 ❯ kubectl --kubeconfig ./llussy-ro-config get pod  -n nginx-wieof-loda
NAME                                    READY   STATUS             RESTARTS   AGE
mysql-1637829625-0                      1/1     Running            0          99d
nginx-prometheus-dpt-595c6dc69f-lbj6v   1/1     Running            0          34d
nginx-prometheus-dpt-748cf76bcd-j2vhz   0/1     CrashLoopBackOff   1505       3d3h

root in k8s  🍣 master
🐳 15:48:18 ❯ kubectl --kubeconfig ./llussy-ro-config -n nginx-wieof-loda delete pod nginx-prometheus-dpt-748cf76bcd-j2vhz
Error from server (Forbidden): pods "nginx-prometheus-dpt-748cf76bcd-j2vhz" is forbidden: User "system:serviceaccount:kube-system:llussy-ro" cannot delete resource "pods" in API group "" in the namespace "nginx-wieof-loda"

```

## 通过CA证书创建kubeconfig
> 建议通过kubectl创建,简单
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

## 参考

[使用 RBAC 控制 kubectl 权限](https://mritd.me/2018/03/20/use-rbac-to-control-kubectl-permissions/)

[Kubernetes RBAC详解](https://www.qikqiak.com/post/use-rbac-in-k8s/)

[Generate a kubeconfig File
](https://docs.d2iq.com/dkp/kommander/2.0/clusters/attach-cluster/generate-kubeconfig/)

