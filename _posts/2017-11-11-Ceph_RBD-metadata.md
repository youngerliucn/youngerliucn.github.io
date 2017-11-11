------------
layout: post
title:  "RBD元数据分析"
date:   2017-11-11 15:10:51 +0800
categories: ceph
tags: [Ceph,RBD,Metadata]
description: RBD作为Ceph提供的块设备，每一个RBD镜像都是由元数据和数据两部分组成，所有的元数据存储在多个特殊的rados对象中，而数据被自动条带化成多个rados对象进行存储。
------------
#RBD元数据分析

​        RBD作为Ceph提供的块设备，每一个RBD镜像都是由**元数据**和**数据**两部分组成，所有的元数据存储在多个特殊的rados对象中，而数据被自动条带化成多个rados对象进行存储。

​        RBD元数据主要有三种存储方式：

1. 将元数据编码后以二进制文件的方式存储在rados对象的数据部分，将该类元数据标记为data(比较让人混淆的一个概念)
2. 将元数据以键值对的方式存储在rados对象的扩展属性中，将该类元数据标记为xattr
3. 将元数据以键值对的方式存储在rados对象omap中，将该类元数据标记为omap

​    在RBD的元数据中，有三个关键的元数据对象

| 元数据类型               | Rados对象    | 备注                                   |
| ------------------- | ---------- | ------------------------------------ |
| rbd_id.<name>       | data       | 记录rbd镜像名称到镜像ID的映射关系                  |
| rbd_header.<id>     | xattr/omap | 记录rbd镜像支持的功能特性，容量大小等基本信息，自定义元数据、锁信息等 |
| rbd_object_map.<id> | data       | 记录组成镜像的所有数据对象的存在状态                   |

​        通过以下命令可以了解：

~~~
[root@younger build]# ./bin/rados -p rbd_pool ls
rbd_object_map.10482ae8944a
rbd_header.10482ae8944a
rbd_id.testimg
...
[root@younger build]#
~~~



### 1. rbd_id

​        如前所述，rbd_id.<name>对象记录着rbd名称所对应的ID，RBD是以ID为基础的，镜像名字发生变化，其他内部结构基本不会发生变化。

​        通过rados命令查询：

~~~
[root@younger build]# ./bin/rados get rbd_id.testimg -p rbd_pool rbd_id_info
[root@younger build]# cat rbd_id_info 
10482ae8944a
[root@younger build]# 
~~~

​       知道rbd_pool/testimg镜像的id为10482ae8944a，那么通过rbd命令可以进一步验证

~~~
[root@younger build]# bin/rbd info rbd_pool/testimg
rbd image 'testimg':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.10482ae8944a
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 
	create_timestamp: Sat Nov 11 14:52:40 2017
[root@younger build]# 
~~~

​    也可以很明确的了解到rbd_pool/testimg镜像的id为10482ae8944a

​    对rbd镜像重命名操作，查看ID是否发生变化

~~~
[root@younger build]# bin/rbd mv rbd_pool/testimg rbd_pool/testimg_new
[root@younger build]# bin/rbd info rbd_pool/testimg
rbd: error opening image testimg: (2) No such file or directory
[root@younger build]# bin/rbd info rbd_pool/testimg_new
rbd image 'testimg_new':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.10482ae8944a
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 
	create_timestamp: Sat Nov 11 14:52:40 2017
[root@younger build]# ./bin/rados get rbd_id.testimg_new -p rbd_pool rbd_id_info_new
[root@younger build]# cat rbd_id_info_new 
10482ae8944a
[root@younger build]# 

~~~

​       rbd镜像重命名操作，并不会改变ID。

### rbd_header



