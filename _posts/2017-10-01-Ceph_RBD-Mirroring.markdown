---
layout: post
navigation: "存储"
category: "Ceph"
title:  "RBD Mirroring - 原理、概念、命令"
tags: [Ceph,RBD,Mirroring]
---

#RBD Mirroring - 原理、概念、命令

​        Ceph采用的是强一致性同步模型，所有副本都必须完成写操作才算一次写入成功，这就导致不能很好地支持跨域部署，因为如果副本在异地，网络延迟就会很大，拖垮整个集群的写性能。因此，Ceph集群很少有跨域部署的，也就缺乏异地容灾。

​        Mirror通过柯林斯词典翻译"镜子、写照、映照、反射"等含义，在ceph集群语音中，mirror翻译为备份更合适一些。在本文中，mirror或者直接采用英文，或者直接翻译为“备份”，不当之处，请指正。

​        RBD Mirror是Ceph Jewel版本引入的新功能，因此**RBD Mirror要求Ceph版本为Jewel或后续版本。**

​        RBD Mirror基于RBD image在两个集群间进行数据同步，其是**基于Journanling Feature来实现的**。

​        RBD Mirror是在集群间实现数据同步，在伙伴的两个集群都需要部署mirror守护进程。**需要说明的是jewel版本中还是一对一的方式。** *在Luminous版本是一对多模式*

注：

​	**如果要在两个集群间配置RBD Mirroring，那么每个集群都要运行RBD-mirror守护线程**



## 原理

​        Ceph RBD mirror原理其实和mysql的主从同步原理非常类似，简单地说就是通过日志进行回放(replay)。

​        RBD mirror必须依赖于journaling特性，且需要额外部署rbd-mirror服务：

![image](flow/rbd-mirror.jpg)

​        再来了解一下Ceph的journal机制（此处的journal是指Ceph RBD的journal，而不是OSD的journal）。

​        当RBD Journal功能打开后，所有的数据更新请求会先写入RBD Journal，然后后台线程再把数据从Journal区域刷新到对应的image区域。RBD journal提供了比较完整的日志记录、读取、变更通知以及日志回收和空间释放等功能，可以认为是一个分布式的日志系统。

![image](flow/rbd-mirror_syly.jpg) 

具步骤如下：

~~~
1. I/O会写入主集群的Image Journal；
2. Journal写入成功后，RBD再把数据写入主集群回复响应；
3. 备份集群的mirror进程发现主集群的Journal有更新后，从主集群的Journal读取数据，写入备份集群；
4. 备份集群写入成功后，会更新主集群Journal中的元数据，表示该I/O的Journal已经同步完成；
5. 主集群会定期检查，删除已经写入备份集群的Journal数据。
~~~

​      RBD Mirroring 依赖两个新的rbd的属性:

~~~
journaling: 启动后会记录image的事件
mirroring: 明确告诉rbd-mirror需要复制这个镜像
~~~

​       需要注意的是，一旦同步出现脑裂情况，rbd-mirror将中止同步操作，此时你必须手动决定哪边的image是有用的，然后通过手动执行rbd mirror image resync命令恢复同步。

优点：

~~~
1. 减少副本在异地间同步的读写延迟。
2. 增强可靠性
~~~

缺点：

~~~
需要解决脑裂情况下的rbd mirror中止同步的问题。
~~~



## 相关概念

### rbd-mirror进程

​	rbd-mirror负责监控远程伙伴集群的镜像日志，并重放日志事件到本地。RBD Image journaling特性会顺序的记录发生的修改事件。可以保证image和备份的crash-consistent。

​        根据备份方式的不同，rbd-mirror可以在单个集群上或者伙伴的两个集群上运行：

单向备份

~~~
当数据从主集群备份到备用的集群的时候，rbd-mirror仅在备份群集上运行
~~~

双向备份

~~~
如果两个集群互为备份的时候，rbd-mirror需要在两个集群上都运行
~~~

注：

~~~
1. 由于需要进行集群间通信，rbd-mirror守护线程必须能够同时连接两个Ceph集群
2. 每个Ceph集群只能运行一个RBD-mirror
~~~

​	每一个rbd-mirror应该使用为一个Ceph用户ID。在创建ceph用户时，需要指定auth get-or-create command, user name, monitor caps, and OSD caps:

~~~
ceph auth get-or-create client.rbd-mirror.{unique id} mon 'profile rbd' osd 'profile rbd'
~~~

​	RBD-mirror线程可以通过systemctl管理，但需要指定userID；如下：

~~~
systemctl enable ceph-rbd-mirror@rbd-mirror.{unique id}
~~~



### Mirror 类型

​        目前ceph支持两种模式的Mirroring，分别为：

pool

~~~
如果配置pool类型，那么存储池中所有RBD镜像的都会备份到远端集群
~~~

image

~~~
如果配置image类型，那么需要对每一个rbd image显示启动mirror功能
~~~



### Mirrored RBD镜像属性

​    备份的RBD image存在两种状态：

primary

~~~
primary的镜像属性是可以更改的。
~~~

non-primary

~~~
non-primary的镜像属性是不可以更改的。
~~~

当第一次对rbd镜像进行mirroring时，image自动晋升为primary。

====================================================================

​       如何连接不同的Ceph集群，参照RBD帮助

​	**在下面的例子中，集群名称对应于一个同名的Ceph配置文件，比如/etc/ceph/remote.conf，如何部署多个集群，参照ceph-conf帮助。**



## Pool配置

​	存储池配置步骤在两个伙伴Ceph集群中均需要配置，为了便于理解和描述，在后续描述中假定两个Ceph集群，一个为local，另一个为remote.

​       指定存储池中的rbd images都会备份到远程伙伴集群的同名存储池中。如果RBD镜像使用的是单独的数据存储池，那么image的镜像数据会备份到远程伙伴集群的同名的数据存储池。例如，如果mirroring image使用存储池为rbd 和 rbd-ec，那么远程伙伴集群也必须有rbd和rbd-ec存储池。

### Enable Mirroring

​	启动rbd Mirroring，需要指定存储池名称和mirror模式

~~~
rbd mirror pool enable {pool-name} {mode}
~~~

比如：

```
rbd --cluster local mirror pool enable image-pool pool
rbd --cluster remote mirror pool enable image-pool pool
```



### Disable Mirroring

​       禁用RBD Mirroring功能，需要指定存储池名称

~~~
rbd mirror pool disable {pool-name}
~~~

当采用这种方式禁用RBD Mirroring时，指定存储池下所有的镜像都会被禁用

例如：

~~~
rbd --cluster local mirror pool disable image-pool
rbd --cluster remote mirror pool disable image-pool
~~~



###增加Cluster Peer

​      为了使rbd-mirror守护线程能够发现其伙伴集群（peer cluster），需要将伙伴集群的注册到存储池。

​      添加mirroring对等集群，需要指定存储池和集群的的名字

~~~
rbd mirror pool peer add {pool-name} {client-name}@{cluster-name}
~~~

如：

~~~
rbd --cluster local mirror pool peer add image-pool client.remote@remote
rbd --cluster remote mirror pool peer add image-pool client.local@local
~~~



### 删除Cluster Peer

​      删除mirroring对等集群，需要指定存储池的名字和伙伴集群的UUID

~~~
rbd mirror pool peer remove {pool-name} {peer-uuid}
~~~

如：

~~~
rbd --cluster local mirror pool peer remove image-pool 55672766-c02b-4729-8567-f13a66893445
rbd --cluster remote mirror pool peer remove image-pool 60c0e299-b38f-4234-91f6-eed0a367be08
~~~



## image配置

​      不像存储池配置，镜像配置只需要在单个对等Ceph集群上执行既可以。

​      需要指定mirrored RBD 镜像的属性——是primary还是non-primary。non-primary属性的镜像是不可以更改的。

​       当一个镜像首次mirror时，mirrored RBD 镜像会自动设置为primary属性，或者显示的指定。



### 启动Journaling特性

​        RBD Mirroring需要启用 RBD Journaling特性确保image的复制能够报春数据一致性。在镜像备份到伙伴集群前，journaling特性必须启用，journaling特性可以在创建rbd镜像时通过命令` --image-feature exclusive-lock,journaling`来启用.


	rbd create image-pool/image-1 --image-feature exclusive-lock,journaling




​        另外，对于已存在的rbd image，也可用动态地为其启用journal特性，命令格式如下：

~~~
rbd feature enable {pool-name}/{image-name} {feature-name}
~~~

如：

~~~
rbd --cluster local feature enable image-pool/image-1 journaling
~~~

**注意：**

​	journaling feature依赖于exclusive-lock feature，如果exclusive-lock feature还没有启用，就应先于 journaling feature而启用.

**技巧：**可以在Ceph配置文件中增加如下配置所有的新镜像启用journaling：

~~~
rbd default features = 125
~~~



### Enable Image Mirroring

​      如果对于镜像存储池，Mirroring类型为image，那么就需要显式地Mirroring（备份）存储池中的每一个镜像。若备份指定RBD镜像，语法格式如下：

~~~
rbd mirror image enable {pool-name}/{image-name}
~~~

例：

~~~
rbd --cluster local mirror image enable image-pool/image-1
~~~



### DIsable Image Mirroring

​      禁用指定image的备份（Mirroring），语法格式：

~~~
rbd mirror image disable {pool-name}/{image-name}
~~~

例如：

~~~
rbd --cluster local mirror image disable image-pool/image-1
~~~



### Image Promotion and Demotion

​      在故障迁移场景下，primary级别的RBD镜像都需要迁移到伙伴Ceph集群，并停止所有的访问请求（ 比如，VM关机、或者删除相应的驱动），然后将primary镜像降级、将新的镜像升级为primary，然后访问伙伴集群上的对应镜像。

**注意：**

​	**RBD只提供了必要的工具，以便镜像可以有序迁移。还需要外部机制来协调整个故障转移过程。**

​      降级指定RBD镜像为non-primary，语法格式如下：

~~~
rbd mirror image demote {pool-name}/{image-name}
~~~

例如：

~~~
rbd --cluster local mirror image demote image-pool/image-1
~~~

​      如果将一个存储池中所有的primary镜像都降级为non-primary，语法格式如下：

~~~
rbd mirror pool demote {pool-name}
~~~

例如：

~~~
rbd --cluster local mirror pool demote image-pool
~~~

   升级指定RBD镜像为primary，语法格式如下：

~~~
rbd mirror image promote [--force] {pool-name}/{image-name}
~~~

例如：

~~~
rbd --cluster remote mirror image promote image-pool/image-1
~~~

​      如果将一个存储池中所有的non-primary镜像都升级为primary，语法格式如下：

~~~
rbd mirror pool promote [--force] {pool-name}
~~~

例如：

~~~
rbd --cluster local mirror pool promote image-pool
~~~



**技巧：**

​	**由于primary和non-primary状态都是基于镜像的，那么就有可能使得两个集群分担IO负载或分别处于failover / failback状态。**

注意：

​       强制升级可以使用选项`--force`,当伙伴集群不能降级时，就需要强制升级（必须Ceph集群故障，通信中断）。这会导致在这两个伙伴集群中产生脑裂，并且给镜像不能继续同步，除非强制同步。

### Force Image Resync

​	如果rbd-mirror线程检测到脑裂事件，不会字啊继续备份受影响的镜像，支持脑裂恢复。

​	唤醒镜像备份：

~~~
1. 首先确定镜像过时；
2. 然后请求与primary镜像resync
~~~

​	请求与镜像重新同步，命令格式如下：

~~~
rbd mirror image resync {pool-name}/{image-name}
~~~

例如：

~~~
rbd mirror image resync image-pool/image-1
~~~

**注意：**

​	RBD命令只标识需要进行resync的镜像，local cluster的rbd-mirror守护线程负责执行异步resync。



## Mirror Status

​	伙伴集群的复制状态保存在每一个primary mirrored镜像中，这个状态能够通过the mirror image status and mirror pool status来获取：

​	获取备份镜像的状态，命令格式如下：

~~~
rbd mirror image status {pool-name}/{image-name}
~~~

例如：

~~~
rbd mirror image status image-pool/image-1
~~~

​	获取备份存储池的状态：

~~~
rbd mirror pool status {pool-name}
~~~

例如：

~~~
rbd mirror pool status image-pool
~~~

注意：

​	在获取存储池状态时，如果增加选项`--verbose`,,可以获取存储池中每一个mirroring镜像的详细信息。



## 参考文献

[1] http://docs.ceph.com/docs/master/rbd/rbd-mirroring/

[2] https://github.com/ceph/ceph/blob/master/doc/rbd/rbd-mirroring.rst

[3] http://blog.csdn.net/benfenge/article/details/70860319

[4] http://docs.ceph.com/docs/master/rbd/rbd-mirroring/#image-promotion-and-demotion

