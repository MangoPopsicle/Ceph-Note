[BlueStore IO 流程代码梳理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/362465290)

### 入口与调用栈

包括write在内的所有涉及数据修改的操作，都需要通过queue_transactions接口。queue_transactions是store统一的入口，各个存储引擎如filestore、kvstore、bluestore都得实现这个入口。
```ad-note
title: 补充：Transaction
collapse: close
Transaction 是bluestore 中用于保证pg上的事务，一批有顺序的操作的集合。Transaction中变量的意义及用途：

- **Op**

代表一次操作
![[Pasted image 20230111164308.png]]


- **TransactionData**

保存transaction的一些关键参数，如ops（操作数量）largest_data_len（最大数据长度）等。
![[Pasted image 20230111164335.png]]


```

queue_transactions()中的处理逻辑如下：
1. 收集transaction中的三种回调函数
```C++
  ObjectStore::Transaction::collect_contexts(
    tls, &on_applied, &on_commit, &on_applied_sync);
```
2. 根据ch（collection的侵入式指针）参数获取collection，然后获取collection对应的OpSequencer。
```C++
  Collection *c = static_cast<Collection*>(ch.get()); 
  OpSequencer *osr = c->osr.get();
```
3. 创建TransContext，关联回调函数。
![[Pasted image 20230112095851.png]]
4. 向TransContext添加事务统计操作数、字节数。遍历事务列表，调用`_txc_add_transaction()`函数，解析Transaction操作，根据op类型调用对应的函数。
![[Pasted image 20230112095827.png]]

5. 计算每条txc的大致消耗，便于后续的限流操作（写抑制）。
6. 更新元数据。
7. 对于延迟写，提交事务到kvdb中。
![[Pasted image 20230112095716.png]]
8. 更新空间分配（freelistmanager），删除失效的kv数据。
9. 如果需要，进行限流操作：休眠，直至被信号量唤醒。
10. 执行状态机，此处会将io请求提交给块设备进行。

以上处理逻辑的流程图可以总结如下：
![[queue_transactions.png]]
```ad-important
title:bluestore写流程的调用栈
在`_txc_add_transaction()`函数中，会根据op的类型调用对应的函数。例如，写操作的操作码为`OP_WRITE`，那么便会进入相应的分支进行处理，此处进入了`_write()`函数并进行逐级调用。
![[Pasted image 20230112112451.png]]

经过分析可以得出**bluestore写流程的调用栈**为：


	->queue_transactions()
		-> txc_add_transaction()
			-> _write()
				-> _do_write()
					-> _do_write_data()
						-> _do_write_big() / _do_write_small()
					-> _do_alloc_write()
```


### 写流程的初步处理

#### _ txc_add_transaction
transaction对象携带着操作的信息（op）传入该函数，其中关于object的操作，先使用op->oid找到collection中对应的onode：
```C++
//创建一个和object数量相同的onode容器
vector<OnodeRef> ovec(i.objects.size()); 
// ... ...
RWLock::WLocker l(c->lock);
OnodeRef &o = ovec[op->oid]; // 获得与op->oid对应位置的onode引用
if (!o) {
	//获得与op->oid对应的ghobject_t(oid)
	ghobject_t oid = i.get_oid(op->oid); 
	o = c->get_onode(oid, create); //根据oid创建一个onode来填充o
}
```
解析op中的信息，和这个onode对象（`o`）一起作为参数传入`_write()`函数中。

#### _ write()
在该函数中检查数据长度是否符合onode(object)的要求，并为o分配一个nid（`bluestore_onode_t`对象的单一bluestore中的唯一标识），进入_do_write()的处理。
```C++
if (offset + length >= OBJECT_MAX_SIZE) { //检查待写对象长度是否超过了限制
	r = -E2BIG;
} else {
	_assign_nid(txc, o); //为onode分配nid
	r = _do_write(txc, c, o, offset, length, bl, fadvise_flags);
	txc->write_onode(o); //向txc中添加处理完成的onode
}
```
> 处理完成后，将o加入到txc->onodes中。

#### _ do_write()
在该函数中，创建一个writeContext（wctx）,调用_choose_write_options()函数，初始化压缩、校验等设置，然后加载涉及范围内的所有extent到extent_map中。
```C++
_choose_write_options(c, o, fadvise_flags, &wctx);
o->extent_map.fault_range(db, offset, length);
_do_write_data(txc, c, o, offset, length, bl, &wctx);
```
随后进入`_do_write_data()`函数进行大小写的处理。

#### _ do_write_data()
在`_do_write_data`中会判断要写的数据是否处于一个min_alloc_size中，如下所示，要写的数据落入一个完整的min_alloc_size的部分将使用大写（write big），不足一个完整的min_alloc_size的部分使用小写（write small）。另参考[[COW与RMW]]
![[Pasted image 20221009155531.png]]
![[写.png]]
具体的处理代码为：
![[Pasted image 20230112161917.png]]


## 大小写的处理

### write small
落入do_write_small的待写区间应该是一个min_alloc_size对齐的子区间，其长度length < min_alloc_size。
1. 首先开始试图寻找一个现有的可用blob，使用seek_lextent函数找到offset对应的lextent迭代器ep，随后调整ep和确认初始的prev_ep，这两个变量将用于在extent_map中分别正向和反向查找合适的blob。
	初始的prev_ep是ep的前一个lextent，除非ep已经是****extent_map的第一个lextent，prev_ep将直接指定为end（`extent_map.end()`），避免进入反向的查找（正常搜索范围：`[offset - target_max_blob_size, offset + target_max_blob_size]`）。

> [!NOTE] 说明：seek_lextent(uint64_t offset)和ep的调整
> ![[Pasted image 20230105151811.png]]
> 如下图，由set::lower_bound()和extent重载的比较运算符决定了,落入(`lextent1.logical_offset()` , `lextent2.logical_offset()`]区间内的offset,都会返回lextent2的迭代器存放到fp中。
> ![[seek.png]]
> 后面的if语句对这个结果进行调整，保证落入lextent1中时（右开区间）返回正确的值，而落入两者之间时仍返回lextent2（ 闭区间，offset=lextent1.logical_end()时返回lextent2）。
> 据此，由于lextent虽然有序但不一定连续，在小写调用seek_lextent()时，offset不一定会落入一个已有的lextent中，如果**落在两个lextent之间返回的是后者的迭代器**。
> **总结seek_offset()的几种特殊情况：**
> - offset均小于lextent的logical_offset返回`extent_map.begin()`
> - offset均大于lextent的logical_offset返回`extent_map.end()`
> - offset在两个lextent之间时返回后者的迭代器
> 
> **==关于ep的调整==**
> ![[Pasted image 20230105154941.png]]
> 这个if语句与seek_lextent()中的类似，此处是为了offset能够找到其落入的blob（就目前的分析来看，ep真正的作用是用于寻找blob，究竟是哪一个lextent并不重要）。
> 如下图，如果offset能落在前面lextent对应的blob中（区间1）或者两个lextent共享一个blob，则会对ep进行调整，否则不变。经过这种处理能够保证如果有的话一定能找到正确的blob（落入区间1和3时）。
> 如果是落入区间2的情况，ep仍然是offset后的第一个lextent，但其对应的blob与其没有重合。
> ![[Pasted image 20230105155856.png]]

2. 进入do-while循环查找有没有可以复用的blob。
寻找可用blob的循环逻辑：
```C++
do{
	any_change = false;
	
	if(ep!=end && ep->logical_offset<offset+max_bsize){
		//正向搜索
		++ep;
		any_change = true;
	}
	
	if(prev_ep!=end && prev_ep->logical_offset>=min_off){
		//反向搜索   min_off = offset>=max_bsize ? offset-max_bsize : 0
		if (prev_ep != begin) {
			--prev_ep;
			any_change = true;
		  } 
		else {
			prev_ep = end;
	    }
	}
} while(any_change)
```
当一轮循环中any_change没有发生任何变化时（没有发生任何改变时），终止这个循环。
终止循环可能的条件：
1) 没有进入正向搜索，也没有进入反向搜索（均超过范围）
2) 没有进入正向搜索，反向搜索到第一个lextent
3) 直接在搜索过程中出现return
除非出现搜索时return的情况，否则只要进入正向搜索就要进入下一轮循环。

3. 进入正向搜索时，获取ep对应的blob引用b，首先判断是否满足：
    1) `bstart<end_offs` blob起始位置要小于目标结束位置，要写数据的区间和blob有重合（因为是向后搜索，有可能会出现不满足的情况 / 不一定是向后搜索的原因，offset落入区间2时blob起始位置会大于目标结束位置。==为什么不比较offset？==）。
    2) blob是可变的。
    3) `ep->logical_offset` % `min_alloc_size` = `ep->blob_offset` % `min_alloc_size`
否则ep后移，继续循环。

4. 当满足以上所有条件时，判断是否向头尾用0填充补齐。用块对齐分别计算出头尾需要填充的字节数head_pad和tail_pad。
    - 在两者不全为0的情况下，使用fault_map确保map的范围被正确加载。
    - 需要向前补齐时，检查是否有lextent在`[offset-head_pad, offset-head_pad+chunk_size]`有区间重叠，若有，则不用0补齐（head_pad=0）；
    - 需要向后补齐时，检查是否有lextent在`[end_offs, end_offs+tail_pad]`有区间重叠，若有，则不用0补齐（tail=0）
    主要是为了确保原有的数据不被覆盖。
经过以上操作后，确认补齐后的待写blob内偏移量b_off和长度b_len。
 ^9bdd4e
5. 判断是否满足以下条件以直接写的方式写入blob：
    1) b_off和b_len块大小对齐（如果因为有重叠lextent而放弃补0的操作，则无法做到块大小对齐）
    2) blob对应的磁盘空间足够放下b_off+b_len长度的数据
    3) 对应磁盘块区间未使用
    4) 对应磁盘块空间已分配
满足以上条件时，向待写数据缓冲区bl中填充头尾的0，根据配置确定是否加入缓存，然后进入写操作的准备。若不满足，则进行[[#^5720da|8*]]中的操作。
 ^1c3a7c
6. 如果待写长度b_len小于强制延迟写入的大小阈值，则进行延迟写准备操作。
```C++
bluestore_deferred_op_t *op = _get_deferred_op(txc, o);
op->op = bluestore_deferred_op_t::OP_WRITE;
b->get_blob().map( 
	b_off, b_len,
	[&](uint64_t offset, uint64_t length) { 
	  op->extents.emplace_back(bluestore_pextent_t(offset, length));
	  return 0;
	}
);
	//获取blob对应的真实物理地址(pextent的地址)，存入op(写操作)的目标extent中
op->data = bl;	//op(写操作)将要写入的数据
```
否则，进行异步写的io准备操作。
```C++
b->get_blob().map_bl( //建立blob对应物理地址上的bl映射
	b_off, bl,
	[&](uint64_t offset, bufferlist& t) {
	  bdev->aio_write(offset, t, &txc->ioc, wctx->buffered);//准备io
	}
);
```
上面的内容可以总结为下图：
①的情况就是正常没有数据时进行直接写（新写）。
②的情况是<=强制延迟写阈值，此时需要进行延迟写。
![[Pasted image 20230112170903.png]]
7. 相关准备操作完成后，计算CSUM。使用set_lextent函数将blob中的`[b_off+head_pad, b_off+head_pad+length]`区间分离出来，作为一个新的lextent插入到extent_map中，并标记blob的对用位置已被使用（mark_used）,更新存储大小。若走到此步，函数直接结束。

8. 回顾[[#^1c3a7c|5*]]中的条件，如果不满足的话，试图准备覆盖写&延迟写。覆盖写需要先读取原来的内容，通过对齐确定前后多读的长度head_read和tail_read。 ^5720da
    1) `head_read||tail_read`，有需要读的内容
    2) blob中已分配的磁盘长度大于`b->get_blob().get_ondisk_length() >= b_off + b_len + tail_read`（head_read部分已经包含在b_off中）
    3) `head_read + tail_read < min_alloc_size`（==这里不明白为什么不包含b_len==）
同时满足以上要求时，才对b_off和b_len的值进行调整，否则`head_read = tail_read = 0`
![[wal写.png]]

9. 判断：
    1) blob的磁盘长度大于等于b_off+b_len
    2) b_off、b_len都已经按照块大小对齐
    3) blob中`[b_off, b_off+b_len]`对应的pextent空间已分配
如果以上条件都满足，向写数据的缓冲区bl首尾补0（依据是[[#^9bdd4e|4*]]中计算的head_pad和tail_pad，仍有可能需要补0）。
如果需要向前读（`head_read>0`），则进行以下处理：
```C++
if (head_read) {
	bufferlist head_bl;
	int r = _do_read(c.get(), o, offset - head_pad - head_read, 
					head_read, head_bl, 0);
					//从头部读数据到head_bl缓冲区，r为实际读取长度
	ceph_assert(r >= 0 && r <= (int)head_read);
	size_t zlen = head_read - r; //应读区域没有数据的长度
	if (zlen) {
	  head_bl.append_zero(zlen); //有的话需要对无数据的部分填充0
	  logger->inc(l_bluestore_write_pad_bytes, zlen);
	}
	head_bl.claim_append(bl); //将bl中的内容剪切到head_bl的尾部
							  //并清空bl中的内容
	bl.swap(head_bl); //交换bl和head_bl中的内容
	logger->inc(l_bluestore_write_penalty_read_ops);
}
```
如果需要向后读（`tail_read>0`），也进行和上面代码类似的处理。
此时，bl中存放着对应块的原有的数据和要新增加的数据，随后准备延迟写，延迟写的流程和前面类似，并计算CSUM。使用set_lextent函数将blob中的`[offset-bstart, offset-start+length]`（原始数据）区间分离出来，作为一个新的lextent插入到extent_map中，并标记blob的对用位置已被使用（mark_used）,更新存储大小。若走到此步，函数直接结束。

10. 如果前面的流程都没有让函数结束，证明没有空闲的空间直接供新数据写入，下一步尝试复用blob。使用`can_reuse_blob()`检查是否可以复用。
> [!NOTE] can_reuse_blob()的说明
> 在该函数中，出现了增加blob空间的情况，同时，为了避免blob的长度超过max_blob_size，对原数据的长度也有可能进行裁剪。
> ![[Pasted image 20230106155601.png]]
> 理想的情况（如下图），即使待写数据与blob有重叠但超过了blob的范围，或是有待写数据完全超出blob范围的出现，也能通过直接延长blob的方式（不超过max_blob_size），让待写数据能够写入。
> ![[Pasted image 20230106161714.png]]
> （个人猜测在该流程种使用这个方法主要是为了测试能否通过延长blob范围的方式重用这个blob） ^c84233

若可以复用，检查此时的alloc_size是否还和min_alloc_size相等，如果相等则可以继续处理，否则抛出ceph_accert。（==这里不明白的是为什么直接使用ceph_accert判断，这种情况不可能写入数据了吗？==）

确认可以reuse blob之后，使用`has_conflict()`检查在一个blob中是否有写到同一个pextent的情况（检查希望重复使用相同范围的待写数据）。
```
Need to check for pending writes desiring to reuse the same pextent. The rationale is that during GC two chunks from garbage blobs(compressed?) can share logical space within the same AU. That's in turn might be caused by unaligned len in clone_range2. Hence the second write will fail in an attempt to reuse blob at do_alloc_write().															
```
（不过==这段注释没看明白==）
如果没有冲突发生，将待写的数据向前向后块对齐补0，将待写数据的区间和extent_map重合的部分分离出来添加到old_extent，随后调用`wxtx->write()`向这个WriteContext对象的writes(`vector<write_item>`)容器种添加一个write_item。
```C++
{
	uint64_t b_off = offset - bstart;
	uint64_t b_off0 = b_off;
	_pad_zeros(&bl, &b_off0, chunk_size); //块对齐补0
		
	o->extent_map.punch_hole(c, offset, length, &wctx->old_extents);
	wctx->write(offset, b, alloc_len, b_off0, bl, b_off, length,
		false, false);
	return;
}
```
走到此步，函数结束直接返回。

12. 如果经过以上流程函数仍未结束，继续进行反向搜索。反向搜索的流程和[[#^c84233|10*]]几乎一致，如果能顺利调用`wctx->write()`返回结束函数。

#Q ==什么情况下这个do_while()循环会走第二次？==

13. 如果没有找到已有的blob可以使用退出了循环，就准备创建一个新的blob准备写入数据。这个过程和reuse_blob比较类似，比较大的不同是在调用`wctx->write`向这个WriteContext对象的writes(`vector<write_item>`)容器种添加write_item时，最后两个参数`_mark_unused`和`_new_blob`均为true。
```C++
{
	BlobRef b = c->new_blob();
	uint64_t b_off = p2phase(offset, alloc_len);
	uint64_t b_off0 = b_off;
	_pad_zeros(&bl, &b_off0, block_size);
	o->extent_map.punch_hole(c, offset, length, &wctx->old_extents);
	wctx->write(offset, b, alloc_len, b_off0, bl, b_off, 
				length, true, true);	
}
```

#### 疑问
1. 为什么搜索范围是`[offset - target_max_blob_size, offset + target_max_blob_size]`？
![[Pasted image 20230104094654.png]]
2. 一个新的extent是如何确定logical_offset的？是不是在OSD层次上处理的？
3. 为什么会出现`ep->logical_offset % min_alloc_size != ep->blob_offset % min_alloc_size`的情况？正常情况下应该是相等的吧
![[Pasted image 20230104145119.png]]

### write big
恰好满足一个min_alloc_size对齐区间时，使用do_write_big()，待写长度length=min_alloc_size。大写的流程和小写类似但比较简单，只是在向前向后搜索时找到可以复用的blob记录下对应的b（blob）和b_off，如果不能复用则新建一个blob，在最后将这些信息都登记到一个WriteContext对象（wctx）中。相比之下，大写没有延迟写、直接写等过程。

### 几种基本情况的总结
![[大小写.png]]
第三种向wctx->writes的处理方式，在后面回到_do_write()进入_do_alloc_write()时进行了处理，提交aio_write()。
为什么④以及⑤的某些情况下同样是新写，不能直接再do_write_small中直接进行处理？
目前猜测是情况①时空间已经分配完成，而延长或者新建blob的情况下，这部分空间还未分配过，需要先进入do_alloc_write中对这部分空间进行分配。或者换句话说，do_alloc_write函数的一个重要作用就是为这些待写的未分配区域进行空间分配。


## do_alloc_write和后续处理
在_`do_write_big`和`_do_write_small`完成了对WriteContext的填写之后，`_do_alloc_write(`)函数被调⽤，首先进行校验和压缩设置，随后该函数逐个处理WriteContext中的每⼀个write_item（writes）。
这个过程中遍历了两次writes：
- 第一遍根据配置进行压缩处理
- 第二遍调用allocator对待写区域分配空间并提交io。
在函数栈逐步退出的过程中更新各个层级结构的元数据。

## 事务状态机
[[事务状态机]]



