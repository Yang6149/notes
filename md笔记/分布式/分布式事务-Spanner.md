# 分布式事务

## 一、事务的棘手问题

可以看之前文章了解事务基本概念[数据库基本知识概论](http://yang614.xyz/blog/61)，基本抄的CYC

这里就是我们经常提到的隔离级别的问题。一下问题是在分布式环境下的脏读。

![1589559304290](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/16/001504-238093.png)

前几个隔离级别我们暂且不考虑，我们这里直接讨论线性一致的隔离级别。

由于我们处在多个集群间进行操作事务，在每一个情况，每一个节点我们都应该考虑失败的可能性。

## 二、分布式事务

首先想要保证事务的线性执行，那么公认的方法就是实现2PL。

2 phase locking，顾名思义就是两段锁，存在一个加锁阶段一个解锁阶段。在解锁阶段开始后就不能加锁了。具体文章一搜一大把。

既然2PL可以解决线性一致那么，还有什么问题呢？别忘了，现在是分布式环境，任何时候的任何节点都可能无法通信，由此产生了2PC算法，其中为节点抽象出身份，一个协调者，几个集群(数据所在地)简要描述为三步：

1. 协调者向几个集群分别发出指令如：set、update、delete、get等，集群会返回该数据
2. 协调者向集群发出prepare请求，询问它们是否准备好commit了，集群会返回yes/no。
3. 如果集群全部返回yes，协调者会发送commit请求，让所有集群去commit该事务。

[mit6.824老师讲解的该部分](https://www.youtube.com/watch?v=aDp99WDIM_4&t=1551s)

所有的集群一旦返回了prepare yes，那么就不会主动放弃锁了。在prepare时需要写日志，防止之后节点挂了，丢失数据。协调者更是要做好日志工作，因为它一旦挂了，group可能永远持有锁不放。具体网上应该也有很多文章。

![1594228679724](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/09/011802-331900.png)

## 三、Spanner

其实这也是我写这篇博客的目的，一下展示一些Spanner的只读事务的思路

### 全球同步时间

Spanner由于是google的，家大业大，通过GPS+原子钟，提供了一组接口，可以告诉你全球的同步时间。当然这个也不是特别精确的，比如从节点到GPS发送到返回还是有一些时间的，那么仍然需要人为解决这微小的误差，核心思想就是错开，如事务1 latest 小于 事务2 earlist，那么事务1 就在事务2之前。

![1594231256692](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1594231256692.png)

### 读写事务

spanner中的coordinator并不是单独的一个用户或是其他节点，一般来说是该事务所在的group的leader(paper提到大部分情况下都是在一个group中)。如果不在一个group中，那么就在这些group中选一个group的replica leader。

这里执行的就是上面提到的2PC+2PL。不同的是，它会在每一个replica 进行更新paxos 数据时，更新该replica的 t_paxos_safe。这个参数的意思是节点最近更新paxos的时间

### 只读事务

这个是spanner的重点，它是无锁，大部分情况下无阻塞的！

spanner 为每个节点中的数据计算一个值 t_safe，这个值可以和一个只读事务的开始时间进行比较，如果t<=t_safe那么就认为这个事务可以直接读，否则就阻塞等待t_safe更新为更大的值。

所以我们现在的任务就是搞懂t_safe怎么更新了。

首先抛出一个定义：

![1594231760114](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/09/020931-573271.png)

那么t_paxos_safe刚才已经解释过了，值得强调的是，只要paxos commit了新数据就去更新该值。

t_tm_safe 是该事务的manager 提供的一个数据，该数据也有一个公式

![1594231908295](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1594231908295.png)

这个时候又出现了 s_prepare_i_g，别怕，这时最后一个数据了，它是该group返回commit yes response时的时间戳。那么t_tm_safe就是代表的第一个准备好的group的时间戳，后面的-1应该是处理一下边界，（我没仔细研究。如果有错请在页脚联系我，谢谢）

现在这些概念都出来了，我们为什么根据这些条件就能得到非阻塞的只读事务呢？

首先我们想如果一个普通的单机事务，如果想要串行化，读事务想要读到写事务的修改，是不是需要读事务开始时间大于写事务提交时间，这样才符合ACID。分布式事务也一样。在2PC阶段，我们认为coordinator发出commit之后读时候开始去读就一定能读的到。

我花了如下一张图，可以看到事务写完成时的非阻塞读以及，写事务没完全完成时的读。

![1594232392870](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/09/021954-988844.png)

重点讲的是我开始产生迷惑的部分，如有错误希望指正