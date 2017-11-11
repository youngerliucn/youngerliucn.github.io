# Ceph and RBD Mirroring：Luminous

Source: Sebastian Han

https://ceph.com/planet/ceph-and-rbd-mirroring-upcoming-enhancements/

​	本文说明对Sebastian Han的上述文章内容进行梳理，了解RBD Mirroring在Luminous版本的变化

## HA Support

​	高可用是一个应用程序的关键能力。理想情况下，应用程序本身是支持高可用的。正由于此，RBD mirroring也需要支持高可用。我们可能在任意数量的机器上运行任意数量的守护进程，将有专门的线程来负责如何分布负载和镜像。

​	HA模型将依赖协作的守护线程，守护线程可以确定需要处理哪一个镜像——依赖于很多因素：当前负载、镜像数量等等

## Multiple peers

​	目前，在Ceph Jewel和Kraken，支持下列关系：

1对1：

~~~
一个primary集群和一个non-primary集群，绝大部分灾备应用场景采用的方案——一个主集群为数据应用或站点提供服务，另外还有一个空闲集群用于接收来自于primary集群的镜像数据。第二个站点不运行任何程序，直到被唤醒。
~~~

1对多：

~~~
与1对1一样，唯一不同的是，有多个不同的non-primary位置，来保存Ceph镜像的副本。
~~~

​	然而，是不支持“多对多”场景的，多对多场景在多站点部署中会起到至关重要的作用。

​	比如，管理人员计划部署3个OpenStack区，他肯定希望在每个站点之间创建网格副本。但是这是不可能的，唯一能做的是，配置链式复制，如A->B->C->A。

~~~
This is not ideal as not every site will have each other data, since we want resiliency we need that.
~~~




## Delayed deletion

​	RBD-mirror的复制是异步的，但是，如果primary image被删除，那么伙伴ceph集群的non-primary image也会被删除。这意味着，如果有人意外删除某个image，那么该image将永远丢失。但是管理员希望希望避免此类事件的发生，这就是为什么延迟删除是非常重要的原因。延迟删除可以使得将意外删除的rbd image回收，比如设置延迟删除时间为4小时，那么在这个时间框架内，任何可能意外删除image都可以被恢复。



## Clone non-primary images

​	Ceph的一个主要应用场景是使用它作为OpenStack的块存储解决方案。云提供商希望部署跨越多个站点的OpenStac。为了实现这种弹性，可以相互部署两个独立的OpenStack云，并在这两个站点之间配置一个交叉复制的Glance和Cinder镜像。
​	由于此设置是采用active-active方式运行，所以希望使用任意站点的Glance images。假设，可以将Glance images的元数据（DB records）从一个云复制到另外一个。同时，rbd-mirror配置将镜像从一个站点复制到另一个站点。目前，正在复制的non-primary镜像不可读的。他们基本上都是只读的，最差的情况是是他们不能克隆，在启动镜像时，克隆是Glance和Nova之间的整合的重点。如果不能克隆，我们将失去了一个最好的Ceph特性——有利将ceph纳入OpenStack。
​	如果我们不能克隆从我们从primary站点复制的image，那么这些images就不能另外一个OpenStack环境使用。因此，他们是没有用的。这就是为什么我们需要在non-primary站点克隆non-primary镜像。



## QoS throttles for replication

​	rbd-mirror QoS

​	根据不同的类型和距离，数据中心之间的联系是非常昂贵的。Putting the monetary aspect on the side they could also serve a different purpose。因此，在两个RBD-mirror之间需要配置一定的QoS



## Configurable replication delay

​	与以前的功能相关联，假定复制可以被延迟是很合理的。在该方案中，我们允许管理员为rbd-mirror配置一个延时时间（如，non-primary站点落后于primary站点X小时）。



## Add a promote-all call on a pool

​	在一个故障场景中，non-primary镜像需要晋升到primary级别（比如故障迁移发生）。OpenStack Cinder目前的复制API是通过遍历存储池中每一个image并升级其为primary来实现的。串行地遍历每一个镜像会话费很多时间，现在的是通过并行化来实现的。
​	由于 primary/non-primary 状态是针对每一个镜像的，而不是针对存储池，不管怎样，都需要遍历存储池中的每一个镜像。cli可以并行地批量地讲X个镜像进行升级，可以通过类似于`concurrent_management_ops`的方式来实现。

