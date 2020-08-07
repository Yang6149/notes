# Query 优化

一般书写SQL的时候，我们不会考虑内部的 query plan 是怎样的，不同的查询计划性能可以查很多

## 1. 两种优化策略

1. 基于规则的：通过重写SQL来消除低效操作
2. 基于成本的：使用成本模型来评估多种等价计算的成本，选择成本最小的。

## 2. 基于规则优化

1. where 下沉，可以在join等操作时处理更少的tuple

   ![1590513213946](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/27/011335-56856.png)

2. 修正表达式

   通过直接修改低效的表达式来达到提升效率，一般都是一些非常 stupid 的SQL造成的。

   ![1590513297461](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/27/011557-538828.png)

   ![1590513362412](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/27/011604-787658.png)

   ![1590513405643](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1590513405643.png)

## 3. 基于成本优化

DBMS的优化器将使用内部成本模型来估计特定查询计划的执行成本。 这提供了一种估计，以确定一个计划是否优于另一个计划而不必实际运行查询（这对于数千个计划来说会很慢）。具体细节这部分我也没怎么理解，不过大概知道个思想。

1. Derivable 统计

2. 存储统计

3. 当然也可以使用抽样法。