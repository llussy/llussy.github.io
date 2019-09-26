---
layout: post
title: "kubernetes delete namespace"
date: 2019-04-13 19:37:34 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
---



### kubernetes delete namespace 处于 Terminating状态

```bash
# kubectl get ns
prometheus                    Active        33d
redis                         Active        7d5h
redis-ha                      Terminating   31d

# kubectl  get all -n redis-ha              
No resources found.

# kubectl  -n redis-ha  get customresourcedefinitions
No resources found.
```
**没有pod资源和crd资源。**

### 删除方法
```bash
kubectl  get ns redis-ha  -o json > tmp.json
# edit tmp.json and remove"kubernetes"
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json https://kubernetes-cluster-ip/api/v1/namespaces/annoying-namespace-to-delete/finalize

```

导出的json文件为：
```json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "creationTimestamp": "2019-03-13T07:47:46Z",
        "deletionTimestamp": "2019-04-13T10:41:37Z",
        "name": "redis-ha",
        "resourceVersion": "68507398",
        "selfLink": "/api/v1/namespaces/redis-ha",
        "uid": "4aa66241-4564-11e9-b0b0-005056bd5fee"
    },
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
    "status": {
        "phase": "Terminating"
    }
}

删除了：
        "finalizers": [
            "kubernetes"
        ]
```

### 参考
[github issues](https://github.com/kubernetes/kubernetes/issues/60807)
