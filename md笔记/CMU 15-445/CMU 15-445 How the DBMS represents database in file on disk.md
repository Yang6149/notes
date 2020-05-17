# CMU 15-445 How the DBMS represents database in file on disk

开新坑了！数据库搞起！

## 第一部分

### 1. DBMS

数据库管理系统是分为很多个组件，一个组件建立在另一个组件之上的，就像上一个我写的，存储系统建立在了 Raft 之上。

![1589633588276](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/16/205308-255734.png)

数据库的目的就是为了减少磁盘的读取数据的时间。我们都直到从 CPU 的一级缓存到磁盘读取数据一次缓慢，我们的数据库就是建立在 磁盘-RAM 之间的工具。

### 2. FILE STORAGE

数据库的表现形式就是一个文件或几个文件。文件是由不同的 PAGE 组成，方便我们进行访问管理。

![1589633889274](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/16/205819-402685.png)

### 3. PAGE

1. Hardware Page 一般位 4KB

2. OS Page 在 Linux 和 Windows 中一般为 4KB

3. DataBase page

   page 是一块固定大小的空间，里面可以包含一些 metadata、index、logRecode 、tuple等等取决于怎么设计 page。

![不同数据库关于 page 的大小](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/16/210028-236542.png)

有两种存储 page heap 的方式:

1. Linked List

   ![1589634286933](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1589634286933.png)

2. Page Directory

![1589634360071](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/141831-298638.png)

### 4. Page data

一个 page 里面一般包含两部分：header和data

data 根据不同设计也可以为：log 和 tuple

tuple 结构中最广为流传的是这种：

![1589634503905](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/16/210825-36533.png)

删除中间的数据可以自动紧凑也可以手动紧凑

日志结构我就不多说了，和我在Raft中操作的差不多形式。

### 5.Tuple

一个 tuple 实际就是一串字节，我们抽象的赋予它意义，比如一个 tuple 我们给他几个属性

```sql
CREATE TABLE table_name (
    column1 datatype,
    column2 datatype,
    column3 datatype,
   ....
);
```

根据这种格式每一行就是一个 tuple。每个 tuple 在最前面还会有一个 header包含一些 metadata

## 第二部分

### 1. 数据表示

不同的数据类型，在表示以及计算时差异很大。在选择数据类型时，需要考虑效率、可扩展性、实用性。

![1589696416978](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/142017-65229.png)

### 2. 可变精度的变量

在即将要完成的 Bustub 中使用IEEE-754标准指定的“本机”C / C ++类型。CPU 可以对这种类型直接进行处理，运算速度更快

如 ：Float

### 3. 定点精度变量

具有任意精度和比例的数字数据类型。 通常存储在精确的可变长度二进制文件中具有附加元数据的表示。

如：：NUMERIC，DECIMAL

### 4. BLOB

一般我们不会允许一个 tuple 超过一个页的大小，但是如果我们想要存储一个 blob（A Binary Large OBjec）或一个 很大的 varchar 的类型怎么办呢？

如果时 varchar 类型 ，会在tuple 上放一个指针指向一个 overflow page

![1589696946049](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/142909-327118.png)

如果是 blob 类型，会引用外部文件，但是 DBMS 无法直接操纵外部文件内容

![1589696931745](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/142852-667728.png)

### 5. OLTP 和 OLAP

1. OLTP
   * 短暂的事务
   * 重复的短读写
   * 占用资源少

2. 
   * 长时间运行处理数据
   * 复杂连接较多
   * 批读取批输入
   * 分析数据使用较多

### 6. 行存储和列存储

#### 行存储

好处：

1. 插入、更新、删除效率高
2. 适用于查询整个元组的数据

缺点：

1. 经常会只需要一个属性的数据却把整行读取了

存储格式为：

![1589697266531](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/143426-826971.png)

#### 列存储

好处：

1. 可以只拿到单一或几列的数据，减少无效列的读取
2. 可以更好的压缩，因为同一列的数据类型相同

缺点：

1. 由于元组拆分/拼接，点查询，插入，更新和删除的速度很慢

![1589697486016](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/17/143806-10998.png)