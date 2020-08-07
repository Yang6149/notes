# TCP sender

我们需要把自己的stream里的数据读取出来然后发送给receiver。

为此我们需要从stream中读取然后放进outstanding queue里面，这个 queue 存放的是我们已经发出去并且还没有收到ack的segment。还要有一个定时器，在指定时间内没有收到ack的话就重传最前面的segment，随着重传次数增多需要放弃这次的连接。

难点为：

1. 对每次传输数据的前期预处理
2. 对窗口间segment填充时的边界处理

一共要完成5个方法供其他人调用

1. fill_window:在窗口内塞满要传过去的数据
2. ack_received:接收到ack信息，用于更新一些ack和窗口大小
3. tick:用于上层调用，传入时间，如果超时就重传outstanding queue中最前面的segment，超时多次可能会断开连接
4. send_empty_segment:发送一个空segment，在后续可以用作 ACK报文

## 1. outstanding queue

把这部分设计为一个队列的原因是，我们我们每一次收到的ACK的seq指的是在seq之前的所有segment都已经确实收到了。所以我们可以把ack大于segment-seq+size的所有queue中的segment删除。我们放入的顺序也是从小到大，所以可以很有效的完成比较和查询功能。

## 2. fill_window

检查window的大小，从stream中读取数据然后切分成不超过1452Byte的 data，然后放入segment 中，需要注意第一次传输数据只包含一个SYN信息，不包含数据。最后一次传输信息可以包含data并包含FIN信号。最后一次传输的判定为stream的input口关闭，并且从中读取完了所有数据。

## 3. ack_received

首先要用作判重，防止stale data 来干扰现在的状态。在接收到 ACK 信息后，改变window大小，在传来相同ACK的术后要注意仍然可以改变大小。

这是我额外定义的一些 metadata，用于辅助判断状态。

```c++
	bool _is_send_syn,_is_finished ;//连接状态
    uint16_t _window_size;//当前窗口大小
    size_t _time_to_out,_time_out;//定时器，定时最大值
    std::queue<TCPSegment> _outstanding{};
    uint64_t _ack_seq;
    unsigned int _consecutive_count;//重传次数
```

