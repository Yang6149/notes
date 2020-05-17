# MIT 6.824 Lab 4

这是最后一个实验了，AB两个实验写在一起吧。整体难度对我来说非常大，期间花费大量时间在找bug。写了一个并发脚本去跑Lab2和Lab3都发现了一些之前没遇到过的错误，Debug 花费了很久时间。

以下内容只提供一些思路，大量细节我没怎么写比如所有RPC后的数据都需要是leader进行操作，在Raft 中通过一致性后再应用、 GC 参数为什么需要版本号及shardID之类的，这些我认为每个人可能有不同的解决办法。由于课程要求不能把代码放在 gitbub 上防止正在上课的学生抄写，我的就放在 private repository了，等课程结束应该是可以放出来？大概。

![1589350328715](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/141209-513384.png)

![1589350669287](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/141749-758629.png)

其中 A 的测试用例比较好跑，几秒就跑一次，并发跑了1000次，没 bug free。B 的测试由于时间太久，只跑了100次，并且由于模拟几十个节点，开的线程数量也很多，并发跑的话有时10分钟都跑不完一个，是由于模拟持久化只是把 metadata 进行 encode 后暂放在内存，假装持久化了，所以这里的瓶颈是由于 CPU 造成的，放在生产环境中，网络以及本地的 IO 过程肯定是更早遇到瓶颈。所以在没达到 IO 瓶颈前，是不会跑那么慢的。

## 实验目标

![1589351177707](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/142619-935154.png)

架构类似 spanner 、BigTable 是一个主从结构，目标就是造一个能跑的 支持动态扩容缩容、负载均衡、保证强一致性、保证容错容灾的 K/V 集群。

![1589351907195](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/143829-768595.png)

其中每个 server 之间都是可以互相 RPC 的，我就没画了。

### 4A 

简要说一下就是，4A 要写一个 Master Group，master 主要是进行配置的管理，我们想要分区来进行优化读写，所以要提前规划好怎么去实现。

![1589352042751](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/14/234815-802102.png)

config的版本号要实现自增，每进行一次操作都要版本号加一，并且保存之前所有config，不用GC，因为配置也占不了多少内存，谁没事天天改个几千次上万次的配置？

shard 这里设计的就比较简单，先通过一个函数对 key 求hash（这里直接对首字符ascii取余了）取余，shard 的个数是固定的，也不用担心固定数量的 shard 有什么坏处。在启动项目前预测会达到的最大数据量所需的节点个数，再乘个好几倍就行了。比如我们认为这个项目最多100个节点，平时可能也就用10个节点，我就分成 1000 个 shard，平时每个节点 拿100 个shard，等到那天数据量大的时候，就动态的扩展到100个节点，然后每个节点拿 10 个shard。具体 shard 在节点中传递是 4B 的任务。

groups 存放的是 gid-> servers[]，servers[]中存放的就是上上图中 server1、2、3的 id，我们可以根据这个id 拿到 rpc 接口然后访问该节点。

思路和 lab3 中差不多，只是把 kv map 转变成了 config 数组，每次拿到新的指令后送进 Raft 中走一圈，然后 根据新的指令来生成新的 config ，再放入 configs 中。

注意一点，在 lab3 中我们为了保证线性执行并且强一致性返回，为每个用户的每个指令分配了唯一的 id，由clientID和SerialID组成，而这里我们由于只有一个用户可以改变 config，所以在 Query 操作中并不需要自增 SerialID，而是直接从config 中返回数据，如果碰巧这个时候 Raft 切换了leader，新的leader还没 commit，证明这个时候的 SerialID 也没更新，我们仍然拿这个 SerialID当作 Query 的ID传入，也会放进 Raft 中走一圈，保证还是会拿到强一致性的数据。

该模块提供了4个接口

1. Join：向该集群中新增 Group，再通过rebalance进行调节负载均衡
2. Leave：剔除掉某一个 Group，再进行 rebalance 进行调节。
3. Move：把其中一个 Shard 移动到某一个 Group 中，不会动态调节
4. Query：获取Config，参数可以提供某个config，-1为最新config。

还有一个问题就是 Rebalance，如何移动更少的shard来rebalance config，这里不多赘述大概leetcode easy~medium 难度。只是注意map取Key为乱序，小心点。

### 4B

4b 就是对 lab3 进行全面升级，实验要求拿到配置后需要进行 Group 之间的 rpc 来传输 shard 的数据。并且保证在用户访问节点中一定拿到正确的数据，不能是过期数据。在后面还有两个 challenge Test，一个是需要GC掉不属于本 Group 的 shard，还有一个就是可以立刻访问传来的 shard ，而不需要等待所有的 shard 都更新好才能同意访问。由于我不想在写 challenge 时再重新设计，就从刚开始时就把这两个考虑进来进行设计了。

首先我们的数据不再只是放入一个`map[string][string]`中了而是放入`map[int]Shard`中，shard的结构为：

![1589354098052](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/151458-128057.png)

首先写一个获取 Config 的线程单独运行，不断的去获取版本号为自己implement的config的版本号+1 的config，判断是否和自己的config相同，不同的话就进行数据迁移。

#### 数据迁移

![1589355593099](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/14/234822-558976.png)

数据迁移最开始考虑有两种方案：

1. Pull 数据，当检测到新的 config 中这个 Group 需要这个 shard ，就回去上个 config 中拥有这个 shard 的group 去请求这个数据。好处是可以及时请求，但是rpc 返回时带有大量数据过慢可能会造成大量重复请求。
2. Push 数据，当检测到新的 config 中这个 Group 不再需要 这个 shard，就把这个 shard 发送给需要的 group，好处是相比上一个不会有大量请求，但是等待数据的一方处于被动状态，无法及时判断对方的状态。

我选择了 Push 数据，因为我考虑到毕竟在真实环境中，绝大部分时间网络通信还是比较畅通的，所以就不需要直到对方状态，不行的话就再发一次，也造成大量重复请求，但只在极端情况下出现。

拿到数据后也是只有在 Raft 中走一圈才会去 apply，再返回，所以这个过程有时会超时（500ms），重发判重。

#### 更新配置

为了保证 Config 和 Shard 的安全，需要选择执行哪个版本的 Config 进行处理，如果我的 Config 中写的是我不需要 shard0，其他的 Group 中的 Config 也不需要 shard0，那么这个shard就会凭空消失，我的处理过程为每一个 Group 只有拥有所有指定版本的 shard 并且没有任何**（不属于自己&&版本号小于等于自己Config 版本号）**的 shard 后才进行请求下一个 config，这样能够保证不会阻塞或丢失数据，并且方便在请求中判断是否是过期请求。

#### GC

由于上一个的设计我们可以清晰的直到哪个 shard 是属于自己的，哪个shard 是过期的，哪个 shard 是超出自己的版本的。我这里写了一个阻塞循环 deamon，当数据迁移成功后，就发送信号量到这个线程中，这里再进行判重然后在 Raft 中走一圈后应用。

#### challenge2

challenge2 在设计之初就有考虑，这里只需要在访问接口的地方进行判断各个 shard 的版本号就可以实现，只有 group 的版本号及该 shard 的版本号>= client 版本号才可以返回。主要还是体现在之前的设计中。

![1589355898537](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/13/154458-266402.png)

## 问题

#### 1. Lab3 的遗留问题

当时写 Lab 3 的snapshot 部分的时候，我看了很久论文，觉得实现起来没问题，但是在我写的时候，我觉得有一个参数不需要，就是 `LastIncludedTerm`在 snapshot中最后一个指令所在的Term，自认为很聪明认为是该协议的一个改进，查了网上其他关于 Raft 的 Snapshot的文章，都有提到这个参数，然而我认为不需要，拿笔构建了半天觉得天衣无缝，甚至觉得马上就可以发表论文震撼30年，并且实现后跑了也有几百次没有出任何bug。然鹅，在Lab4中由于没有 LastIncludedTerm，在带动宕机很久的节点重新连接时，需要判断新leader是否接收，Term=0会一直拒绝。现在也想不明白为什么 Lab3 能跑通几百遍，用git回退到lab3版本后再跑，才几遍就出bug了，细思恐极。

所谓改一处而动全身，因为这个地方，我甚至差不多修改了一半的 Snaoshot 相关代码，debug起来及其费劲。

#### 2. Lab4 忘记再Client针对不同 Err做出不同回应

一个技巧，由于 4b 中有很多节点，但是同一 Group 内状态都是相同的。只需要先判断Leader再打 Log 就好了，不然3个集群3个Server，就是9个Log。

这个我Lab4b时出现的Bug，极为迷惑无从下手，以为 4a Bug Free，后来用脚本并发跑了以下 4A ，大概1/200的概率会出现一次，修改后跑了1000次 bug free。

#### 3. 各种奇形怪状的 corner case

首先一定要保证只要在Lock的范围内不能出现任何 RPC 通信或阻塞信号量，如果需要信号量写成信号量队列来用，大概这个样子

![1589365761892](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1589365761892.png)

我在有一个 Group 向另一个 Group 的几个 Server 发送RPC的过程中使用同一个线程同一个锁，造成的结果就是仍然可以运行，但是奇慢无比。修改后就快了5~6倍。

还有很多奇怪的 corner case ，都是因为设计不够周全或者实现起来操之过急漏掉，这些问题都通过慢慢 debug 一一解决了（写bug10分钟，debug 1整天）。

## 总结

至此所有实验写完了，从不会Go 到写完集群KV系统花了快2个月时间，课还差4节，前面有些论文也只是囫囵吞枣的看完，接下来一段时间好好沉淀以下。好运!