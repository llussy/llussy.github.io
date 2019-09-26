---
layout: post
title: "Cloudera Manager Server迁移"
date: 2019-01-30 09:38:34 +0800
catalog: ture  
multilingual: false
tags: 
    - cdh
---

> 主要步骤：[Moving the Cloudera Manager Server to a New Host](https://www.cloudera.com/documentation/enterprise/5-15-x/topics/cm_ag_restore_server.html) 官方文档最好啊



### 前提

要备份mysql的数据库



### 主要步骤

1、在新的机器上安装Cloudera Manager 

2、copy `/var/lib/cloudera-scm-server` 到新的机器

3、安装好数据库，还原备份，下载mysql-connector-java-5.1.34.jar ，更新db.properties

4、修改所有的agent的config.ini文件，重启agent

5、停掉之前的server,启动新的server



### 注意事项

- 新的机器安装java环境、创建用户

- 拉取CDH相关软件包（可能不需要）

- 修改hosts文件

- db.properties  **INIT** 改为 **EXTERNAL**

  ```bash
  # The database host
  # If a non standard port is needed, use 'hostname:port'
  com.cloudera.cmf.db.host=10.21.8.61
  
  # The database name
  com.cloudera.cmf.db.name=scm
  
  # The database user
  com.cloudera.cmf.db.user=scm
  
  # The database user's password
  com.cloudera.cmf.db.password=xxxxxx
  
  # The db setup type
  # By default, it is set to INIT
  # If scm-server uses Embedded DB then it is set to EMBEDDED
  # If scm-server uses External DB then it is set to EXTERNAL
  #com.cloudera.cmf.db.setupType=INIT
  com.cloudera.cmf.db.setupType=EXTERNAL
  ```

[Cloudera manager fails to start trying to fetch parcels](https://community.cloudera.com/t5/Cloudera-Manager-Installation/Cloudera-manager-fails-to-start-trying-to-fetch-parcels/td-p/65146)



## Cloudera Management Service

**Cloudera Managerment Service 的迁移**

把Cloudera Managerment Service 服务停止删掉，重新添加就行。添加需要输入cm库相关的数据库用户名密码。



[如何迁移Cloudera Manager节点](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==&mid=2247483970&idx=1&sn=6d847a32e90ab485713f02f0b676110c&chksm=ec2ad24bdb5d5b5d24b51349494d8f3e0a5873a8ea739d283d5eea41d9aac6d96bbcedda5d1f&scene=21#wechat_redirect)

