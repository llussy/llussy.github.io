---
layout: post
title: "prometheus查询"
date: 2020-02-04 16:40:30 +0800
catalog: ture
multilingual: false
tags:
    - prometheus
---

[toc]

### PromQL

#### matching

```bash
= != =~ !~ * |
```

#### function

```bash
sum() min() max() avg() count() rate()
```

#### query

```bash
http_requests_total{code="200"}
# 正则
prometheus_http_requests_total{code!="200"} // 表示查询 code 不为 "200" 的数据
prometheus_http_requests_total{code=～"2.."} // 表示查询 code 为 "2xx" 的数据
prometheus_http_requests_total{code!～"2.."} // 表示查询 code 不为 "2xx" 的数据

rate(prometheus_http_requests_total{code!="200"}[5m])    #5分钟内，平均每秒数据

by（） # 分组 好像sum后面才能跟by()

sum(rate(traefik_backend_requests_total{code=~"4.."}[5m])) by(backend) / sum(rate(traefik_backend_requests_total[5m])) by(backend)  # 4xx比例

#pod cpu
sum(rate(container_cpu_usage_seconds_total{namespace="monitoring",pod_name="prometheus-operator-prometheus-node-exporter-d5gpm"}[5m])) by (pod_name)

#pod mem
sum(rate(container_memory_working_set_bytes{namespace="monitoring",pod_name="prometheus-operator-prometheus-node-exporter-d5gpm"}[5m])) by (pod_name)
```

#### rate irate

irate和rate都会用于计算某个指标在一定时间间隔内的变化速率。但是它们的计算方法有所不同：irate取的是在指定时间范围内的最近两个数据点来算速率，而rate会取指定时间范围内所有数据点，算出一组速率，然后取平均值作为结果。

### Prometheus API

<https://prometheus.io/docs/prometheus/latest/querying/api/>

```bash
curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
```

### 参考

[PromQL](https://www.cnblogs.com/gschain/p/11697281.html)
