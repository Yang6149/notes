# MIT6.824

* parallism
* fault tolerance
* physical
* security/isolated

### Lab

1. MapReduce
2. Raft for fault tolerant
3. use lab2 to build tolerant key-value server
4. Sharded K/V service(use lab3 and clone it into a number of independent groups and split the data in key-value storage system)

### Infrastructure

| Abstractions  | Impl                           |
| ------------- | ------------------------------ |
| Stroage       | RPC\Threads\concurrent Control |
| Communication |                                |
| Computation   |                                |

Performance

Scalability

fault Tolerant

​	Avaliability	| non-volatile storage

​	Recoverability	| Replication

Consistency

​	

### MapReduce

this is a simple Word Count process by MapReduce

![1584524379876](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/18/174004-146925.png)

![1584524662930](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/18/174425-753647.png)

The Map k could be the title,it's not important .And the Map V could be the content ,we put the k/v to map ,then we get a table .

The Reduce k coule be a vector. Then we can get the number of the word.

![1584586954470](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/19/110241-839054.png)