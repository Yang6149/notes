# HashTable & B+Tree

## HashTable

### 1. Hash Function

需要考虑的问题有，如何把较大的 key 映射到一个域上，然后还要保证速度块和低碰撞。在接下来的实验中使用到[xxhash](https://github.com/Cyan4973/xxHash)

### 2. Static Hashing Schemes

这个代表 hash table 的大小是固定的。这就代表当我们随着存储数据过大而 overflow 之前，我们需要及时的 rebuild 一个 hash table。

为了减少冲突，优化 Hash Table，以下是一些常见方案

#### 2.1 Linear Probe Hashing

这是最基本的 Hash 方案，一般情况也是最快的方案。如名线性探索：就是一旦我们发现key hash 的位置上已经有值了，就把它插入到往下数第一个空 slot 上。

#### 2.2 Robin Hood Hashing

针对 linear probe hashing 的优化，为了减少出现某一个 hash 后向下跳 slot 的次数过多，为每一个 key 增加一个 count 保存与本来 hash 位置的举例，如果key 在向下跳的时候发现，count 大于当前位置的key 的count，就让后面的一次往后挪（插队）。

![1589978092646](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/203453-93220.png)

#### 2.3 Cuckoo Hashing

用多个表+多个Hash函数来防止发生冲突，通常是两个表。

例如：Hash1(x)=0，插入到了表1的0，然后来一个Hash1(y) = 0,发现表1冲突，就用Hash2(y),插入到 表2中，如果再来一个Hash1(z)==Hash1(x) && Hash2(z)==Hash(y),则y插入到表1中，x 用 Hash2 插入到 表2中。

特别极端情况，x、y、z的Hash1和Hash2 的结果都相同，就会造成死锁。

![1589978585482](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/204305-175042.png)

### 3. Dynamic Hashing Schemes

动态方案可以有效减少 rebuild Hash的需求。

#### 3.1 Chained Hashing

这个就不多BB了，Java 的 HashMap 就是用的这个。

#### 3.2 Extendible Hashing

现在我们还可以进行优化，为了防止上一个方案链长一直边长(Java 中超过阈值就 扩容了但这并不是我们想要的)，我们可以分裂表格，直接上图。

![1589984443829](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/222044-278611.png)

假设我们现在图是这样的，00和01目前的所有数据都指向同一个表，并且表示深度为1，就是只去最前面一位。其余两个深度为2，取前两位。

这个时候我们插入B

![1589984610575](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/222331-406817.png)

可以看到第二个表格满了，但这个时候如果再插入C会怎么办？

![1589984644605](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/222405-647202.png)

没错，它就会分裂，一个深度为2的表格分裂为两个深度为3的表格。一部分数据不用移动，一部分数据移动到另一个表格，其余所有表格不受影响，Hash Function 的全局决策发生略微变化。

#### 3.3Linear Hashing

这种方法是有一个指针指向某一位index，一旦有一个表要 overflow 了，就分裂 split Pointer 指向的表，及时这个表并没有 overflow，然后马上 overflow的表采用链式拓展。

![1589985488090](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/223808-561907.png)

比如 现在我要插入一个 17，那么 1指向的表要 overflow 了，那么我就拓展 1 指向的表，然后把 0 分为两个表，并把 index 拓展到4

并且增加一个 hash function

![1589985718944](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/224201-925671.png)

而且这是个可逆操作，当减少一个 table 时，就把split pointer 往前移一位。

这是前两种方案的折中策略，防止链表过长和pointer table 过长。

## B+Tree

讨论B+Tree 前我们先说一下索引，我们想要找某一个 tuple ，如果是线性扫描，效率会极其底下。我们要做的就是定义某一个、几个、一组属性为一个索引，那么我们就可以根据这个索引快速的定位到这个tuple。

B+树就是一个极其效率的结构。

B+树是一个 M-Way 平衡搜索树，区别与B-树只有子节点保存数据。

这里给一个可视化的网站 [Data Structure Visualizations](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

我这里就不总结什么插入删除这些实际很难用到的操作了，想要了解的在网站里实际插入删除几个值就能马上明白了。

关键我们是要知道为什么用它，什么时候用它。

### 1. Node Size

在 MySQL 中一个 Node 的size 刚好是 16KB 也是它的一个页的大小（每个页是一个node）。

但理论上是数据读取速度越慢，node size 越大。所以在设计大小时，应该考虑是数据存放在哪？

在内存中，就可以设计的小一点（512B），在机械硬盘中就要设计大一点（1MB）。正常来说固态硬盘设计10KB是推荐的。

### 2. Merge Threshold

在 B+树的一个节点分裂后，需要设一个阈值防止马上融合为一个节点，这样频繁分裂融合会导致性能过差。

### 3. Variable Length Key

可变的 Index 有以下解决方法：

1. key 作为 Index 的指针（很少有人用）
2. 正常使用 Index 作为 Key，要好好管理内存防止泄漏或溢出，容易出现碎片。
3. 填充
4. 构建一个排好序的map Key->offset，指向本node内的KV所在偏移量。这也是大多数人的选择。

![1589989084460](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/20/233804-999016.png)

在node内搜索可以顺序搜也可以二分搜索。

### 4. B+Tree Optimizations

#### 4.1 Prefix Compression

通常数据量大的时候，一个 node 中的所有Key 都具有相同的前缀，使用前缀压缩可以减少 Key 的大小来存放更多 degree。

#### 4.2 Suffix Truncation

截去后缀和上一个刚好相反，在前缀都不相同的情况下，截取后缀可以粗略的区分不同的 Key ，也可以节省空间。

#### 4.3 Bulk Inserts

海量数据排好序一起插入，只需要构建一次 B+树，而One-By-One 的插入，每一次插入都要进行维护，速度会满。

### 5. Covering Indexes

查询所需要的所有属性都在索引当中，当然要遵循最左原则，因为根据索引排序有优先级，先根据最左边的索引排。类比Stringbi大小。

### 6. Trie Index 和 Radix Tree

![img](https://upload-images.jianshu.io/upload_images/10803273-8df9472eaa3730a7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https://upload-images.jianshu.io/upload_images/10803273-864a36c671ccf97d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https://upload-images.jianshu.io/upload_images/10803273-30d7f6d8202fce3f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

