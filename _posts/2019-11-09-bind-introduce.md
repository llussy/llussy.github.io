---
layout: post
title: "bind介绍"
date: 2019-11-08 21:31:30 +0800
catalog: ture  
multilingual: false
tags: 
    - linux
---
[toc]

### DNS介绍

​		DNS（Domain Name System，域名服务系统）是Internet上用得最频繁的服务之一，它是一个分布式数据库，组织成域层次结构的计算机和网络服务命名系统。通过它人们可以将域名解析为IP地址，从而使人们能够通过简单好记的域名来代替枯燥难记的IP地址来访问网络。

**域名的正向解析**

将主机域名转换为对应的IP地址，以便网络程序能够通过主机域名访问到对应的服务器主机

**域名的反向解析**

将主机的IP地址转换为对应的域名，以便网络（服务）程序能够通过IP地址查询到主机的域名



DNS服务采用树型层次结构,全世界的DNS服务器具有共同的根域（.）

对域名的查询是分层次进行的,对域名www.sina.com.cn域名的解析需要依次经过：

根(.)域的DNS服务器 

"cn."域的DNS服务器 

"com.cn."域的DNS服务器

"sina.com.cn."域的DNS服务器 



**协议使用端口**
udp 53   正常查询解析情况下使用udp53
tcp53    当进行主从之间的区域传送时使用tcp53



#### 域的空间划分

![img](https://llussy.github.io/images/0.2781905767043533.png)

#### 查询方式

##### 递归查询

当主机A要向DNS服务器发送查询主机D的请求时，服务器返回给A最终结果，这种方式就是递归查询，如果客户端要查找的内容直接在服务器上得到结果，刚给出的答案是一个权威答案，否则就是一个参考答案。

##### 迭代查询

NS服务器接收到A的请求后，本地没有D的解析，则会通过以下过程获得

1、NS向根域询问D，根域让他去找一级域.com

2、NS向一级域.com询问D，.com让他去找二级域.baidu

3、NS向二级域.baidu询问D，.baidu返回D的结果，NS获得D的解析

以上过程则为迭代查询

![image-20191016211950436](https://llussy.github.io/images/image-20191016211950436.png)

#### 资源记录

资源记录是DNS数据库中用于答复客户端的条目，资源记录的格式一般如下：

```bash
Name   [ttl]   IN   RRtype   Value
RRtype是指资源类型，常见的资源类型有SOA、A、AAAA、CNAME、MX、NS、PTR 
```

| 资源类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| SOA      | 是指定区域的起点，一般包含区域名、区域管理员的电子邮箱以及设置从服务器如何更新区域数据文件的信息 |
| A        | 主机记录，将域名映射为ipv4地址                               |
| AAAA     | 主机记录，将域名映射为ipv6地址                               |
| CNAME    | 别名记录，可为一个域名其一个别名，达到一个ip地址对应两个域名的效果 |
| MX       | 邮件记录，列出了域中的邮件服务器，需要设置优先级             |
| NS       | 指定负责域中域名解析的服务器                                 |
| PTR      | 与A记录相反，将IP地址映射成域名                              |

### bind

#### 主配置文件

Bind的主配置文件是/etc/named.conf，该文件只包括Bind的基本配置，并不包含任何DNS区域数据。 

在主配置文件中主要可分为两部分

全局声明（options）

options语句在每个配置文件中只有一处。如果出现多个options,则第一个options的配置有效，并会产生一个警告信息。

区域声明（zone）

zone语句作用是定义DNS区域，在此语句中可定义DNS区域选项。

当DNS服务器处理递归查询时，如果本地区域文件不能进行查询的解析，就会转到根DNS服务器查询，所以在主配置文件named.conf文件中还要定义根区域。

```bash
	zone "." { 
		type hint;	
		file "named.ca"; 
	};
	
# type:设置域的类型
# file:设置根服务列表文件名
```

**/etc/named/conf简单介绍**

```bash
//设置全局配置
options {
        listen-on port 53 { 192.168.10.1; };    //设置DNS服务器监听的ipv4地址和>端口
        listen-on-v6 port 53 { ::1; };          //设置DNS服务器监听的ipv6地址和>端口
        directory       "/var/named";           //设置区域文件存放的目录
        dump-file       "/var/named/data/cache_dump.db";        
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { localhost; };         //指定允许进行查询操作的主机
        allow-transfer  {none;};                        //指定允许接受区域传送的从服务器
        allow-recursion {aqlist;};                //指定允许为哪些主机进行递归查询
        
logging {
        channel default_debug {
                file "data/named.run";          //指定日志
                severity dynamic;
        };
};
acl aqlist {                //配置acl
  202.0.0.0/8; 
  221.0.0.0/8; 
};
//设置局部配置内容
zone "." IN {
        type hint;      //指定区域类型
        file "named.ca";        //指定该区域的区域文件名称
};
zone "test.com." IN {
        type master;
        file "test.com.zone";
};
zone "10.168.192.in-addr.arpa" IN {
        type master;
        file "10.168.192.in-addr.local";
};
```

**type: 用于定义区域类型，此时只有一个DNS服务器，所以为master，type可选值为：hint(根的)|master(主的)|slave(辅助的)|forward(转发)**

**file：用于定义区域数据文件路径，默认该文件保存在/var/named/目录。**



#### 正向区域文件

```bash
vim /var/named/test.com.zone

$TTL 38400
@       IN      SOA     ns.test.com.    admin.test.com. (
                        2016082601      #序列号，不能超过10位
                        2H           #指定从服务器每隔多久到主服务器上刷新一次
                        4M            #刷新失败时，多久重试
                        1D          #主服务器down后，从服务器能继续工作多久
                        2D           #否定答案的ttl
）
@       IN      NS      ns.test.com.
ns      IN      A       192.168.10.1
@       IN      MX  90  mail.test.com.
mail    IN      A       192.168.10.3
```

$TTL为定义的宏，表示下面资源记录ttl的值都为38400秒。

@符号可代表区域文件/etc/named.conf里面定义的区域名称，即："test.com."。

每个区域的资源记录第一条必须是SOA，SOA后面接DNS服务器的域名和电子邮箱地址，此处电子邮箱地址里的@因为有特殊用途，所以此处要用点号代替。

#### 反向区域文件

```bash
vim /var/named/10.168.192.in-addr.local

$TTL 38400
@       IN      SOA     ns.test.com.    admin.test.com. (
                        2016082601
                        10800
                        3600
                        604800
                        38400
）
@       IN      NS      ns.test.com.
1       IN      PTR     ns.test.com.
3       IN      PTR     mail.test.com.
```



#### check command

```bash
# 检查配置文件的语法是否正确
named-checkconf /etc/named.conf

# 对区域文件进行检查 语法格式：named-checkzone 域名 区域文件
named-checkzone test.com /var/named/test.com.zone

```



#### DNS主从同步

从服务器复制主服务器的区域文件，与主服务器一起提供地址解析功能;从服务器上的资源记录只能读取，不能修改和删除。

**从服务器**

```bash
vim /etc/named.conf

zone "test.com." IN {
        type slave;
        masters { 192.168.10.1; };
        file "slave/test.com.zone";
};
zone "10.168.192.in-addr.arpa" IN {
        type slave;
        masters { 192.168.10.1; };
        file "slave/10.168.192.in-addr.local";
};
```

**主服务器**

1、 为 从服务器添加一条NS记录和对应的A或PTR记录。

2、编辑/etc/named.conf 

```bash
    allow-transfer{ 192.168.10.2; };
```

#### 转发DNS服务器

服务器收到客户端的解析请求后，将请求转发给其他DNS服务器进行解析，称为转发

转发服务器分为完全转发和条件转发

完全转发：服务器会将全部的请求都转发给其他服务器进行解析

编辑/etc/named.conf

```bash
forwarders { 192.168.0.30; };    \\转发给192.168.0.30进行解析
```

条件转发：当客户端请求特定域的解析时才进行转发查询

编辑/etc/named.conf

```bash
zone “lzs.com" IN {            \\对lzs.com域的查询请求进行转发
    type forward;        \\类型为forward
    forwarders { 192.168.0.30; };    \\转发给192.168.0.30进行解析
    };
```

### 参考

[DNS协议原理、安装及主从同步、负载均衡和转发、缓存的详细配置](https://www.tuicool.com/articles/Rna6V3F)

[利用dns解析来实现网站的负载均衡](https://segmentfault.com/a/1190000002578457)

[负载均衡之DNS域名解析](https://blog.csdn.net/cywosp/article/details/38017027)

