#ceph学习

对象 Object是默认4MB大小的数据块，在代码实现中，有 object、sobject、hobject、ghobject等不同的类名。

### object
```C++
//定义在object.h中
struct object_t {
	string name;  //文件/对象名，代表一个文件
}
```

### soject
```C++
//定义字object.h中
struct sobject_t {
	object_t oid;
	snapid_t snap;  //snap号，用于标识该对象是否为快照对象
}
```
soject在object_t的基础上增加了snap变量，snap就是该对象对于snapshot的snap号，用于标识一个对象是否为快照对象，如果一个对象不是快照，那么这个对象的snap字段就设置为`CEPH_NOSNAP`，ceph称之为head对象。
```ad-quote
title: snap-id
collapse: closed
作为文件系统中最经典地数据备份机制，快照和克隆对所有的存储系统而言几乎都是必备功能。在将一切客户端数据对象化之后，很容易理解ceph的快照和克隆功能，其实现粒度也应当是对象级的。因此为了区分原始对象的和由其产生的一众克隆对象，必须通过向对象添加全局唯一的快照标识（`snap-id`）加以解决。

>这里的全局唯一并非同存储池标识一样要求全局唯一，而是取决于快照产生的粒度，例如快照是存储池级别的，那么快照标识只要做到存储池内唯一即可。当然也可以有更小粒度的快照，例如卷（image）级别的快照。
```


### hobject
(hash object)
头文件：hobject.h
```C++
//定义在hobject.h中
struct hobject_t {
public:
  static const int64_t POOL_META = -1;
  static const int64_t POOL_TEMP_START = -2; // and then negative
  object_t oid;   //对象的名字
  snapid_t snap;  //snap号
private:
  uint32_t hash;  //hash和key不能同时设置，一般hash为pg的id值
  bool max;
  uint32_t nibblewise_key_cache;
  uint32_t hash_reverse_bits;
public:
  int64_t pool;   //所在pool的id
  string nspace;  //一般为空，用于标识特殊对象
private:
  string key;  //对象的特殊标记
};
```
hobject是hash object的缩写。



### ghobject
```C++
//定义在hobject.h中
struct ghobject_t {
	hobject_t hobj;
	gen_t generation;    //用于记录对象的版本号
	shard_id_t shard_id; //用于标识对象所在osd在EC类型下PG中的序号
	bool max;
public:
	  static const gen_t NO_GEN = UINT64_MAX;
};
```
ghobject_t 在对象 hobject_t的基础上，添加了generation字段 和shard_id字段，这个用于 EC（Erasure Code，纠删码）模式 下的PG
- shard_id用于标识对象所在的osd在EC类型的PG中的序号，对应EC来说，每个osd在PG中的序号在数据恢复时非常关键；如果时Replicate（多副本）类型的PG，那么字段就设置为 `NO_SHARED`（=-1），该字段对于replicate是没用的
- generation用于记录对象的版本号；当PG为EC时，写操作需要区分写前后两个版本的object，写操作保存对象的上一个版本（generation）的对象，当EC写失败时，可以rollback到上一个版本
```ad-cite
title:shard-id和generation
collapse: closed
在宣布支持纠删码这种数据备份机制后，ceph的数据强一致性设计必然要求其支持对象粒度的数据回滚功能，为此还必须能够区分某个纠删码对象的当前版本和由其产生的一众历史版本。类似地，这通过向对象中添加分片标识（shard-id）和对象版本号（generation）加以解决。
```






参考
[Object#yyds干货盘点#_仗剑走天涯 -- 石破天的技术博客_51CTO博客](https://blog.51cto.com/u_11495268/5004386)