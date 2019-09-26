---
layout: post
title: "Kubernetes 日志收集"
date: 2019-07-08 12:08:34 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
---

[TOC]

日志流： **filebeat --> kafka--> logstash-->es**

k8s应用日志输出到标准输出，最好是json格式日志。

### filebeat

**关键配置**

1. **output.kafka** `topic: 'k8s-%{[kubernetes.namespace]}'`  按照ns打入topic
2. filebeat 传输meta信息落盘。 **pod** `/usr/share/filebeat/data` 目录 挂载到宿主机`/data/filebeat`目录

### kafka

正常kafka集群即可，只做日志的话，可以将副本数设置为1。

### logstash

消费kafka数据，打入不同es的index （按namespace）

**output关键配置**

```index => "k8s-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"```

### es

动态调试索引模板。

