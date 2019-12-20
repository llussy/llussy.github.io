---
layout: post
title: "prometheus-pushgateway使用"
date: 2019-12-20 12:40:30 +0800
catalog: ture
multilingual: false
tags:
    - prometheus
---

[toc]

### Pushgateway介绍



[Pushgateway](https://github.com/prometheus/pushgateway) 是 Prometheus 生态中一个重要工具，使用它的原因主要是：

- Prometheus 采用 pull 模式，可能由于不在一个子网或者防火墙原因，导致 Prometheus 无法直接拉取各个 target 数据。

- 在监控业务数据的时候，需要将不同数据汇总, 由 Prometheus 统一收集。

​		Pushgateway并不是将Prometheus的pull改成了push，它只是允许用户向他推送指标信息，并记录。而Prometheus每次从 Pushgateway拉取的数据并不是期间用户推送上来的所有数据，而是最后一次push上来的数据。所以设置推送时间与Prometheus拉取的时间相同(<=)一般是较好的方案。

### prometheus增加pushgateway

```bash
  - job_name: pushgateway
    static_configs:
      - targets: ['192.168.91.132:9091']
        labels:
          instance: pushgateway
```

### 数据管理

正常情况我们会使用 Client SDK 推送数据到 pushgateway, 但是我们还可以通过 API 来管理。

#### shell脚本

shell脚本 --data-binary 表示发送二进制数据，注意：它是使用POST方式发送的！ 
**增加数据**

```bash
cat <<EOF | curl --data-binary @- http://pushgateway.test.com/metrics/job/some_job/instance/some_instance
ifeng_a_some_metric{label="test",IDC="SYQ"} 100
EOF

注意：必须是指定的格式才行！ 
pushgateway 中的数据我们通常按照 job 和 instance 分组分类，所以这两个参数尽量写全


echo "some_metric 3.14" | curl --data-binary @- http://pushgateway.example.org:9091/metrics/job/some_job


cat <<EOF | curl --data-binary @- http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance
# TYPE some_metric counter
some_metric{label="val1"} 42
# TYPE another_metric gauge
# HELP another_metric Just an example.
another_metric 2398.283
EOF
```

**删除数据**

```bash
#删除某个组下的某实例的所有数据：
curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance

#删除某个组下的所有数据：
curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job
```

#### ps aux推送到pushgateway

```bash
If you want the same script for memory usage, simply change the ‘cpu_usage’ label to ‘memory_usage’ and the $3z to $4z

vim better-top 
#!/bin/bash
z=$(ps aux)
while read -r z
do
   var=$var$(awk '{print "cpu_usage{process=\""$11"\", pid=\""$2"\"}", $3z}');
done <<< "$z"
curl -X POST -H  "Content-Type: text/plain" --data "$var
" http://localhost:9091/metrics/job/top/instance/machine


while sleep 1; do ./better-top; done;
```



### 参考

[pushgateway](https://www.cnblogs.com/xiao987334176/p/9933963.html)

[Monitoring Linux Processes using Prometheus and Grafana](https://medium.com/schkn/monitoring-linux-processes-using-prometheus-and-grafana-113b3e271971)