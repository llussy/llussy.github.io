---
layout: post
title: "CentOS7利用systemctl添加自定义系统服务"
date: 2019-02-02 15:45:34 +0800
catalog: ture  
multilingual: false
tags: 
    - linux
---
​	



​	CentOS7的服务`systemctl`脚本存放在:/usr/lib/systemd/,有系统（system）和用户（user）之分,需要开机不登陆就能运行的程序，存在系统服务里，即：`/usr/lib/systemd/system`目录下.

​	CentOS7的每一个服务以.service结尾，`权限为754 `一般会分为3部分：[Unit]、[Service]和[Install]

##### **[Unit]**

[Unit]部分主要是对这个服务的说明，内容包括Description和After，Description用于描述服务，After用于描述服务类别

##### **[Service]**

[Service]部分是服务的关键，是服务的一些具体运行参数的设置

- Type=forking是后台运行的形式

- PIDFile为存放PID的文件路径

- ExecStart为服务的具体运行命令

- ExecReload为重启命令

- ExecStop为停止命令

- PrivateTmp=True表示给服务分配独立的临时空间

  

  注意：[Service]部分的启动、重启、停止命令全部要求使用绝对路径，使用相对路径则会报错！

##### **[Install]**

[Install]部分是服务安装的相关设置，可设置为多用户的

##### tomcat例子

```shell
# cat /usr/lib/systemd/system/tomcat.service
[Unit]  
Description=tomcatapi  
After=network.target  
   
[Service]  
Type=forking  
ExecStart=/usr/local/soft/tomcat/tomcat8/bin/startup.sh  
ExecReload=  
ExecStop=/usr/local/soft/tomcat/tomcat8/bin/shutdown.sh  
PrivateTmp=true  
   
[Install]  
WantedBy=multi-user.target  
```

##### nginx例子

```bash
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```



**参考**

[在CentOS 7上利用systemctl添加自定义系统服务](https://blog.csdn.net/yuanguozhengjust/article/details/38019923 )
[CentOS 7 systemd添加自定义系统服务](https://my.oschina.net/liucao/blog/470458)


