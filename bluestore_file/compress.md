[Red Hat Ceph Storage 3.3 BlueStore compression performance](https://www.redhat.com/zh/blog/red-hat-ceph-storage-33-bluestore-compression-performance)
[bluestore压缩设置 | 李厅 (lovethegirl.github.io)](https://lovethegirl.github.io/2020/02/11/compression/)

BlueStore支持在线的压缩方式，即在读或写I/O的路径中进行同步的压缩和解压缩。`compression_mode`、`compression_algorithm`、`compression_required_ratio`、`compression_min blob_size`和`compresion_max_blob_size`参数可以对每个pool属性或全局配置选项进行设置。可以使用以下命令设置pool属性：
```
ceph osd pool set <pool-name> compression_algorithm <algorithm>
ceph osd pool set <pool-name> compression_mode <mode>
ceph osd pool set <pool-name> compression_required_ratio <ratio>
ceph osd pool set <pool-name> compression_min_blob_size <size>
ceph osd pool set <pool-name> compression_max_blob_size <size>
```
源码中的相关配置项具体为：
```C++
OPTION(bluestore_compression_mode, OPT_STR)  
// force|aggressive|passive|none

OPTION(bluestore_compression_algorithm, OPT_STR)

//小于该值的块不进行压缩，原文存储（后附默认值）
OPTION(bluestore_compression_min_blob_size, OPT_U32) // 0
OPTION(bluestore_compression_min_blob_size_hdd, OPT_U32) // 8k
OPTION(bluestore_compression_min_blob_size_ssd, OPT_U32) // 128k

//大于该值的块进行分片处理后压缩，分成若干个数据块进行存储（后附默认值）
OPTION(bluestore_compression_max_blob_size, OPT_U32) // 0
OPTION(bluestore_compression_max_blob_size_hdd, OPT_U32) // 64k
OPTION(bluestore_compression_max_blob_size_ssd, OPT_U32) // 512k

/*
 * Require the net gain of compression at least to be at this ratio,
 * otherwise we don't compress.
 * And ask for compressing at least 12.5%(1/8) off, by default.
 * 压缩的收益至少达到这个值否则就不压缩
 */
OPTION(bluestore_compression_required_ratio, OPT_DOUBLE)
```

### bluestore_compression_mode
BlueStore提供不同的压缩模式包括：
- **none**: 从不压缩数据.
- **passive**: 除非写入操作设置了可压缩提示，否则不要压缩数据。
- **aggressive**: 压缩数据，除非写入操作设置了不可压缩的提示。
- **force**: 无论如何都要尝试压缩数据。在所有情况下都使用压缩，即使客户端提示数据不可压缩也是如此。

### bluestore_compress_algorithm
算法包括snappy、none、zstd、zlib、lz4，默认值是snappy  
压缩算法的使用场景可以分为以下几个场景：  
- 如果需要较好的压缩率，请使用 zstd。注意，由于 zstd 在压缩少量数据时 CPU 开销较高，建议不要将其用于 BlueStore。  
- 如果需要较低的 CPU 使用率，请使用 lz4 或 snappy。  
针对实际数据的样本运行这些算法的基准测试，观察集群的 CPU 和内存使用率。

### bluestore_compression_required_ratio
Require the net gain of compression at least to be at this ratio, otherwise we don't compress. And ask for compressing at least 12.5%(1/8) off, by default.
压缩的收益至少达到这个值否则就不压缩。目前为0.875，即至少压缩掉原始数据长度的1/8。

采用数据压缩算法需要固化两个关键信息：一是采用的压缩算法；二是压缩后的数据长度。BlueStore使用压缩头来保存这两个信息。
需要注意的是，不同于其他元数据，压缩头是与压缩后的数据一起保存的，这主要是因为压缩头需要额外的磁盘空间，所以需要纳入到压缩后的数据之中，据此一并计算压缩收益（压缩率）。当压缩率小于bluestore_compression_required_ratio时，采用压缩算法获得的净收益太小，bluestore放弃本次压缩。


