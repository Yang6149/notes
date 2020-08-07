# CS144 TCP Receiver

现在我们已经有了一个整合器Reassembler，可以完成我们对不同数据片段的整合处理。现在开始 TCP reveiver的逻辑处理部分

## Wrapper

在 Reassembler 中我们是使用的无符号64位的整数类型来表示序号，但是我们可以看一下TCP的header

![img](https://camo.githubusercontent.com/e873847d8d54eaeb4351a19a5114eb5431c84241/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f323031393033313731373531323931352e706e673f782d6f73732d70726f636573733d696d6167652f77617465726d61726b2c747970655f5a6d46755a33706f5a57356e6147567064476b2c736861646f775f31302c746578745f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c3164314d4441774f546b352c73697a655f31362c636f6c6f725f4646464646462c745f3730)

在64位的情况下我们不用担心空间被用完，32位的情况下很有可能空间被用完，而且我们也知道在第一次建立连接的时候seq是随机的一个值，我们称为ISN（ Initial Sequence Number）

![1590771526923](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/30/005847-513416.png)

首先要弄明白这几个的意思，比如建立连接时附带syn的seq位2^32-2，之后传了一个data为‘cat’的segment那么关系就如下图

![1590771697009](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1590771697009.png)

那么我们就需要一个函数对这个操作进行封装，把32位的seq转化为64位的abs_seq,把64位的abs_seq转化为32位的seq。

从64位转化为32位很简单，这一步叫做包装，把真实seq转化为可以放入segment的形式。只需要加上ISN然后取余就好。

从32位转化为64位比较麻烦，需要考虑一些special case。比如我们根据传过来的值来计算这次大小

```c++
uint32_t offset = seq - wrap(checkpoint, isn);
```

那么 offset 很有可能是一个很大的数字。造成这种状况有两个可能

1. 传过来的 seq 的真实abs_seq比 checkpoint还要小，由于是无符号位，所以负数就会变得特别大，这个时候就要进行特殊处理

   ```c++
   	uint32_t offset = n - wrap(checkpoint, isn);
       uint64_t ret = checkpoint + offset;
       if (offset >= (1u << 31) && ret>=(1ul<<32))
           ret -= (1ul << 32);
       return ret;
   ```

2. 第二种情况就显的不真实了，可以看到我们上一步判断是否为上一种情况是比较 offset时候大于1<<31，因为可以看 TCPheader，报文最大也就是窗口大小，窗口大小一次最大为16位也就是1<<16，所以基本不可能是真的数据太大。

## TCP receiver

这里主要是处理接收到数据之后对数据的不同类型进行处理。

定义 TCP的header数据

```
uint16_t sport = 0;         //!< source port
    uint16_t dport = 0;         //!< destination port
    WrappingInt32 seqno{0};     //!< sequence number
    WrappingInt32 ackno{0};     //!< ack number
    uint8_t doff = LENGTH / 4;  //!< data offset
    bool urg = false;           //!< urgent flag
    bool ack = false;           //!< ack flag
    bool psh = false;           //!< push flag
    bool rst = false;           //!< rst flag
    bool syn = false;           //!< syn flag
    bool fin = false;           //!< fin flag
    uint16_t win = 0;           //!< window size
    uint16_t cksum = 0;         //!< checksum
    uint16_t uptr = 0;          //!< urgent pointer
```

主要对这一个函数进行处理`bool TCPReceiver::segment_received(const TCPSegment &seg) {}`

1. 在接收到数据之后首先判断，自身状态和header.syn以及header.fin状态之间的关系。注意要把重复操作杜绝掉。
2. 第二部就是对自己的数据进行处理，首先根据ackno()计算出窗口开始地点，根据window_size()计算窗口结束的地方。如果传来的segment的数据和窗口没有相交部分就返回false
3. 第三个要注意的点就是判断segment 中data的size的时候，需要考虑syn、fin等信息，也要算入在内，然后在把数据交给Reassembler的时候还要把这部分排除在外。
4. 在得到 abs_seq之后，需要转化位reassembler能接收的streamIndex形式，所以要进行一次转换。
5. 在设置了SYN、和FIN 的Segment时，需要进行单独判断是否在窗口来并且判断是否包含来返回正确结果。

代码后续会贴在github中。