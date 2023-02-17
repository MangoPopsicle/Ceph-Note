##### StupidAllocator
基于区间树，类似于Linux的[[伙伴系统（buddy system）算法]]

```c++
class StupidAllocator : public Allocator {
	CephContext* cct;
	std::mutex lock; //分配空间用的互斥锁
	int64_t num_free;     ///< total bytes in freelist 空闲空间总大小
	int64_t num_reserved; ///< reserved bytes 保留字节
	typedef mempool::bluestore_alloc::pool_allocator<
		pair<const uint64_t,uint64_t>> allocator_t; //extent: offset, length
	typedef btree::btree_map<uint64_t,uint64_t,std::less<uint64_t>,allocator_t> 
		interval_set_map_t; //有序的 btree map，按顺存放extent。
	typedef interval_set<uint64_t,interval_set_map_t> interval_set_t; // 区间树
	std::vector<interval_set_t> free;  
	///< leading-edge copy 区间树数组，初始化的时候，free数组的长度为10，即有十颗区间树
	
	uint64_t last_alloc; //最后一次分配空间的位置
	
	//... ...
}
```
![[Pasted image 20221009104152.png]]

每颗区间树的下标为index(0-9)，index(1-9)表示的空间大小为：(2^(index-1) * bdev_block_size, 2^(index) * bdev_block_size)

	free[0]: [0, 4k)
	free[1]: [4k, 8k)
	free[2]: [8k, 16k)
	free[3]: [16k, 32k)
	free[4]: [32k, 64k)
	free[5]: [64k, 128k)
	free[6]: [128k, 256k)
	free[7]: [256k, 512k)
	free[8]: [512k, 1024k)
	free[9]: [1024k, 2048k)

###### 插入到区间树
增加空闲空间
```c++
//根据区间长度，选择将要存放的区间树（确定其所属的区间树）
unsigned StupidAllocator::_choose_bin(uint64_t orig_len)

//插入空间空间到区间树
void StupidAllocator::_insert_free(uint64_t off, uint64_t len)
```
- 伙伴算法更加严格，只允许伙伴关系的区间合并
- Stupid Extent合并较为松散，只要两个Extent大小相等即可


###### 从区间树删除
空闲空间减少
一个extent被使用，在原来的extent中删除被传入的extent，然后计算剩余的extent是否落入其它区间，如果是则从此区间删除，添加到新的区间树。
被使用的extent变小，有可能不在符合此区间的大小，换到其他空间。


###### 分配和释放
**分配**
核心流程为4个2层for循环大致为：优先从hint地址依次向高级区间树开始分配长度大于等于want_size的连续空间，如果没有，则优先从hint地址依次向低级区间树开始分配长度大于等于alloc_unit的连续空间(长度会大于alloc_unit)。
![[Pasted image 20221009150230.png]]

#todo 具体的分配流程还是没搞明白，另：区间树上挂载的是否的相同大小的extent？
![[Pasted image 20221009150606.png]]

**释放**
```c++
void release(const interval_set<uint64_t>& release_set) override;
```
先加锁，然后循环调用_insert_free插入到对应区间树里面，会涉及到相邻空闲空间的合并，但是会导致分配空间碎片的问题。
