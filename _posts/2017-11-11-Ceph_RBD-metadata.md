---
layout: post
title:  "RBD元数据分析"
date:   2017-11-13
categories: storage
tags: RBD
description: RBD镜像都是由元数据和数据两部分组成，元数据存储在多个特殊的rados对象中.
---

[TOC]

​        RBD作为Ceph提供的块设备，每一个RBD镜像都是由**元数据**和**数据**两部分组成，所有的元数据存储在多个特殊的rados对象中，而数据被自动条带化成多个rados对象进行存储。

​        RBD元数据主要有三种存储方式：

1. 将元数据编码后以二进制文件的方式存储在rados对象的数据部分，将该类元数据标记为data(比较让人混淆的一个概念)
2. 将元数据以键值对的方式存储在rados对象的扩展属性中，将该类元数据标记为xattr
3. 将元数据以键值对的方式存储在rados对象omap中，将该类元数据标记为omap

## RBD镜像元数据

​        在RBD的元数据中，有三个关键的元数据对象


| 元数据类型                | Rados对象    | 备注                                   |
| :------------------- | :--------- | :----------------------------------- |
| rbd_id.\<name>       | data       | 记录rbd镜像名称到镜像ID的映射关系                  |
| rbd_header.\<id>     | xattr/omap | 记录rbd镜像支持的功能特性，容量大小等基本信息，自定义元数据、锁信息等 |
| rbd_object_map.\<id> | data       | 记录组成镜像的所有数据对象的存在状态                   |

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

​        如前所述，rbd_id.\<name>对象记录着rbd名称所对应的ID，RBD是以ID为基础的，镜像名字发生变化，其他内部结构基本不会发生变化。

​        通过rados命令查询：

	[root@younger build]# ./bin/rados get rbd_id.testimg -p rbd_pool rbd_id_info
	[root@younger build]# cat rbd_id_info 
	10482ae8944a
	[root@younger build]# 

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

### 2. rbd_header

​        rbd_header.\<id>对象是RBD镜像最主要的元数据对象，包含以下信息：

| Key                 | type | 字节数    | 描述                                       |
| ------------------- | ---- | ------ | ---------------------------------------- |
| data_pool_id        | omap | 64bits | 记录镜像数据存储的存储池，若数据和元数据使用同一存储池，则该key不存在     |
| features            | data | 64bits | 启用的功能特性：layering(0)、striping(1)、exclusive-lock(2)、object-map(3)、fast-diff(4)、deep-flatten(5)、journaling(6)、data-pool(7) |
| object_prefix       | omap | string | 数据对象的名称前缀                                |
| order               | omap | 8bits  | 组成镜像的数据对象大小，2^order                      |
| parent              | omap |        | 存在克隆关系时，父镜像的快照信息                         |
| size                | omap | 64bits | 镜像容量大小                                   |
| snap_seq            | omap | 64bits | 最近一次的快照ID                                |
| snapshot_\<snap_id> | omap |        | ID为snap_id快照的基本信息                        |
| strip_count         | omap | 64bits | 条带宽度，数据对象间进行条带化的参数                       |
| strip_unit          | omap | 64bits | 条带大小，                                    |
| lock_rbd_lock       | omap |        | 控制镜像互斥访问的锁                               |

参考：

~~~
[root@younger build]# ./bin/rados getomapval -p rbd_pool rbd_header.10482ae8944a order
value (1 bytes) :
00000000  16                                                |.|
00000001
[root@younger build]# 
~~~

....

### 3. rbd_object_map

​        RBD镜像采用精简配置（thin provisioning）的形式对存储空间进行分配，新创建的镜像只存在少量的元数据对象，在RBD镜像之上的文件系统或快设备应用在对镜像进行读写时，才会进行数据空间的分配。

​       通过对一个10G镜像testimg进行验证：

1. 初次创建：

~~~
[root@younger build]# bin/rados get -p rbd_pool rbd_object_map.10482ae8944a rbd_data_obj_map
[root@younger build]# bin/ceph-dencoder type 'BitVector<2>' import rbd_data_obj_map decode dump_json | head
{
    "size": 2560,
    "bit_table": [
        "0x00",
        "0x00",
        "0x00",
        "0x00",
        "0x00",
        "0x00",
        "0x00",
[root@younger build]# 
~~~

所有的内容均为0，说明此时没有任何数据，也没有分配数据对象。

2. 写入部分数据后

~~~
[root@younger build]# bin/ceph-dencoder type 'BitVector<2>' import rbd_data_obj_map_1 decode dump_json | head
{
    "size": 2560,
    "bit_table": [
        "0x00",
        "0x00",
        "0x05",
        "0x54",
        "0x00",
        "0x00",
        "0x00",
~~~

数据layout发生了变化。

## RBD管理元数据

​        在RBD的存储池中，需要一些独立的元数据对象记录和管理RBD镜像：

| 对象名           | 类型   | 描述                    |
| ------------- | ---- | --------------------- |
| rbd_directory | omap | 记录存储池的rbd镜像列表         |
| rbd_children  | omap | 记录父镜像快照到克隆镜像之间的单项映射关系 |

### rbd_directory

​        每一个创建了rbd镜像的存储池下都会有rbd_directory对象，记录当前存储池中的rbd镜像列表。镜像列表信息以键值对的形式记录在rbd_directory对象的omap中，如下：

| Key          | Type | 描述          |
| ------------ | ---- | ----------- |
| name_\<name> | omap | 镜像名称对应的ID   |
| id_\<id>     | omap | 镜像ID对应的镜像名称 |

​        列出存储池的镜像列表：

~~~
[root@younger build]# ./bin/rados listomapvals -p rbd_pool rbd_directory
id_10482ae8944a
value (11 bytes) :
00000000  07 00 00 00 74 65 73 74  69 6d 67                 |....testimg|
0000000b

id_fa7774b0dc51
value (13 bytes) :
00000000  09 00 00 00 74 65 73 74  69 6d 67 5f 30           |....testimg_0|
0000000d

id_fa7a643c9869
value (13 bytes) :
00000000  09 00 00 00 74 65 73 74  69 6d 67 5f 31           |....testimg_1|
0000000d

name_testimg
value (16 bytes) :
00000000  0c 00 00 00 31 30 34 38  32 61 65 38 39 34 34 61  |....10482ae8944a|
00000010

name_testimg_0
value (16 bytes) :
00000000  0c 00 00 00 66 61 37 37  37 34 62 30 64 63 35 31  |....fa7774b0dc51|
00000010

name_testimg_1
value (16 bytes) :
00000000  0c 00 00 00 66 61 37 61  36 34 33 63 39 38 36 39  |....fa7a643c9869|
00000010

[root@younger build]# 
~~~

​        两种信息都列了出来。

### rbd_children

​        当镜像之间存储克隆关系时，在存储池中会产生rbd_children对象（如果未创建过克隆关系，则不存在rbd_children对象）（查看对象*rados -p rbd_pool ls*）

​        rbd_children用于记录父镜像快照到克隆镜像之间的单向映射关系：

| Key       | Type | 描述                          |
| --------- | ---- | --------------------------- |
| \<parent> | omap | 当前存储池下基于父镜像快照创建的一个或多个克隆镜像列表 |

​        关键字由\<pool_id,image_id,snap_id>三个字段组，用于表示父镜像快照的信息。

~~~
[root@younger build]# ./bin/rbd create rbd_pool/testimg_0
[root@younger build]# ./bin/rbd snap create rbd_pool/testimg_0@snapshot1
[root@younger build]# ./bin/rbd snap protect rbd_pool/testimg_0@snapshot1
[root@younger build]# ./bin/rbd clone rbd_pool/testimg_0@snapshot1 rbd_pool/clone_testimg_0
[root@younger build]# ./bin/rados listomapvals -p rbd_pool rbd_children
key (32 bytes):
00000000  03 00 00 00 00 00 00 00  0c 00 00 00 66 61 37 37  |............fa77|
00000010  37 34 62 30 64 63 35 31  04 00 00 00 00 00 00 00  |74b0dc51........|
00000020

value (20 bytes) :
00000000  01 00 00 00 0c 00 00 00  66 61 38 61 32 65 62 31  |........fa8a2eb1|
00000010  34 31 66 32                                       |41f2|
00000014

[root@younger build]# 
~~~

解析：

​        pool id: 0x0000000000000003

​        image id: fa7774b0dc51

​        snap id: 0x0000000000000002

