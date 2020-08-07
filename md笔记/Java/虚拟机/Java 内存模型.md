# Java 内存模型

总结自《java并发编程的艺术》

## 1. volatile 底层实现

在对 volatile 修饰的变量进行写操作时，汇编指令会发送一条 lock 前缀的指令，作用为：

* 将当前处理器缓存行的数据写回到系统内存。
* 这个写回内存的操作会使在其他 CPU 里缓存了该内存地址的数据无效。

在读取 volatile 修饰的变量，多处理器下，为了保证各个处理器的缓存一致，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是否过期。当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置为无效状态，当处理器对这个数据进行修改操作时，会重新从系统内存中读取到处理器缓存里。

## 2. synchronized

Java 中每个对象都可以作为锁。

* 对于普通同步方法，锁是当前实例对象。
* 对于静态同步方法，锁是当前类的Class对象
* 对于同步方法块，锁是 synchronized 括号里配置的对象，

## 3. Java 内存模型

### 3.1 内存模型基础

线程之间的通信方式是有两种：共享内存和消息传递

Java 的并发采用的是共享内存模型。

在 Java 中，所有实例域、静态域、和数组元素都存储在堆内存中，堆内存在线程之间共享。JMM 定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读、写共享变量的副本。本地内存是 JMM 的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

![1585465263943](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585465263943.png)

如果想要线程间通信的话需要如下两步

![1585465320131](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/150201-773016.png)

在执行程序时，为了提高性能，编译器和处理器通常会对指令做重排序。重排序分为三类：

1. 编译器优化的重排序。
2. 指令级并行的重排序
3. 内存系统的重排序

![1585465644222](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/150734-166936.png)

上述的1属于编译器重排序，2和3属于处理器重排序。

**happens-before**

遵循 as-if-serial语义，不管怎么重排序，（单线程）程序的执行结果不能改变。编译器、runtime、和处理器都必须遵守 as-if-serial

Java 不会对存在数据依赖关系的指令进行重排，如果一个操作的结果对另一个操作可见，那么这两个操作之间必须存在 happens-before 关系。既可以在同一线程之内也可以是在不同线程之间。

程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作

监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读

传递性：a->b  b->c   a->c

> 注意两个操作间有happens-before并不意味着前一个操作必须要在后一个操作之前执行，它仅仅要求前一个操作对后一个操作可见，且前一个操作按顺序排在第二个操作之前

![1585467232722](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/153354-429626.png)

### 3.2 重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列重新排序的手段。

在一个 A happens-before B的情况中，A的结果不需要对于B可见，而重拍前后结果一致则允许重排。

在多线程中，由于指令重排，会出现逻辑上的错误比如

![1585467660433](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/154101-131598.png)

线程一执行writer、线程二执行reader，线程二不一定能看到a=1.

### 3.3 顺序一致性

这里我们的目的也明确了，**程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同**

顺序一致性内存模型为程序员提供的视图如图所示。 

![1585468365713](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/155309-133223.png)

在任意时间点最多只能有一个线程可以连接到内存。

顺序一致性模型中，所有操作完全按程序的顺序串行执行。而在JMM中，临界区内的代码可以重排序

![1585468690304](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/155810-103558.png)

虽然线程A在临界区内做了重排序，但由于监视器互斥执行的特性，这里的线程B根本无法“观察”到线程A在临界区内的重排序。这种重排序既提高了执行效率，又没有改变程序的执行结果。

非同步程序在两个模型中的差异：

1. 顺序一致性模型保证单线程内的操作会按程序的顺序执行，而 JMM 不保证单线程内的操作会按程序的顺序执行。
2. 顺序一致性模型保证所有线程只能看到一致的操作执行顺序，而JMM 不保证所有线程看到的一致的操作执行顺序。
3. JMM 不保证对 64 位的 long 型和 double 型变量的写操作具有原子性，而顺序一致性模型保证对所有内存读写操作具有原子性。

### 3.4 volatile 应用

volatile变量的写-读可以实现线程之间的通信。

![1585469178421](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/160620-851136.png)

假设线程 A 执行writer() 方法之后，线程 B 执行 reader。根据 happens-before 规则，这个过程建立的 happens-before 关系可以分为 3 类：

1. 根据程序次序规则，1 happens-before 2;3 happens-before 4。 
2. 根据volatile规则，2 happens-before 3。 
3. 根据happens-before的传递性规则，1 happens-before 4。

volatile写的内存语义：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内 

存。 

volatile读的内存语义：当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主 

内存中读取共享变量。

**volatile语义实现**

* 在每个volatile写操作的前面插入一个StoreStore屏障。 
* 在每个volatile写操作的后面插入一个StoreLoad屏障。 
* 在每个volatile读操作的后面插入一个LoadLoad屏障。 
* 在每个volatile读操作的后面插入一个LoadStore屏障。 

### 3.5 锁的内存语义

锁除了可以让临界区互斥执行，还可以让释放锁的线程向同一个锁的线程发送消息。

![1585470871288](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/163432-934680.png)

假设线程 A 执行writer() 方法，随后线程 B 执行 reader() 方法。

1. 根据程序次序规则，1 happens-before 2,2 happens-before 3;4 happens-before 5,5 happens-before 6。 

2. 根据监视器锁规则，3 happens-before 4。 
3. 根据happens-before的传递性，2 happens-before 5。 

**锁的释放和获取的内存语义**

A线程释放锁后，共享数据的状态示意图如图所示。

![1585471032727](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585471032727.png)

这里和volatile一样，就不累述了。

### 3.6 final

1. 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。 
2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。 

### 3.7 happens-before

happens-before是JMM最核心的概念。对应Java程序员来说，理解happens-before是理解JMM的关键。

JMM把happens-before要求禁止的重排序分为了下面两类。

* 会改变程序执行结果的重排序（JMM禁止这种行为）
* 不会改变程序执行结果的重排序（JMM 允许这种行为）

![1585472397413](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/165959-320897.png)

**定义：**

1. If *x* and *y* are actions of the same thread and *x* comes before *y* in program order, then *hb(x, y)*.
2. There is a *happens-before* edge from the end of a constructor of an object to the start of a finalizer ([§12.6](https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.6)) for that object.
3. If an action *x* ***synchronizes-with*** a following action *y*, then we also have *hb(x, y)*.
4. If *hb(x, y)* and *hb(y, z)*, then *hb(x, z)*.

**Synchronizes-with**

Synchronization actions induce the ***synchronized-with*** relation on actions, defined as follows:

- An unlock action on monitor *m* *synchronizes-with* all subsequent lock actions on *m* (where "subsequent" is defined according to the synchronization order).
- volatile 读写，不多说
- An action that starts a thread *synchronizes-with* the first action in the thread it starts.
- 各个变量初始化默认值  synchronized-with  线程执行第一个指令
- 如果线程1发现线程2 terminated，线程2 synchronized-with 线程1
- If thread `T1` interrupts thread `T2`, the interrupt by `T1` *synchronizes-with* any point where any other thread (including `T2`) determines that `T2` has been interrupted (by having an `InterruptedException` thrown or by invoking `Thread.interrupted` or `Thread.isInterrupted`).

### 3.8 双重检查锁问题

![1585473440368](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/171726-413550.png)

这里有一个错误的优化！在线程执行到第4行，代码读取到instance不为null时，instance引用的对象有可能还没有完成初始化。

正常行为为：

![1585473605037](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/172005-131144.png)

有可能重排为:

![1585473627044](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585473627044.png)

结果就是：

![1585473694393](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585473694393.png)

在还没初始化时，就开始访问了。

解决：

* 可以使用 volatile 修饰 instance。在写入的时候不允许指令重排



## 功利行为：



1. JMM 是什么？

Java内存模型，是java虚拟机规范中所定义的一种内存模型，Java内存模型是标准化的，屏蔽掉了底层不同计算机的区别。主要是为了在提高性能的前提下，解决指令重排导致的执行结果不一致和线程间的可见性的问题。它屏蔽了底层复杂的实现过程，定义了一个 happens-before原则规范，来帮助我们解决这些问题。happens-before原则就是，在单线程内环境下，遵循 as-if-serial ，在不违背原则的情况下进行指令重排，通过 volatile 或 synchronizes-with 进行规定它的 happens-before的顺序，然后在多线程状态下通过它的传递性来实现多线程状态下的 happens-before。





- 

