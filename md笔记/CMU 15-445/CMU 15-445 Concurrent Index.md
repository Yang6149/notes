# Concurrent Index

## 1. 为什么我们要并发

像是 Redis 在读写数据时就是单线程读写数据，但是我们这个数据是存储在磁盘上的。瓶颈当然是在 disk I/O。那么我们就需要多个线程，在某些线程进行 IO 时其他线程仍然可以进行操作而不是全部阻塞。

## 2. Lock vs Latchs

![1590056806553](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/182648-282416.png)

### 2.1latchs 的实现

1. OS 提供的锁 Mutex
2. [test-and set](https://en.wikipedia.org/wiki/Test-and-set)实现的锁
3. 读写锁

## 3. Hash Table

### 3.1 Page Latches

![1590057730209](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/184211-633781.png)

### 3.2 Slot Latches

hash(d)指向A所在位置，然后根据 Linear Probe往下找。

![1590057779228](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/184302-792086.png)

## 4. B+Tree

比如我们现在想要delete44，同时另一个线程在 find 41，delete 44后会进行rebalance

![1590057973576](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/184622-194461.png)

![1590058034107](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/184714-136649.png)

这个时候虽然 44 握有写锁，但是仍然会出现错误，所以需要引入一些规则。

### 4.1 获取锁算法

这个因为太早，也没名字，就这么叫了。

![1590058249027](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/185049-38457.png)

每个节点都从父节点中获取锁，当父节点保证其子节点绝对安全后才会释放锁。

这里我也没有掌握细节，只知道大体规则。

想要在这种复杂数据结构上进行复杂操作并保证线程安全确实挺难的。一个范围查找+插入删除我就懵了。