---
layout: post
title: "linux性能分析"
date: 2020-02-18 12:40:30 +0800
catalog: ture
multilingual: false
tags:
    - linux
---
[toc]

### io相关

```bash
iotop
iostat -x 1 10
dstat  --top-io --top-bio
pidstat -d 1 10
```
