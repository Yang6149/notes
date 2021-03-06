# MIT6.824 Lab2A

本来打算写完整个lab2 之后再写博客记录地，无奈这个2A真的是把我搞炸了。

## 实验目标

阅读 Raft 论文，并实现 Raft 算法。拆分为三次小实验，2A 的目的只是实现论文中的选举功能，目标是可以达到：

1. 高效选举
2. 保持一致性
3. 具有容错机制，断开可以保证重连

## 遇到的问题

#### 三次重构

这里就要好好唠唠了，由于原先的多线程经验真的少，并且这里还是分布式多线程，按着代码骨架以及论文上的一些要点就实现了一遍。里面的一个要点就是需要实现两个计时器，如果一定时间没有接收到消息就触发某个方法，接收到消息就延迟触发。follower 就是不断接收 heartbeat 来控制自己的election timeout。刚开始我也不熟悉 go 的语言特性以及库，就谨慎的记录时间戳，并且循环检查当前时间以及比较时间戳、短暂的sleep、更新时间戳。

伪代码大概是这个样子

```go
for{
    if time.Now().Unix()>rf.timeout{
        election()
    }else{
        sleeep(10)
    }
}
func appendRpc(args,reply){
	//...
    rf.timeout=time.Now().Unix()+randTimeout()
    //...
}
```

最后写完后，折腾了 2 点测试每一次 2A 的测试达到了 8s - 8.6s

由于看了[Raft Structure Advice](https://github.com/Yang6149/MIT-6.824-Distributed-Systems/blob/master/Lectures/LEC03/raft-structure.txt.md)之后，说

![1586249975456](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/07/165935-544458.png)

我就不服了，查time的文档，然后马上重构。内心觉得肯定要比傻傻的sleep要高效啊。想当然的看到timer有一个Reset方法。就想当然的觉得这个就是延迟的作用。折腾半天重构完后后来跑了一遍测试。 老是出现一个 warning 大概是在没有错误的情况下，依然出现 term 的递增。这怎么行呢？毕竟term 只能在 election 的过程中才能递增啊，一直竞争 leader 会导致后来的 replica 的过程很多无 commited 的传输以及大量 overwrite logEntry。然后老老实实的看了官方文档。

![1586250494878](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/07/170815-848598.png)

当时心态就崩了，又写了测试代码，在 Stop()之后也是需要等待上一个 channel 的数据读出之后才能,如下：

```go
if !t.Stop() {
	<-t.C//需要等待
}
t.Reset(yourTime)
```

那意思不就是永远不会延迟timer吗？只能复用。

这个时候心灰意冷，git reset 回原来的版本后，又写了个测试脚本跑，想着如果跑100 遍没有错误的话，效率低就效率低吧。然后大概跑到50+遍的时候，出错了！没错第50多遍才出错，我寻思着，我用 test -race 也没检测出Race问题，然后也没出现死锁，就是在一个节点断开后再连接，老leader重新连接接收到新leader之后的一次rpc操作后，就失联了。鬼知道为什么，我甚至翻遍了测试代码，在每个操作后面都打上日志。跑一次代码只要不到10秒，看一次日志要半小时。最惨的要跑几十次才能跑出一次错误，还分析不出什么东西重新跑？

一气之下代码全删了，重构！

这一次我翻了翻 go 的其他特性，之前的 timer 好像是用的 channel，那么我也用 channel，正好看到 select 等待 channel 的过程中，可以使用 time.After打断,这不正是我想要的效果吗？

看到这时间已经接近晚上3点了，第二天还得上课，先睡了。

再次动手就是第二天的下午2点多，挂着网课继续搞。这一次重构一遍就直接跑通了测试（熟练的心疼自己）。继续挂脚本跑测试，大概跑到了7-8遍的时候，又出错了，代码停不下来，这个时候就熟练多了，只会出现两个问题，一是连接阻塞，二是锁被channel阻塞。

又是通过打满log之后进行分析，连接没有阻塞，但是锁老是进不去，而且编译器报发生死锁，-race 也没问题。那么就肯定是我的代码逻辑问题。又是检查了半天，由于我是在每一次切换状态之后都重新设置每个节点的 channel，然后出现了这样的一种情况

```go
1. 
select{
    case <-rf.chan:
    //lock
    //...
    //unlock
}

2.
//切换状态调用
rf.chan=make(chan int)

3.
//其他各种条件
rf.chan<-1
```

由于 1 在等待 channel 时没有加锁，2和1不在同一线程

正常来说在调用3 前1一定是在等待的顺序为1>3

但可能出现，毕竟无阻塞循环也有极小的概率在两个指令之间并行执行了其他指令

大概顺序就是1>3>3>2>1

然后就发生阻塞，以及如果发生这种行为的时候就代表3发送的信息已经过期，没有用了，所以最终解决办法是`rf.chan=make(chan int,10000)`多塞几个不会造成阻塞，并且之间塞进去的也直接丢弃，go 也有垃圾回收机制，空间也不大放心丢弃。



期间也出现过各种各样的小问题，究其原因也都是没有好好遵循paper

真实应了这句

![1586252251069](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/07/173731-292356.png)

好好看（指paper）好好学。

还值得一提的就是，把代码分割开来可以有效的提升效率。1000+行的全逻辑代码，互相交互、各种参数啊、锁啊、channel啊、goroutine 啊，上下翻看头晕。

把东西分开就舒服多了

![1586252440249](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/07/174043-609836.png)

现在的版本不仅比最初版本优雅，效率也高多了。

## 开发环境

另外说一下开发环境，这一次改用了 WSL+VScode。原先用的Goland，无奈的是好像不支持 wsl，每天打开电脑时都要重新设置一下项目的 path，而且它的检错机制也不友好。后来用了 vscode 后，哇真个世界都量了。轻量！老年机特别友好，各种有效的插件，自带wsl插件，完美提高效率，原来都是要双屏来回看然后切切切，毕竟还要查看其他东西例如文档、paper、[Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)等一系列其他读物，现在好多了，一个屏幕读东西，一个屏幕写代码+运行

大概这个样子![1586252841626](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/07/174722-289546.png)

期间也解决了 WSL 的咳血上网问题

## 总结

其实从看论文到第一次跑通2A，花了两天，但是后来出现的各种错误，翻看各种资料以及重构，最后花费的时间大概也是有1个星期了。多线程还是要深思熟虑再写，不然出现错误，光是发现错误在哪看日志就要半天，再接再厉吧，讨论组里的人都说 2B 是最难的。希望5天内能稳妥的结束2B。