# MySQL 主从操作防迷路指南

### docker 安装mysql

```
docker run \
 -p 3306:3306 \
 --name mysql \
 -v $PWD/mysql/conf:/etc/mysql/conf.d \
 -v $PWD/mysql/logs:/logs \
 -v $PWD/mysql/data:/var/lib/mysql \
 -e MYSQL_ROOT_PASSWORD=123456 \
 -e TZ=Asia/Shanghai \
 -d mysql:5.7
```

注意改name、端口、版本号，根据需求修改挂载

### master配置

在 conf 下创建 my.cnf

```
## 同一局域网内注意要唯一
server-id=100  
## 开启二进制日志功能，可以随便取（关键）
log-bin=mysql-bin
## 想要设置专门的复制数据库需要专门指定
## 也可以指定忽略哪个数据库
```

初始化

```sql
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';

GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```

创建一个`slave` 用户给slave们使用，以便它们连接主数据库直接读取 binlog

配置完成后重启 docker

### 配置 slave

```
server-id=1
log-bin=mysql-bin
relay_log=edu-mysql-relay-bin

```

初始化

```sql
stop slave;

change master to master_host='192.168.23.129', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos= 2830, master_connect_retry=30;

start slave;

show slave status;
```

注意 pos 要和 master 中的 pos 一致，不然老出现找不到 binlog

`Got fatal error 1236 from master when reading data from binary log: 'Binary log is not open'`

配置这个东西配了超级久，网上配置每一个相同的，还是自己摸索出来一个，记下来防迷路。

只是完成了一个功能，但 linux 操作越来越熟了，挺开心的。