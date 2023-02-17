#ceph学习 

### 整体架构
![[Pasted image 20220923091837.png]]
#### Allocator
管理磁盘空间：bluestore通过Allocator进行裸设备管理，进而将数据持久化至裸设备空间。支持StupidAllocator和BitmapAllocator两种分配器。

#### RocksDB (kvdb)
[欢迎使用RocksDB RocksDB中文网 | 一个持久型的key-value存储](http://rocksdb.org.cn/doc/Home.html)
一个kv数据库，保存元数据（包括存储WAL、对象元数据、对象扩展属性omap、磁盘分配器元数据）、日志等信息。rocksdb是默认的kvdb，只要能提供相应接口，也可以使用其他kvdb。

> RocksDB是使用C++编写的嵌入式kv存储引擎，其键值均允许使用二进制流。由Facebook基于levelDB开发， 提供向后兼容的levelDB API。RocksDB针对直接使用SSD作为后端存储介质的场景做了大量优化

> **特点**
> 1. 专为使用本地SSD设备作为存储后端且存储容量不超过几个TB的应用程序设计，是一种内嵌式的非分布式数据库
> 2. 适合于存储小型或者中型键值对，性能随键值对长度上升下降很快。
> 3. 性能随CPU核数以及后端存储设备的I/O能力呈线性扩展。

rocksdb保留了SST（Static Sort Table? , SSTable）文件相关管理逻辑，将SST分层存储。

#### BlueFS
一个精简的用户态日志型文件系统，服务于rocksdb，实现了RocksDB::Env所定义的全部API，为rocksdb提供操作裸设备相关接口（操作系统自带的本地文件系统对rocksdb而言很多功能不是必须）。bluefs可通过调用块设备驱动将数据和日志持久化。

> rocksdb基于文件系统，不是直接操作裸设备的（无法直接操作裸设备），需要通过抽象接口Env进行。

![[Pasted image 20221128135917.png]]
BlueFS只用于存放单个bluestore实例的元数据，数据量有限且文件类型单一（主要是sst文件），只采用两类表（dir_map和file_map）来进行管理，定位一个文件只需要经过两级查找：dir_map->file_map。
>每一个dir都是绝对路径，没有隶属关系。如上图中的“/var/db”和“/var/log/ceph”只是两个不同名字的dir，两者属于同一层级。

![[Pasted image 20221116173925.png]]
超级块作为固定入口，用于索引日志所对应的存储位置。上电时，总是先通过日志重放来获取bluefs所有的元数据，此时还原出完整的dir_map和file_map。

#### BlueRocksEnv
**Env**: environment

rocksdb用来实现访问操作系统功能（如文件系统功能）的抽象接口

``` C++
// An Env is an interface used by the rocksdb implementation to access
// operating system functionality like the filesystem etc.  Callers
// may wish to provide a custom Env object when opening a database to
// get fine gain control; e.g., to rate limit file system operations.
```
BlueRocksEnv是RocksDB与BlueFS交互的接口；RocksDB提供了文件操作的接口EnvWrapper，用户可以通过继承实现该接口来自定义底层的读写操作，BlueRocksEnv就是继承自EnvWrapper实现对BlueFS的读写。


#### BlockDevice
最底层的块设备，BlueStore直接操作块设备，抛弃了本地文件系统。BlockDevice在用户态直接以linux系统实现的AIO直接操作块设备文件（使用Libaio操作裸设备）。
	%%AsyncIO 异步IO%%

在使用bluefs+rockesdb的情况下**最多有3个块设备**。
>BlueStore在设计上将元数据和用户数据严格分离，并支持分别存储到不同的设备。使用更好的设备来存储BlueStore的元数据通常可以获得更好的性能。

==**Data / SLOW**== (block path)：bluestore数据（用户业务数据/对象数据） 常用低速盘，如HDD，由BlueStore管理

==**WAL**==(wal path)：超高速空间，存放rocksdb内部内部产生的journal（日志数据，.log文件）和BlueFS自身的日志文件，常用高速盘/超高速盘，如SATA/SAS SSD、NVMe SSD、NVRAM，由BlueFS管理。
>理论上数据写入缓存即可向客户端返回写入完成应答，但是由于存在内存数据掉电后丢失的可能，将数据先写入相较普通磁盘性能更好并且掉电后不会丢失的中间设备，等待后续数据写入普通磁盘后再释放中间设备上的空间，是一种可行的替代方案。
>这个写中间设备的过渡过程称为**写日志**，中间设备称为**日志设备**，包含日志的存储系统也被称为日志型存储系统。

>涉及数据修改的相关操作，要么全部完成，要么没有变化，不能是介于两者之间的状态(All or Nothing)，保证数据一致性。符合上述语义的存储系统称为**事务型存储系统**，所有的操作都符合ACID。

==**DB**==(db path)：高速空间，用于存储BlueStore内部产生的元数据，存放rocksdb SST等，常用高速盘，如SSD，由BlueFS管理（BlueStore的元数据都交由RocksDB管理，而RocksDB最终通过BlueFS保存数据）

|  名称   | 数据类型  | 空间类型 | 设备类型 | 管理方 |
|  ----  | ----  | ----| ---- | ---- |
| Data(Slow)  | 用户业务数据 | 低速空间 | HDD | BlueStore |
| WAL | 日志数据 | 超高速空间 | NVMe SSD, NVRAM | BlueFS |
| DB | 元数据 | 高速空间 | SSD | BlueFS |

写入速度要求：日志数据 > 元数据 > 用户业务数据
>元数据设备选择优先级：DB > Slow
>日志数据设备选择优先级：WAL > DB > Slow
-   1个块设备
    **Data**，bluestore和bluefs共享这个设备
	![[2.png]]
-   2个块设备
    **Data**(bluestore数据、rocksdb SST) + **WAL**
    **Data** + **DB**(rocksdb SST + WAL)
    
-   3个块设备
    **Data** + **DB** + **WAL**
	![[1.png]]

容量需求：Slow > DB ≈ WAL
>注：DB和WAL的空间需求不是固定的，而是与Slow的使用情况密切相关。
>        - bluestore的元数据数量与对象数量、对象中数据的稀疏程度、被覆盖写的次数等相关。
>        - rocksdb的WAL的数量则与记录（即BlueStore中的元数据）数量、访问习惯、自身的压缩策略等相关。

在设计上，bluestore将自身管理的部分slow与bluefs共享，并在运行中实时监控和动态调整。![[bluefs.png]]
与bluefs共享的部分也是由bluestore直接管理，已分配的段会写入bluefs_extents集合，并且从Allocator中扣除。
```C++
interval_set<uint64_t> BlueStore::bluefs_extents
///< block extents owned by bluefs (private)
```
bluestore上电时，需要先加载bluefs_extents才能挂载bluefs。bluefs上电时，通过bluestore传递预先从数据库中加载的bluefs_extents，以正确由自身初始化这部分空间对应的Allocator。DB和WAL由bluefs自身进行管理（bluestore不可见）。
bluefs上电时会初始化三个allocator实例，以管理对应的空间。
bluefs不保存已分配/未分配空间列表，而是通过上电时遍历所有文件的元数据信息来生成完整的已用空间列表。

![[Pasted image 20220923091837.png]]
根据上图，可以很直观的了解到，BlueStore 把数据分成两条路径：
- 一条是 data 直接通过 Allocator分配磁盘空间，然后写入 BlockDevice。
- 另一条是 metadata 先写入 RocksDB，通过 BlueFs来管理 RocksDB 数据，经过 Allocator 分配磁盘空间后落入 BlockDevice。

**BlueStore的写操作综合运用直接写、COW 和 RMW 策略**
[[COW与RMW]]
%%这一部分可以参考一下书里提供的案例%%

---
第一层的观念：
BlueStore 把元数据和对象数据分开写，对象数据直接写入硬盘，而元数据则先写入超级高速的内存数据库，后续再写入稳定的硬盘设备，这个写入过程由 BlueFS 来控制。
- **非覆盖写**直接分配空间写入即可；
- **块大小对齐的覆盖写**采用 COW 策略；**小于块大小的覆盖写**采用 RMW 策略。
---
第二层的观念：
- 非覆盖写直接通过 Allocator 分配空间后写入硬盘设备。
- 覆盖写分为两种：一种是可以 COW，也是直接通过 Allocator 分配空间后写入硬盘设备。另一种是需要 RMW，先把数据写入 Journal，在 BlueStore 中就是 RocksDB，再后续通过 BlueFS 控制，刷新写入硬盘设备
![[Pasted image 20230111142706.png]]
---

### 参考
[欢迎使用RocksDB RocksDB中文网 | 一个持久型的key-value存储](http://rocksdb.org.cn/doc.html)
[BlueStore 存储引擎介绍](https://www.cnblogs.com/hukey/p/11910741.html)
[ceph源码分析BlueStore - Earfire's Blog - El Psy Congroo](http://earfire.me/post/ceph/bluestore/)
[BlueStore 架构及原理分析](https://blog.csdn.net/DeamonXiao/article/details/120866790)
[ceph核心理论_勤学-365的博客](https://blog.csdn.net/qq_23929673/category_9656388.html)
[ictfox blog (yangguanjun.com)](http://www.yangguanjun.com/)
[Ceph BlueStore 和双写问题 (qq.com)](https://mp.weixin.qq.com/s/dT4mr5iKnQi9-NEvGhI7Pg)
[tags - Emperorlu’s Site](https://emperorlu.github.io/tags/)
[Ceph中Bufferlist的设计与使用](https://blog.csdn.net/bandaoyu/article/details/113250328)
[Categories (wjin.org)](http://blog.wjin.org/categories.html)
[BlueStore-先进的用户态文件系统《二》-BlueFS - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/46362124)
[BlueStore - Justice的小站](https://justice.bj.cn/post/40.storage/ceph/ceph-bluestore/)

- BlueFS:
[Ceph BlueStore BlueFS__51CTO博客](https://blog.51cto.com/u_15265005/2888363)

SSD:
[关于SSD存储原理的介绍 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/426265253)
[Bluestore源码分析2 NVME Device - 简书 (jianshu.com)](https://www.jianshu.com/p/d27f382aa00e)

同步IO
int **KernelDevice::_ sync_write**(uint64_t off, bufferlist &bl, bool buffered)

### 数据结构

#### ObjectStore
位置：src/os/ObjectStore.h
``` c++
class ObjectStore {
protected:
    string path;
public:
    CephContext* cct; 
    
    /**
    *构造函数，在初始化时调用一次
    *
    *@param type type存储类型，这是来自配置文档的字符串，filestore bluestore memstore
    *@param data 数据的数据路径（或其他描述符）
    *@param journal 日志journal path (or other descriptor) for journal (optional)
    *@param flag 标记哪些filestore应检查是否适用
    */
    static ObjectStore *create(CephContext *cct, 
            const string& type,
            const string& data,
            const string& journal,
            osflagbits_t flags = 0);

    struct CollectionImpl : public RefCountedObject { 
	    //同一Collection下的排队的事务会顺序处理，而不同Collection下的事务存在并行处理
        const coll_t cid;
    };
    typedef boost::intrusive_ptr<CollectionImpl> CollectionHandle;

    class Transaction { //用来实现相关的事务
    private:
        TransactionData data;

        map<coll_t, __le32> coll_index; //coll_t -> coll_id的映射关系
        map<ghobject_t, __le32> object_index; //ghobject -> object_id的映射关系

        __le32 coll_id {0}; //当前分配的coll_id的最大值
        __le32 object_id {0}; //当前分配的object_id的最大值

        bufferlist data_bl; //数据
        bufferlist op_bl; //元数据操作
        //一个事务中，对应如下三类回调函数，分别在事务不同的处理阶段调用
        list<Context *> on_applied; 
        //事务应用完成之后的回调函数，在Finisher线程里异步调用执行
        list<Context *> on_commit; 
        //事务提交完成之后调用的回调函数
        list<Context *> on_applied_sync; 
        //事务应用完成之后的回调函数，同步调用执行
    };

    virtual int queue_transactions( 
    //所有ObjectStore更新操作的接口，更新相关的操作
    //(如创建一个对象，修改属性，写数据等)都是以事务的方式提交给ObjectStore。
            CollectionHandle& ch, vector<Transaction>& tls,
            TrackedOpRef op = TrackedOpRef(),
            ThreadPool::TPHandle *handle = NULL) = 0;

    virtual bool test_mount_in_use() = 0;
    virtual int mount() = 0; //加载objectstore相关的系统信息
    virtual int umount() = 0;
    virtual int mkfs() = 0;  // wipe //创建objectstore相关的系统信息
    virtual int mkjournal() = 0; // journal only

    virtual int getattr(CollectionHandle &c, const ghobject_t& oid, 
				    const char *name, bufferptr& value) = 0; //获取对象的扩展属性
    virtual int omap_get( //获取对象的omap信息
            CollectionHandle &c,     ///< [in] Collection containing oid
            const ghobject_t &oid,   ///< [in] Object containing omap
            bufferlist *header,      ///< [out] omap header
            map<string, bufferlist> *out /// < [out] Key to value map
            ) = 0;
};
```


#### BlockDevice & KernelDevice
BlockDevice定义块设备基类
KernelDevice继承于BlockDevice，对应的物理设备为HDD和SATA

>Ceph 较新版本中，把 设备 模块单独放到 blk 文件夹中。
>
>	├── aio
>	│   ├── aio.cc                    # 封装 aio 操作
>	│   └── aio.h
>	├── BlockDevice.cc                # 块设备基类
>	├── BlockDevice.h
>	├── CMakeLists.txt
>	├── kernel                        # kernel 设备
>	│   ├── io_uring.cc   # bluestore 或者 存在 libaio 的情况下，不适用 libio_uring
>	│   ├── io_uring.h
>	│   ├── KernelDevice.cc
>	│   └── KernelDevice.h


#### aio
aio 封装了libaio相关操作。
具体有三个结构：  
- **aio_t**：一次io操作，pwrite，pread，不写到磁盘，是io的最小结构体   %% aio.h %%
- **IOContext**：每个IO请求都会生成一个IO上下文，里面包含多个aio_t  %% IOContext.h %%
- **io_queue_t**：io提交队列，基本不使用，只用作基类  
- **aio_queue_t**：继承自io_queue_t，提交 io 操作，真正在此写入磁盘，用于初始化、提交、收割IO，一个裸设备一个aio队列。 %% aio.h %%
![[Pasted image 20220930174507.png]]

##### aio读写

^a3c908

- **写**(KernelDevice::aio_write)：调用io_prep_pwritev将写的数据放入buf中，此时还在内存，没有写入磁盘。
#todo aio_write分析

- **读**(KernelDevice::aio_read)：调用io_prep_pread，并开辟一块block对齐的内存，准备读数据，还没有从磁盘读。  
#todo aio_read分析

- **提交**(KernelDevice::aio_submit)：准备完读写之后，就需要调用aio_submit提交准备的IO了。
![[Pasted image 20221008144447.png]]
	 **io_submit系统调用**
	 函数原型：int io_submit(io_context_t ctx_id, long nr, struct iocb ** iocbpp) 提交异步io块进行处理
	 io_submit()系统调用将nr个io请求块排队，以便在AIO上下文ctx_id中进行处理。 iocbpp参数应为nr个AIO控制块的数组，并将其提交给上下文ctx_id。 
	 成功时，io_submit()返回提交的iocb的数量(可以小于nr，如果nr为零，则为0)。

> 错误EAGAIN: 没有足够的资源来排队任何iocb
> ![[Pasted image 20221008143338.png]]
> errno.h中定义的错误值均为正值，为避免与正常的返回值冲突，错误返回值取相反数

##### aio线程
IO提交之后就可以返回了，BlueStore起了一个单独的线程检查IO的完成情况，当真正完成的时候，执行回调函数通知调用方。

**KernelDevice::_ aio_thread**
![[Pasted image 20221008144534.png]]
#todo %%aio_thread代码分析，后面的部分还不明白用处%%


**io_getevents系统调用**
从完成队列中读取异步I / O事件
函数原型：io_getevents (io_context_t ctx_id, long min_nr, long nr,struct io_event * events, struct timespec * timeout)
尝试从ctx_id指定的AIO上下文的完成队列中读取至少min_nr个事件，最多读取nr个事件。

返回值
	成功后，io_getevents()返回读取的事件数。
	如果超时到期，则可以为0或小于min_nr的值。
	如果调用被信号处理程序中断，它也可能是一个小于min_nr的非零值。

[IO_GETEVENTS - Linux手册页-之路教程 (onitroad.com)](https://www.onitroad.com/jc/linux/man-pages/linux/man2/io_getevents.2.html)

#todo sync和discard

#### Allocator
Allocator用来分配磁盘空间，只在内存做标记，实现包含Stupid和BitMap两种，Stupid即为基于extent的方式（[[StupidAllocator]]）。*目前重点关注Bitmap的实现方式*
![[Pasted image 20221008153159.png]]

  
##### BitMapAllocator
BitmapAllocator实现了一个块粒度的内存版本磁盘空间分配器。和BitmapFreelistManager不同，因为BitmapAllocator中的所有段信息不需要使用kvdb存盘，所以可以采用非扁平方式进行组织，以提升索引效率。实现上，BitmapAllocator中的块是以树状的形式进行组织的。
>需要注意的是，因为BlueStore将不同类型的数据严格分开并且允许使用不同的设备存储，所以一个BlueStore实例中可能存在多个BitmapAllocator实例。
###### 数据结构
BitAllocator是BitMapAllocator成员，位分配器实例。
osd启动时加载kv中的记录，在内存中构造以下树结构：
	相关函数：`BlueStore::_open_alloc()`
![[Pasted image 20221008171121.png]]
树中每个节点都会统计自己子树中包含的空闲磁盘空间和已分配磁盘空间，这在分配连续大块的磁盘空间时可以跳过空间不足的子树，快速定位到剩余空间能够满足要求的子树，从而提高分配效率。但是各个节点都单独开辟内存空间，使用指针连接，使用的是离散的内存空间。
%%_ps：这种已经是ceph的旧版位分配方式，在后来的版本中对其进行的调整以达到更好的效果，并将默认分配方式改回位分配方式。13.2.5使用的仍然是旧版的位分配器_%%

>按《ceph之rados设计原理与实现》:
>![[Pasted image 20221111005354.png]]
> BitmapAreaInternal可以存在多层，相关设置可以进行修改，前图的层级表示并不完全准确。

![[AEZF47A4I22%)HPVT1DFE9K.jpg]]

该拓扑结构是可以进行配置的，主要受两个参数约束：
- **`bluestore_bitmapallocator_blocks_per_zone`**
	BitmapZone大小
	BitmapZone是单次（连续）最大的可分配单位，大小可配置，拥有独立的锁逻辑，所有API都被设计成原子的，因此不同BitMapZone之间可以并发操作。
	该参数决定了单次最大可分配空间和并发操作粒度。
```C++
OPTION(bluestore_bitmapallocator_blocks_per_zone, OPT_INT) 
// must be power of 2 aligned, e.g., 512, 1024, 2048...
```
- **`bluestore_bitmapallocator_span_size`**
	BitmapArea中单个节点（中间/叶子）的跨度 span-size
	决定了BitmapArea的高度，影响索引效率。
```C++
OPTION(bluestore_bitmapallocator_span_size, OPT_INT) 
// must be power of 2 aligned, e.g., 512, 1024, 2048...
```

此外，BitmapAllocator可以自定义最小可分配空间（分配单元，alloc unit， AU），即其可分配空间的最小粒度。通过BitmapAllocator分配空间必须是最小可分配空间的整数倍。
其值为`BlueStore::min_alloc_size`，确定方式和freelistmanager的[[# ^810892 |byte_per_block]]
一样，在`BlueStore::mkfs()`中，由配置项`bluestore_min_alloc_size`、`bluestore_min_alloc_size_ssd`、`bluestore_min_alloc_size_hdd`决定。在`allocate()`及其子类的重写函数中作为第二个参数`uint64_t alloc_unit`传递。
```C++
int64_t BitMapAllocator::allocate(
  uint64_t want_size, uint64_t alloc_unit, uint64_t max_alloc_size,
  int64_t hint, PExtentVector *extents);
```
![[Pasted image 20230109165514.png]]

###### 公共接口

- **`init_add_free()`**
	BlueStore 上电时，通过 FreelistManager 读取磁盘中空闲的段，然后调用本接 口将 BitMapAllocator 中相应的段空间标记为空闲。

- **`init_rm_free()`**
	将BitMapAllocator 指定范围的空间（`[offdset， offset + length]`）标记为已分配。

- **`reserve()` /  `unreserve()`**
	预留/释放预留空间
	因为BitmapAllocator支持多线程访问，所以通过 BitmapAllocator 进行空间分配时，需要先 调用`reserve()`接口进行空间预留，以保证后续通过`allocate()`接口能够分配到所请求的空间。
	如果通过`allocate()`分配的空间比之前`reserve()`少，那么差值需要通过`unreserve()`返还。

- **`allocate()` / `release()`**
	分配/释放空间
	分配的空间不一定是连续的，有可能是一些离散的片段。
	allocator()接口可以同时指定hint参数，用于对下次开始分配的起始地址（例如可以是上一次成功分配后返回空间的结束地址）进行预测，以提升分配速率。
	分配的结果会放在一个`PExtentVector`结构中

> allocate()调用栈：
> `BitMapAllocator::allocate()
> -> `BitMapAllocator::allocate_dis()`
>     -> `BitAllocator::alloc_blocks_dis_res()`
>           -> `BitAllocator::alloc_blocks_dis_work()`
>                 -> `BitAllocator::alloc_blocks_dis_int()`
>                      -> `BitMapAreaIN::alloc_blocks_dis_int_work()`

- **`get_free()`**
	返回当前BitmapAllocator实例中的所有空闲空间大小。







#### FreeListManager
管理空闲空间
任意时刻，已知空闲表或分配表其一，都可以推导出另外一张表，因此不需要将两张表全部存盘。BlueStore选择将空闲表存盘，系统上电时，通过加载空闲表，可以在内存中还原出完整的分配表。
>最开始有extent和bitmap两种实现，现在已经默认为bitmap实现，并将extent的实现废弃。

##### BitmapFreeListManager
该结构以块为粒度，将数量固定、物理上连续的多个块进一步组成段，从而将整个磁盘空间划分为若干连续的段进行管理。以每个段在磁盘中的起始位置进行编号可以获得一个唯一的索引，从而可以使用kvdb固化BitmapFreeListManager中的所有段数据，其中key为每个段的起始地址，value为段内块使用状态的固定长度比特流（段内块的位图）。
相关配置项：`bluestore_freelist_blocks_per_key` : 段大小，默认为128
过大过小的段大小都会影响数据库的性能。
![[bitmapfreelistmanager 2.jpg]]
```ad-question
title: 功能上和allocator有什么区别？
collapse: none
Allocator负责在内存中计算和分配空间、管理磁盘，FreeListManager记录各分配单元的空闲状态，并持久化到db中（持久化到磁盘），进程启动时，从db中加载这些信息，并按照具体分配器逻辑构建内存结构。

```


###### Create & Init
BlueStore在初始化osd的时候，会执行mkfs，初始化FreelistManager(create/init）,（第一次）在create()中固化一些meta参数（如块大小、每个段包含的块数目等）到kvdb中。后续进程重启重新上电时，会执行mount操作，init()的时候从kvdb中读取这些参数（不再执行create()函数），防止因为配置变化导致BitmapFreelistManager无法正常工作。
%% mount  v.  登/爬上，组织，开展，**装载，装入** %%
![[{91KADGWK}_(G6T3%IDGNSC.jpg]]

相关meta参数：
- **`byte_per_block`** 块大小（分配块大小）
	create()函数的第二个参数，其值为`(int64_t)min_alloc_size`
	`min_alloc_size`的值取决于配置项`bluestore_min_alloc_size`、`bluestore_min_alloc_size_ssd`、`bluestore_min_alloc_size_hdd` ^810892
	
	此处的块大小并不是物理设备的块大小，只是bitmapfreelistmanager的**分配层面的块大小**，两者并不等同。bitmapfreelistmanager的块大小由min_alloc_size确定，有可能是物理设备块的几倍，不过一般情况下两者是相等的。（包括Allocator结构中涉及的块大小，指的也是分配层面上的块大小，要注意与物理设备层面上的块大小进行区分，在源码中两者一般也都是用block_size进行指代。）
```c++
  // BlueStore::mkfs (BlueStore.cc)
  // choose min_alloc_size
  if (cct->_conf->bluestore_min_alloc_size) {
    min_alloc_size = cct->_conf->bluestore_min_alloc_size;
  } else {
    ceph_assert(bdev);
    if (bdev->is_rotational()) {
      min_alloc_size = cct->_conf->bluestore_min_alloc_size_hdd;
    } else {
      min_alloc_size = cct->_conf->bluestore_min_alloc_size_ssd;
    }
  }
```

- **`block_per_key`** 段大小(段中包含块的数量)
	取决于配置项`bluestore_freelist_blocks_per_key`
``` C++
 //BitmapFreelistManager::create (BitmapFreelistManager.cc)
 blocks_per_key = cct->_conf->bluestore_freelist_blocks_per_key
```

- **`size`** 设备总大小(byte)
	根据create()第一个参数与块大小(`byte_per_block`)对齐，第一个参数`new_size`值为`bdev->get_size()`
```C++
//BitmapFreelistManager::create (BitmapFreelistManager.cc)
size = p2align(new_size, bytes_per_block);
```

- **`blocks`** 设备包含的总块数
```c++
//BitmapFreelistManager::create (BitmapFreelistManager.cc)
blocks = size / bytes_per_block;
if (blocks / blocks_per_key * blocks_per_key != blocks) {
blocks = (blocks / blocks_per_key + 1) * blocks_per_key;
dout(10) << __func__ << " rounding blocks up from 0x" << std::hex << size
	 << " to 0x" << (blocks * bytes_per_block)
	 << " (0x" << blocks << " blocks)" << std::dec << dendl;
// set past-eof blocks as allocated
_xor(size, blocks * bytes_per_block - size, txn);
}
```

从日志中可以看到meta参数的值及相关函数的输出内容：
![[Pasted image 20221109171241.png]]
![[3e792760a0277bbaaae9702193ca180.jpg]]

block/key掩码的生成，将在allocate和release中发挥作用：
![[Pasted image 20221110221213.png]]

###### Allocate & Release
- `allocator()`从BitmapFreelistManager中分配指定范围（`[offset, offset+length]`）空间
- `release()`从BitmapFreelistManager中释放指定范围（`[offset, offset+length]`）空间

分配和释放操作一样，都是将段的bit位和当前的值进行异或运算。
相关实现在`BitmapFreelistManager::_xor()`中，release()和allocate()均调用此函数。
结合日志提供的信息对代码进行分析：
```c++ 
//BitmapFreelistManager.cc
void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length, 
  KeyValueDB::Transaction txn)
  //假设 简单情况：offset=0x 0001 0000 , length=0x 0001 0000 占用一个block
  //     复杂情况：offset=0x 007F 0000 , length=0x 0082 0000 占用130个block
  //				 分别位于第一段的最后一块、整个第二段、第三段的第一块
{
  // must be block aligned
  // offset和length都必须以块为边界对齐
  ceph_assert((offset & block_mask) == offset);   
  ceph_assert((length & block_mask) == length);   

  uint64_t first_key = offset & key_mask;   			
  //起始段位置 简单：0x00		复杂：0x00
  uint64_t last_key = (offset + length - 1) & key_mask;	
  //终止段位置 简单：0x00		复杂：0x100 0000
  dout(20) << __func__ << " first_key 0x" << std::hex << first_key
	   << " last_key 0x" << last_key << std::dec << dendl;
  
  // 最简单的情况，对应一个段的操作 
  //e.g. offset=0x 1 0000 length=0x 1 0000 占用一个block
  if (first_key == last_key) { 
    bufferptr p(blocks_per_key >> 3); 
    // 16字节大小的buffer  128 >> 3 = 16 
    //(一个段内用16个字节，共128位表示block的使用状态)
    p.zero();  // 全置为0 0x00
    unsigned s = (offset & ~key_mask) / bytes_per_block;  
    // 段内开始block的编号 =1
    unsigned e = ((offset + length - 1) & ~key_mask) / bytes_per_block; 
    // 段内结束block的编号 =1
    for (unsigned i = s; i <= e; ++i) { // 生成此次操作的掩码
      p[i >> 3] ^= 1ull << (i & 7);  	
      // i>>3定位block对应位的字节 (1>>3=0，相应的block状态位于第一个字节中)
      // 1ull<<(i&7)定位bit，然后异或将位设置位1 
      // (1&7=1,相应的block状态位于第一个字节的第二位，
      //  类似地，11&7=3，相当于除8求余数。
      //  1ull<<1,将1ull左移相应的位数，表示该位的状态将改变，
      //  然后对该字节进行异或运算，将该位置1。
      //  之所以还要进行异或运算，是因为这是一个循环，将所有发生变化的位都置1。)
      // ps. ULL后缀：unsigned long long 64位
    }
    string k;
    make_offset_key(first_key, &k);		
    // 将内存内容(起始段位置)转换为16进制的字符
    bufferlist bl;
    bl.append(p);
    dout(30) << __func__ << " 0x" << std::hex << first_key 
	       << std::dec << ": ";
    bl.hexdump(*_dout, false);
    *_dout << dendl;
    txn->merge(bitmap_prefix, k, bl);  	
    // 和目前的value进行异或操作，更新kvdb中的内容
  } 
  
  //对应多个段，分别处理第一个段，中间段，和最后一个段，
  //首尾两个段和前面情况一样，操作和前面类似，只有范围有区别
  // e.g. offset=0x 007F 0000 , length=0x 0082 0000 占用130个block
  else {
    // first key
    {
      bufferptr p(blocks_per_key >> 3);
      p.zero();
      unsigned s = (offset & ~key_mask) / bytes_per_block; 	
      //段内开始block的编号 =127
      unsigned e = blocks_per_key;		
      //段内结束block的编号 =block_per_key = 128 (直到段的最后)
	  // ... ... 
      first_key += bytes_per_key; // = 0x80 0000
    }
    // middle keys
    while (first_key < last_key) {// 0x0080 0000 < 0x0100 0000 
	  // ... ...
      txn->merge(bitmap_prefix, k, all_set_bl); //全部进行异或
      first_key += bytes_per_key;
    }
	// last key
    ceph_assert(first_key == last_key);
    {
	  // ... ...
      unsigned e = ((offset + length - 1) & ~key_mask) / bytes_per_block; 
      // 段内结束block的编号 = 0
      // ... ...
    }
  }
}

```
对应的截图：
![[Pasted image 20221110221451.png]]

merge实现：
```C++
//BitmapFreelistManager.cc(XorMergeOperator::merge())
void merge(
	const char *ldata, size_t llen,
	const char *rdata, size_t rlen,
	std::string *new_value) override {
	ceph_assert(llen == rlen);
	*new_value = std::string(ldata, llen);
	for (size_t i = 0; i < rlen; ++i) {
		(*new_value)[i] ^= rdata[i]; //按位异或
	}
}
```
分配和释放基本全是位操作，可以关注简单情况的实现，涉及多个段的复杂情况只是对不同段及块的范围分别进行了处理，具体实现是一样的。

###### Reset & Next
接口：`enumerate_reset()` & `enumerate_next()`
上电时，BlueStore通过这两个接口遍历BitmapFreelistManager中所有空闲段，并将其从Allocator中同步移除，从而还原得到上一次下电时Allocator对应的内存结构。


#### 对象 / Onode

主要定义在BlueStore.h和bluestore_types.h
![[Pasted image 20221028101755.png]]
BlueStore的每个对象对应一个Onode结构体，每个Onode包含一张extent-map，extent-map包含多个extent(lextent)，每个extent负责管理对象内的一个逻辑段数据并且关联一个Blob，Blob包含多个pextent，最终将对象的数据映射到磁盘上。

[Ceph BlueStore Write Analyse | ictfox blog (yangguanjun.com)](http://www.yangguanjun.com/2018/09/06/ceph-blueStore-write-analyse/)
##### 📌Onode
代表一个Object（一个bluestore对象），通过Onode里的ExtentMap来查询Object数据到底层的映射。
```C++
struct Onode {  
    Collection *c;              // 对应的Collection，对应PG  
    ghobject_t oid;             // Object信息  
    bluestore_onode_t onode;    // Object存到kv DB的元数据信息  
    ExtentMap extent_map;       // 映射lextents到blobs  
};

struct bluestore_onode_t { //bluestore对象
  uint64_t nid = 0;   // 逻辑标识，单个bluestore实例内唯一
  uint64_t size = 0;  // 对象大小
  map<mempool::bluestore_cache_other::string, bufferptr> attrs;         
			          // 拓展属性对
  //客户端下发的访问提示信息，用于优化对象的读、写、压缩控制等策略
  uint32_t expected_object_size = 0;
  uint32_t expected_write_size = 0;
  uint32_t alloc_hint_flags = 0;

  uint8_t flags = 0; //对象关联的omap是否使用

  struct shard_info { //extent_map的分片
    uint32_t offset = 0;  // 分片逻辑对应的起始地址
    uint32_t bytes = 0;   // 分片编码后的长度
  };
  vector<shard_info> extent_map_shards; // 对象关联的分片概要信息
};

struct ExtentMap {  
    Onode *onode;                   // 指向Onode指针  
    extent_map_t extent_map;        // Extents到Blobs的map  
    blob_map_t spanning_blob_map;   // 跨越shards的blobs  
   
    struct Shard {  
        bluestore_onode_t::shard_info *shard_info = nullptr;  
        unsigned extents = 0;  ///< count extents in this shard  
        bool loaded = false;   ///< true if shard is loaded  
        bool dirty = false;    
        ///< true if shard is dirty and needs reencoding  
    };  
     mempool::bluestore_cache_other::vector<Shard> shards;    ///< shards  
};
```

>**ExtentMap**
>ExtentMap的主要成员extent_map_t是Extent的set集合（有序关联式容器），是有序的（但可能不连续）。其顺序由Extent的logical_offset确定。
>它同时提供了分片功能，防止文件在碎片化严重、ExtentMap很大时，防止生成的校验和过大，影响其在数据库中的索引效率。
>ExtentMap会随着写入数据的变化而变化，连续的小段会合并为大，覆盖写也会导致ExtentMap分配新的Blob。

建立上层对象到onode的映射关系后，可以通过对象名索引到onode。
```ad-info
title: 两种对象概念的区分
collapse: closed
bluestore看到的对象和上层看到的对象（所谓ceph中的对象），两者实际上并不完全相同。

bluestore中所有对象相关的元数据都用kvdb存储，bluestore将上层对象进行转义后的对象名作为kvdb表中的唯一索引存储（二者是一一对应的？只是放在不同层次上的一个概念）。

```

onode包含四个部分：数据、扩展属性、omap头部、omap条目
其中omap存储的内容和文件系统中的扩展属性类似，两者的不同有：
- 位于不同的地址空间（使得两者可以拥有相同的键但不同的值）。
- 一般文件系统对扩展属性有长度限制，omap没有。从这个角度看，扩展属性适合保存少量小型属性对，omap适合保存大量大型属性对。
基于以上内容，bluestore中扩展属性和onode是一起保存的，omap则分开保存。
![[Pasted image 20230110174741.png]]


 
##### 📌Extent  (lextent)
logical extent，逻辑段，**对象内的基本数据管理单元**，数据压缩、数据校验、数据共享等功能都是基于Extent粒度实现的。这里的Extent是对象内的，并不是磁盘内的，持久化在RocksDB。
```C++
struct Extent : public ExtentBase {  
uint32_t logical_offset = 0;    // 对应Object的逻辑偏移  
uint32_t blob_offset = 0;       // 对应Blob上的偏移  
uint32_t length = 0;            // 数据段长度  
BlobRef  blob;                  
	// 指向对应Blob的指针
	// 负责将逻辑数据段映射到磁盘
};
```

> [!NOTE] 特别说明
> 将extent抽象化，它应该包含以下三种信息：
> 1.**offset** 对象内逻辑偏移，从0开始编址
> 2.**length**  逻辑段长度
> 3.**data** 逻辑段包含的数据，为抽象数据类型
> 以上offset和length都可以只用数字表示，data中存储着extent包含的数据。但是为了实现逻辑与物理空间的映射，以及满足数据校验和压缩的需求，还需要一些其他的参数以保证这些功能的实现，因此data并不能只负责存储具体的数据内容。Extent中的blob结构就是为了满足以上这些需求（但不止于此）。

Extent会根据`block_size`对齐，其中`block_size` <= `length` <= `max_blob_size`，即不能超过blob的大小，也不能小于最小物理块的大小。

逻辑段与磁盘上的物理空间是一对多的关系。
> 对象的组织形式效仿了文件，基于逻辑段（逻辑块进行）
> #Q Onode或文件系统，物理分段的情况下，为什么要使用逻辑段？
> 具体文件系统所操作的基本单位是逻辑块，只在需要进行I/O操作时才进行逻辑块到物理块的映射，避免了大量的I/O操作，因而文件系统能够变得高效。
> ➡避免进行操作/管理的数据过大或者过小划分的管理单位

extent中逻辑段与物理段之间的关系：
![[extent.png]]
pextent的offset和length必须是块大小对齐的，而与pextent不同，lextent的offset和length则有可能不是块大小对齐的，但是由于pextent物理空间的强制约束，使得lextent的logical_offset和对应的物理起始地址之间可能产生一个偏移。相应地，如果extent的length不是块大小对齐的，则extent的逻辑结束地址和对应的物理结束地址也可能产生一个偏移。

##### 📌Blob
```C++
struct Blob {  
    int16_t id = -1;  
    SharedBlobRef shared_blob;      // 共享的blob状态  
    mutable bluestore_blob_t blob;  // blob的元数据  
};  
struct bluestore_blob_t {  
    PExtentVector extents;          // 对应磁盘上的一组数据段  
    uint32_t logical_length = 0;    // blob的原始数据长度  
    uint32_t compressed_length = 0; // 压缩的数据长度  
};
```
Blob是BlueStore里引入的处理块设备数据映射的中间层，包含磁盘上物理段的集合/数据结构blue_pextent_t的集合，可以由多段不连续的pextent组成（每个Blob会对应一组 PExtentVector，它就是 bluestore_pextent_t 的一个数组，指向从Disk中分配的物理空间）。
Blob中pextent的个数最多为：`max_blob_size / min_alloc_size`

Blob真正负责执行用户数据与磁盘空间之间的映射。
Onode申请的一组磁盘空间，可以被多个lextent映射；一个blob的磁盘空间可以由一段或多段不连续的pextent组成。


##### 📌pextent
physical extent，一段连续的物理磁盘空间。
```C++
struct bluestore_pextent_t {
  uint64_t offset = 0; // 磁盘上的物理偏移
  uint32_t length = 0; // 长度
}
```
pextent中的offset和length必须是块大小对齐的。
（pextent并不是磁盘上的最小分配空间/不是块大小/不是固定长度）
无法保证为每个lextent分配物理上也连续的一段空间，逻辑段与磁盘上的物理空间应该是一对多的关系。
pextent是lextent在磁盘上无法连续分配产生的物理段，并非磁盘读写/分配单位，每段大小可能不同。

#Q ==lextent是min_alloc_len的整数倍，但不一定块大小对齐。pextent是块大小对齐，那是min_alloc_size的整数倍吗？==
通过wirte_small的流程来看，

##### Onode内部结构关系图
![[onode代码结构.jpg]]


![[onode结构.drawio2 1.png]]

![[onode-disk.jpg]]

![[Pasted image 20221229145919.png]]





#### 缓存
[Ceph bluestore中的缓存管理_lizhongwen1987的博客](https://blog.csdn.net/lzw06061139/article/details/105656938)

##### 缓存对象
需要缓存的对象：
- 元数据
	- Collection PG上下文的内存管理结构（Cnode） ^092514
	- Onode 对象上下文的内存管理结构（[[#对象 / Onode]]）
	- *osdmap等...*
- 数据：object data对应的buffer

类似于页面置换算法，bluestore使用了两种Cache淘汰算法：==LRU==(Least Recently Use, 最近最久未用)和==2Q==(two queue(s))。
collection体积较小，采用全部缓存，不存在淘汰算法。
onode和data采用部分缓存，部分缓存存在淘汰算法。其中onode等元数据（如osdmap等）采用LRU（bluestore中，即使是在2Q的实现里，元数据的处理也只使用了LRU相关的队列，即只使用了LRU而并未使用2Q），data采用2Q。

**对象的元数据（onode）**
一个onode唯一归属一个collection，两者是多对一的关系。为了方便PG(C ollection)级别的操作，建立Collection和Onode的中间结构OnodeSpace，作为Collection的成员，便于查找属于其的onode。
![[Pasted image 20221102102838.png]]
>关于`Cache *cache`：
>每个bluestore包含多个cache实例。由于不同PG之间的客户端请求可以并发处理，为了提升性能，每个osd会设置多个pg工作队列（op_sharddwq包括多个子队列），bluestore中的cache实例个数与之对应。

**对象的数据（data）**
Blob是真正负责用户数据与磁盘空间之间映射的结构，因此基于Blob引入一个中间结构BufferSpace，作为连接用户数据（Blob）与Cache之间的桥梁。([[#^195118]])
![[Pasted image 20221102150601.png]]
默认情况下数据不会进行缓存。
当数据写完时，会根据标记决定是否缓存。`BlueStore::BufferSpace::finish_write`
在函数`BlueStore::BufferSpace::_finish_read()`中，根据flag确定是否加入缓存。
#todo *还没找到flag具体是根据哪个结构或配置项确定的，目前看到的就是将该值作为参数传入的。*
```C++
//根据标志确定是否加入缓存
if (b->flags & Buffer::FLAG_NOCACHE) {
  writing.erase(i++);
  ldout(cache->cct, 20) << __func__ << " discard " << *b << dendl;
  buffer_map.erase(b->offset);
} else {
  b->state = Buffer::STATE_CLEAN;
  writing.erase(i++);
  b->maybe_rebuild();
  b->data.reassign_to_mempool(mempool::mempool_bluestore_cache_data);
  cache->_add_buffer(b, 1, nullptr); //加入缓存
  ldout(cache->cct, 20) << __func__ << " added " << *b << dendl;
}
```

在函数`BlueStore::_do_read()`中根据设置决定读取后是否使用缓存，相关配置项为`bluestore_default_buffered_read`
```C++
//BlueStore.cc: BlueStore::_do_read()
// generally, don't buffer anything, unless the client explicitly requests
// it.
// 设置是否缓存
bool buffered = false;
if (op_flags & CEPH_OSD_OP_FLAG_FADVISE_WILLNEED) {
dout(20) << __func__ << " will do buffered read" << dendl;
buffered = true;
} else if (cct->_conf->bluestore_default_buffered_read []&&
	 (op_flags & (CEPH_OSD_OP_FLAG_FADVISE_DONTNEED |
		  CEPH_OSD_OP_FLAG_FADVISE_NOCACHE)) == 0) {
dout(20) << __func__ << " defaulting to buffered read" << dendl;
buffered = true;
}
```
加入缓存是在`BlueStore::BufferSpace::did_read()`中
```C++
//BlueStore.cc: BlueStore::_do_read()
if (buffered) {
  bptr->shared_blob->bc.did_read(bptr->shared_blob->get_cache(),
				 req.r_off, req.bl);
}

//BlueStore.h: struct BufferSpace
void did_read(Cache* cache, uint32_t offset, bufferlist& bl) {
  std::lock_guard<std::recursive_mutex> l(cache->lock);
  Buffer *b = new Buffer(this, Buffer::STATE_CLEAN, 0, offset, bl);
  b->cache_private = _discard(cache, offset, bl.length());
  _add_buffer(cache, b, 1, nullptr); //加入缓存
}
```

Buffer存在三种状态：
![[onodespace 1.jpg]]

##### 2Q算法

>《Ceph之RADOS设计原理与实现》p99
 >![[Pasted image 20221102144402.png]]

与三个LRU队列对应的三个TwoQCache类成员：
```c++
buffer_list_t buffer_hot;      ///< "Am" hot buffers
buffer_list_t buffer_warm_in;  ///< "A1in" newly warm buffers
buffer_list_t buffer_warm_out; ///< "A1out" empty buffers we've evicted
```
![[R)V$}_`Y26F4CSOOZXU0MWW.jpg]]
1. 新加入的数据总是进入A1in队列。如果在A1in队列中被不断访问到，则按照LRU规则不断提升到A1in队首，没有热度提升，因为这些访问可能是相关的。
2. A1in队列中正常淘汰的数据进入到A1out队列中。
3. 数据在A1out中被再次访问到时，进入Am队列。这是一个热度提升操作，认为这种访问与之前的访问不再是相关的。
4. Am中的数据被再次访问到时会按照LRU规则将该数据提升到队首。
5. Am队列中正常淘汰的数据进入到A1out队列中。

三种队列各司其职，均按照规则各自插入、提升和淘汰数据，可以分开来看。
	A1in和Am队列数据来源不同，前者来自第一次被访问的数据，后者来自在A1out中命中的数据。各自队列中的数据均严格按照LRU算法进行顺序调整，淘汰的数据均进入A1out中。
	A1out接收来自A1in和Am淘汰的数据。无论来源，再次命中的数据进入到Am队列中，或直至正常淘汰。

> [!NOTE] #Summary
> - osd启动的时候，提供参数初始化BlueStore的cache分片大小，供后续pg对应的collection使用。osd从磁盘读取collection信息，将pg对应的collection全部加载到内存，并分配一个负责缓存的cache给collection。
> - 元数据缓存：执行对象操作的时候，会首先读取Onode元数据信息并将其加入缓存管理。
> - 数据缓存：写入对象的时候，会根据标志，将对象数据的Buffer加入缓存管理。
> - Onode/Buffer等对象统一用内存池分配，后台线程定期检查内存使用情况，并将超出的部分trim掉。
> - *另：虽然TwoQCache内部对Onode和Buffer分开进行管理，但是整个缓存空间是共享的，因此需要为两者指定一个合适的分割比例。*

#### BlueFS
[BlueStore-先进的用户态文件系统《二》-BlueFS - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/46362124)
![[Pasted image 20230110175129.png]]
BlueFS主要有三部分数据，superblock、journal、以及data。superblock主要存放BlueFS的全局信息以及日志的信息，其位置固定在BlueFS的头部；journal中存放日志记录，一般会预分配一块连续区域，写满以后从剩余空间再进行分配；data为实际的文件数据存放区域，每次写入时从剩余空间分配一块区域。




### 处理流程

#### 初始化: mkfs & mount

![[Bluestore-初始化-mkfs流程.docx]]

[[BlueStore mkfs & mount]]
![[未命名绘图.drawio.png]]
>还有个 is_rotational 是用来判断设备是否为HDD，HDD 支持覆盖写，所以是不用discard的。 在一般linux系统中，可通过读取/sys/block/sdX/queue/rotational 来判断sdX是否为HDD盘，对应源码src/common/blkdev.cc block_device_is_rotational

由于现有文件系统管理大量文件性能不好，并且在存储系统中，往往不需要一个完整语义的文件系统，所以基于裸盘来实现一个objectstore。跟文件系统类似，基于裸盘来管理数据，需要对磁盘空间进行管理，包括分配和回收，也有元数据需要管理。元数据管理这里用到了rocksdb。rocksdb 在设计时就考虑了允许用户自定义后端的存储，只需要实现他的Env接口即可。所以bluestore 基于裸盘实现了一个bluefs，并在此基础上实现了rocksdb的Env接口跟rocksdb对接。bluefs里面看到的文件是很少的，就是rocksdb里面的sst、manifest等文件。bluefs 会用到一些空间，包括WAL、DB、SLOW。WAL、DB的空间是单独管理的，SLOW的空间是通过freelistmanager管理。rocksb 也可能会用到slow的空间，这部分空间放在设备的中间部分。freelistmanager 维护了设备的空间使用情况，设备按照min_alloc_size划分，记录各个单元的是空闲还是在用。bitmapfreelistmanager是用位图来记录空间使用情况。有了这个list后，有各种分配器对应不同的分配策略。如bitmapallocator, stupid, bit 等等，这些暂时了解一个bitmap就行。针对不同的设备类型，提供了不同的IO接口实现。如kernedevice, pmem, nvme， 这种也是了解一个就行，其他都差不多。

#### read
[[BlueStore read]]

![[bluestore_do_read.drawio.png]]

合并相邻的blob区域：
![[Pasted image 20221214143728.png]]
![[do_read.png]]
```C++
//BlueStore::_do_read() (BlueStore.cc)
{// merge regions 合并区域
  uint64_t r_off = b_off; // 要读取数据在blob内的偏移
  uint64_t r_len = l;     // 在本次extent内读取的大小
  uint64_t front = r_off % chunk_size; //求余结果，为了对齐在前方多读的长度
  if (front) { // 若r_off与块大小不对齐
	r_off -= front; //从第一个最小可读块的起始地址读起  在前面对齐 
	r_len += front; //同时调整读取大小
  }
  unsigned tail = r_len % chunk_size; //为了对齐在后方多读的长度
  if (tail) { // 若r_len与块不对齐
	r_len += chunk_size - tail; 
	//读到最后一个最小可读块的结束地址止 在后面对齐 (chunk_size-tail 多读的长度)
  }
  
  //可能会在后面的循环中出现会同一个blob的信息的情况，
  //此时需要将这些要读取段的信息合并到blob的待读列表中
  bool merged = false; 
  regions2read_t& r2r = blobs2read[bptr]; 
  //当前blob对应的regions2read (list<read_req_t>) 
  if (r2r.size()) { //该blob已经在前面的循环出现过
	read_req_t& pre = r2r.back(); //上一个(最后的)read request
	if (r_off <= (pre.r_off + pre.r_len)) { 
	//对齐后的起始偏移量小于等于同blob内上一个read request的结束位置(有重合部分)
	  front += (r_off - pre.r_off); //头部多读部分(front)扩展到上一个的起始位置
	  pre.r_len += (r_off + r_len - pre.r_off - pre.r_len); 
	  //上一个read request的读取长度扩展到本段的结束位置
	  pre.regs.emplace_back(region_t(pos, b_off, l, front)); 
	  //在上一个read request的regs(原始读取区域)尾部生成一个元素
	  //pos为要读取数据在对象内开始的逻辑地址(去掉空洞)，
	  //b_off为要读取数据在blob内的偏移，l为在本次extent内读取的大小
	  //front为到整个连续区域的起始位置的距离
	  merged = true;
	}
  }
  if (!merged) { //如果不合并，直接新增
	read_req_t req(r_off, r_len);
	req.regs.emplace_back(region_t(pos, b_off, l, front));
	r2r.emplace_back(std::move(req)); 
  }
}
```




#### write
[[ceph存储引擎bluestore解析]]
[ceph bluestore源码分析：非对齐写逻辑_z_stand的博客](https://blog.csdn.net/Z_Stand/article/details/97160702)
[[BlueStore write]]

![[Pasted image 20221009155649.png]]


###### 事务状态机
[BlueStore写流程事务实现梳理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/387927597)
[BlueStore源码分析之事物状态机 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/139063574)

[[事务状态机]]






