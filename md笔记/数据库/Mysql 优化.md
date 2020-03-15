# Mysql 优化

1. 优化在数据库层

   * 表结构适合吗？列类型适合吗？表有适合工作的列数吗？比如经常更新就需要表的列少点，分析大量数据经常要少量表with大量的列
   * 有使用正确的索引使查询更效率？
   * 是否为每个表使用合适的引擎，更好的利用每个引擎的特性？比如在事务型引擎`InoDB`和非事务型引擎`MyISAM`中的选择对性能和可扩展性很重要。
   * 每个表都用了适合的 row format 吗？这也取决于表使用的存储引擎的选择。在时间中，压缩表使用更少的磁盘空间，因此使用更少的 I/O 去读写数据。压缩表可用于工作中的所有 InnoDB 表，也可用于只读 MyISAM 表

   * 应用是否使用适合的锁策略？分享锁提高并发
   * 内存空间使用正确的缓存大小？

2. 在物理层优化，为了 avoid bottleneck
   * 在磁盘搜索中，现代磁盘查一次约 10 ms，因此理论上每秒执行 100 次搜索。优化寻道的方法是把数据分散在多个磁盘上。
   * 磁盘读写。磁盘在正确的地方的时候，现代磁盘，能提供 10-20M/s 的吞吐量。与查找相比优化简单，因为可以在多个磁盘上并行读数据。
   * CPU 周期。当数据存储在主内存中，这经常成为大表的限制因素。
   * 内存带宽：cpu 需要的数据大于 cpu 缓存的时候，内存带宽就成为了 bottleneck。但这很少出现。

## 1. SQL 语句优化

## 2. Index 优化

 Most MySQL indexes (`PRIMARY KEY`, `UNIQUE`, `INDEX`, and `FULLTEXT`) are stored in [B-trees](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_b_tree). Exceptions: Indexes on spatial data types use R-trees; `MEMORY` tables also support [hash indexes](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_hash_index); `InnoDB` uses inverted lists for `FULLTEXT` indexes. 

### 2.1 怎么用索引

* 通常 Mysql 会在多个索引中寻找查询行数最少的索引进行使用
* 如果表有多列索引，优化器可以用最左前缀索引 For example, if you have a three-column index on `(col1, col2, col3)`, you have indexed search capabilities on `(col1)`, `(col1, col2)`, and `(col1, col2, col3)`. 
* 使用表连接时，如果声明相同的类型和大小，MySQL 可以更加有效的使用索引 For example, `VARCHAR(10)` and `CHAR(10)` are the same size  , but `VARCHAR(10)` and `CHAR(15)` are not. 

### 2.2 Primary Key

主要时得益于 NOT NULL 的优化

### 2.3 空间索引优化

可用于在 NULL 值上创建 `SPATIAL` indexes 

### 2.4 列索引

索引涉及单个列，可以在 WHERE 字句中快速找到与运算符 `=`, `>`, `≤`, `BETWEEN`, `IN`, and so on 相应的一个或一组值。

所有存储引擎每个表至少支持 16 个索引，并且限制长度至少为 256 个字节。

* 前缀索引，可以仅仅使用该列的前 n 个字符作为索引。

 ```sql
CREATE TABLE test (blob_col BLOB, INDEX(blob_col(10))); 
 ```

* #### FULLTEXT Indexes

仅 InnoDB 和 MyISAM 存储引擎支持 FULLTEXT 索引，仅支持 CHAR，VARCHAR 和 TEXT 列。

仅返回文档 ID 或搜索等级很有效

## 3. 磁盘优化

 MRR，全称「Multi-Range Read Optimization」。 

 简单说：**MRR 通过把「随机磁盘读」，转化为「顺序磁盘读」，从而提高了索引查询的性能。** 