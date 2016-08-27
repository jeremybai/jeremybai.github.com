---
layout: post
title: "DigitalOcean主机增加Swap分区"
description: ""
categories: 
- linux
- web
tags: [SecureCRT,DigitalOcean,VPS,Swap]
---

　　在cmake mysql源码的时候出现下面的错误：  

	[ 46%] Building CXX object sql/CMakeFiles/sql.dir/geometry_rtree.cc.o
	c++: internal compiler error: Killed (program cc1plus)
	Please submit a full bug report,
	with preprocessed source if appropriate.
	See <http://bugzilla.redhat.com/bugzilla> for instructions.
	make[2]: *** [sql/CMakeFiles/sql.dir/geometry_rtree.cc.o] Error 4
	make[1]: *** [sql/CMakeFiles/sql.dir/all] Error 2
	make: *** [all] Error 2
　　通过查找，[可能是因为内存不够的原因](https://bitcointalk.org/index.php?topic=304389.0)，使用`free -h`查看了下，发现DO的主机连Swap分区都没有，Swap分区是当物理内存不够用的时候，把物理内存中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间被临时保存到Swap分区中，等到那些程序要运行时，再从Swap分区中恢复保存的数据到物理内存中。Swap的调整对Linux服务器，特别是Web服务器的性能至关重要，通过调整Swap，有时可以越过系统性能瓶颈，节省系统升级的费用。  
　　SWAP分区设置多大是我们需要关心的问题，关于设置的规则可以参考下面，实际情况可以根据业务需求进行调整，选择合适的Swap分区大小：  

	4G以内的物理内存，SWAP 设置为内存的2倍。 
	4-8G的物理内存，SWAP 等于内存大小。 
	8-64G 的物理内存，SWAP 设置为8G。 
	64-256G物理内存，SWAP 设置为16G。 
　　接下来我们看下如何设置Swap分区。  
## 检查是否存在Swap分区 
　　输入`swapon -s`，如果没有任何的信息显示，也就是还没有划分Swap分区。  
## 检查文件系统 
　　如果没有创建Swap分区，再看下硬盘还剩下多少空间可以使用，使用`df`命令查看。因为我先创建了1G的Swap分区，还是报错，于是我选择创建一个2GB大小的Swap分区。  
## 创建Swap分区文件
　　创建swap文件。   

    dd if=/dev/zero of=/swapfile bs=2048 count=1M
　　该命令将创建一个大小为2GB，文件名为swapfile的Swap分区文件，`of=/swapfile`参数指定了文件的创建位置和文件名；`bs=2048`指定了文件的大小，`count=1M`代表单位。  
## 格式化swap分区 

    mkswap /swapfile

## 激活swap分区   

    swapon /swapfile

## 查询swap分区 

    swapon -s
　　你会发现在重启之后Swap分区就没了，那是因为上面的设置是一次性的，想要一直启动Swap分区，可以编辑fstab文件。

    nano /etc/fstab
　　在最后一行添加上下面一条：  

    /swapfile     swap     swap     defaults     0  0
　　添加成功后给swap赋予相关权限：

    chown root:root /swapfile
    chmod 0600 /swapfile  
## 配置swappiness
　　实际上，并不是等所有的物理内存都消耗完毕之后，才去使用swap的空间，什么时候使用是由swappiness 参数值控制。

	cat /proc/sys/vm/swappiness
　　默认值是60，swappiness=0 的时候表示最大限度使用物理内存，然后才是Swap空间;swappiness＝100 的时候表示积极的使用Swap分区，并且把内存上的数据及时的搬运到swap空间里面。  
### 临时性修改

	sysctl vm.swappiness=10
	cat /proc/sys/vm/swappiness  
　　这里我们的修改已经生效，但是如果我们重启了系统，又会变成60。  
### 永久修改
　　在`/etc/sysctl.conf`文件里添加如下参数：`vm.swappiness=10`，保存重启就可以了。