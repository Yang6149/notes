# Synchronized 相关

在1.6之后引入了偏向锁和锁升级

![1584349226059](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/11/001047-446896.png)

## 1. Java 头对象和 Monitor

### 对象头

32位

![1584350763471](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/16/172604-513442.png)

64位：

![1584350732526](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/29/223804-919554.png)

Mark Word用于存储对象自身的运行时数据，如哈希码(HashCode)、GC分代年龄、锁状态标志、线程持有的 锁、偏向线程 ID、偏向时间戳等等。Java对象头一般占有两个机器码(在32位虚拟机中，1个机器码等于4字节， 也就是32bit)

### Monitor

Monitor 可以理解为一个同步工具或一种同步机制，通常被描述为一个对象。每一个 Java 对象就有一把看不见的锁，称为内部锁或者 Monitor 锁。

对于synchronized语句当Java源代码被javac编译成bytecode的时候，会在同步块的入口位置和退出位置分别插入monitorenter和monitorexit字节码指令。

**monitor 依赖于 操作系统的 Mutex Lock，需要进入内核态，会消耗一些cpu资源。**



synchronize 同步方法实现synchronized 方法会被翻译成普通的方法调用和返回指令如: invokevirtual、areturn 指令，在 VM 字节码层面并没有任何特别的指令来实现被 synchronized 修饰的方法，而是在 Class 文件的方法表中将该方法的 access_flags 字段中的 synchronized 标志位置 1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的 Class 在 JVM 的内部对象表示 Klass 做为锁对象。

```java
public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
```

从字节码中可以看出，synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是 synchronized 锁在同步代码块和同步方法上实现的基本原理。

## 2. 无锁

无锁是指没有对资源进行锁定，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。

无锁的特点是修改操作会在循环内进行，线程会不断的尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。如果有多个线程修改同一个值，必定会有一个线程能修改成功，而其他修改失败的线程会不断重试直到修改成功。

**轻量锁**

当代码进入同步块时，如果同步对象为无锁状态时，当前线程会在栈帧中创建一个锁记录(`Lock Record`)区域，同时将锁对象的对象头中 `Mark Word` 拷贝到锁记录中，再尝试使用 `CAS` 将 `Mark Word` 更新为指向锁记录的指针。

如果更新**成功**，当前线程就获得了锁。

如果更新**失败** `JVM` 会先检查锁对象的 `Mark Word` 是否指向当前线程的锁记录。



## 3. 偏向锁

当线程1访问代码块并获取锁对象时，**会在java对象头和栈帧中记录偏向的锁的threadID**，因为偏向锁不会主动释放锁，因此以后**线程1再次获取锁的时候，需要比较当前线程的threadID和Java对象头中的threadID是否一致**，如果一致（还是线程1获取锁对象），则无需使用CAS来加锁、解锁；如果不一致（其他线程，如线程2要竞争锁对象，而偏向锁不会主动释放因此还是存储的线程1的threadID），那么需要查看Java对象头中记录的线程1是否存活，如果没有存活，那么锁对象被重置为无锁状态，其它线程（线程2）可以竞争将其设置为偏向锁；如果存活，那么立刻查找该线程（线程1）的栈帧信息，如果还是需要继续持有这个锁对象，那么暂停当前线程1，撤销偏向锁，升级为轻量级锁，如果线程1 不再使用该锁对象，那么将锁对象状态设为无锁状态，重新偏向新的线程。
![1584351346919](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/16/173548-159079.png)

偏向锁在 JDK 6 及之后版本的 JVM 里是默认启用的。可以通过 JVM 参数关闭偏向锁：-XX:-UseBiasedLocking=false，关闭之后程序默认会进入轻量级锁状态。

## 4. 轻量级锁

轻量级锁是指当锁是偏向锁的时候，却被另外的线程所访问，此时偏向锁就会升级为轻量级锁。

加轻量级锁过程为：在当前线程创建一个空间存放对象的Mark Word ，然后同过 CAS 操作把自己的对象头的 Mark Word 替换为指向 锁记录的指针。

其他线程会通过自旋的形式尝试获取锁，线程不会阻塞，从而提高性能。基本是通过自旋+CAS的方式

轻量级锁的获取主要由两种情况：

1. 当关闭偏向锁功能时

2. 多个线程竞争导致偏向锁升级为轻量级锁

   

**自旋次数过多，或有三个以上线程竞争锁**，就会膨胀为重量锁

## 5.重量锁

从用户态到内核态，去挂起线程。
