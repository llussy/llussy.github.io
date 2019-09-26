---
layout: post
title: "ceph 架构和概念"
date: 2019-08-17 15:41:30 +0800
catalog: ture  
multilingual: false
tags: 
    - ceph
---
[TOC]

 Ceph是基于`RADOS`无限扩展的分布式存储集群，提供同时包括对象、块设备、和文件的存储服务，拥有高可靠、易于管理、开源等特点。Ceph有着极易扩展的特性——上千数量的客户端可同时访问PB~EB数量级的数据。 

### Ceph架构

  Ceph的整体架构由三个层次组成：**最底层也是最核心的部分是RADOS对象存储系统。第二层是librados库层；最上层对应着Ceph不同形式的存储接口实现。**

![img](https://llussy.github.io/images/Ceph/ceph-rados.png)

#### RADOS

​	从架构图中可以看到最底层的是RADOS（reliable，autonomous，distributed object store），RADOS自身是一个完整的分布式对象存储系统，它具有可靠、智能、分布式等特性，Ceph的高可靠、高可拓展、高性能、高自动化都是由这一层来提供的，用户数据的存储最终也都是通过这一层来进行存储的，RADOS可以说就是Ceph的核心。

​	**RADOS系统主要由两部分组成，分别是OSD和Monitor。**

#### LIBRADOS

​	基于RADOS层的上一层是LIBRADOS，LIBRADOS是一个库，它允许应用程序通过访问该库来与RADOS系统进行交互，支持多种编程语言，比如C、C++、Python等。

#### RADOSGW RBD CEPH FS

​	最上层面向应用提供3种不同的存储接口：

​         l  块存储接口，通过**librbd库**提供了块存储访问接口。RBD通过Linux内核客户端和QEMU/KVM驱动来提供一个分布式的块设备。它可以为虚拟机提供虚拟磁盘，或者通过内核映射为物理主机提供磁盘空间。

​          l  对象存储接口，**RADOSGW**  目前提供了两种类型的API，一种是和AWS的S3接口兼容的API，另一种是和OpenStack的Swift对象接口兼容的API。

​           l  文件系统接口，目前提供两种接口，一种是标准的posix接口，另一种通过**libcephfs库**提供文件系统访问接口。文件系统的元数据服务器MDS用于提供元数据访问。数据直接通过librados库访问。

### Ceph核心组件 

**Ceph的核心组件包括Ceph OSD、Ceph Monitor和Ceph MDS**

#### Ceph OSD

OSD的英文全称是**Object Storage Device**，它的主要功能是存储数据、复制数据、平衡数据、恢复数据等，与其它OSD间进行心跳检查等，并将一些变化情况上报给Ceph Monitor。`一般情况下一块硬盘对应一个OSD`。

##### filestore

伴随OSD的还有一个概念叫做Journal盘，一般写数据到Ceph集群时，都是先将数据写入到Journal盘中，然后每隔一段时间比如5秒再将Journal盘中的数据刷新到文件系统中。一般为了使读写时延更小，Journal盘都是采用SSD，一般分配10G以上，当然分配多点那是更好，Ceph中引入Journal盘的概念是因为Journal允许Ceph OSD能很快做小的写操作；一个随机写入首先写入在上一个连续类型的journal，然后刷新到文件系统，这给了文件系统足够的时间来合并写入磁盘，一般情况下使用SSD作为OSD的journal可以有效缓冲突发负载。(**bluestore中去掉了**)

##### bluestore

bluestore的诞生是为了解决filestore自身维护一套journal并同时还需要基于系统文件系统的写放大问题，并且filestore本身没有对SSD进行优化，因此bluestore相比于filestore主要做了两方面的核心工作：

- 去掉journal，直接管理裸设备

- 针对SSD进行单独优化

#### Ceph Monitor

Ceph Monitor：由该英文名字我们可以知道它是一个监视器，负责监视Ceph集群，维护Ceph集群的健康状态，同时维护着Ceph集群中的各种Map图，比如OSD Map、Monitor Map、PG Map和CRUSH Map，这些Map统称为Cluster Map，Cluster Map是`RADOS`的关键数据结构，管理集群中的所有成员、关系、属性等信息以及数据的分发，比如当用户需要存储数据到Ceph集群时，OSD需要先通过Monitor获取最新的Map图，然后根据Map图和object id等计算出数据最终存储的位置。

查看各种Map的信息可以通过如下命令：ceph osd(mon、pg) dump

##### map

- **Monitor map**：包括有关monitor 节点端到端的信息，其中包括 Ceph 集群ID，监控主机名和IP以及端口。并且存储当前版本信息以及最新更改信息，通过 "ceph mon dump" 查看 monitor map。
- **OSD map**：包括一些常用的信息，如集群ID、创建OSD map的 版本信息和最后修改信息，以及pool相关信息，主要包括pool 名字、pool的ID、类型，副本数目以及PGP等，还包括数量、状态、权重、最新的清洁间隔和OSD主机信息。通过命令 "ceph osd dump" 查看。
- **PG map**：包括当前PG版本、时间戳、最新的OSD Map的版本信息、空间使用比例，以及接近占满比例信息，同事，也包括每个PG ID、对象数目、状态、OSD 的状态以及深度清理的详细信息。通过命令 "ceph pg dump" 可以查看相关状态。
- **CRUSH map**： CRUSH map 包括集群存储设备信息，故障域层次结构和存储数据时定义失败域规则信息。通过 命令 "ceph osd crush map" 查看。
- **MDS map**：MDS Map 包括存储当前 MDS map 的版本信息、创建当前的Map的信息、修改时间、数据和元数据POOL ID、集群MDS数目和MDS状态，可通过"ceph mds dump"查看。

#### Ceph MDS

Ceph MDS：全称是Ceph MetaData Server，主要保存的文件系统服务的元数据，但对象存储和块存储设备是不需要使用该服务的。

### Ceph数据存储过程

Ceph存储集群从客户端接收文件，每个文件都会被客户端切分成一个或多个对象，然后将这些对象进行分组，再根据一定的策略存储到集群的OSD节点中，其存储过程如图所示：

![image-20190623224741004](/Users/lisai/go/src/llussy.github.io/images/image-20190623224741004.png)

图中，对象的分发需要经过两个阶段的计算，才能得到存储该对象的OSD，然后将对象存储到OSD中对应的位置。

(1)  **对象到PG的映射。**PG(PlaccmentGroup)是对象的逻辑集合。PG是系统向OSD节点分发数据的基本单位，相同PG里的对象将被分发到相同的OSD节点中(一个主OSD节点多个备份OSD节点)。对象的PG是由对象ID号通过Hash算法，结合其他一些修正参数得到的。

(2)  **PG到相应的OSD的映射。**RADOS系统利用相应的哈希算法根据系统当前的状态以及PG的ID号，将各个PG分发到OSD集群中。OSD集群是根据物理节点的容错区域(比如机架、机房等)来进行划分的。

Ceph中的OSD节点将所有的对象存储在一个没有分层和目录的统一的命名空间中。**每个对象都包含一个ID号、若干二进制数据以及相应的元数据。**

ID号在整个存储集群中是唯一的；元数据标识了所存储数据的属性。一个对象在OSD节点中的存储方式大致如图所示。

![image-20190623224819249](https://llussy.github.io/images/image-20190623224819249.png)

而对存储数据的语义解释完全交给相应的客户端来完成，比如，Ceph FS客户端将文件元数据(比如所有者、创建日期、修改日期等)作为对象属性存储在Ceph中。



### CRUSH 算法

**Controlled Replication Under Scalable Hashing 可扩展哈希算法的可控复制**

**PG到OSD的映射的过程算法叫做crush 算法**

​	Ceph作为一个高可用、高性能的对象存储系统，其数据读取及写入方式是保证其高可用性及高性能的重要手段。对于已知的数据对象，Ceph通过使用CRUSH(ControlledReplication Under Scalable Hashing)算法计算出其在Ceph集群中的位置，然后直接与对应的OSD设备进行交互，进行数据读取或者写入。

#### 写入数据

例如其写入数据的其主要过程如图所示。

![image-20190623225350763](https://llussy.github.io/images/image-20190623225350763.png)

​	首先，客户端获取Ceph存储系统的状态信息Cluster Map，然后根据状态信息以及将要写入的Pool的CRUSH相关信息，获取到数据将要写入的OSD，最后OSD将数据写入到其中相应的存储位置。其中相关概念的解释如下：

​	(1)   集群地图(Cluster Map)：Ceph依赖于客户端以及OSD进程中保存有整个集群相关的拓扑信息，来实现集群的管理和数据的读写。整个集群相关的拓扑信息就称之为“Cluster Map”。Cluster Map主要保存Monitor集群、OSD集群、MDS集群等相关的拓扑结构信息以及状态信息。

​	(2)   存储池(pool)：是对Ceph集群进行的逻辑划分，主要设置其中存储对象的权限、备份数目、PG数以及CRUSH规则等属性。

​	在传统的存储系统中，要查找数据通常是依赖于查找系统的的文件索引表找到对应的数据在磁盘中的位置。而在Ceph对象存储系统中，客户端与OSD节点都使用CRUSH算法来高效的计算所存储数据的相关信息。相对于传统的方式，CRUSH提供了一种更好的数据管理机制，它能够将数据管理的大部分工作都分配给客户端和OSD节点，这样为集群的扩大和存储容量的动态扩展带来了很大的方便。CRUSH是一种伪随机数据分布算法，它能够在具有层级结构的存储集群中有效的分发对象副本。

​	CRUSH算法是根据集群中存储设备的权重来进行数据分发的，数据在各个OSD设备上近似均匀概率分布。CRUSH中，数据在存储设备上的分布是根据一个层次化的集群地图(Cluster Map)来决定的。集群地图是由可用的存储资源以及由这些存储资源构建的集群的逻辑单元组成。比如一个Ceph存储集群的集群地图的结构可能是一排排大型的机柜，每个机柜中包含多个机架，每个机架中放置着存储设备。数据分发策略是依照数据的存放规则(placement rules)进行定义的，存放规则是指数据在备份以及存放时应该遵循的相关约定，比如约定一个对象的三个副本应该存放在三个不同的物理机架上。

给定一个值为x的整数，CRUSH将根据相应的策略进行哈希计算输出一个有序的包含n个存储目标的序列：

CRUSH(x)=(osd1,osd2,osd3,osdn)

CRUSH利用健壮的哈希函数，其得到的结果依赖于集群地图Cluster Map、存放规则(placementmles)和输入x。并且CRUSH是一个伪随机算法，两个相似的输入得到的结果是没有明显的相关性的。这样就能确保Ceph中数据分布是随机均匀的。

​	在分布式存储系统中比较关注的一点是如何使得数据能够分布得更加均衡，常见的数据分布算法有一致性Hash和Ceph的Crush算法。**Crush是一种伪随机的控制数据分布、复制的算法**，Ceph是为大规模分布式存储而设计的，数据分布算法必须能够满足在大规模的集群下数据依然能够快速的准确的计算存放位置，同时能够在硬件故障或扩展硬件设备时做到尽可能小的数据迁移，Ceph的CRUSH算法就是精心为这些特性设计的，可以说CRUSH算法也是Ceph的核心之一。

在说明CRUSH算法的基本原理之前，先介绍几个概念和它们之间的关系。

**存储数据与object的关系**：

​	当用户要将数据存储到Ceph集群时，存储数据都会被分割成多个`object`，每个object都有一个object id，每个object的大小是可以设置的，默认是4MB，`object可以看成是Ceph存储的最小存储单元`。

**object与pg的关系：**

​	由于object的数量很多，所以Ceph引入了pg的概念用于管理object，每个object最后都会通过hash计算映射到某个pg中，**一个pg可以包含多个object**。

**pg与osd的关系：**

​	pg也需要通过CRUSH计算映射到osd中去存储，如果是二副本的，则每个pg都会映射到二个osd，比如[osd.1,osd.2]，那么osd.1是存放该pg的主副本，osd.2是存放该pg的从副本，保证了数据的冗余。

**pg和pgp的关系：**

​	pg是用来存放object的，pgp相当于是pg存放osd的一种排列组合，我举个例子，比如有3个osd，osd.1、osd.2和osd.3，副本数是2，如果pgp的数目为1，那么pg存放的osd组合就只有一种，可能是[osd.1,osd.2]，那么所有的pg主从副本分别存放到osd.1和osd.2，如果pgp设为2，那么其osd组合可以两种，可能是[osd.1,osd.2]和[osd.1,osd.3]，是不是很像我们高中数学学过的排列组合，pgp就是代表这个意思。一般来说应该将pg和pgp的数量设置为相等。[一个实验](https://www.cnblogs.com/luohaixian/p/8087591.html)

通过实验总结：
（1）PG是指定存储池存储对象的目录有多少个，PGP是存储池PG的OSD分布组合个数
（2）PG的增加会引起PG内的数据进行分裂，分裂相同的OSD上新生成的PG当中
（3）PGP的增加会引起部分PG的分布进行变化，但是不会引起PG内对象的变动

**pg和pool的关系**：pool也是一个逻辑存储概念，我们创建存储池pool的时候，都需要指定pg和pgp的数量，逻辑上来说pg是属于某个存储池的，就有点像object是属于某个pg的。

以下这个图表明了存储数据，object、pg、pool、osd、存储磁盘的关系

![ceph](https://llussy.github.io/images/Ceph/ceph-storage.png)

 	本质上CRUSH算法是根据存储设备的权重来计算数据对象的分布的，权重的设计可以根据该磁盘的容量和读写速度来设置，比如根据容量大小可以将1T的硬盘设备权重设为1，2T的就设为2，在计算过程中，CRUSH是根据Cluster Map、数据分布策略和一个随机数共同决定数组最终的存储位置的。

​	Cluster Map里的内容信息包括存储集群中可用的存储资源及其相互之间的空间层次关系，比如集群中有多少个支架，每个支架中有多少个服务器，每个服务器有多少块磁盘用以OSD等。

​	数据分布策略是指可以通过Ceph管理者通过配置信息指定数据分布的一些特点，比如管理者配置的故障域是Host，也就意味着当有一台Host起不来时，数据能够不丢失，CRUSH可以通过将每个pg的主从副本分别存放在不同Host的OSD上即可达到，不单单可以指定Host，还可以指定机架等故障域，除了故障域，还有选择数据冗余的方式，比如副本数或纠删码。

### 参考

[Ceph整体架构](http://www.voidcn.com/article/p-njiezowl-beg.html)

[ceph分布式存储](https://www.cnblogs.com/yangxiaoyi/p/7795274.html)

[Ceph基础知识和基础架构认识](https://www.cnblogs.com/luohaixian/p/8087591.html)

[Ceph分布式存储系统](<https://blog.51cto.com/tasnrh/1761437>)


