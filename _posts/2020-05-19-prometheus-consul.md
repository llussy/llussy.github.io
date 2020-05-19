---
layout: post
title: "prometheus结合consul"
date: 2020-05-19 09:32:30 +0800
catalog: ture
multilingual: false
tags:
    - prometheus
---

> promethues结合consul可以实现自动发现,可以把一些需要收集的信息(metrics)上报到consul,prometheus会自动去consul中拉取。

### 启动一个consul
```bash
# 这里只作测试,线上确保consul高可用
consul agent -server -dev -ui -data-dir /data/consul -client=192.168.1.1 -bind=192.168.1.1 -config-dir /etc/consul.d -enable-script-checks=true
```
### prometheus配置
```bash
- job_name: consul_collect
  metrics_path: /metrics
  scheme: http
  consul_sd_configs:
    - server: 192.168.1.1:8500
      services:
        - consul_test
```
### consul创建service
```bash
# 写入配置文件
echo '{"service": {"name": "consul_test", "tags": ["exporter"], "port": 9100}}' \
    | sudo tee /etc/consul.d/consul_test.json
# 通过接口
curl -X PUT -d '{"id": "node-192.168.1.2","name": "consul_test","address": "192.168.1.2","port": 9100,"tags": ["exporter"],"checks": [{"http": "http://192.168.1.2:9100/metrics","interval": "35s"}]}' http://192.168.1.1:8500/v1/agent/service/register
```

![consul](https://llussy.github.io/images/WX20200519-091927.png)



