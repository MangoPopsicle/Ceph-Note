## RocksDB数据文件
rocksdb的数据文件类型：
- **.sst**：rocksDB中存放数据的file。
- **CURRENT**：保存了最新的manifest文件信息。
- **IDENTITY**：db的uuid。
- **MANIFEST**：保存了rocksDB中LSM结构的变化日志。
- **LOCK**：db锁持有信息，同时只允许有一个进程打开db。
- **OPEIONS**：db的配置属性。
- **.log**：rocksdb数据写时记录的wal数据。

## 数据结构

#### bluefs_extent_t
```C++
class bluefs_extent_t {
public:
  //物理磁盘的位移和长度，代表块设备的一个存储区域
  uint64_t offset = 0; //BlockDevice的物理地址
  uint32_t length = 0; //长度
  uint8_t bdev; //属于哪个Block device
};
```

#### bluefs_fnode_t
```C++
struct bluefs_fnode_t { //文件的inode
  uint64_t ino;		//fnode编号，唯一标识一个fnode
  uint64_t size; 	//文件大小
  utime_t mtime;	//修改时间
  uint8_t prefer_bdev; 	//优先使用哪个block device
  mempool::bluefs::vector<bluefs_extent_t> extents; 
  //文件对应的磁盘空间 //磁盘上的物理段集合
  uint64_t allocated; //文件实际占用的空间大小 extents的length之和，应该<=size
};
```

#### bluefs_super_t
记录文件系统的全局信息，也是文件系统加载的入口，其位置固定存放于BlueFS的第二个block。  其中的log_fnode为journal的文件。
```C++
//超级块 类似于文件索引超级块，此处用于索引日志
//（日志本身采用一个单独的文件进行保存）
struct bluefs_super_t { 
  uuid_d uuid;      
	  ///< unique to this bluefs instance 唯一的 bluefs关联的uuid
  uuid_d osd_uuid;  
	  ///< matches the osd that owns us bluefs关联的osd对应的uuid
  uint64_t version; //版本 超级块当前版本
  uint32_t block_size;//块大小 DB/WAL设备块大小，固定为4kb

  bluefs_fnode_t log_fnode; //记录文件系统日志的文件 日志文件对应的fnode
};
```

#### bluefs_transaction_t
文件系统操作，需要封装成事务记录在日志中。
journal实际上是一种特殊的文件，其fnode记录在superblock中，而其他文件的fnode作为日志内容记录在journal文件中。  
journal是由一条条的事务transaction组成。
```c++
struct bluefs_transaction_t { //日志事务的磁盘数据结构
  typedef enum {
    OP_NONE = 0,
    OP_INIT,        ///< initial (empty) file system marker

	// 给文件分配和释放空间
    OP_ALLOC_ADD,   ///< add extent to available block storage (extent)
    OP_ALLOC_RM,    ///< remove extent from availabe block storage (extent)

	// 创建和删除目录项
    OP_DIR_LINK,    ///< (re)set a dir entry (dirname, filename, ino)
    OP_DIR_UNLINK,  ///< remove a dir entry (dirname, filename)

	// 创建和删除目录
    OP_DIR_CREATE,  ///< create a dir (dirname)
    OP_DIR_REMOVE,  ///< remove a dir (dirname)

	// 文件更新
    OP_FILE_UPDATE, ///< set/update file metadata (file)
    OP_FILE_REMOVE, ///< remove file (ino)

	// bluefs日志文件的compaction操作
    OP_JUMP,        ///< jump the seq # and offset
    OP_JUMP_SEQ,    ///< jump the seq #
  } op_t;

  uuid_d uuid;          ///< fs uuid //日志事务归属bluefs实例对应的uuid
  uint64_t seq;         ///< sequence number //全局唯一序列号
  bufferlist op_bl;     ///< encoded transaction ops 
  //编码后的日志事务条目，可以包含多条，每个条目由操作码和操作涉及的相关数据组成
};
```


## 参考
[Ceph BlueStore_easonwx的博客-CSDN博客](https://blog.csdn.net/easonwx/category_11932276.html)



## 写文件

[ceph bluefs 写操作 源码解析_帮我起个网名的博客-CSDN博客_ceph bulefs](https://blog.csdn.net/u014104588/article/details/87886764)

创建一个新的可写文件：
`BlueRocksEnv::NewWritableFile()`
-> `BlueFS::open_for_write() `

写缓存：
`BlueRocksWritableFile::Append()`
-> `BlueFS::FileWriter::append()`
    -> `buffer_appender.append(buf, len)` 写数据
    -> `buffer.claim_append(bl)` 写（BlueFS）日志，内部使用

将缓存刷到磁盘：
`BlueRocksWritableFile::Sync()`
-> `BlueFS::fsync()`
    -> `BlueFS::_fsync()`
         -> `BlueFS::_flush()`
              -> `BlueFS::_flush_range()` 此步提交异步写请求
         -> `BlueFS::_flush_bdev_safely()` 等待请求返回
         -> `BlueFS::_flush_and_sync_log()`
              -> `BlueFS::_flush()`