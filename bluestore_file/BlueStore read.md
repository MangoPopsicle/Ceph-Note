
## read
BlueStore重写/实现了ObjectStore::read方法，在这个函数中通过oid获取对象的onode信息，并在正常的情况下进行读取，调用`_do_read()`函数进行真正的读操作的处理。
>目前暂时不确定用户提交的读请求入口，和write不同，在transaction中没找到读操作。但是有一些OSD方法调用了该函数。


## do_read
![[ceph/bluestore_file/attachments/bluestore_do_read.drawio.png]]
在_do_read()中有四个主要的大循环，其中有两个嵌套的循环。
1. 进行一些变量的定义和赋值，以及其他的预处理操作。
> [!NOTE] 重要变量
> - **==offset==** `uint54_t` 待读数据的逻辑起始偏移量（函数参数）
> - **==length==**  `size_t` 待读数据的长度（函数参数）
> - **ready_regions** `ready_regions_t` 即`map<uint64_t, bufferlist>`地址和读取内容缓冲区的映射表，用于暂存读取完成的部分数据。

2. 找到第一个包含offset的lextent
> [!NOTE] 重要变量
> - **==pos==** `uint_64_t` 考虑空洞校正过的待读数据的逻辑起始偏移量，初始值=offset。
> - **==left==** `unsigned` 考虑空洞校正过的待读数据的的长度，初始值=length。
> - **==lp==** `set<Extent>::iterator` 当前查看的lextent，经过此步操作后初值为第一个包含offset的lextent。
> - **==num_regions==** `unsigned` 统计读取区间分片的数量，初值=0。
> - **==blobs2read==** `blobs2read_t` 即`map<BlueStore::BlobRef, regions2read_t>`，blob和读取区间列表的映射表

##### 第一个外部大循环：遍历lp及后面的所有extent
3. 考虑空洞，参考`lp->logical_offset`校正真正的逻辑起始偏移量和长度。（可能会有存在空洞的情况，即有可能在lextent的offset前读到未写过的数据）
```C++
if (pos < lp->logical_offset) { 
	unsigned hole = lp->logical_offset - pos;
	if (hole >= left) {
	break;
	}
	dout(30) << __func__ << "  hole 0x" << std::hex << pos << "~" << hole
	   << std::dec << dendl;
	pos += hole; //=lp->logical_offset
	left -= hole;//读取长度减去空洞
}
```

4. blob循环的准备工作：定义变量，读取缓存信息。
> [!NOTE] 重要变量
> - **==bptr==** `BlobRef&` =lp->blob，当前lextent对应的blob
> - **==l_off==** `unsigned` =pos-lp->logical_offset，pos有可能大于当前extent的logical_offset（空洞是排除小于的情况）,l_off就是获得pos相对于logical_offset在当前lextent内开始读取的位置
> - **==b_off==** `unsigned` = l_off+lp->blob_offset，加上extent在blob中的偏移，就可以得到当前读取的起始位置在blob中的偏移，该值会在后面的内部循环中随着区间的不断确定而向后推移。
> ![[b_off.png]]
> - **==b_len==** `unsigned` =min(left, lp->length - l_off)，blob中的剩余区间长度，初始时根据pos的位置取值有可能不同，当pos<logcial_offset时，left为真正的长度；当pos>logical_offset时，后者为真正的长度，仅当pos=logical_offset时两者相同。该值会在后面的内部循环中随着区间的不断确定而不断减小直至为0。
> - **==cache_res==** `ready_regions_t` 即`map<uint64_t, bufferlist>`，存放缓存内容和对应的地址。
> - **==cache_interval==** `interval_set<uint32_t>` 已被缓存的区间范围


##### 第二个内部循环：在一个lextent中查找所有的待读内容/区间，最大为b_len
5. 如果当前位置b_off在缓存中命中，则确定本段区间长度`l`为本段缓存长度，并将缓存中的内容暂存到ready_regions中。
```C++
if (pc != cache_res.end() &&  pc->first == b_off) { //如果本段在缓存中命中
	l = pc->second.length(); //本段区间长度为本段缓存长度
	ready_regions[pos].claim(pc->second); //读到ready_regions
	dout(30) << __func__ << "    use cache 0x" << std::hex << pos << ": 0x"
		 << b_off << "~" << l << std::dec << dendl;
	++pc;
  } 
```
6. 如果缓存未命中，则选择在磁盘中读，找到对应的磁盘位置。此时，本段区间的长度`l`为b_off到下一次命中缓存位置的距离。
> [!NOTE] 重要变量
> - **==r_off==** `uint64_t` 要读的数据在blob内的偏移，后面需要进行最小可读块的对齐，初值=b_off
> - **==r_len==** `uint64_t` 本次确定的读取区间的长度，后面需要进行最小可读块的对齐，初值为`l`
> - **==front==** `uint64_t` 为了对齐在前方多读的长度。

分别向前向后将读取区间与blob的最小可读块大小对齐，调整r_off和r_len的值和确定front的大小。
此外在循环中可能出现同一个blob相邻区间在不同循环出现的情况，此时需要将这些相邻的区间进行合并，更新到blob的待读区间列表中去。
> [!NOTE] 重要变量
> - **==r2r==** `regions2read_t&` 当前blob对应的regions2read(`list<read_req_t>`)待读区间列表
> - **==pre==** `read_req_t` r2r中的上一个/最后的读取请求信息/区间信息，在read_req_t结构中存储着连续的区间信息以及组成这些区间的小区间（原始读取区间）列表regs。

- 如果r2r(当前blob对应的待读区间列表)不为空，即已经登记过相关的区间，且当前对齐后的起始偏移量r_off小于等于pre的结束位置，则需要将当前区间与pre进行合并。
![[do_read.png]]
将头部多读部分的长度front扩展到pre的起始位置，并将pre的读取长度延长到本次区间的结束位置。并在pre的regs尾部生成一个元素，将当前区间的信息作为原始读取区间进行登记。
- 如果无需进行合并，则直接在r2r的尾部生成一个元素，登记本段区间的信息。
本区间的信息登记结束后，num_regions+1。

7. 调整变量以继续循环向后查找登记。本blob中的信息全部登记完毕后结束内部循环。
8. 遍历过剩余lextent后结束外部大循环。

##### 第三个循环：准备io
9. 准备一个名为compressed_blob_bls的bufferlist容器，用于存放已经压缩后的数据，遍历blobs2read。
	- 如果当前blob对应的数据经过了压缩，第一次进到此分支时，向compressed_blob_bls分配blobs2read大小的容量，避免在随后的循环中因容量不足重新分配。每次遍历到此步，向容器末尾添加新元素，取得这个元素，随后建立blob到物理设备的映射，准备io（即每次准备一个读到compressed_blob_bls末端的io）。
	- 如果当前blob对应的数据没有经过压缩，则遍历当前blob对应的region2read待读区间列表，依次准备io。
循环结束后，检查是否有待执行的io，进行提交并等到io完毕。

##### 第四个循环：枚举并解压所有的blob（解压、校验）
> [!NOTE] 重要变量
> - **==ready_regions==** `map<uint64_t, bufferlist>` 读取完成的区间及数据映射表
> - 
10. 再次遍历一整个blobs2read：
	- 如果blob对应的数据已经经过了压缩，首先校验数据（如遇到假性读取错误，重试读取，即重新进入`_do_read()`函数），然后从compressed_blob_bls取得对应的待解压数据进行解压，根据配置进行缓存，将解压后的数据存放到ready_regions结构中。
	- 类似地，如是未压缩数据，跳过压缩相关过程，进行校验、缓存处理，存放到ready_regions结构中。

11. 遍历ready_regions，将所有结果存放到一个连续的缓冲区中（即_do_read()的bl参数），读取结束。