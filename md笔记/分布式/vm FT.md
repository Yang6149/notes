# LEC 4

在副本挂掉后，应该重新创建一个副本，并且也只能使用 State Transfer,把所有信息clone 出去。

![1585819043778](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202004/02/171724-851519.png)

工作原理：

1. primary 和 backup 都跑在虚拟机上，好处是可以有效的监控不确定性的 event ，维护两者的一致性（cpu时钟不一致、中断）。
2. 首先 client 先发送一条操作信息到 primary 。
3. primary 把信息通过 logging channal 同步到 backup 上（相比复制全部状态省时省力）
4. 等 backup 上也执行完后， primary 返回，backup 的返回信息在 VMM 这一层被抛弃。
5. 当 primary 挂掉之后，backup 担当 primary。

![1585819264791](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585819264791.png)

Primary 会通过 log 来通知 Backup 的一些非确定性的指令以及时间，比如中断，所以 primary 一定要比 backup先执行

为了防止 backup 在 primary 之间执行

![1585820339930](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585820339930.png)

这有个 buffer ，只有当 backup 的 buffer 中至少有一条事件指令时才会执行，以免发生 Backup 先执行