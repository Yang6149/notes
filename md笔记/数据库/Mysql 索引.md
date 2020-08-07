# Mysql 索引

## 覆盖索引

覆盖索引就是只通过索引就可以得到结果不用查那一行，也就是**不用回表**

现在设主键为：`individual_user_general_id`

然后我们有两个列 `nickname` ,  `biography`,建立一个联合索引

![1592553660369](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/19/160100-495354.png)

现在我们根据普通的索引去查主键以及通过nickname去查biography就是覆盖索引

```sql
EXPLAIN SELECT biography FROM individual_user_general WHERE nickname='AQXOPZXUSVOOIYJ'

EXPLAIN SELECT individual_user_general_id FROM individual_user_general WHERE biography='TGKHDRLRKILJWBNQIKZB'
```

![1592553786604](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/19/160307-511215.png)



## 索引下推

Index Condition Pushdown

```sql
ALTER TABLE individual_user_general ADD INDEX bio(biography);

ALTER TABLE individual_user_general ADD INDEX pic(picture);

EXPLAIN SELECT * FROM individual_user_general WHERE biography='UIOEENWLGFIGLLELSKSH' AND picture='ZYYEHAHEEXKCSOAQPE'
```

我们先构建两个索引，然后通过这两个单列索引联合起来搜索整行数据

现在有三种办法：

1. 只用第一个索引，然后回表遍历
2. 只用第二个索引，然后回表遍历
3. 分别用两个索引，先遍历一个，再把结果在第二个where条件上进行过滤，然后回表。比前两种回表次数少。

## 普通索引（回表）

如果只是利用非聚簇索引查正行数据，就需要回表

```sql
ALTER TABLE individual_user_general ADD INDEX bio(biography);

EXPLAIN SELECT * FROM individual_user_general WHERE biography='UIOEENWLGFIGLLELSKSH'
```

![1592556606880](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1592556606880.png)

* using where;using index：代表先通过索引拿到数据集，在此基础上进行where筛选。不回表
* using index condition：先通过索引查找，再回表