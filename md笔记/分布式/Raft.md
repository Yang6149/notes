# Raft

<http://thesecretlivesofdata.com/raft/>神仙描述

## 一、intro

1. 保持 replicated log  一致

   一致模块接收到用户的commands 然后加到 log 中。

   互相communicated 保证 log 最终一种，即使一些servers挂了。

特点：

1. 安全：不会返回错误的结果
2. 高可用：五台机器挂两个没问题，修复后还能rejoin。
3. 不依赖于时间去保证log一致性：错误的时钟或极端的消息延迟可能会造成的问题。
4. 通常来说，一个命令在集群的majority响应一轮rpc就可以完成了。少数速度慢的服务器不会影响性能。

分解为三个子问题

1. leader election 
2. log replicated
3. Safety