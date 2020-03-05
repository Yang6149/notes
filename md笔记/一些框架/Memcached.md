# Memcached

memcached 是高性能的分布式内存缓存服务器。它通过缓存数据库查询结果，减少数据库访问次数，以提高动态Web应用的速度、提高可扩展性。memcached 的 API 使用 32 位元的循环冗余校验（CRC-32）计算键值后，将资料分散在不同的机器上。当表格满了以后，接下来新增的资料会以LRU机制替换掉。Memcached基于一个存储键/值对的 hashmap 。其守护进程（daemon）是用 C 写的，但是客户端可以用任何语言来编写，并通过 memcached 协议与守护进程通信

## 分布式哈希算法

* 余数 hash
* 一致性哈希

## Memcached 的数据清除算法

 当 Memcached 使用内存大于设置的最大内存使用时，使用 slabs_alloc 函数申请内存失败时，就开始淘汰数据了。 

## Memcached 的工作流程

先检查客户端请求的数据是否在 memcached 中，如果有，直接把请求数据返回，不再对数据库进行任何操作；如果请求的数据不在 memcached 中，就去查数据库，把从数据库中获取的数据返回给客户端，同时把数据缓存一份到 memcached 中（memcached 客户端不负责，需要程序明确实现）；每次更新数据库的同时更新 memcached 中的数据，保证一致性；当分配给 memcached 内存空间用完之后，会使用LRU 策略加上到期失效策略，失效数据首先被替换，然后再替换掉未使用的数据（即使该数据还没有到期）。

## Memcached 和 Redis 的区别

1. Redis 不仅支持简单的 k/v 类型的数据，同时还提供 String、List、Set、zset 和 hash 等数据结构的存储。memcached 支持简单的数据类型、String.
2. Redis 支持数据的备份，即 master-slave 模式的数据备份。?
3. Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候还可以再次加载使用，而 Memcached 把数据全部存放在内存中
4. Redis 的速度比 Memcached 快很多
5. Memcached 是多线程，非阻塞 IO 复用的网络模型，Redis 使用单线程的 IO 复用模型。



**有持久化需求或者对数据结构和处理有高级要求的应用，选择 Redis，其他简单的 key/value 存储，先择 Memcached**。对于两者的选择需要要看具体的应用场景，如果需要缓存的数据只是 key-value 这样简单的结构时，则还是采用 Memcached，它也足够稳定可靠。如果涉及到存储，排序等一系列复杂的操作时，毫无疑问选择 Redis。