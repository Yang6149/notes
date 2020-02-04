# sql语句  
## DDL  
用来定义数据库对象：库、表、列  
创建、删除、修改 结构

* 数据库  
    1. 查看所有数据库：`SHOW DATABASES`
    2. 切换（要操作的）数据库：`USE 数据库名`
    3. 创建数据库:`CREATE DATABASES [IF NOT EXISTS] mydb [CHARSET=utf8]`
    4. 删除数据库：`DROP DATABASES [IF EXISTS] mydb`
    5. 修改数据库编码:`ALTER DATABASES mydb CHARACTER SET utf8`  
    
* 数据类型  
  
     *  文本类
        1. CHAR(size)
        2. VARCHAR(size)  大于size255转化为TEXT
        3. TINYTEXT
        4. TEXT 65535个字符
        5. BLOB 65535个字节
        6. MEDIUMTEXT 16 777 215个字符
        7. LONGTEXT 4 294 967 295个字符
        8. ENUM(x,y,z) 只能输入预定好的值65535
        9. SET 与MEUM类似，最多包含64个列表项，可存储一个以上的值
     *  数据类
        1. TINYINT(size) -128 to 127      ro 0 to 255
        2. SMALLINT(size) -32768 to 32767
        3. MEDIUMINT(size) -8388608 to 8388607
        4. INT(size)    -2147483648 to 2147483647
        5. BIGINT(size)  -9223372036854775808 to 9223372036854775807
        6. FLOAT(size,d) d规定小数点右侧的最大位数
        7. DOUBLE(size,d) 
        8. DECIMAL  字符串存储的DOUBLE类型，允许固定的小数点
     *  时间类
        1. DATE()  YYYY-MM-DD
        2. DATETIME()  YYYY-MM-DD HH:MM:SS
        3. TIMESTAMP() YYYY-MM-DD HH:MM:SS使用Unix纪元‘1970-01-01 00:00:00’至今的描述存储
        4. TIME()  HH:MM:SS
        5. YEAR()   1901-2155   70-69
    
* 表
    1. 创建表：
        ```
        CREATE TABLE [IF NOT EXISTS] 表名(
            列名 列类型,
            列名 列类型, 
            列名 列类型,
            列名 列类型
        )[ charset = utf8];
        ```

    2. 查看当前数据库中所有表名称：SHOW TABLES;
    3. 查看表结构： DESC 表名;
    4. 删除表 DROP TABLE 表名;
    5. 修改表 前缀：ALTER TABLE 表名  
        * 添加：
        ```
        ALTER TABLE 表名 ADD(
            列名 列类型,
        ...
        )
        ```
        * 修改列类型
        ```sql
        ALTER TABLE 表名 MODIFY 列名 列类型
        ```
        * 修改列名称
        ```sql
        ALTER TABLE 表名 CHANGE 原列名 新列名 列类型;
        ```
        * 删除列
        ```sql
        ALTER TABLE 表名 DROP 列名;
        ```
        * 修改表名
        ```sql
        ALTER TABLE 表名 RENAME 新表名
        ```

## DML
数据操作语言  
对表的记录进行更新（增删改）  

* 插入数据  
     `INSERT INTO 表名(列名，列名..)VALUES('','',..);`
     
* 修改数据
    `UPDATE  表名 SET 列名=值1，列名=值2，...[WHERE 条件]；`不加 where 该列全部修改
    
* 删除数据
    `DELETE FROM 表名 [WHERE 条件];`
    



## DCL
对用户的创建，授权
* 创建用户
`CREATE USER 用户名@ip地址('%') IDENTIFIED BY '密码';`
* 给用户授权
`GRANT 权限1，权限n ON 数据库.* TO 用户名@ip地址`
* 撤销授权
`REVOKE 权限1，权限n ON 数据库.* TO 用户名@ip地址`
* 查看权限
`SHOW GRANTS FOR 用户名@ip地址`
* 删除用户
`DROP USER 用户名@ip地址`
## DQL
* distinct  去重

  `select distinct  col_name  from table_name`

* order by   默认为asc

  `select * from table_name [where] order by col_name [asc/desc][,col_name [asc/desc]...]`

* limit(dialect)

  `select * from table_name [where] [order by col_name] limit[offset,]rowCount `

* insert into and select

  ` insert into table1 [(col1,col2)]select col3,col4 from table2` 

* in 语法

  `select * from table_name col_name in (value1,value2)`value 可用select col_name获得

* between      （a<=b<=c）  not between

* like

  `select * from table_name where col_name [not] like pattern`

  "%"可以匹配任何字符串

* count

  `select count(*) 自定义列名 from table_name` 用于查询该列有多少条值，为Null不计

* sum

  `select sum(slary) from table_name `  例 计算工资（该列值）总和  null为0

* max、min、avg

* **group by**   

  `select job,count(*) from emp [where] group by job`与之前的聚合函数配合使用，每一份组的聚合值

* **having**

  `select job,count(*) from emp [where] group by job having count(*)>=2`

### 连接查询

* 内连接

  `select * from table_1,table_2 ` 笛卡尔积

  `inner join` 加条件   `nature join`自动寻找条件

* 外连接

  左外连接  有一主一次，当不满足条件时，左表完全显示，不满足内容为NULL

  ```sql
  SELECT e.ename , d.dname
  FROM emp e LEFT OUT JOIN dept d
  ON e.deptid=d.deptid
  ```

* 右外连接同理左外连接，右表为主
* 全外为full outer join（mysql不支持）可以通过左外加右外加union关键字实现

### 子查询

子查询可以在 **from** 后，也可以在 **where** 后

* 多行单列`sal > ALL(查找语句)` or`sal > ANY (查找语句)`  `IN`
* 单行多列`select * from table_1 where (col_1,col_20) IN (select col_1,col_2 from table_2 where 条件)`
* 多行多列`select * from table_1 别名1,(select ...)别名2 where 条件`





关键字执行顺序（**select、from、where、group by、having、order by**）

## 数据的备份与恢复

* `mysqldump -p用户名 -u密码 数据库名>路径`
* `mysqldump -p用户名 -u密码 数据库名<路径`
* `在客户端里面 source 路径名`

## 约束

* PRIMARY KEY
* NOT NULL
* UNIQUE
* AUTO_INCREMENT
* `CONSTRAINT 约束名fk_从表_主表 FOREIGN KEY(本表被约束属性) CONFERENCES 主表(主表的主键)`

#### 合并结果集

```sql
select a , b from table_1
union all
select c , d from table_2
on 条件
```

加上`all`全部显示，不加的话完全相同的行会消除

