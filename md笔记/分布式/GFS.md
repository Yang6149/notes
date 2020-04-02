# GFS

## 2 设计概要

### 2.1假设

* 做到不断的检查，能够容忍左无并且可以迅速的从错误中恢复。

* 通常存大文件、几个G很正常。
* 主要有两种读取方式，大的流式读取，小的随机读取。流式读取通常一次读几百KB到1MB多，随机读在任意偏移量上读几KB。一般为了性能会把很多随机读排序然后进行批处理。
* 对应读，也会有很多顺序写，操作大小一般和读一样大。
* 系统必须高效的实现，多个用户去同时对一个文件进行读写。
* 持续的高宽带要比低延迟要好。

### 2.2 interface

GFS 提供了常见接口，文件在组织的层级目录中被文件名标识。提供了有常见的 create、delete、open、close、read和write 操作。

而且，GFS还有 snapshot 和 记录 append 的操作。

### 2.3 架构

GFS 集群有一个 master 多个 chunkservers。可以被多个 clients 连接

![1585661342261](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/31/213046-649945.png)

可以很轻松的做到 clients 和 chunkservers 在一台机器上。

文件被分为固定大小，在创建时被master 用一个 64bit 的 chunk handle 来标识。默认每个 chunk 在三台机器上存放。用户可以对不同的文件命名空间设置不同的复制级别。

master 维护全部的文件系统元数据，包括 namespace，access control info，文件到 chunk 的映射，当前 chunks 的位置。也控制系统层的行为如：chunk 的租约管理，孤儿 chunk 的回收，以及 chunk 迁移。master 周期性的向 chunkserver 发送信息，来发送指令以及收集server的状态，曰：HeartBeat

GFS client 被嵌入到应用当中，通过API来访问 master 和 chunkservers，与 master 只有元数据的交互，所有 file 数据与 chunkservers 交互。

client 和 chunkservers 都不缓存文件。

### 2.4 单 Master

单 Master 极大的简化了设计，但是应该尽可能的减少它的读和写操作，让这不会称为它的瓶颈。

读写交互的过程为：

1. client 通过 文件名和偏移量计算出 chunk 的 index。然后向 master 发送一个包含 filename  和 index 的请求，master 回应符合的 chunk handle 和所在位置（包括replicas）。
2. 用户把得到的信息缓存起来并当作key向其中一个 replicas （一般最近的）发起请求，这个请求制定了 chunk handle 和访问的大小。这个时候和 master 没有半毛钱关系。

通常 client 会请求多个块，master 也会在请求之后立刻返回这些信息。获取这些信息可以减少 client 和 master 之间的交互。

### 2.5 chunk size

选了 64 MB，比典型的文件系统的 block 大得多。

采用这么大的块有这几点好处：

1. 减少了client 和 master 的交互
2. 在块中可以进行更多的操作，通过TCP长连接可以减小网络开销。
3. 减小元数据在 master 上存储的量。

当然也有坏处，比如一个小文件比如只有一个 chunk ，然后大量 client 访问这个文件的话就会从固定的三个 chunkserver 上读取，造成很大的压力。我们通过提高可执行文件的复制因数和批队列处理系统应用交错启动的方法解决了这个问题。一个可能的长期解决方案是在这种情况下让客户端能够从其它的客户端读取数据。

### 2.6 元数据

主要存三种元数据：file 和 chunk 的 namespace，files 到 chunks 的映射，还有每一个 chunk 的 replicas 的位置。

所有数据都在内存中，前两种还会通过 log 持久化，并且远程机器上也有备份。有日志可以简单、可靠的更新 master 的状态。master 不持久化每个 chunk 的位置，而是在启动时以及有新的 server 加入集群问每个 chunkserver 它的 chunks。

#### 2.6.1 内存数据结构

元数据存内存中可以很快的周期性的扫描状态，周期性的扫描用于：实现垃圾回收，块服务器出错的再复制，负载均衡和磁盘潜移。

文件的 namespace 一般少于 64byte，所以内存管够

#### 2.6.2 chunk Locations

前面也说了，通过刚启动时来 poll chunk location。然后用 HeartBeat 来维护状态信息。还有一点是这些信息主要由 chunkserver 来决定，例如节点挂了。

#### 2.6.3 操作日志

操作日志包含了元数据改变的关键的历史记录。这不仅是元数据的唯一持久记录，还可以用作定义并发操作顺序的逻辑时间表。File 和 chunks 以及版本都是由逻辑创建时间唯一且永恒标识的。

既然操作日志很关键，我们必须可靠的存储它，在元数据改变持久化前不能让 client 看到改变过的信息。因此我们在远程机器上也复制操作日志，在本地和远程都 flush 符合的日志到磁盘上后才向 client 返回对操作结果。主节点会在刷新前对日志进行批处理操作来减少吞吐量。

master 通过日志来恢复文件系统状态。为了减少启动时间，我们尽可能让 log 小。当 log 超过一定大小，master 就保存 checkpoints 它的状态，这样的话就可以通过载入最近一次的 checkpoint 去恢复。这些通过一个 B-tree 维护。

创建 checkpoint 可能会很慢，这个时候创建一个新的线程去创建 checkpoint，并不阻塞原来的任务，在创建好后去持久化到本地和远程。

### 2.7 一致性模型

#### 2.7.1 GFS 保障

文件 namespace 的改变的原子性由 master 保证，通过 namespace lock；总执行顺序由 master 的操作日志决定。

![1585673959339](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/01/005920-467550.png)

一个文件区域（region）在一个数据修改后的状态依赖于该修改的类型、成功或失败、以及是否为同步的修改。不管从哪读的，所有的 client 读到的数据都相同，就认为这个文件区域是一致的。

1. 非并行文件数据 write ，如果它是一致性的就是被 defined，这时客户端就可以看到写入的全部内容。

2. 当并行的 write 在没有被干扰的情况下成功了，也是 defined：全部的 client 都能看到写入的内容。

3. 并行成功的 write 使区域处于 undefined 但是一致：所有的 client 都看到相同的数据，但是可能不会反映出每个人都修改了啥。一般来说这由多个修改片段组成。

4. 一个失败的修改造成不一致当然 undefined：不同用户看到不同数据不同。

我们接下来会说怎么区分 defined 和 undefined。

数据修改有两种操作：write 和 record appends。write 可以在任意偏移量上进行写数据。 record append 至少一次的原子操作把数据追加在GFS指定偏移量后面（一般是文件末尾），即使是并发情况下。把偏移量返回给 client 然后标记包含 record 的定义区域的开头。此外，GFS 还可以插入占据空字符到 record 中，来弥补出现非一致性导致的记录信息大小不等。

![1585719389140](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585719389140.png)

在连续的成功修改后，被修改的区域都认为是 defined，并且包含最后一次数据写入的数据。GFS 通过相同的数据对 replicas chunks 进行修改和使用 chunk 版本号去检测其他 replica。过期的数据就不会出现任何修改和提供数据，这个时候就垃圾回收掉。

clients 会缓存 chunk location，可能会在数据更新前读到过期数据，通过缓存的过期以及下次访问时再次更新缓存来实现，而且一般文件是 append 操作，所以是读不完而很少读过期数据。

数据修改很长时间之后，可能发生数据损坏，GFS 通过握手的方式，检测数据。一旦数据出现问题，就立刻通过其他 replicas 进行补救数据。只有在同时所有数据都失效才会出现不可逆的错误。但也是返回 error 而不是错误的数据。

#### 2.7.2 应用实现

一般来说应用是通过 append 操作而不是 overwrite 操作来修改数据。从头到尾写完后再改个名字。周期性的检查写成功了多少。checkpoint 也包含应用层的checksum。读取器只核对处理最后一次 checkpoint，就是 defined 的状态的checkpoint。

另一种使用是，很多 writer 并行 append 一个文件，然后融合结果。前面的至少一次原子写保证了输出。每个被 writer 准备的 record 包含了一次 checksum，用来核对修改，一个 reader 能够识别和抛弃一次额外的 padding 和 record fragment 通过 checksum。

## 3. 系统交互

### 3.1 租约（Leases）和修改顺序

我们通过 租约（Leases）去维护 replicas 之间的一致性修改。master 再几个 replicas 中选出一个 primary。然后 primary 来控制修改的顺序。所有 replicas 都遵循这个顺序。

这个机制可以有效减少 master 的开销，lease 的初始化超时时间为 60 s。而且只要有一个 chunk 正在修改，那么就会有续约，这个操作是通过 HeartBeat 来进行的。GFS 也可能会再到期前解约（比如重名命操作期间不能写入），即使一个 primary 与 master 丢失联系，也可以重新找一个来签约。

![1585724951864](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/01/150912-360868.png)

1. client 向 master 哪个 chunkserver 是 primary，以及其他 replicas 在哪，如果没有 primary 就签一个（上图没描述）

2. master 返回 primary 和 其他secondary 的一些信息。然后 client 就缓存这些数据，进行对数据的修改，只有连接不到 primary 以及节点不再是 primary 时才去连接 master。
3. client 把数据传给全部的 replicas，可以是任意顺序，每个服务器把数据保存在内部的 LRU 缓存当中，直到数据被使用或被替换掉。在传数据时怎么快怎么来，不用管谁是 primary。
4. 一旦所以普数据都传完了，就向 primary 发一条信息，让 primary 去核对刚才的信息。primary 为刚才的传入操作分配连续的序号，来确保一定的顺序，因为这些操作可能来自不同的客户端，这提供了连续性。应用按照这些操作修改状态，

5. primary 向所有的 secondary 发起 write 请求。每一个 secondary 都按照线性的顺序写入。
6. secondary 回复 primary 完成操作啦。
7. primary 回复 client 。在 replicas 的错误都发送给 client。走到这一步代表至少 primary 成功了。secondary 可能失败了，这个时候 client 会再尝试去修改 secondary 来保持一致性。（回到第3步-7步）。

如果写操作过大或跨过了 chunk 的边界，GFS 的 client 会把这一次操作分成多次操作，并且都遵循上面步骤，这样的话共享的文件区域可能包含不同的碎片，但所有 replicas 都相同，因为遵循了上面步骤，这样的话就处于一种一致但 undefined 的状态。

### 3.2 数据流

通过网络有效的把数据流从控制流中分开。控制流是 client - master，数据流是 client - chunkserver。目的就是有效利用带宽，避免网络瓶颈和高延迟连接。为了有效利用带宽，数据沿着 chunkserver 线性推送（3.1 的 step 3），为了尽可能的优化，优先把数据传到最近的节点。并且使用了管道的技术来进行数据的传输，在接收到数据之后立刻进行传输，因为实现了全双工，并不会减少接收数据的速度。

在没有网络拥塞的情况下，将B个字节传输到R个副本的理想时间消耗为B/T+RL，这里T是网络的吞吐量，L是两台机器间的传输延迟。我们的网络链路通常为100Mbps（T），并且L远小于1ms，因此，在理想情况下，1MB的数据能够在80ms内分发完成。

### 3.3 原子 record append

GFS 提供了原子的 record append 操作，传统的写中，client 要说明写入的 偏移量，并行的写向同一个区就不是线性化的，该区域最终可能包含多个 client 发送的片段，append 操作就不担心，client 只需要发送数据就行，不用给偏移量，GFS将数据至少一次的原子性的 append 到 GFS 选定的偏移量上（末尾）

在并发情况下，多个 client 对同一个文件进行操作经常用到这种方法。如果传统写的方法的话，就需要花费大量资源在分布式锁上。这样的模式通常用于多生产则单消费者队列或合并来自不同 client 的数据的操作。primary 会检查这一次 append 的大小会不会大于最大的 chunk size ，如果大于的话就把当前 chunk 填满（padding 空数据），并通知 replicas 也做相同的操纵，并返回给 client ，表明这个操作会再下个 chunk 上重新执行。一般来说严格限制每一次的 append 量少于最大大小的1/4，来减少这种事情的发生。

replicas 如果同步失败的话，先repush一次，可能包含部分传入的数据，这样就会造成它们结尾的偏移量不同，通过这个可以观察到同步是否失败，我们可以利用2.7 中的方法解决（padding）。

### 3.4 快照

快照操作几乎立刻拷贝一份文件或目录，这个不会被其他操作打断，我们用户经常要大数据集的分支拷贝或保存 checkpoint 现在的状态再执行改变的操作前，能够轻易的commit 或 rollback。

当 master 收到一个快照请求，它首先回收相关块的租约，保证确保 client 对 chunkserver 的写需要先与 maser 交互。这就给 master 一个机会区首先拷贝 chunk。

当租约被回收或过期， master 就把这个操作写进日志保存到硬盘，通过复制的文件和目录tree来作用于内存中的记录状态。

开始一个 client 想要在 快照操作后写 chunk C，它发送请求去寻找 primary，Master注意到 chunk C的引用计数大于1，它将推迟回复客户端的请求，并选择一个新的 chunk handleC’，然后通知每个拥有chunk C replicas的chunkserver，创建一个新的 chunk C’。通过在同一个块服务器上创建一个新的 chunk，我们能确保这个拷贝是本地的，不需要通过网络进行的（我们的磁盘速度是100MB以太网链路的3倍）。master 重新为C‘ 选一个 primary，并返回，client 并不知道拿到的数据是一个新的 chunk。

## 4 Master 操作

master 执行所有的 namespace 操作，另外它管理系统的复制吞吐量：决定 chunk 存储的位置，协调不同系统范围内的行为，来保持由足够的副本，再 chunkserver 间负载均衡，回收不用的空间。

### 4.1 namespace 管理和锁

很多 master 的操作能花很长时间：例如一个 快照操作 必须在要操作的块所属的 chunkserver 回收租约。我们不想在其他 master 操作运行时去推迟它们。因此，我们允许其他的 master 操作同时执行，使用 namespace 的锁来顺序的执行。

不像传统的文件系统有目录结构，GFS 逻辑上表现为一个 全路径名映射元数据的查找表。通过前缀压缩，这个表能够有效的在内存中表示。在 namespace tree 上的每一个节点都有一个 读写锁。

每个 master 的操作都需要获得一系列的锁。通常如果调用 /d1/d2/.../dn/leaf ，就需要在（/d1）,（ /d1/d2）, ..., （/d1/d2/.../dn） 上的读写锁。

现在描述怎么在 `/home/user`写快照到 `/save/user`时，通过锁机制保护`/home/user/foo`的创建,保存快照需要 `/home` 和 `/save`上的读锁以及 `/home/user` 和 `/save/user`上的写锁。文件创建需要在获得 `/home` 和 `/home/user`的读锁，以及 `/home/user/foo` 的写锁，这两个操作会适当的线性化，因为 一个要 `/home/user`的读锁，一个要写锁。文件上的读锁足够防止目录被修改。

好处就是可以在同一目录下并行操作，例如多个文件在同一目录下的并行创建。读锁足够目录被修改或删除以及快照。

为了防止死锁，先通过层级排序，在逐次获得锁。

### 4.2 Replica Placement

GFS 集群采用高度分布式的多层结构。通常几百台机器在多个机架上。两台机器间交流可能要跨多个交换机，进出机架的宽带可能小于机架上总宽带。

chunk replicas 的位置有两个标准：最大化数据的可靠性和可用性、最大化宽带的利用率。我们把 chunk replica 放在多个机架上，读起来就很舒服。不怕整个机架出错，也能有效利用带宽。

### 4.3 Creation，Re-replication，Rebalancing

chunk replica 的创建有三个原因：chunk 创建、重复制、重负载均衡。

在 master 创建一个 chunk时，它选择哪去放初始化的空 replicas。这包含几个元素：

1. 我们想要在空间使用率低的磁盘上创建副本
2. 限制每个 chunkserver 上的最近创建数量
3. 想要把 replica 放在不同的机架上。

当副本数量少于用户指定数量时,master 会再创建新的副本.可能有好几个原因：

1. 一个chunkserver 不可用了
2. 磁盘坏了
3. 指定副本数量变多

少两个 chunk 的比少一个 chunk 的优先级要高。近期活跃的优先级也高。

最后，Master周期性的重新均衡副本的负载：它检查当前的副本分布情况，并将副本移动到更好的磁盘空间上，达到负载均衡。

### 4.4 Garbage Collection

在一个文件被删除后，GFS不会立即回收可用的物理存储空间。它只有在文件和块级别上定期的垃圾回收时才会进行。我们发现，这个方法使系统更加简单，更加可靠。



当一个文件被应用删除时，Master会像其他操作一样立即记录下这个删除操作。然而，文件仅仅会被重命名为一个包含删除时间戳的、隐藏的名字，而不是立即回收资源。然后定期扫描，删除隐藏三天的文件。也可以通过重命名来救回来。

master 通过时别元数据中的孤儿 chunk 也进行删除。

### 4.5 过期副本的检测

在 chunkserver 失效或宕机的时候，副本可能失效，master 维护一个版本号去区别是否过期。

不管啥时候，master 授予一个新的 lease 给 chunk，就会增加这个 chunk 的版本号。如果其它副本当前不可用，它的块版本号将不会被更新。

Master在定期的垃圾回收操作中清除过期的副本。在这之前，当它回复 client 的 chunk 信息请求时，它将认为一个过期的副本实际上根本并不存在。作为另一个安全措施，当Master通知 client 哪个 chunkserver 持有这个 chunk 的租约时，或者在进行克隆操作期间它通知一个 chunkserver 从另一个 chunkserver 上读取数据时，包含块版本号。客户端或块服务器在执行操作时会验证这个版本号，以保证总是可访问的最新的数据。

