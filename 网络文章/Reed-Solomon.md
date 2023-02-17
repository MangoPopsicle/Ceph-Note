[![Go Reference](https://pkg.go.dev/badge/github.com/klauspost/reedsolomon.svg)](https://pkg.go.dev/github.com/klauspost/reedsolomon) [![Go](https://github.com/klauspost/reedsolomon/actions/workflows/go.yml/badge.svg)](https://github.com/klauspost/reedsolomon/actions/workflows/go.yml)

Reed-Solomon Erasure Coding in Go, with speeds exceeding 1GB/s/cpu core implemented in pure Go.

This is a Go port of the [JavaReedSolomon](https://github.com/Backblaze/JavaReedSolomon) library released by 
[Backblaze](http://backblaze.com), with some additional optimizations.

For an introduction on erasure coding, see the post on the [Backblaze blog](https://www.backblaze.com/blog/reed-solomon/).

For encoding high shard counts (>256) a Leopard implementation is used.
For most platforms this performs close to the original Leopard implementation in terms of speed. 

Package home: https://github.com/klauspost/reedsolomon

Godoc: https://pkg.go.dev/github.com/klauspost/reedsolomon

# Installation
To get the package use the standard:
```bash
go get -u github.com/klauspost/reedsolomon
```

Using Go modules is recommended.

# Changes

## 2022

* [GFNI](https://github.com/klauspost/reedsolomon/pull/224) support for amd64, for up to 3x faster processing.
* [Leopard GF8](https://github.com/klauspost/reedsolomon#leopard-gf8) mode added, for faster processing of medium shard counts.
* [Leopard GF16](https://github.com/klauspost/reedsolomon#leopard-compatible-gf16) mode added, for up to 65536 shards. 
* [WithJerasureMatrix](https://pkg.go.dev/github.com/klauspost/reedsolomon?tab=doc#WithJerasureMatrix) allows constructing a [Jerasure](https://github.com/tsuraan/Jerasure) compatible matrix.

## 2021

* Use `GOAMD64=v4` to enable faster AVX2.
* Add progressive shard encoding.
* Wider AVX2 loops
* Limit concurrency on AVX2, since we are likely memory bound.
* Allow 0 parity shards.
* Allow disabling inversion cache.
* Faster AVX2 encoding.

<details>
	<summary>See older changes</summary>

## May 2020

* ARM64 optimizations, up to 2.5x faster.
* Added [WithFastOneParityMatrix](https://pkg.go.dev/github.com/klauspost/reedsolomon?tab=doc#WithFastOneParityMatrix) for faster operation with 1 parity shard.
* Much better performance when using a limited number of goroutines.
* AVX512 is now using multiple cores.
* Stream processing overhaul, big speedups in most cases.
* AVX512 optimizations

## March 6, 2019

The pure Go implementation is about 30% faster. Minor tweaks to assembler implementations.

## February 8, 2019

AVX512 accelerated version added for Intel Skylake CPUs. This can give up to a 4x speed improvement as compared to AVX2.
See [here](https://github.com/klauspost/reedsolomon#performance-on-avx512) for more details.

## December 18, 2018

Assembly code for ppc64le has been contributed, this boosts performance by about 10x on this platform.

## November 18, 2017

Added [WithAutoGoroutines](https://godoc.org/github.com/klauspost/reedsolomon#WithAutoGoroutines) which will attempt 
to calculate the optimal number of goroutines to use based on your expected shard size and detected CPU.

## October 1, 2017

* [Cauchy Matrix](https://godoc.org/github.com/klauspost/reedsolomon#WithCauchyMatrix) is now an option. 
Thanks to [templexxx](https://github.com/templexxx) for the basis of this.

* Default maximum number of [goroutines](https://godoc.org/github.com/klauspost/reedsolomon#WithMaxGoroutines) 
has been increased for better multi-core scaling.

* After several requests the Reconstruct and ReconstructData now slices of zero length but sufficient capacity to 
be used instead of allocating new memory.

## August 26, 2017

*  The [`Encoder()`](https://godoc.org/github.com/klauspost/reedsolomon#Encoder) now contains an `Update` 
function contributed by [chenzhongtao](https://github.com/chenzhongtao).

* [Frank Wessels](https://github.com/fwessels) kindly contributed ARM 64 bit assembly, 
which gives a huge performance boost on this platform.

## July 20, 2017

`ReconstructData` added to [`Encoder`](https://godoc.org/github.com/klauspost/reedsolomon#Encoder) interface. 
This can cause compatibility issues if you implement your own Encoder. A simple workaround can be added:

```Go
func (e *YourEnc) ReconstructData(shards [][]byte) error {
	return ReconstructData(shards)
}
```

You can of course also do your own implementation. 
The [`StreamEncoder`](https://godoc.org/github.com/klauspost/reedsolomon#StreamEncoder) 
handles this without modifying the interface. 
This is a good lesson on why returning interfaces is not a good design.

</details>

# Usage

This section assumes you know the basics of Reed-Solomon encoding. 
A good start is this [Backblaze blog post](https://www.backblaze.com/blog/reed-solomon/).

This package performs the calculation of the parity sets. The usage is therefore relatively simple.

First of all, you need to choose your distribution of data and parity shards. 
A 'good' distribution is very subjective, and will depend a lot on your usage scenario. 

To create an encoder with 10 data shards (where your data goes) and 3 parity shards (calculated):

首先，你需要选择你的数据块和奇偶校验块的分布。
一个 "好的 "分布是非常主观的，而且将在很大程度上取决于你的使用场景。

要创建一个有10个数据块和3个奇偶校验块的编码器：

```Go
    enc, err := reedsolomon.New(10, 3)
```
This encoder will work for all parity sets with this distribution of data and parity shards. 

If you will primarily be using it with one shard size it is recommended to use 
[`WithAutoGoroutines(shardSize)`](https://pkg.go.dev/github.com/klauspost/reedsolomon?tab=doc#WithAutoGoroutines)
as an additional parameter. This will attempt to calculate the optimal number of goroutines to use for the best speed.
It is not required that all shards are this size. 

Then you send and receive data that is a simple slice of byte slices; `[][]byte`. 
In the example above, the top slice must have a length of 13.

这个编码器将适用于所有具有这种数据和奇偶校验块分布的奇偶校验集。

如果你主要使用一个块大小，建议使用[`WithAutoGoroutines(shardSize)`](https://pkg.go.dev/github.com/klauspost/reedsolomon?tab=doc#WithAutoGoroutines)作为一个额外的参数。它将试图计算出最佳的goroutines数量以获得最佳速度。
并不要求所有的块都是这个大小。

然后你发送和接收的数据是一个简单的字节分片；`[][]byte`。
在上面的例子中，最上面的片断的长度必须是13。

```Go
    data := make([][]byte, 13)
```
You should then fill the 10 first slices with **equally sized** data, 
and create parity shards that will be populated with parity data. In this case we create the data in memory, 
but you could for instance also use [mmap](https://github.com/edsrzf/mmap-go) to map files.

然后，你应该用**同样大小**的数据填充前10个块，并创建奇偶校验块，这些块将被填充奇偶校验数据。在这种情况下，我们在内存中创建数据，但你也可以使用如mmap来映射文件。

```Go
    // Create all shards, size them at 50000 each
    for i := range input {
      data[i] := make([]byte, 50000)
    }
    
    // The above allocations can also be done by the encoder:
    // data := enc.(reedsolomon.Extended).AllocAligned(50000)
    
    // Fill some data into the data shards
    for i, in := range data[:10] {
      for j:= range in {
         in[j] = byte((i+j)&0xff)
      }
    }
```

To populate the parity shards, you simply call `Encode()` with your data.

为了填充奇偶校验块，你只需用你的数据调用`Encode()`。

```Go
    err = enc.Encode(data)
```

The only cases where you should get an error is, if the data shards aren't of equal size. 
The last 3 shards now contain parity data. You can verify this by calling `Verify()`:

你可能得到错误的唯一情况是，数据块的大小不相等。
最后3个分片现在包含奇偶校验数据。你可以通过调用`Verify()`来验证这一点。

```Go
    ok, err = enc.Verify(data)
```

The final (and important) part is to be able to reconstruct missing shards. 
For this to work, you need to know which parts of your data is missing. 
The encoder **does not know which parts are invalid**, so if data corruption is a likely scenario, 
you need to implement a hash check for each shard. 

最后（也是重要的）部分是去重建丢失的块。
要做到这一点，你需要知道你的数据中哪些部分是丢失的。
编码器**不知道哪些部分是无效的**，所以如果数据损坏发生，你需要为每个块进行散列检查。

If a byte has changed in your set, and you don't know which it is, there is no way to reconstruct the data set.

To indicate missing data, you set the shard to nil before calling `Reconstruct()`:

如果你的数据集里有一个字节发生了变化，而你又不知道是哪个字节，那么就没有办法重建数据集了。

为了表示缺失的数据，你需要在调用`Reconstruct()`之前将分片设置为nil：

```Go
    // Delete two data shards
    data[3] = nil
    data[7] = nil
    
    // Reconstruct the missing shards
    err := enc.Reconstruct(data)
```

The missing data and parity shards will be recreated. If more than 3 shards are missing, the reconstruction will fail.

缺少的数据和奇偶校验块将被重新创建。如果缺少超过3个块，重建将失败。

If you are only interested in the data shards (for reading purposes) you can call `ReconstructData()`:

如果你只对数据块感兴趣（用于读取），你可以调用`ReconstructData()`。

```Go
    // Delete two data shards
    data[3] = nil
    data[7] = nil
    
    // Reconstruct just the missing data shards
    err := enc.ReconstructData(data)
```

If you don't need all data shards you can use `ReconstructSome()`:

```Go
    // Delete two data shards
    data[3] = nil
    data[7] = nil
    
    // Reconstruct just the shard 3
    err := enc.ReconstructSome(data, []bool{false, false, false, true, false, false, false, false})
```

So to sum up reconstruction:
* The number of data/parity shards must match the numbers used for encoding.
* The order of shards must be the same as used when encoding.
* You may only supply data you know is valid.
* Invalid shards should be set to nil.

总结：
* 数据/奇偶校验块的数量必须与编码时使用的一致。
* 块的顺序必须与编码时所用的相同。
* 你只能提供你知道是有效的数据。
* 无效的碎片应该被设置为nil。

For complete examples of an encoder and decoder see the 
[examples folder](https://github.com/klauspost/reedsolomon/tree/master/examples).

# Splitting/Joining Data

You might have a large slice of data. 
To help you split this, there are some helper functions that can split and join a single byte slice.

你可能有一个大的数据片。
为了帮助你分割，有一些辅助函数可以分割和连接单个的字节片断。

```Go
   bigfile, _ := ioutil.Readfile("myfile.data")
   
   // Split the file
   split, err := enc.Split(bigfile)
```
This will split the file into the number of data shards set when creating the encoder and create empty parity shards. 

这将把文件分割成创建编码器时设定的数据块数量，并创建空的奇偶校验块。

An important thing to note is that you have to **keep track of the exact input size**. 
If the size of the input isn't divisible by the number of data shards, extra zeros will be inserted in the last shard.

To join a data set, use the `Join()` function, which will join the shards and write it to the `io.Writer` you supply: 

需要注意的一个重要问题是，你必须**获得准确的输入大小**。
如果输入的大小不能被数据块的数量整除，那么在最后一个块中就会插入额外的零。

要join一个数据集，请使用`Join()`函数，它将join这些块并将其写入你提供的`io.Writer`：
```Go
   // Join a data set and write it to io.Discard.
   err = enc.Join(io.Discard, data, len(bigfile))
```

## Aligned Allocations

For AMD64 aligned inputs can make a big speed difference.

This is an example of the speed difference when inputs are unaligned/aligned:

对于AMD64来说，对齐的输入可以带来很大的速度差异。

这是一个例子，说明输入不对齐/对齐时的速度差异：

```
BenchmarkEncode100x20x10000-32    	   7058	    172648 ns/op	6950.57 MB/s
BenchmarkEncode100x20x10000-32    	   8406	    137911 ns/op	8701.24 MB/s
```

This is mostly the case when dealing with odd-sized shards. 

To facilitate this the package provides an `AllocAligned(shards, each int) [][]byte`. 
This will allocate a number of shards, each with the size `each`.
Each shard will then be aligned to a 64 byte boundary.

Each encoder also has a `AllocAligned(each int) [][]byte` as an extended interface which will return the same, 
but with the shard count configured in the encoder.   

It is not possible to re-aligned already allocated slices, for example when using `Split`.
When it is not possible to write to aligned shards, you should not copy to them.

这主要是在处理奇数大小的块时的情况。

为了方便，这个包提供了一个`AllocAligned(shards, each int) [][]byte`方法，它将分配若干每个大小为`each`的块，然后将每个块按照64字节对齐。

每个编码器也有一个`AllocAligned(each int) [][]byte`作为扩展接口，将返回同样的结果，但在编码器中配置了块数量（shard count）。  

不可能重新对齐已经分配的分片，例如使用`Split`时。
当不可能写到对齐的分片时，你不应该复制到它们。

# Progressive encoding

It is possible to encode individual shards using EncodeIdx:
可以使用EncodeIdx对单个块进行编码：

```Go
	// EncodeIdx will add parity for a single data shard.
	// Parity shards should start out as 0. The caller must zero them.
	// Data shards must be delivered exactly once. There is no check for this.
	// The parity shards will always be updated and the data shards will remain the same.
	EncodeIdx(dataShard []byte, idx int, parity [][]byte) error
```

This allows progressively encoding the parity by sending individual data shards.
There is no requirement on shards being delivered in order, 
but when sent in order it allows encoding shards one at the time,
effectively allowing the operation to be streaming. 

The result will be the same as encoding all shards at once.
There is a minor speed penalty using this method, so send 
shards at once if they are available.

这允许通过发送单个数据块来逐步进行奇偶校验编码。
它没有要求块按顺序发送，但当按顺序发送时，它允许一次性对块进行编码，
有效地使操作成为流式。

其结果将与一次性编码的所有块相同。
使用这种方法会有一个小的速度损失，所以如果块是可用的话，就一次性发送这些块。

## Example

```Go
func test() {
    // Create an encoder with 7 data and 3 parity slices.
    enc, _ := reedsolomon.New(7, 3)

    // This will be our output parity.
    parity := make([][]byte, 3)
    for i := range parity {
        parity[i] = make([]byte, 10000)
    }

    for i := 0; i < 7; i++ {
        // Send data shards one at the time.
        _ = enc.EncodeIdx(make([]byte, 10000), i, parity)
    }

    // parity now contains parity, as if all data was sent in one call.
}
```

# Streaming/Merging

It might seem like a limitation that all data should be in memory, 
but an important property is that **as long as the number of data/parity shards are the same, you can merge/split data sets**, and they will remain valid as a separate set.

所有的数据都应该在内存中，这似乎是一个限制。
但一个重要的属性是，**只要数据/奇偶校验块的数量是相同的。你可以合并/拆分数据集**，并且它们将作为一个独立的集合保持有效的状态。

```Go
    // Split the data set of 50000 elements into two of 25000
    splitA := make([][]byte, 13)
    splitB := make([][]byte, 13)
    
    // Merge into a 100000 element set
    merged := make([][]byte, 13)
    
    for i := range data {
      splitA[i] = data[i][:25000]
      splitB[i] = data[i][25000:]
      
      // Concatenate it to itself
	  merged[i] = append(make([]byte, 0, len(data[i])*2), data[i]...)
	  merged[i] = append(merged[i], data[i]...)
    }
    
    // Each part should still verify as ok.
    ok, err := enc.Verify(splitA)
    if ok && err == nil {
        log.Println("splitA ok")
    }
    
    ok, err = enc.Verify(splitB)
    if ok && err == nil {
        log.Println("splitB ok")
    }
    
    ok, err = enc.Verify(merge)
    if ok && err == nil {
        log.Println("merge ok")
    }
```

This means that if you have a data set that may not fit into memory, you can split processing into smaller blocks. 
For the best throughput, don't use too small blocks.

This also means that you can divide big input up into smaller blocks, and do reconstruction on parts of your data. 
This doesn't give the same flexibility of a higher number of data shards, but it will be much more performant.

这意味着，如果你有一个无法全部装入内存的数据集，你可以将它分成较小的块。
但为了获得最佳吞吐量，不要使用太小的块。

这也意味着你可以把大的输入分成更小的块，并对部分数据进行重构。
这并不像更多的数据碎片那样具有灵活性，但它的性能会好很多。

# Streaming API

There has been added support for a streaming API, to help perform fully streaming operations, 
which enables you to do the same operations, but on streams. 
To use the stream API, use [`NewStream`](https://godoc.org/github.com/klauspost/reedsolomon#NewStream) function 
to create the encoding/decoding interfaces. 

已经增加了对Streaming API的支持，以帮助执行完全的streaming操作，这使你在streaming上能获得同样的操作。要使用流Streaming API，使用[`NewStream`](https://godoc.org/github.com/klauspost/reedsolomon#NewStream)函数 
来创建编码/解码接口。

You can use [`WithConcurrentStreams`](https://godoc.org/github.com/klauspost/reedsolomon#WithConcurrentStreams) 
to ready an interface that reads/writes concurrently from the streams.

You can specify the size of each operation using 
[`WithStreamBlockSize`](https://godoc.org/github.com/klauspost/reedsolomon#WithStreamBlockSize).
This will set the size of each read/write operation.

Input is delivered as `[]io.Reader`, output as `[]io.Writer`, and functionality corresponds to the in-memory API. 
Each stream must supply the same amount of data, similar to how each slice must be similar size with the in-memory API. 
If an error occurs in relation to a stream, 
a [`StreamReadError`](https://godoc.org/github.com/klauspost/reedsolomon#StreamReadError) 
or [`StreamWriteError`](https://godoc.org/github.com/klauspost/reedsolomon#StreamWriteError) 
will help you determine which stream was the offender.

你可以使用[`WithConcurrentStreams`](https://godoc.org/github.com/klauspost/reedsolomon#WithConcurrentStreams)来准备一个接口，从流中并发地读/写。

你可以使用[`WithStreamBlockSize`](https://godoc.org/github.com/klauspost/reedsolomon#WithStreamBlockSize)指定每个操作的大小。
这将设置每个读/写操作的大小。

输入以`[]io.Reader`的形式交付，输出以`[]io.Writer`的形式交付，功能对应于内存中的API。每个流必须提供相同的数据量，类似于内存API中每个块的大小必须相似。如果一个流发生错误，就会出现`StreamReadError`或`StreamWriteError`帮助你确定是哪个流出错的。

There is no buffering or timeouts/retry specified. If you want to add that, you need to add it to the Reader/Writer.

For complete examples of a streaming encoder and decoder see the 
[examples folder](https://github.com/klauspost/reedsolomon/tree/master/examples).

GF16 (more than 256 shards) is not supported by the streaming interface. 

没有指定缓冲或超时/重试。如果你想添加这些，你需要把它添加到Reader/Writer中。

关于流媒体编码器和解码器的完整例子，见 [examples文件夹](https://github.com/klauspost/reedsolomon/tree/master/examples)。

流媒体接口不支持GF16（超过256个碎片）

# Advanced Options

You can modify internal options which affects how jobs are split between and processed by goroutines.

To create options, use the WithXXX functions. You can supply options to `New`, `NewStream`. 
If no Options are supplied, default options are used.

Example of how to supply options:

你可以修改内部选项，这些选项影响作业如何在goroutines之间分割和处理。

要创建选项，请使用WithXXX函数。你可以为`New`, `NewStream`提供选项。
如果没有提供选项，将使用默认选项。

如何提供选项的例子：

 ```Go
     enc, err := reedsolomon.New(10, 3, WithMaxGoroutines(25))
 ```

# Leopard Compatible GF16

When you encode more than 256 shards the library will switch to a [Leopard-RS](https://github.com/catid/leopard) implementation.

This allows encoding up to 65536 shards (data+parity) with the following limitations, similar to leopard:

* The original and recovery data must not exceed 65536 pieces.
* The shard size *must*  each be a multiple of 64 bytes.
* Each buffer should have the same number of bytes.
* Even the last shard must be rounded up to the block size.

当你编码超过256个碎片时，库将切换到[Leopard-RS](https://github.com/catid/leopard)实现。

这允许编码多达65536个碎片（数据+奇偶性），有以下限制，类似于leopard：

* 原始数据和恢复数据不得超过65536块。
* 分片大小**必须**是64字节的倍数。
* 每个缓冲区都应该有相同的字节数。
* 即使是最后一个碎片也必须四舍五入到块的大小。

|                 | Regular | Leopard |
| --------------- | ------- | ------- |
| Encode          | ✓       | ✓       |
| EncodeIdx       | ✓       | -       |
| Verify          | ✓       | ✓       |
| Reconstruct     | ✓       | ✓       |
| ReconstructData | ✓       | ✓       |
| ReconstructSome | ✓       | ✓ (+)   |
| Update          | ✓       | -       |
| Split           | ✓       | ✓       |
| Join            | ✓       | ✓       |

* (+) Same as calling `ReconstructData`.

The Split/Join functions will help to split an input to the proper sizes.

Speed can be expected to be `O(N*log(N))`, compared to the `O(N*N)`. 
Reconstruction matrix calculation is more time-consuming, 
so be sure to include that as part of any benchmark you run.  

相比于`O(N*N)`，速度可望达到`O(N*log(N))`。
重构矩阵的计算更耗时，因此，请务必将其作为您运行的任何基准测试的一部分。

For now SSSE3, AVX2 and AVX512 assembly are available on AMD64 platforms.

目前，SSSE3、AVX2 和 AVX512 组件可在 AMD64 平台上使用。

Leopard mode currently always runs as a single goroutine, since multiple 
goroutines doesn't provide any worthwhile speedup.

Leopard 模式目前总是作为单个 goroutine 运行，因为多个 goroutines 不提供任何有价值的加速。

## Leopard GF8

It is possible to replace the default reed-solomon encoder with a leopard compatible one.
This will typically be faster when dealing with more than 20-30 shards.
Note that the limitations listed above also applies to this mode. 
See table below for speed with different number of shards.

To enable Leopard GF8 mode use `WithLeopardGF(true)`.

Benchmark Encoding and Reconstructing *1KB* shards with variable number of shards.
All implementation use inversion cache when available.
Speed is total shard size for each operation. Data shard throughput is speed/2.
AVX2 is used.

可以用一个与leopard兼容的编码器来替换默认的reed-solomon编码器。
在处理20-30个以上的碎片时通常会更快。
请注意，上面列出的限制也适用于这种模式。
关于不同块数的速度，见下表。

要启用Leopard GF8模式，请使用`WithLeopardGF(true)`。

基准编码和重建1KB的块，块数量可变
所有的实现都使用反转缓存，如果有的话。
速度是每个操作的总碎片大小。数据碎片的吞吐量是速度/2。
使用AVX2。

| Encoder      | Shards      | Encode         | Recover All  | Recover One    |
| ------------ | ----------- | -------------- | ------------ | -------------- |
| Cauchy       | 4+4         | 23076.83 MB/s  | 5444.02 MB/s | 10834.67 MB/s  |
| Cauchy       | 8+8         | 15206.87 MB/s  | 4223.42 MB/s | 16181.62  MB/s |
| Cauchy       | 16+16       | 7427.47 MB/s   | 3305.84 MB/s | 22480.41  MB/s |
| Cauchy       | 32+32       | 3785.64 MB/s   | 2300.07 MB/s | 26181.31  MB/s |
| Cauchy       | 64+64       | 1911.93 MB/s   | 1368.51 MB/s | 27992.93 MB/s  |
| Cauchy       | 128+128     | 963.83 MB/s    | 1327.56 MB/s | 32866.86 MB/s  |
| Leopard GF8  | 4+4         | 17061.28 MB/s  | 3099.06 MB/s | 4096.78 MB/s   |
| Leopard GF8  | 8+8         | 10546.67 MB/s  | 2925.92 MB/s | 3964.00 MB/s   |
| Leopard GF8  | 16+16       | 10961.37  MB/s | 2328.40 MB/s | 3110.22 MB/s   |
| Leopard GF8  | 32+32       | 7111.47 MB/s   | 2374.61 MB/s | 3220.75 MB/s   |
| Leopard GF8  | 64+64       | 7468.57 MB/s   | 2055.41 MB/s | 3061.81 MB/s   |
| Leopard GF8  | 128+128     | 5479.99 MB/s   | 1953.21 MB/s | 2815.15 MB/s   |
| Leopard GF16 | 256+256     | 6158.66 MB/s   | 454.14 MB/s  | 506.70 MB/s    |
| Leopard GF16 | 512+512     | 4418.58 MB/s   | 685.75 MB/s  | 801.63 MB/s    |
| Leopard GF16 | 1024+1024   | 4778.05 MB/s   | 814.51 MB/s  | 1080.19 MB/s   |
| Leopard GF16 | 2048+2048   | 3417.05 MB/s   | 911.64 MB/s  | 1179.48 MB/s   |
| Leopard GF16 | 4096+4096   | 3209.41 MB/s   | 729.13 MB/s  | 1135.06 MB/s   |
| Leopard GF16 | 8192+8192   | 2034.11 MB/s   | 604.52 MB/s  | 842.13 MB/s    |
| Leopard GF16 | 16384+16384 | 1525.88 MB/s   | 486.74 MB/s  | 750.01 MB/s    |
| Leopard GF16 | 32768+32768 | 1138.67 MB/s   | 482.81 MB/s  | 712.73 MB/s    |

"Traditional" encoding is faster until somewhere between 16 and 32 shards.
Leopard provides fast encoding in all cases, but shows a significant overhead for reconstruction.

Calculating the reconstruction matrix takes a significant amount of computation. 
With bigger shards that will be smaller. Arguably, fewer shards typically also means bigger shards.
Due to the high shard count caching reconstruction matrices generally isn't feasible for Leopard. 

"传统"编码在16和32个块之间时会更快。
Leopard在所有情况下都能提供快速的编码，但在重建方面显示出明显的开销。

算出重建矩阵需要大量的计算。
对于较大的块，这个计算量会更小。可以说，更少的碎片通常也意味着更大的碎片。
由于高分片数，缓存重建矩阵对Leopard来说通常是不可行的。

# Performance

Performance depends mainly on the number of parity shards. 
In rough terms, doubling the number of parity shards will double the encoding time.

Here are the throughput numbers with some different selections of data and parity shards. 
For reference each shard is 1MB random data, and 16 CPU cores are used for encoding.

| Data | Parity | Go MB/s | SSSE3 MB/s | AVX2 MB/s |
| ---- | ------ | ------- | ---------- | --------- |
| 5    | 2      | 20,772  | 66,355     | 108,755   |
| 8    | 8      | 6,815   | 38,338     | 70,516    |
| 10   | 4      | 9,245   | 48,237     | 93,875    |
| 50   | 20     | 2,063   | 12,130     | 22,828    |

The throughput numbers here is the size of the encoded data and parity shards.

If `runtime.GOMAXPROCS()` is set to a value higher than 1, 
the encoder will use multiple goroutines to perform the calculations in `Verify`, `Encode` and `Reconstruct`.


Benchmarking `Reconstruct()` followed by a `Verify()` (=`all`) versus just calling `ReconstructData()` (=`data`) gives the following result:
```
benchmark                            all MB/s     data MB/s    speedup
BenchmarkReconstruct10x2x10000-8     2011.67      10530.10     5.23x
BenchmarkReconstruct50x5x50000-8     4585.41      14301.60     3.12x
BenchmarkReconstruct10x2x1M-8        8081.15      28216.41     3.49x
BenchmarkReconstruct5x2x1M-8         5780.07      28015.37     4.85x
BenchmarkReconstruct10x4x1M-8        4352.56      14367.61     3.30x
BenchmarkReconstruct50x20x1M-8       1364.35      4189.79      3.07x
BenchmarkReconstruct10x4x16M-8       1484.35      5779.53      3.89x
```

The package will use [GFNI](https://en.wikipedia.org/wiki/AVX-512#GFNI) instructions combined with AVX512 when these are available.
This further improves speed by up to 3x over AVX2 code paths.

## ARM64 NEON

By exploiting NEON instructions the performance for ARM has been accelerated. 
Below are the performance numbers for a single core on an EC2 m6g.16xlarge (Graviton2) instance (Amazon Linux 2):

```
BenchmarkGalois128K-64        119562     10028 ns/op        13070.78 MB/s
BenchmarkGalois1M-64           14380     83424 ns/op        12569.22 MB/s
BenchmarkGaloisXor128K-64      96508     12432 ns/op        10543.29 MB/s
BenchmarkGaloisXor1M-64        10000    100322 ns/op        10452.13 MB/s
```

# Performance on ppc64le

The performance for ppc64le has been accelerated. 
This gives roughly a 10x performance improvement on this architecture as can be seen below:

```
benchmark                      old MB/s     new MB/s     speedup
BenchmarkGalois128K-160        948.87       8878.85      9.36x
BenchmarkGalois1M-160          968.85       9041.92      9.33x
BenchmarkGaloisXor128K-160     862.02       7905.00      9.17x
BenchmarkGaloisXor1M-160       784.60       6296.65      8.03x
```


# Links
* [Backblaze Open Sources Reed-Solomon Erasure Coding Source Code](https://www.backblaze.com/blog/reed-solomon/).
* [JavaReedSolomon](https://github.com/Backblaze/JavaReedSolomon). Compatible java library by Backblaze.
* [ocaml-reed-solomon-erasure](https://gitlab.com/darrenldl/ocaml-reed-solomon-erasure). Compatible OCaml implementation.
* [reedsolomon-c](https://github.com/jannson/reedsolomon-c). C version, compatible with output from this package.
* [Reed-Solomon Erasure Coding in Haskell](https://github.com/NicolasT/reedsolomon). Haskell port of the package with similar performance.
* [reed-solomon-erasure](https://github.com/darrenldl/reed-solomon-erasure). Compatible Rust implementation.
* [go-erasure](https://github.com/somethingnew2-0/go-erasure). A similar library using cgo, slower in my tests.
* [Screaming Fast Galois Field Arithmetic](http://www.snia.org/sites/default/files2/SDC2013/presentations/NewThinking/EthanMiller_Screaming_Fast_Galois_Field%20Arithmetic_SIMD%20Instructions.pdf). Basis for SSE3 optimizations.
* [Leopard-RS](https://github.com/catid/leopard) C library used as basis for GF16 implementation.

# License

This code, as the original [JavaReedSolomon](https://github.com/Backblaze/JavaReedSolomon) is published under an MIT license. See LICENSE file for more information.