创建：mkfs -> mount
重启：mount
![[未命名绘图.drawio.png]]
## mkfs
mkfs主要固化一些用户指定的配置项到磁盘，后续bluestore上电时，这些配置项可以直接从磁盘读取，从而不受配置文件改变的影响。避免配置项改变时，bluestore重新上电因为使用了不同配置项造成磁盘数据发生永久损坏。
在osd的路径下存放了bluestore的元数据文件，以字符串的形式存放元数据。
![[Pasted image 20221128011752.png]]
上图的路径即为程序操作的路径。
vstart创建的虚拟环境路径如下：
![[Pasted image 20221128011920.png]]
1. 读取mkfs_done文件，检查是否已经进行过格式化。
2. 读取type文件，检查objectstore类型。若该文件不存在，则创建文件固化类型“bluestore”到磁盘。
3. 固定使用freelist_type=bitmap。用于后面_open_fm()创建freelistmanager。
4. 打开path路径，获取文件句柄`path_fd`，保存在成员变量中。
	（对应上图/var/lib/ceph/osd/ceph-0和/home/store/ceph/build/dev/osd0）
5. 打开fsid文件，得到fsid文件的句柄`fsid_fd`，保存在成员变量中。fsid唯一标识一个bluestore实例。
6. 对fsid文件加锁。锁定fsid的目的是为了防止对应的磁盘被多个bluestore实例打开和访问，从而引起数据一致性问题。
	`F_WRLCK`: 写入锁/互斥锁
7. 读取fsid文件中的内容到old_fsid。
	如果fsid文件为空：
	- 如果未指定fsid，则使用随机生成器生成一个。
	- 否则使用已经指定的fsid（为osd的uuid，在OSD::mkfs中指定）
8. 建立path/block文件到配置指定的数据存储设备的符号链接/软链接（对应第一张图中的青色高亮block）
	如果是使用bluestore，建立path/block.wal到配置指定的日志存储设备的软链接、
	建立path/block.db到DB存储设备的软链接。
![[Pasted image 20221128014044.png]]
9. 打开path/block设备（主设备）,如果不存在则创建。创建BlockDevice对象，并调用open方法，获得设备大小dev_size，如果设备支持标签的话（NVME设备不支持），设置标签为main；初始化全局参数block_size、block_mask、block_size_order等，并且根据设备类型设置缓存大小。
10. 从配置项`bluestore_min_alloc_size`里面获取最小配置单元`min_alloc_size`。
	如果没有配置，则判断设备是否HDD（通过设备类的is_rotational方法来判断，在一般linux系统中，可通过读取/sys/block/sdX/queue/rotational 来判断是否为HDD盘，对应源码src/common/blkdev.cc block_device_is_rotational），如果是HDD,则使用`bluestore_min_alloc_size_hdd`配置项作为`min_alloc_size`，否则使用`bluestore_min_alloc_size_ssd`。
![[Pasted image 20221128014547.png]]
> [!NOTE] 注意
> **block_size（磁盘块大小）和min_alloc_size是两个不同的参数**，即使两者一般情况下是相等的。
> block_size是一个确定的数值，由磁盘的物理性质决定，无法进行修改；而min_alloc_size是一个可配置项。一般情况下min_alloc_size和block_size大小相等，但是出于分配效率的考虑，也可以将min_alloc_size提升为block_size的整数倍。
11. 打开db，不存在则创建。
12. 创建freelistmanager。
13. 持久化SUPER元数据到磁盘。 ^1541ea
	- `nid_max`：每个对象拥有实例内唯一的nid。
			nid_max用于标识当前bluestore最小未被分配的nid，新建对象的nid总是从当前的nid_max开始分配。
	- `blobid_max`： blob对象的id，在bluestore实例内唯一
	- `min_alloc_size`
1. 固化元数据到磁盘文件：kv_backend、bluefs、fsid
2. 依据配置项`bluestore_fsck_on_mkfs`，扫描并检验所有的**元数据**。若`bluestore_fsck_on_mkfs_deep`也为真，同时扫描并检验所有的**对象数据**。
3. 更新mkfs_done文件：yes。

## mount
osd上电时，BlueStore通过mount操作完成正常上电前的检查和准备工作。
1. 读取type文件，校验objectstore类型。
2. 依据配置项`bluestore_fsck_on_mkfs`，扫描并检验所有的**元数据**。若`bluestore_fsck_on_mkfs_deep`也为真，同时扫描并检验所有的**对象数据**。
3. 打开path路径。
4. 打开fsid文件。
5. 读取fsid文件。
6. 对fsid文件加锁。
7. 加载主设备。slow类型，由bluestore直接进行管理。
8. 加载db。
9. 从SUPER超级块中读取元数据。
	- nid_max
	- blobid_max
	- freelist_type
	- bluefs_extents
	- ondisk_format
	- min_compat_ondisk_format
	- min_alloc_size
10. 加载freelistmanager
11. 初始化allocator
12. 加载 [[bluestore#^092514|collection]]
	pg的上下文管理结构，体积小，在上电时全部预加载并常驻内存。
13. 重载性能计数器PrefCounter
14. 一致化bluefs空间
15. 创建所有线程/线程池
``` C++
void BlueStore::_kv_start()
{
  // 用于执行回调函数的finisher线程
  for (int i = 0; i < m_finisher_num; ++i) { 
    ostringstream oss;
    oss << "finisher-" << i;
    Finisher *f = new Finisher(cct, oss.str(), "finisher");
    finishers.push_back(f);
  }

  deferred_finisher.start();
  for (auto f : finishers) {
    f->start();
  }
  //用于数据同步的同步线程
  kv_sync_thread.create("bstore_kv_sync"); 
  kv_finalize_thread.create("bstore_kv_final");
}

//BlueStore::mkfs():
mempool_thread.init(); //用于监控内存使用
```
17. 日志重放
18. 成员变量mount = true

## open_db
![[opendb.drawio1.png]]
1. 打开db如果不存在则创建,打开db时，默认不进行修复操作。
2. 如果是新建DB，则从配置里面读取bluestore_kvbackend和bluestore_bluefs，否则，从元数据中读取kv_backend（对应上图kv_backend文件、bluefs文件，bluefs文件的内容为1对应程序变量do_bluefs=true）。
只考虑使用bluefs的情况：
3. 创建一个BlueFS对象
4. 判断是否有专门的path/block.db设备
	- 如果没有，则跟path/block共享，需要将path/block设备作为BDEV_DB
	- 如果有专门的path/block.db设备，则先调用bluefs->add_block_device加入BDEV_DB设备，如果设备支持标签，则将设备加上bluefs db标签，新创建时调用bluefs->add_block_extent将DB设备的extent加入的`block_all[BDEV_DB]`。
5. 加载path/block设备
	- 如果是共用情况，则将block的空间登记到BDEV_DB下面
	- 否则登记到BDEV_SLOW下面。
	Block设备的中间部分留出一定的空间给bluefs使用，这部分空间的大小inital由（bluestore_bluefs_min_ratio+bluestore_bluefs_gift_ratio）决定，并取initial和bluestore_bluefs_min的最大值，避免这部分空间小于bluestore_bluefs_min，并按最小分配单元对齐。
	需要注意的是，这部分空间除了调用add_block_extent加入之外，还会加入bluefs_extents（Bluefs_extents会持久化到super的元数据中）。
		#todo 还有哪部分空间会加入bluefs_extents? 
		目前只看到block共享的空间加进来了。
5. 处理WAL空间：
	- 如果有独立的WAL设备，则调用add_block_device将改设备空间登记到BDEV_WAL，如果设备支持标签，则设置bluefs wal标签。新建DB时加入到`block_all[BDEV_WAL]`，预留BDEV_LABEL_BLOCK_SIZE空间。
	- 如果没有独立的WAL设备，将去除DB的separate_wal_dir选项。
1. 调用bluefs->mkfs(fsid)
2. Bluefs->mount()
3. 创建Env 和设置option
4. 创建DB（KeyValueDB::create）对象得到db，并db->init(options)初始化配置项
5. 调用db->create_and_open()或db->open()打开db