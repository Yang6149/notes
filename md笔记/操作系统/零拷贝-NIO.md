# 零拷贝-NIO

Java中，通常情况下我们想要修改一个文件，会先 new 一个`InputStream`，然后一个个chunk的去读取，然后再通过`OutputStream`输出。为了优化，可以用一些如`BufferInputStream`之类的工具类，内置了缓存。但是在底层，这种操作的代价还是很大的。

## 底层操作

1. JVM 发出了 raed() 系统调用
2. 内核空间从硬件中读数据
3. 数据通过DMA传回内核空间
4. 数据从内核空间copy到用户空间

![1592830719509](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1592830719509.png)

## sendfile

显然我们不想在用户态和内核态间因为拷贝数据浪费时间。

考虑一个场景，从硬盘中读取文件并通过网络发送出去。

我们想这样：![1592830888994](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/22/210131-887539.png)

比如我们想要从读取文件并发送出去，那么路径就是硬盘->内核空间->网关。完全不需要用户态参与，所以只有一次上下文切换。

中间只有一次`write data to target socket buffer`

![1592833077245](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/22/213757-479284.png)

因为DMA block 传输需要连续的空间，直接在内核缓冲区中读的话不是连续的空间，

那么为了减少这一次在内核空间中的copy，可以通过把内核缓冲区的空间用链表连接起来，使用scatter-n-gather技术来读取。

![1592833305592](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1592833305592.png)

Java中是通过`transferTo`来实现。[Doc](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/FileChannel.html#transferTo-long-long-java.nio.channels.WritableByteChannel-)

## mmap

但上述场景只是直接读取文件并输出网络，并不涉及到修改文件。

那么我们就像修改文件并保存，就是最普通的单机读写文件操作，这要怎么办呢。

我们也已使用稍微昂贵一点的方法。这种方式会在`map()`,`write()`在用户态和内核态中切换2次,切换4次上下文,进行1次数据拷贝.并且不一定比普通IO效率高,因为在对页的管理及分配拆解时,可能花费更多时间。使用时需要做好测试。

1. 用户进程通过mmap(),创建虚拟地址映射空间。发起系统调用，进入内核态。
2. 内核调用mmap，实现文件物理地址和进程虚拟地址的映射关系
3. 进程去访问映射空间，发生缺页异常，在交换缓存空间（swap cache）中寻找需要访问的内存页。
4. 没有就去物理磁盘上读，并存进内存中。
5. CPU利用DMA从磁盘中读数据，读到内核缓冲区。
6. 切换回用户态，并进行操作如`write()`
7. 拷贝到网络缓冲区 再DMA输出
8. 或DMA scatter-n-gather 直接输出

这种方式并不安全，需要做好并发安全处理。如果该文件被其他文件访问，那么 write 系统调用会因为访问非法地址而被 SIGBUS 信号终止。

关于这点的其他[参考](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

> SIGBUS 默认会杀死进程并产生一个 coredump

![1592834057455](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/02/001704-375133.png)

## NIO DirectByteBuffer

Java NIO 提供了`ByteBuffer`，来探究一下实现原理：

**HeapByteBuffer**：在调用`ByteBuffer.allocate()`时，会使用。维护在JVM的堆空间，受益于GC。由于不是页对其的，所以需要通过JNI，拷贝数据到缓冲空间。

**DirectByteBuffer**：在调用`ByteBuffer.allocateDirect()`时，会使用。通过调用`malloc()`在堆外空间申请空间，由于不在JVM中，不享受GC但页对齐不用进行拷贝数据。写底层很舒服。但是就会成Java程序员转为了C程序员，需要手动防止内存泄漏。

**MappedByteBuffer**：在调用`FileChannel.map()`时，会使用。

也是在对外空间，本质上是封装的系统调用`mmap()`，直接管理映射在物理内存上的数据。