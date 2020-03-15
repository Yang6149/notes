# Spring 事务的隔离级别和传播行为

## 1. 事务的隔离级别

1.  **DEFAULT** 这是一个这是一个PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别. 
2.  **未提交读（read uncommited）** :脏读，不可重复读，虚读都有可能发生 
3.  **已提交读 （read commited）**:避免脏读。但是不可重复读和虚读有可能发生 
4.  **可重复读 （repeatable read）** :避免脏读和不可重复读.但是虚读有可能发生. 
5.  **串行化的 （serializable）** :避免以上所有读问题. 

## 2. 事务的传播行为

在同一个事务当中

1.  PROPAGATION_REQUIRED 支持当前事务，如果不存在 就新建一个(默认) 
2.  PROPAGATION_SUPPORTS 支持当前事务，如果不存在，就不使用事务 
3.  PROPAGATION_MANDATORY 支持当前事务，如果不存在，抛出异常 

保证没有在同一个事务当中

1.  PROPAGATION_REQUIRES_NEW 如果有事务存在，挂起当前事务，创建一个新的事务 
2.  PROPAGATION_NOT_SUPPORTED 以非事务方式运行，如果有事务存在，挂起当前事务 
3.  PROPAGATION_NEVER 以非事务方式运行，如果有事务存在，抛出异常 
4.  PROPAGATION_NESTED 如果当前事务存在，则嵌套事务执行 