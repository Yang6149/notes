# Sort & Aggregations

## 1. Sort

排序是 DBMS 中很重要的一个环节，在进行一些操作如 ORDER BY、GROUP BY、JOIN、DISTINCT 操作和 Bulk Insert 插入构建B+树的过程中都需要运用到排序。

很多时候我们还不能用 B+树 的索引来排序。聚簇索引还好说，我们可以根据叶子节点遍历，毕竟空间在一个页中是连续的。非聚簇索引来排序就会造成大量的随机IO，速度慢的一批。

### 1.1 External Merge Sort

#### 1.1.1 2-Way Sort

这个有点像面试时经常问道到的大文件排序问题。就归并排序嘛。

![1590059305958](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/190827-185107.png)

每一个节点是一个 page，page 内部就自己内部排序了。

（1 + log2 N ）* 2N 是总 IO 次数。一共只需要3个Page

#### 1.1.2 k-Way Sort

这种方法可以有效降低Number of passes = 1 + ⌈ logB-1
⌈N / B⌉ ⌉从而降低总 IO 次数

### 1.2. Using B+Tree

在非聚簇索引的情况下排序就离谱，每一次IO一个页的一个 slot

![1590059735509](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/191536-454765.png)

## 2. Aggregations

聚合操作包括 sum、count、max等。实现方式有排序和hash两种

### 2.1 Sort

![1590060133676](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/192215-946199.png)

### 2.2 Hash

hash 相比 排序要快得多。分为两个不知 partition 和 ReHash

* Partition

  先进行一次Hash1，把相同Hash key 的tuple 放入同一个 Bucket，如果太大的话就写进磁盘中。

  ![1590072201391](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/21/224322-937623.png)

* ReHash

  一个一个的读取partition ，Hash2构建一个Hash 表，在这一步我们可以进行聚合操作比如求平均数

  ![1590072242296](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1590072242296.png)