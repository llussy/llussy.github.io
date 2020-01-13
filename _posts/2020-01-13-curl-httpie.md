---
layout: post
title: "curl和httpie使用"
date: 2020-01-14 16:40:30 +0800
catalog: ture
multilingual: false
tags:
    - linux
    - shell
---

### curl

#### parameter

```bash
-I --head  								输出head信息
-x host:port  						使用HTTP代理访问；如果未指定端口，默认使用8080端口
-X 												指定请求方法
-d, --data 								指定请求数据
-H --header    						自定义请求头
-s 												减少输出的信息
-w, --write-out FORMAT  	完成后输出什么
-u	user:passwd						指定用户名
```

#### API test

```bash
curl -I -o /dev/null -s -w "http_code:%{http_code}\ntime_dns:%{time_namelookup}\ntime_total:%{time_total}\n" http://www.baidu.com

curl -sL -w "%{http_code}" "www.baidu.com" -o /dev/null

curl www.baidu.com -x10.21.8.64:8118 -XGET 

# while
while true; do curl -sL -w "%{http_code}" "www.baidu.com" -o /dev/null;sleep 0.5;echo "";done

# for
for i in {1..1000}
do
 sleep 0.5
 curl -sL -w "%{http_code}" "www.baidu.com" -o /dev/null
 echo " "
done
```

#### header

```bash
curl -H 'Content-Type:application/json' -H 'Authorization: bearer eyJhbGciOiJIUzI1NiJ9' itbilu.com
```

#### http2

```bash
curl --http2 -I https://nghttp2.org
-k, --insecure      Allow insecure server connections when using SSL
-v, --verbose       Make the operation more talkative
```

#### download

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.10.4/bin/linux/amd64/kubectl
```

#### submit data

```bash
curl -XPOST -d'{"username":"someone","password":"p@ssword"}' localhost:8080/login

curl -u admin:xxxxxxxx -XPUT 'http://es.cloud.com:9200/_template/layer7-template?pretty' -H 'Content-Type: application/json' -d'
{
    "template" : "layer7*",
    "order": 100,
    "settings" : {
        "index.number_of_shards" : 30,
        "number_of_replicas" : 1,
        "routing.allocation.total_shards_per_node": 8,
        "refresh_interval" : "60s"
    }
}'
```

#### Specify the certificate

```bash
curl --cacert /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem https://10.21.8.24:2379/metrics
```



### HTTPie

[github](https://github.com/jakubroztocil/httpie)

```bash
http HEAD www.baidu.com  or  http -h www.baidu.com  # curl -I 

# 自定义HTTP方法，HTTP标头和JSON数据
http PUT example.org X-API-Token:123 name=John

# 提交表格
http -f POST example.org hello=World

#download
http --download example.org/file

http -v --json POST localhost:8081/login username=admin password=admin

http -v -f GET localhost:8081/auth/refresh_token "Authorization:Bearer xxxtokenxxx"  "Content-Type: application/json"

http -f GET localhost:8081/auth/hello "Authorization:Bearer xxxtokenxxx"  "Content-Type: application/json"
```



### reference

[curl](https://itbilu.com/linux/man/4yZ9qH_7X.html)
