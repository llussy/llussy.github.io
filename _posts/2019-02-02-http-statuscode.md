---
layout: post
title: "http 状态码"
date: 2019-02-02 09:50:34 +0800
catalog: ture
multilingual: false
tags:
    - linux
    - http
---



### http状态码分类

| 分类 | 分类描述                                       |
| ---- | ---------------------------------------------- |
| 1**  | 信息，服务器收到请求，需要请求者继续执行操作   |
| 2**  | 成功，操作被成功接收并处理                     |
| 3**  | 重定向，需要进一步的操作以完成请求             |
| 4**  | 客户端错误，请求包含语法错误或无法完成请求     |
| 5**  | 服务器错误，服务器在处理请求的过程中发生了错误 |

### 常见2xx状态码

#### 204
```
If the client is a user agent, it SHOULD NOT change its document view from that which caused the request to be sent. This response is primarily intended to allow input for actions to take place without causing a change to the user agent’s active document view, although any new or updated metainformation SHOULD be applied to the document currently in the user agent’s active view.
```
意思等同于请求执行成功，但是没有数据，浏览器不用刷新页面.也不用导向新的页面。如何理解这段话呢。还是通过例子来说明吧，假设页面上有个form，提交的url为http-204.htm，提交form，正常情况下，页面会跳转到http-204.htm，但是如果http-204.htm的相应的状态码是204，此时页面就不会发生转跳，还是停留在当前页面。另外对于a标签，如果链接的页面响应码为204，页面也不会发生跳转。

### 常见3xx状态码

#### 304
304状态码或许不应该认为是一种错误，而是对客户端有缓存情况下服务端的一种响应。
自从上次请求后，请求的网页未修改过。服务器返回此响应时，不会返回网页内容。
客户端在请求一个文件的时候，发现自己缓存的文件有 Last Modified ，那么在请求中会包含 If Modified Since ，这个时间就是缓存文件的 Last Modified 。因此，如果请求中包含 If Modified Since，就说明已经有缓存在客户端。服务端只要判断这个时间和当前请求的文件的修改时间就可以确定是返回 304 还是 200 。


### 常见4xx状态码

#### 401  Unauthorized

表示发送的请求需要有HTTP认证信息或者是认证失败了

#### 403  Forbidden

表示对请求资源的访问被服务器拒绝了

#### 404  Not Found

表示服务器找不到你请求的资源



### 常见5xx状态码

#### 502  Bad Gateway

作为网关或者代理工作的服务器尝试执行请求时，从远程服务器接收到了一个无效的响应。

产生原因： 连接超时 我们向服务器发送请求 由于服务器当前连接太多，导致服务器方面无法给于正常的响应,产生此类报错。

#### 503  Service Unavailable

由于超载或系统维护，服务器暂时的无法处理客户端的请求。延时的长度可包含在服务器的Retry-After头信息中。

#### 504  Gateway Timeout

服务器作为网关或代理，但是没有及时从上游服务器收到请求。





### 一些文章

#### [Nginx 502错误触发条件与解决办法汇总](http://os.51cto.com/art/201011/233698.htm)

#### [502 Bad Gateway 怎么解决？](https://www.zhihu.com/question/21647204)

#### [服务器返回的14种常见HTTP状态码](https://blog.csdn.net/q1056843325/article/details/53147180)

