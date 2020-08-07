# Query

## 1. Query Plan

我们在接收到一段SQL后需要对语句进行解释，变成让计算机可以理解的格式。这个格式就是 query plan，是由不同的操作组成的Tree结构。

根节点是我们要返回的数据，中间节点是数据的处理过程，子节点是读取数据的过程。

![1590325886471](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/211127-656099.png)

## 2. Processing Models

在有了Query Plan 后就知道需要进行什么操作了。现在考虑怎么执行，执行的顺序和方式。

### 1. Iterator Model

每一个节点都是一个 Iterator，不断的调用next，next的结果就是它的子树的内容。调用完后进行emit返回给自己父级。

![1590326111591](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/211512-974350.png)

通过自顶向下的调用来完成查询。毕竟数据可能大过缓冲池，多次进行IO读取速度太慢，而且生产太多中间数据不太好。会用到pipeline的方式，尽量减少中间数据产生，当然在进行join，subQuery、order 等操作时，会发生 pipeline break。

### 2. Materialization Model

这种方法是Iterate 的过程中，直接获取全部的子数据，再进行计算。这种方式比较适合 OLTP ，因为这种操作一般获取少量的 tuple，获取完数据再处理更加方便。不使用于 OLAP ，因为海量数据读不完，可能需要其他的分盘、分数据等操作。

### 3. Vectorization Model

我们知道数据是存储再一个个的 Page 当中的，所以我们读数据也应该是按Page为单位。所以我们在 Iterator Model 的基础上进行进行一些优化，每次自上而下的调用不是读一个tuple就返回，而是读完整个Block才返回。

## 3. Access Methods

现在让我们讨论读取表中数据的方法。

### 3.1 顺序扫描

就硬扫描，横向遍历读取Table中每个Page的数据。但我们可以做一些优化

1. 预读取
2. 并行读取
3. 读完数据不写回缓冲池，因为我们只扫描一遍就过了
4. Zone Map：通过在Block上写一些每个重要属性的范围来判断是否要读

### 3.2 Index Scan

顾名思义根据索引来读取数据，不多说。

### 3.3 Multi-Index Scan

可以通过对每个索引进行扫描读取到一个Set中，然后在对它们进行去重。期间可以运用 bitmap、bloom filter 的数据结构进行去重和减少无效读取

# Parallel Query

## 1. Parallel database 和 distribute database

**parallel database** wiki：Parallel databases improve processing and [input/output](https://en.wikipedia.org/wiki/Input/output) speeds by using multiple [CPUs](https://en.wikipedia.org/wiki/CPU) and disks in parallel.

![1590391623644](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/25/152736-306482.png)

可以知道并行数据库就是同时使用多个CPU core 和多个 磁盘进行读写来横向扩展速度。

根据不同数据库可以分为两种水平分区和垂直分区

下图是根据列划分

![1590391753315](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1590391753315.png)

误区：SSD多通道主控多个闪存可以同时读写同一个文件，并且突破了机械硬盘的物理移动地址限制，所以更快，和上面不是同一概念。

**Distributed DBMS:**数据本身存储在多个地区，详细可以看之前写的mit6.824中的一些实现部分。

## 2. 并行类型

1. 单个查询的并行操作

![1590390849433](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/25/151409-594975.png)

![1590391517437](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/25/152519-603670.png)

2. 并行多个查询操作

   利用多核优势同时进行多个 查询 操作。

## 3. Process Model

### 3.1 Process per worker

这种方式依靠 OS 的调度，共享相同的数据结构，在一个进程 crash 掉后不会影响其他的进程，IBM的DB2和Oracle 以及 postgreSQL使用这种

![1590391180715](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/25/151940-963868.png)

### 3.2 Process Pool

每当想使用一个进程时都会去进程池中调用一个进程，也是上几家公司在用

### 3.3 Thread per worker

在单进程中创建一个线程池，想要调度时就去线程池中取，MySQL、SQLServer以及Oracle都在用

![1590391411472](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/25/152331-402586.png)