# MySQL Explain



先解析一条 sql 语句：

```sql
EXPLAIN SELECT s.uid,s.username,s.name,f.email,f.mobile,f.phone,f.postalcode,f.address

FROM uchome_space AS s,uchome_spacefield AS f

WHERE 1 

AND s.groupid=0

AND s.uid=f.uid
```



 ![img](https://img-blog.csdnimg.cn/20181229105132839.png) 


### 1. id

 select 识别标识符。这是select 查询序列号 , 查询序列号即为sql语句执行的顺序 

1. id 相同时，执行顺序由上至下

2. 如果是子查询，id 的序号会递增，id 值越大优先级越高，越先被执行

3. id 如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id 值越大，优先级越高，越先执行

 ![img](https://img-blog.csdnimg.cn/2018122910520737.png) 

### 2. select_type

1.  SIMPLE(简单 SELECT，不使用 UNION 或子查询等) 
2.  PRIMARY(子查询中最外层查询，查询中若包含任何复杂的子部分，最外层的 select 被标记为 PRIMARY) 
3.  UNION(UNION 中的第二个或后面的 SELECT 语句) 
4.  DEPENDENT UNION(UNION 中的第二个或后面的 SELECT 语句，取决于外面的查询) 
5.  UNION RESULT(UNION 的结果，union 语句中第二个 select 开始后面所有 select) 
6.  SUBQUERY(子查询中的第一个 SELECT，结果不依赖于外部查询) 
7.  DEPENDENT SUBQUERY(子查询中的第一个 SELECT，依赖于外部查询) 
8.  DERIVED(派生表的 SELECT, FROM 子句的子查询) 
9.  UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行) 

```sql
explain select * from uchome_space limit 10 union select *from uchome_space limit 10,10
```

 ![img](https://img-blog.csdnimg.cn/20181229105225974.png) 

### 3. table

 显示这一步所访问数据库中表名称（显示这一行的数据是关于哪张表的），有时不是真实的表名字，可能是简称

### 4. type

对表访问方式，表示 MySQL 在表中找到所需行的方式，又称 “访问类型”。

常用的类型有： **ALL、index、range、 ref、eq_ref、const、system、NULL**（从左到右，性能从差到好）

**ALL**：Full Table Scan， MySQL 将遍历全表以找到匹配的行

**index**: Full Index Scan，index 与 ALL 区别为 index 类型只遍历索引树

**range**: 只检索给定范围的行，使用一个索引来选择行

**ref**: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

**eq_ref**: 类似 ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用 primary key 或者 unique key 作为关联条件

**const、system**: 当 MySQL 对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于 where 列表中，MySQL 就能将该查询转换为一个常量，system 是 const 类型的特例，当查询的表只有一行的情况下，使用 system

**NULL**: MySQL 在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

### 5. possible_keys

**指出 MySQL 能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用（该查询可以利用的索引，如果没有任何索引显示 null）**

该列完全独立于 EXPLAIN 输出所示的表的次序。这意味着在 possible_keys 中的某些键实际上不能按生成的表次序使用。
如果该列是 NULL，则没有相关的索引。在这种情况下，可以通过检查 WHERE 子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用 EXPLAIN 检查查询

### 6. Key

**key 列显示 MySQL 实际决定使用的键（索引），必然包含在 possible_keys 中**

如果没有选择索引，键是 NULL。要想强制 MySQL 使用或忽视 possible_keys 列中的索引，在查询中使用 FORCE INDEX、USE INDEX 或者 IGNORE INDEX。

### 7. key_len

**表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len 显示的值为索引字段的最大可能长度，并非实际使用长度，即 key_len 是根据表定义计算而得，不是通过表内检索出的）**

不损失精确性的情况下，长度越短越好 

### 8. ref

 **列与索引的比较，表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值** 

### 9. rows

 **估算出结果集行数，表示 MySQL 根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数** 

### 10. Extra

该列包含 MySQL 解决查询的详细信息, 有以下几种情况：

Using where: 不用读取表中所有信息，仅通过索引就可以获取所需数据，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示 mysql 服务器将在存储引擎检索行后再进行过滤

Using temporary：表示 MySQL 需要使用临时表来存储结果集，常见于排序和分组查询，常见 group by ; order by

Using filesort：当 Query 中包含 order by 操作，而且无法利用索引完成的排序操作称为 “文件排序”

```sql
-- 测试Extra的filesort
explain select * from emp order by name;
```

Using join buffer：改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

Impossible where：这个值强调了 where 语句会导致没有符合条件的行（通过收集统计信息不可能存在结果）。

Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行

No tables used：Query 语句中使用 from dual 或不含任何 from 子句

```sql
-- explain select now() from dual;
```