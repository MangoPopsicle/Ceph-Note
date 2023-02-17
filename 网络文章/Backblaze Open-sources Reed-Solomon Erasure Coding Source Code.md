At Backblaze we have built an extremely cost-effective storage system that enables us to offer a great price on our online [backup service](https://www.backblaze.com/b2/solutions/backup-and-archive.html). Along the path to building our storage system, we have used time-tested technologies off the shelf, but we have also built in-house technologies ourselves when things weren’t available, or when the price was too high.

We have taken advantage of many open-source projects, and want to do our part in contributing back to the community. Our first foray into open-source was our [original Storage Pod design](https://www.backblaze.com/b2/storage-pod.html), back in September of 2009.

Today, we are releasing our latest open-source project: Backblaze Reed-Solomon, a Java library for erasure coding.

An [erasure code](https://en.wikipedia.org/wiki/Erasure_code) takes a “message,” such as a data file, and makes a longer message in a way that the original can be reconstructed from the longer message even if parts of the longer message have been lost. [Reed-Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction) is an erasure code with exactly the properties we needed for file storage, and it is simple and straightforward to implement.

## Erasure Codes and Storage

Erasure coding is standard practice for systems that store data reliably, and many of them use Reed-Solomon coding.

The RAID system built into Linux uses Reed-Solomon. It has a carefully tuned Reed-Solomon implementation in C that is part of the RAID module. Microsoft Azure uses a similar, but different, erasure coding strategy. We’re not sure exactly what Amazon S3 and Google [Cloud Storage](https://www.backblaze.com/b2/cloud-storage.html) use because they haven’t said, but it’s bound to be Reed-Solomon or something similar. Facebook’s new cold storage system also uses Reed-Solomon.

If you want reliable storage that can recover from the loss of parts of the data, then Reed-Solomon is a well-proven technique.

擦除编码是可靠的数据存储系统的标准做法，其中许多系统使用Reed-Solomon编码。

Linux内置的RAID系统使用了Reed-Solomon。它有一个精心调整的C语言的Reed-Solomon实现（impelmentation，实现），是RAID模块的一部分。微软Azure使用一个类似的，但不同的擦除编码策略。我们不确定亚马逊S3和谷歌云存储到底使用什么，因为他们没有说，但它一定是Reed-Solomon或类似的东西。Facebook的新冷存储系统也使用了Reed-Solomon。

如果你想要可靠的存储，能够从部分数据的丢失中恢复，那么Reed-Solomon是一种经过充分验证的技术。

## Backblaze Vaults Utilize Erasure Coding

Earlier this year, I wrote about [Backblaze Vaults](https://www.backblaze.com/blog/vault-cloud-storage-architecture/), our new software architecture that allows a file to be stored across multiple Storage Pods, so that the file can be available for download even when some Storage Pods are shut down for maintenance.

To make Backblaze Vaults work, we needed an erasure coding library to [compute](https://www.backblaze.com/b2/solutions/compute.html) “parity” and then use it to reconstruct files. When a file is stored in a Vault, it is broken into 17 pieces, all the same size. Then three additional pieces are created that hold parity, resulting in a total of 20 pieces. The original file can then be reconstructed from any 17 of the 20 pieces.

We needed a simple, reliable, and efficient Java library to do Reed-Solomon coding, but didn’t find any. So we built our own. And now we are releasing that code for you to use in your own projects.

今年早些时候，我写了关于Backblaze Vaults的文章，这是我们的新软件架构，允许一个文件存储在多个存储舱中，这样，即使一些存储舱因维护而关闭，该文件也可以被下载。

为了使Backblaze Vaults发挥作用，我们需要一个擦除编码库来计算 "奇偶性"，然后用它来重建文件。当一个文件被存储在Vault中时，它被分成17块，所有的大小都一样。然后再创建三个额外的片段，以支持奇偶校验，结果是总共20个片段。然后，原始文件可以从这20个碎片中的任何17个碎片中重构出来。

我们需要一个简单、可靠、高效的Java库来进行Reed-Solomon编码，但没有找到。所以我们建立了自己的库。现在我们发布了这些代码，供你在自己的项目中使用。

## Performance

Backblaze Vaults store a vast amount of data and need to be able to ingest it quickly. This means that the Reed-Solomon coding must be fast. When we started designing Vaults, we assumed that we would need to code in C to make things fast. It turned out, though, that modern Java virtual machines are really good, and the just-in-time compiler produces code that runs fast.

Our Java library for Reed-Solomon is as fast as a C implementation, and is much easier to integrate with a software stack written in Java.

A Vault splits data into 17 shards, and has to calculate three parity shards from that, so that’s the configuration we use for performance measurements. Running in a single thread on [Storage Pod](https://www.backblaze.com/b2/storage-pod.html) hardware, our library can process incoming data at 149 megabytes per second. (This test was run on a single processor core, on a Pod with an Intel Xeon E5-1620 v2, clocked at 3.70GHz, on data not already in cache memory.)

Backblaze Vaults存储了大量的数据，并需要能够快速获取这些数据。这意味着Reed-Solomon编码必须是快速的。当我们开始设计Vaults时，我们认为我们需要用C语言编码来使事情变得快速。然而，事实证明，现代的Java虚拟机真的很好，及时编译器产生的代码运行速度也很快。

我们的Reed-Solomon的Java库和C语言实现的速度一样快，而且更容易与用Java编写的软件栈集成。

一个Vault将数据分割成17个碎片，并要从中计算出三个奇偶校验碎片，所以这就是我们用于性能测量的配置。在Storage Pod硬件上以单线程运行，我们的库可以以每秒149兆字节的速度处理传入数据。(这个测试是在一个单核处理器上运行的，一个装有 Intel Xeon E5-1620 v2的Pod上，时钟为3.70GHz，数据未缓存。)

## Where Is the Open-source Code?

You can find the source code for Backblaze Reed-Solomon on the [Backblaze website](https://www.backblaze.com/open-source-reed-solomon.html), and also on [GitHub](https://github.com/Backblaze/JavaReedSolomon).

The code is licensed with the [MIT License](https://en.wikipedia.org/wiki/MIT_License), which means that you can use it in your own projects for free. You can even use it in commercial projects.

We’ve put together a video titled: [“Reed-Solomon Erasure Coding Overview”](https://youtu.be/jgO09opx56o) to get you started.

If you’re interested in an overview of the math behind the code, keep reading. If not, you already have what you need to start using the Backblaze Reed-Solomon library. Just download the code, read the documentation, look at the sample code, and you’re good to go.

## Reed-Solomon Encoding Matrix Example

Feel free to skip this section if you aren’t into the math.

We are fortunate that mathematicians have been working on matrix algebra, group theory, and information theory for centuries. Reed and Solomon used this body of knowledge to create a coding system that seems like magic. It can take a message, break it into n pieces, add k “parity” pieces, and then reconstruct the original from n of the (n+k) pieces.

我们很幸运，数学家们几个世纪以来一直在研究矩阵代数、群论和信息论。里德和所罗门利用这些知识创造了一个看起来非常神奇的编码系统。它可以把一条信息分成n个部分，再加上k个 "奇偶校验 "部分，然后从n+k中的n个部分中重建原始信息。

The examples below use a “4+2” coding system, where the original file is broken into four pieces, and then two parity pieces are added. In Backblaze Vaults, we use 17+3 (17 data plus three parity). The math—and the code—works with any numbers as long as you have at least one data shard and don’t have more than 256 shards total. To use Reed-Solomon, you put your data into a matrix. For computer files, each element of the matrix is one byte from the file. The bytes are laid out in a grid to form a matrix. If your data file has “ABCDEFGHIJKLMNOP” in it, you can lay it out like this:

下面的例子使用的是 "4+2 "编码系统，原始文件被分解成四块，然后添加两个奇偶校验块。在Backblaze Vaults中，我们使用17+3（17个数据加三个奇偶校验）。只要你至少有一个数据分片，并且总分片数不超过256个，那么这个数学和代码就可以用于任何数字。为了使用Reed-Solomon，你把你的数据放到一个矩阵中。对于计算机文件，矩阵的每个元素是文件中的一个字节。这些字节被排列在一个网格中，形成一个矩阵。如果你的数据文件中有 "ABCDEFGHIJKLMNOP"，你可以这样布局。

**The Original Data**  
![The Original Data](https://www.backblaze.com/blog/wp-content/uploads/2015/06/blog-rs-1.png)

In this example, the four pieces of the file are each four bytes long. Each piece is one row of the matrix. The first one is “ABCD.” The second one is “EFGH.” And so on.  
The Reed-Solomon algorithm creates a coding matrix that you multiply with your data matrix to create the coded data. The matrix is set up so that the first four rows of the result are the same as the first four rows of the input. That means that the data is left intact, and all it’s really doing is computing the parity.

在这个例子中，文件的四个片断分别是四个字节长。每一块都是矩阵的一行。第一块是 "ABCD"。第二块是 "EFGH"。以此类推。 
里德-所罗门算法创建了一个编码矩阵，你将其与你的数据矩阵相乘，以创建编码数据。矩阵的设置是为了使结果的前四行与输入的前四行相同。这意味着数据是完整的，而它真正做的是计算奇偶校验。

**Applying the Coding Matrix**  
![Erasure Coding](https://www.backblaze.com/blog/wp-content/uploads/2015/06/blog-rs-2.png)

The result is a matrix with two more rows than the original. Those two rows are the parity pieces.

Each row of the coding matrix produces one row of the result. So each row of the coding matrix makes one of the resulting pieces of the file. Because the rows are independent, you can cross out two of the rows and the equation still holds.

结果是一个比原来多两行的矩阵。这两行是奇偶校验块。

编码矩阵的一行生成对应结果的一行。因此，编码矩阵的每一行都会产生文件中的一个结果片。因为这些行是独立的，你可以划掉其中两行，方程仍然成立。

**Data Loss: Two of the Six Rows Are “Lost”**  
![Data Loss: 2 of the 6 rows are lost](https://www.backblaze.com/blog/wp-content/uploads/2015/06/blog-rs-3.png)

With those rows completely gone, it looks like this:

随着这些行的完全消失，它看起来像这样：

**Data Loss: The Matrix Without the Two “Lost” Rows**  
![Data Loss: The matrix without the 2 "lost" rows](https://www.backblaze.com/blog/wp-content/uploads/2015/06/RS-post-pic-4.png)

Because of all the work that mathematicians have done over the years, we know the coding matrix, the matrix on the left, is invertible. There is an inverse matrix that, when multiplied by the coding matrix, produces the identity matrix. As in basic algebra, in matrix algebra you can multiply both sides of an equation by the same thing. In this case, we’ll multiply on the left by the identity matrix:

由于数学家多年来所做的所有工作，我们知道编码矩阵，即左边的矩阵，是可逆的。有一个反转矩阵，当它与编码矩阵相乘时，会产生一个身份矩阵。与基础代数一样，在矩阵代数中，你可以将一个方程的两边都乘以相同的东西。在这种情况下，我们将在左边乘以身份矩阵。

**Multiplying Each Side of the Equation by the Inverse Matrix**  
![Multiplying Each Side of the Equation by the Inverse Matrix](https://www.backblaze.com/blog/wp-content/uploads/2015/06/RS-post-pic-5.png)**The Inverse Matrix and the Coding Matrix Cancel Out**  
![The Inverse Matrix and the Coding Matrix Cancel Out](https://www.backblaze.com/blog/wp-content/uploads/2015/06/RS-post-pic-5-5.png)

This leaves the equation for reconstructing the original data from the pieces that are available:

这就留下了从现有的碎片中重构原始数据的方程式：

**Reconstructing the Original Data**  
![Reconstructing the Original Data](https://www.backblaze.com/blog/wp-content/uploads/2015/06/blog-rs-7.png)

So to make a decoding matrix, the process is to take the original coding matrix, cross out the rows for the missing pieces, and then find the inverse matrix. You can then multiply the inverse matrix and the pieces that are available to reconstruct the original data.

所以要做一个解码矩阵，过程就是把原始的编码矩阵，划掉缺失部分的行，然后找到反矩阵。然后，你可以将反矩阵和可用的碎片相乘，重建原始数据。

## Summary

That was a quick overview of the math. Once you understand the steps, it’s not super complicated. The Java code goes through the same steps outlined above.

There is one small part of the code that does the actual matrix multiplications that has been carefully optimized for speed. The rest of the code does not need to be fast, so we aimed more for simple and clear.

If you need to store or transmit data, and be able to recover it if some is lost, you might want to look at Reed-Solomon coding. Using our code is an easy way to get started.