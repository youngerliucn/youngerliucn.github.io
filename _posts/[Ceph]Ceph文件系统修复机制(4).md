# Ceph文件系统修复机制ceph-data-scan(3)

​      前述章节已经介绍过:

​      ceph-data-scan是通过函数data_scan.main(args)解析并执行用户命令的。

​      data_scan.main(args)的参数检查和解析及其ceph-data-scan init的执行过程

​      本章节主要介绍cephfs-data-scan scan_extents [--force-pool][--worker_n N --worker_m M] <data pool name>的实现过程

当用户输入命令如下命令

```
# cephfs-data-scan scan_extents --force-pool [--worker_n N --worker_m M] <data pool name>
```

当<data pool name>为文件系统数据池，必须指定--force-pool ，定义最大可启动的线程数为M，默认启动的是N

其思想是：扫描存储池中的每一个object，

## 1. 参数解析

当进入data_scan.main中时，首先会解析参数，当解析到--force-pool时，会设置全局变量force_pool=true

```
  if (arg == "--force-pool") {
    force_pool = true;
    return true;
  } 
  ...
  } else {
    return false;
  }
```

解析worker_n和worker_m，

如下代码块所示，其中m,n均为DataScan的成员变量

~~~
bool DataScan::parse_kwarg(
    const std::vector<const char*> &args,
    std::vector<const char *>::const_iterator &i,
    int *r)
{
  ...
  if (arg == std::string("--output-dir")) {
   ...
  } else if (arg == std::string("--worker_n")) {
    std::string err;
    n = strict_strtoll(val.c_str(), 10, &err);
    if (!err.empty()) {
      std::cerr << "Invalid worker number '" << val << "'" << std::endl;
      *r = -EINVAL;
      return false;
    }
    return true;
  } else if (arg == std::string("--worker_m")) {
    std::string err;
    m = strict_strtoll(val.c_str(), 10, &err);
    if (!err.empty()) {
      std::cerr << "Invalid worker count '" << val << "'" << std::endl;
      *r = -EINVAL;
      return false;
    }
    return true;
  }
  ...
~~~



## 2. 初始化

然后会初始化fs、 driver，与rados建立通信联系



## 3. 存储池的查找

通过存储池的名字获取存储池的ID

~~~
    data_pool_id = rados.pool_lookup(data_pool_name.c_str());
    if (data_pool_id < 0) {
      std::cerr << "Data pool '" << data_pool_name << "' not found!" << std::endl;
      return -ENOENT;
    } else {
      dout(4) << "data pool '" << data_pool_name
        << "' has ID " << data_pool_id << dendl;
    }
~~~

pool_lookup调用函数

~~~
int64_t librados::RadosClient::lookup_pool(const char *name)
~~~

其流程如下：

![ControlFlow-lookup_pool](E:\03_电子藏书馆_试建\502_cookbook\flow\ControlFlow-lookup_pool.png)

其语义很简单：

1. 首先是获取osdmap，同步请求
2. 从osdmap查找存储池的ID
3. 如果前两步，无法查找，则更新osdmap，再一次查找

继续判断，如果不是文件系统的数据池，则直接返回

~~~c
    if (!fs->mds_map.is_data_pool(data_pool_id)) {
      std::cerr << "Warning: pool '" << data_pool_name << "' is not a "
        "CephFS data pool!" << std::endl;
      if (!force_pool) {
        std::cerr << "Use --force-pool to continue" << std::endl;
        return -EINVAL;
      }
    }
~~~



最后调用scan_extents

## 4 scan_extents分析

其流程如下：（有待于仔细斟酌和完善）

1. 获取对象分片列表的起始、终止焦点
2. 判断是否为旧版本osd，旧版本osd是不支持cephfs pgls filtering mode
3. 依次遍历每一个object，对每一个object进行处理





![ControlFlow-forall_objects](E:\03_电子藏书馆_试建\502_cookbook\flow\ControlFlow-forall_objects.jpg)

