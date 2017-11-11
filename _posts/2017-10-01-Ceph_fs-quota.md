#  通用文件系统的quota实践

##1 理论基础

###1.1 Quota的概念

Quota即限额的意思，用来限制用户、组、文件夹的空间使用量。

###1.2 用途范例

- web服务器控制站点可用空间大小

- mail服务器控制用户可用空间大小

- file服务器控制用户可用空间大小

###1.3 限制模式

- 根据用户（UID）控制每个用户的可用空间大小

- 根据组（GID）控制每个组的可用空间大小

- 根据目录(directory,project)控制每个目录的可用空间大小(xfs可用project模式)

###1.4 使用条件

- EXT格式只能对文件系统进行限制，xfs可用对project进行限制

- 内核需要预开启对Quota支持

- Quota限制只对非管理员有效

- 默认只开启对/home使用Quota，其他需要配置SELinux

###1.5 限制的可配置对象

- 根据用户(User)、组(Group)、特定目录（project）

- 容量限制或文件数量限制(block/inode)

- 限制值soft（超过空间用量给予警告和宽限时间）和hard（超过空间用量则剥夺用户使用权）

- 宽限时间(grace time)，空间用量超出soft限定而未达到hard限定给予的处理时限（超出时限soft值变成hard值）

##2 实际操作

###2.1 配置前准备

####2.1.1 建立用户组

	groupadd gp1

####2.1.2 添加组成员

	useradd -g gp1 user1
	echo"pwd1"|passwd--stdin user1
	useradd -g gp1 user2
	echo"pwd1"|passwd--stdin user2

####2.1.2 创建用户目录并变更所有组

	mkdir /home/gp1 
	chgrp gp1 /home/gp1
	chmod 2770 /home/gp1

####2.1.2 检查文件系统类型
	df -hT /home

显示如下：

	Filesystem             Type Size Used Avail Use% Mounted on
	/dev/mapper/centos-home xfs 5.0G 67M  5.0G   2%   /home
###2.2 启用文件系统的quota功能

####2.2.1 编辑fstab

	vim /etc/fstab
修改内容如下：

	/dev/mapper/centos-home /home xfs defaults,usrquota,grpquota 0 0
注，类型如下：

- 根据用户(uquota/usrquota/quota)

- 根据组(gquota/grpquota)

- 根据目录(pquota/prjquota)(不能与grpquota同时设定)

####2.2.2 卸载并重新挂载

	umount /home
	mount-a
####2.2.3 检查

	mount | grep home
显示如下：

	/dev/mapper/centos-home on /home type xfs rw,relatime,seclabel,attr2,inode64,usrquota,grpquota)

###2.3 查阅Quota信息

####2.3.1 命令格式

	xfs_quota -x -c "指令" [挂载点]

选项与参数：

	-x  ：专家模式，后续才能够加入 -c 的指令参数喔！
	-c  ：后面加的就是指令，这个小节我们先来谈谈数据回报的指令

指令：

      print ：单纯的列出目前主机内的档案系统参数等资料
      df    ：与原本的 df 一样的功能，可以加上 -b (block) -i (inode) -h (加上单位) 等
      report：列出目前的 quota 项目，有 -ugr (user/group/project) 及 -bi 等资料
      state ：说明目前支援 quota 的档案系统的资讯，有没有起动相关项目等

####2.3.2 查询支持Quota的分区

	xfs_quota -x -c "print"

####2.3.3 查询Quota目录的使用情况

	xfs_quota -x -c"df-h"/home

####2.3.4 显示用户的Quota的限制信息

	#xfs_quota -x -c "report -ubih" /home
	User quota on /home (/dev/mapper/centos-home)
	                        Blocks                            Inodes
	User ID      Used   Soft   Hard Warn/Grace     Used   Soft   Hard Warn/Grace
	---------- --------------------------------- ---------------------------------
	root           4K      0      0  00 [------]      4      0      0  00 [------]
	dmtsai      34.0M      0      0  00 [------]    432      0      0  00 [------]
	.....(中间省略).....
	myquota1      12K      0      0  00 [------]      7      0      0  00 [------]
	myquota2      12K      0      0  00 [------]      7      0      0  00 [------]
	myquota3      12K      0      0  00 [------]      7      0      0  00 [------]
	myquota4      12K      0      0  00 [------]      7      0      0  00 [------]
	myquota5      12K      0      0  00 [------]      7      0      0  00 [------]


	# 所以列出了所有用户的目前的档案使用情况，并且列出设定值。注意，最上面的 Block
	# 代表这个是 block 容量限制，而 inode 则是档案数量限制喔。另外，soft/hard 若为 0，代表没限制

注，显示项目加参数“-u”

###2.4 配置限制

####2.4.1 命令格式：

	# xfs_quota -x -c "limit [-ug] b[soft|hard]=N i[soft|hard]=N name"
	# xfs_quota -x -c "timer [-ug] [-bir] Ndays"

选项与参数：

limit ：
	实际限制的项目，可以针对 user/group 来限制，限制的项目有
	bsoft/bhard : block 的 soft/hard 限制值，可以加单位
	isoft/ihard : inode 的 soft/hard 限制值
	name        : 就是用户/群组的名称啊！

timer ：
	用来设定 grace time 的项目喔，也是可以针对 user/group 以及 block/inode 设定

####2.4.2 根据用户和块大小限制

	# xfs_quota -x -c "limit -u bsoft=250M bhard=300M myquota1" /home
	# xfs_quota -x -c "limit -u bsoft=250M bhard=300M myquota2" /home
	# xfs_quota -x -c "limit -u bsoft=250M bhard=300M myquota3" /home
	# xfs_quota -x -c "limit -u bsoft=250M bhard=300M myquota4" /home
	# xfs_quota -x -c "limit -u bsoft=250M bhard=300M myquota5" /home


检查配置：

	# xfs_quota -x -c "report -ubih" /home

####2.4.3 根据组和块大小限制

	# xfs_quota -x -c "limit -g bsoft=950M bhard=1G myquotagrp" /home

检查配置：

	# xfs_quota -x -c "report -gbih" /home

####2.4.5 配置宽限时间

	# xfs_quota -x -c "timer -ug -b 14days" /home

验证配置：

	# xfs_quota -x -c "state" /home

####2.4.6 验证Quta

	su - myquota1
	$ dd if=/dev/zero of=123.img bs=1M count=310
	dd: error writing ‘123.img’: Disk quota exceeded
	300+0 records in
	299+0 records out
	314552320 bytes (315 MB) copied, 0.181088 s, 1.7 GB/s
	$ ll -h
	-rw-r--r--. 1 myquota1 myquotagrp 300M Jul 24 21:38 123.img
	
	$ exit
	# xfs_quota -x -c "report -ubh" /home

###2.5 根据project限制

首先，要将 grpquota 的参数取消，然后加入 prjquota ，并且卸载 /home 再重新挂载才行
####2.5.1 修改fstab

	vim /etc/fstab

内容：
	/dev/mapper/centos-home /home xfs  defaults,usrquota,prjquota  0 0

记得， grpquota 与 prjquota 不可同时设定喔！所以上面删除 grpquota 加入 prjquota

	# umount /home
	# mount -a
	# xfs_quota -x -c "state"




####2.5.1 规范目录、专案名称(project)与专案 ID

1 指定专案识别码与目录的对应在 /etc/projects

	# echo "11:/home/myquota" >> /etc/projects

2 规范专案名称与识别码的对应在 /etc/projid

	# echo "myquotaproject:11" >> /etc/projid

3 初始化专案名称

	# xfs_quota -x -c "project -s myquotaproject"
	Setting up project myquotaproject (path /home/myquota)...
	Processed 1 (/etc/projects and cmdline) paths for project myquotaproject with recursion 
	depth infinite (-1).   

会闪过这些讯息！是 OK 的！别担心！
​	
	[root@study ~]# xfs_quota -x -c "print " /home
	Filesystem          Pathname
	/home               /dev/mapper/centos-home (uquota, pquota)
	/home/myquota       /dev/mapper/centos-home (project 11, myquotaproject)

print 功能很不错！可以完整的查看到相对应的各项档案系统与 project 目录对应	

	# xfs_quota -x -c "report -pbih " /home
	Project quota on /home (/dev/mapper/centos-home)
	                        Blocks                            Inodes
	Project ID       Used   Soft   Hard Warn/Grace     Used   Soft   Hard Warn/Grace
	---------- --------------------------------- ---------------------------------
	myquotaproject      0      0      0  00 [------]      1      0      0  00 [------]
喔耶！确定有抓到这个专案名称啰！接下来准备设定吧！


####2.5.3 根据块大小配置限制

依据本章的说明，将 /home/myquota 指定为 500M 的容量限制，那假设到 450M 为 soft 的限制好了！ 那么设定就会变成这样啰：

1. 先来设定好这个 project 吧！设定的方式同样使用 limit 的 bsoft/bhard 喔！：

   # xfs_quota -x -c "limit -p bsoft=450M bhard=500M myquotaproject" /home
   	# xfs_quota -x -c "report -pbih " /home
   	Project quota on /home (/dev/mapper/centos-home)
   	                        Blocks                            Inodes
   	Project ID       Used   Soft   Hard Warn/Grace     Used   Soft   Hard Warn/Grace
   	---------- --------------------------------- ---------------------------------
   	myquotaproject      0   450M   500M  00 [------]      1      0      0  00 [------]
   	# dd if=/dev/zero of=/home/myquota/123.img bs=1M count=510
   	dd: error writing '/home/myquota/123.img': No space left on device
   	501+0 records in
   	500+0 records out
   	524288000 bytes (524 MB) copied, 0.96296 s, 544 MB/s


##XFS quota 的管理与额外指令对照表


##2.6 Quota的管理
- disable：暂时取消 quota 的限制，但其实系统还是在计算 quota 中，只是没有管制而已！应该算最有用的功能啰！
- enable：就是回复到正常管制的状态中，与 disable 可以互相取消、启用！
- off：完全关闭 quota 的限制，使用了这个状态后，你只有卸载再重新挂载才能够再次的启动 quota 喔！也就是说， 用了 off 状态后，你无法使用 enable 再次复原 quota 的管制喔！注意不要乱用这个状态！一般建议用 disable 即可，除非你需要执行 remove 的动作！
- remove：必须要在 off 的状态下才能够执行的指令～这个 remove 可以『移除』quota 的限制设定，例如要取消 project 的设定， 无须重新设定为 0 喔！只要 remove -p 就可以了！


####2.6.1  临时禁用 XFS quota

	# xfs_quota -x -c "disable -up" /home
	# xfs_quota -x -c "state" /home
	User quota state on /home (/dev/mapper/centos-home)
	  Accounting: ON
	  Enforcement: OFF   <== 意思就是有在计算，但没有强制管制的意思
	  Inode: #1568 (4 blocks, 4 extents)
	Group quota state on /home (/dev/mapper/centos-home)
	  Accounting: OFF
	  Enforcement: OFF
	  Inode: N/A
	Project quota state on /home (/dev/mapper/centos-home)
	  Accounting: ON
	  Enforcement: OFF
	  Inode: N/A
	Blocks grace time: [7 days 00:00:30]
	Inodes grace time: [7 days 00:00:30]
	Realtime Blocks grace time: [7 days 00:00:30]
	
	# dd if=/dev/zero of=/home/myquota/123.img bs=1M count=520
	520+0 records in
	520+0 records out  # 见鬼！竟然没有任何错误发生了！
	545259520 bytes (545 MB) copied, 0.308407 s, 180 MB/s
	
	# xfs_quota -x -c "report -pbh" /home
	Project quota on /home (/dev/mapper/centos-home)
	                        Blocks
	Project ID       Used   Soft   Hard Warn/Grace
	---------- ---------------------------------
	myquotaproject   520M   450M   500M  00 [-none-]

其实，还真的有超过耶！只是因为 disable 的关系，所以没有强制限制住就是了！

####2.6.2  临时启动 XFS quota

	# xfs_quota -x -c "enable -up" /home  # 重新启动 quota 限制
	# dd if=/dev/zero of=/home/myquota/123.img bs=1M count=520
	dd: error writing ‘/home/myquota/123.img’: No space left on device

又开始有限制！这就是 enable/disable 的相关对应功能喔！暂时关闭/启动用的！

####2.6.3 完全关闭Quota限制

	# xfs_quota -x -c "off -up" /home
	# xfs_quota -x -c "enable -up" /home
	XFS_QUOTAON: Function not implemented

没有办法重新启动！因为已经完全的关闭了 quota 的功能！所以得要 umouont/mount 才行！

	# umount /home; mount -a

这个时候使用 report 以及 state 时，管制限制的内容又重新回来了！好！来瞧瞧如何移除project

	# xfs_quota -x -c "off -up" /home
	# xfs_quota -x -c "remove -p" /home
	# umount /home; mount -a
	# xfs_quota -x -c "report -phb" /home
	Project quota on /home (/dev/mapper/centos-home)
	                        Blocks
	Project ID       Used   Soft   Hard Warn/Grace
	---------- ---------------------------------
	myquotaproject   500M      0      0  00 [------]


##参阅文档
http://linux.vbird.org/linux_basic/0420quota.php
http://www.centoscn.com/CentOS/config/2013/1103/2043.html
http://www.jb51.net/os/RedHat/400503.html