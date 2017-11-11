# Ceph文件系统修复机制ceph-data-scan(2)#

​      前述章节已经介绍过，ceph-data-scan是通过函数data_scan.main(args)解析并执行用户命令的。本章节主要介绍data_scan

=========================

## 1. ceph-scan-data的用法

通过代码来看，如下：

~~~C++
  std::cout << "Usage: \n"
    << "  cephfs-data-scan init [--force-init]\n"
    << "  cephfs-data-scan scan_extents [--force-pool] [--worker_n N --worker_m M] <data pool name>\n"
    << "  cephfs-data-scan scan_inodes [--force-pool] [--force-corrupt] [--worker_n N --worker_m M] <data pool name>\n"
    << "  cephfs-data-scan pg_files <path> <pg id> [<pg id>...]\n"
    << "  cephfs-data-scan scan_links\n"
    << "\n"
    << "    --force-corrupt: overrite apparently corrupt structures\n"
    << "    --force-init: write root inodes even if they exist\n"
    << "    --force-pool: use data pool even if it is not in FSMap\n"
    << "    --worker_m: Maximum number of workers\n"
    << "    --worker_n: Worker number, range 0-(worker_m-1)\n"
    << "\n"
    << "  cephfs-data-scan scan_frags [--force-corrupt]\n"
    << "  cephfs-data-scan cleanup <data pool name>\n"
    << std::endl;

  generic_client_usage();
}
~~~



## 2. 参数解析

 (1) 判断参数个数，参数个数至少为1，否则datascan是无法理解用户想做什么的.

~~~
  if (args.size() < 1) {
    usage();
    return -EINVAL;
  }
~~~

(2)通用RADOS初始化，及初始化全局变量g_ceph_context

​    g_ceph_context是CephContext *的类型，

~~~
  // Common RADOS init: open metadata pool
  // =====================================
  librados::Rados rados;
  int r = rados.init_with_context(g_ceph_context);
  if (r < 0) {
    derr << "RADOS unavailable" << dendl;
    return r;
  }
~~~

(3) 通过for循环，依次解析每一个参数，在解析过程中会验证各个参数，然后以全局变量进行标识

~~~
  for (std::vector<const char *>::const_iterator i = args.begin() + 1;
       i != args.end(); ++i) {
       ......
~~~

(4) 检查namespace，如果没有指定并且本地只有一个，则选择这一个，否则报错

~~~
  // If caller didn't specify a namespace, try to pick
  // one if only one exists
  if (fscid == FS_CLUSTER_ID_NONE) {
    if (fsmap->filesystem_count() == 1) {
      fscid = fsmap->get_filesystem()->fscid;
    } else {
      std::cerr << "Specify a filesystem with --filesystem" << std::endl;
      return -EINVAL;
    }
  }
  auto fs =  fsmap->get_filesystem(fscid);
  assert(fs != nullptr);
~~~

(5)  如果没有初始化output目录，则初始化为MetadataDriver(),否则初始化为LocalFileDriver()

~~~C++
  // Default to output to metadata pool
  if (driver == NULL) {
    driver = new MetadataDriver();
    driver->set_force_corrupt(force_corrupt);
    driver->set_force_init(force_init);
    dout(4) << "Using metadata pool output" << dendl;
  }
~~~



## 3. 相关初始化

(1) 与Rados建立通信联系

~~~
  dout(4) << "connecting to RADOS..." << dendl;
  r = rados.connect();
  if (r < 0) {
    std::cerr << "couldn't connect to cluster: " << cpp_strerror(r)
              << std::endl;
    return r;
  }
~~~

(2) 初始化driver

~~~
  r = driver->init(rados, metadata_pool_name, fsmap, fscid);
  if (r < 0) {
    return r;
  } 
~~~



接下来是对用户的各种操作分析



## 4. init操作分析

当用户输入命令如下命令

~~~
# cephfs-data-scan init [--force-init]
~~~



### 1.解析参数

当进入data_scan.main中时，首先会解析参数，当解析到--force-init时，会设置全局变量force_init=true

~~~
  if (arg == "--force-pool") {
    ...
  } else if (arg == "--force-init") {
    force_init = true;
    return true;
  } else {
    return false;
  }
~~~



### 2. 初始化

然后会初始化fs、 driver，与rados建立通信联系



### 3. 执行init操作

然后调用init_rootfs来初始化

~~~
  // Finally, dispatch command
  if (command == "scan_inodes") {
    ....
  } else if (command == "init") {
    return driver->init_roots(fs->mds_map.get_first_data_pool());
  } else {
    std::cerr << "Unknown command '" << command << "'" << std::endl;
    return -EINVAL;
  }
~~~



MetadataDriver->init_rootfs

![ControlFlow-init_roots](E:\03_电子藏书馆_试建\502_cookbook\flow\ControlFlow-init_roots.png)

LocalFileDriver->init_rootfs

![ControlFlow-init_roots_LocalFileDriver](E:\03_电子藏书馆_试建\502_cookbook\flow\ControlFlow-init_roots_LocalFileDriver.png)





