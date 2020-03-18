# ThreadLocal

今天看了一下源码，原理出奇的简单

大概流程就是这样

### 流程

![1584328011970](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/16/134857-407175.png)

一个Thread 对象里面有一个 ThreadLocalMap

在每个一个ThreadLocal 对象中，调用 set 对象会把自己注册道**Thread**对象上的 **ThreadLocalMap**上。

![1584328231755](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/16/134838-537541.png)

这里可以把 map 简单的当作一个hashmap理解。

get 也是同样的方法，在ThreadLocalMap 中把 ThreadLocal 对象当作 Key ，来取出对象。

![1584328339067](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/16/134846-811295.png)

### 内存泄漏问题

与 HashMap 不同的是，ThreadLocalMap 的每个 Entry 都是一个对 **键** 的弱引用，这一点从`super(k)`可看出。另外，每个 Entry 都包含了一个对 **值** 的强引用。

使用弱引用的原因在于，（没有强引用 ThreadLocal 时，每个 Thread 中的 ThreadLocalMap 对它弱引用。）及时回收，因为 Thread 只能从 ThreadLocal 中进入 Map 然后获取值。

但这可能会导致该 Entry 无法被移除。解决方法为：在 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null。