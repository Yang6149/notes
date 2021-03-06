# MIT6.824 Lab2B

2B的内容要比2A的内容复杂得多，先贴成果吧。这个博客并没有Raft 的实现过程以及解释，如果有想交流的可以加在QQ或发邮件什么的，都在最下面。

花了快两个小时跑了 100 遍 -race 的测试，所以应该没什么问题了。

![1586434731849](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/09/201852-700034.png)

最开始的难处，还是不知道论文中参数的具体意思，可能是我英语太菜了吧。找到了 Raft 的[可视化过程](https://raft.github.io/raftscope/index.html)，光是各种情况的传输过程的调试以及做笔记，就花了至少2个小时。然后好好再对照着论文看明白  Figure 2，又花了好久，迟迟不敢动手。

![1586436844352](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1586436844352.png)

最后到了晚上，思维很不清晰的情况下开始下手写代码，还是基本每写一段代码就想半天有没有想错以及写错。可以说很慎重了，就怕写出了啥逆天bug，后患无穷。

## 实验目标

这一次的实验目标是完成对数据的复制。

主要是实现Figure2，在2A 中没实现的部分。![1586437222208](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/09/210051-64220.png)

完成所有测试用例，测试用例包括：

1. 基本复制实现
2. 传输的数据保证完全正确
3. 在出现错误的情况下依然可以运行
4. 在多数节点挂掉后保证不commit，数据安全
5. 并发向leader传输数据
6. 再连接分区leader（测试直接过了，就没往里研究）
7. 未提交覆盖数据
8. 不能出现太多 RPC

## 遇到的问题

第一个就是不知道怎么提交数据

没有好好看代码：

![1586437302329](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1586437302329.png)

注释上写的好好的，这里卡了一段时间，蠢啊

阻塞问题：

最早遇到的问题依然是阻塞，但由于有了先前的经验，很快就冲日志中发现问题，以及确认错误点。

异常判断问题：

这个可以说是一堆问题，因为确实我在 code 的时候，很多条件没有想明白，导致出现各种错误，比如重复发送 chan 数据、错误判断对对 nextIndex[] 的错误改变，初始化问题、等等一系列。

这些其实全都是没思考明白就往里写，出现错误再排查，前期还好，代码量很少，我打的 log 输出也很少，在后面第七个测试函数中，我下午3点多就写好跑通了，然后用脚本打算跑个一百遍，恰巧这个测试函数是跑的最慢的一个，平均要25秒才能跑一遍，然后在第30 多遍的时候，出现了错误，运行时间过长自动退出，我看了一下打出的log 妈耶 30W行。好在从第10W行开始才是错误代码的log。为了精准是哪个环节出错，又在测试用例上面打满了log，后面就是不断地写log，跑测试，这个过程极为痛苦，毕竟想跑出一次错误要花很久。最后确认是，两次相同 appendEntry 在rpc 的过程中，同时被处理，多 commit 了一次，进行了chan 阻塞。经过判断过滤后，又跑了一百遍，终于过了。

## 担忧

其实这一次我对上一次的 2A 代码进行了很多改动，使它更加高效了，同时我也觉得我越写越与论文中的细节处理有稍许偏差，当然我觉得自己的思考没有问题，目前跑了那么多测试也过了，希望以后不会因为这个吃亏。

## 总结

这一次比想象中进行的顺利，但是逻辑也逐渐复杂了起来，现在代码、注释、log的比例大概是 2：4：4。我这里也其实有一些效率问题，比如没有进行 leader 领先的情况下连续请求。目前的想法是对每个 follower 都进行单独计时。

马上也要进行 2c 了，祝好运。