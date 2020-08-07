# CAP

CAP分为口嗨及严格定理

口嗨：可以是定义模糊、涵盖广泛、启发分布式系统设计的原则

定理：属于每个性质都有严格数学定义、结论有完善推导的**理论计算科学**范畴的概念

我们经常说的大概就是口嗨类型的。

## CAP 理论概述

一个分布式系统最多只能同时满足 一致性（Consistency）、可用性（Availability ）、分区容错性（Partition tolerance）这三项中的两项

![1585631137104](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/31/130537-936975.png)

## CAP 的定义

### Consistency 一致性

“`Consistent is: “Atomic, linearizable, consistency [...]. There must exist a total order on all operations such that each operation looks as if it were completed at a single instant. This is equivalent to requiring requests of the distributed shared memory to act as if they were executing on a single node, responding to operations one at a time.”`”

一致性指所有节点在同一时间的数据完全一致。这是在多个数据拷贝下进行并发读写时会出现的问题。

从两个视角来看

* 客户端

从客户端来看，一致性主要指的是多并发访问时更新过的数据如何获取的问题。

* 服务端

从服务端来看，则是更新如何分布到整个系统，以保证数据最终一致。

强一致性：要求更新后立刻能看到。

弱一致性：容忍后续的部分或全部访问不到

最终一致性：经过一段时间后能访问到数据

### Availability 可用性

“`Available is: “For a distributed system to be continuously available, every request received by a non-failing node in the system must result in a response.”`”

去任何一个没有失败的节点的请求，系统一定要给一个成功应答。

### Partition Tolerance 分区容错性

“`Partition is: “The network will be allowed to lose arbitrarily many messages sent from one node to another. When a network is partitioned, all messages sent from nodes in one component of the partition to nodes in another component are lost.”`”

在任意节点挂掉或者消息丢失的情况下，系统还能够运行。

##  CAP 定理的证明

[论文地址](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.6951&rep=rep1&type=pdf)

![1585632687286](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/31/133128-626567.png)

假设现在有两个节点 N1、N2

如果这个时候N1和N2间发生了网络异常，数据不能同步了，但是我们的系统设计的还是要支持这种网络异常的，就相当于满足了分区容错性，这个时候就单独考虑一致性和可用性了

* 满足一致性

  我们在网络异常时还想要满足一致性，client1向N1节点写入一条数据，这个时候我们访问节点N2去读取这条数据，那肯定是访问不到的呀，很可能会进行错误返回或阻塞，毕竟网络异常了嘛。所以这个时候不满足可用性。

* 满足可用性

  还是拿上一行举例子，我们想满足可用性，就是访问N2读取数据的时候一定成功，并且能拿到数据，但这个时候拿到的数据就不是client1传向N1的数据，因为还没同步，所以不满足一致性。

## CAP 权衡

引用DDIA作者的一些话。

> #### **CAP**定理没有帮助 
>
> CAP有时以这种面目出现：一致性，可用性和分区容忍：三者只能择其二。不幸的是这 
>
> 种说法很有误导性，因为网络分区是一种错误，所以它并不是一个选项：不管你 
>
> 喜不喜欢它都会发生。 
>
> 在网络正常工作的时候，系统可以提供一致性（线性一致性）和整体可用性。发生网络 
>
> 故障时，你必须在线性一致性和整体可用性之间做出选择。因此，一个更好的表达CAP 
>
> 的方法可以是一致的，或者在分区时可用。一个更可靠的网络需要减少这个选 
>
> 择，但是在某些时候选择是不可避免的。 
>
> 在CAP的讨论中，术语可用性有几个相互矛盾的定义，形式化作为一个定理并不 
>
> 符合其通常的含义。许多所谓的“高可用”（容错）系统实际上不符合CAP对可用 
>
> 性的特殊定义。总而言之，围绕着CAP有很多误解和困惑，并不能帮助我们更好地理解 
>
> 系统，所以最好避免使用CAP。

* CA without P

  要求满足一致性和可用性，但是分区错误一定会存在的，因此CA的系统更多的是允许分区后各子系统保持CA。

* CP without A

  相当于保持强一致性，但是P会让数据同步时间无限期的延迟，这样 CP 是可以保证的。很多传统数据库分布式事务都属于这种模式

* AP without C

  在高可用并且允许分区，则需要放弃一致性。一旦分区错误，节点之间失去联系，在访问时就会各个节点间数据不同步。众多 NoSQL 都属于此类。

大多数网络服务基本会保证AP，然后保证一个最终一致性

在涉及到金钱财务方面，应该要保证CA，即使网络错误停止服务也不能出现一致性的问题。还有就是保证CP不保证A，可以只读不写。