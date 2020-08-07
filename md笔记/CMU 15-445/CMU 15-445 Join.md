# Join

这里讨论的都是自然连接

## 1. OutPut

首先我们输入是两个表或者说 tuple list，我们的目的就是筛选出需要的数据，构建成一个新的 tuple list

![1590253685954](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/010808-34558.png)

## 2. JOIN ALGORITHMS

### 2.1 Nested Loop Join

直接嵌套的方式进行连接，假设R包含M个page，S包含N个page

![1590253802663](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011002-509102.png)

这样的话，我们每一个 tuple r 都要和S中所有的page进行连接。那么IO cost为

![1590253888846](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011129-860567.png)

也可以根据Block进行嵌套，读取两个Block后再根据tuple嵌套，这样可以减少IO次数

![1590253959426](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011248-45746.png)

![1590253978698](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011259-198311.png)

### 2.2 Sort and Merge

可是上一种还是太慢，现实中没人会使用。

第二种方法先进行排序，再根据两个排序号的结果进行读取，相同数据match进行输出。

![1590254074761](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011435-467.png)

当S index 向下移动不在match R index 数据，S index 复位，R向下移动。

![1590254150103](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011550-713588.png)

![1590254169006](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011609-815901.png)

### 2.3 Hash Join

第三种方法为进行Hash，两个表分别使用相同Hash function，hash 到同一张表中，相同key当然会进入同一 partition

![1590254272701](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/24/011753-800260.png)

其中还可以进行一些优化，比如我们构建一个bloom filter，在对R进行Hash之前先insert 进 bloom filter，这样就可以降低S进行Hash时占用的多余空间以及无效次数。