# ReentrantLock() 分析

此代码基于 JDK 12

首先 new 一个 ReentrantLock() 是调用了

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认空参数为 NonfairSync();

然后网上一层一层找，发现 继承关系为

 ![img](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNe9WmkGVYDYOTmfVtaiaicMC9mxQ89DqFylIR5UUjhRRtTcibCIox3MoTKg/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

这个 AbstractQueuedSynchronizer （简称 AQS ）继承于  AbstractOwnableSynchronizer(AOS) ，AOS 主要保存获取当前锁的线程对象。 FairSync 与 NonfairSync的区别在于，是不是保证获取锁的公平性，因为默认是NonfairSync，我们以这个为例了解其背后的原理。 

 ![img](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNeXQXPJJnVVuE5w60FFE0QCABvYnicbbZb1JbxcpLAWelHJjOiasFUgQsA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

再看 Node 是什么？

 ![img](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNeRz24CqfiaoekbpibKfYUJaHKUnEr9loicmiaRwyNSUVIicUVjDw5YCudE7w/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

 **最后我们可以发现锁的存储结构就两个东西:"双向链表" + "int类型状态"。** 

 需要注意的是，他们的变量都被"`transient`和`volatile`修饰。 

 ![img](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNeEt6nua7h39OIREUaNrEDhQMicKJDlsVHhKicvicrIyHJUr184fBF6PRng/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

 我们接下来看看，如何获取锁？ 

```java
public void lock() {
    sync.acquire(1);
}
```

 可以看到调用的是，`NonfairSync.acquire(1) `  此时如果是公平锁不会立刻 CAS 而是先查询 `hasQueuedPredecessors()`

此时 jdk 8 版本：在 acquire() 里面进行判断`hasQueuedPredecessors()`队列为空？ 是否排队

jdk 12 中在 tryAcquire 中进行判断`hasQueuedPredecessors()`

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&//拿到锁 返回 !true,拿不到返回 ！false继续判断
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

* tryAcquire ( arg )   会尝试通过CAS获取一次锁。 
* addWaiter : 将当前线程加入上面锁的双向链表 （等待队列） 中。
*  acquireQueued：通过自旋，判断当前队列节点是否可以获取锁。 

tryAcquire 调用 nonfairTryAcquire

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();//获得当先线程
    int c = getState();//得到同步状态值//默认为0，有人持有不为0
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {//自旋一波//公平锁会先判断队列再自旋，把state改为1
            setExclusiveOwnerThread(current);//设置当前线程获得锁
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//重入
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;//拿不到锁返回false
}
```





 **addWaiter 添加当前线程到等待链表中** 

```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

acquireQueued 进入 当当前线程到头部的时候，尝试 CAS 更新锁状态，更新成功就从头部移除。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
```

 ![img](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNe2WBQibNRtn2oYbjzicvJxsibN811WNg4U5owU6TwTczUCOkiciaWZREoXFw/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

概括

 ![img](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNeX3L0W2SWKGawoEEnUt5R8iaMX41ss6C06iccARY6q7qhDDzIYgLdfbJQ/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

## lock.unlock()

```java
public void unlock() {
    sync.release(1);
}
```

 可以看到调用的是，NonfairSync.release() 

 ![img](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNezNjmbT05MYI1s1FWSGIibaWWc31JacNUNUrN25NOpUdNdxKYgmL1YLA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

 最后有调用了NonfairSync.tryRelease() 

 ![img](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KZL3KHm7AEo7iaGm3v3brNe74KPgZfKlGWicYW4o3yK3Rmibyl44NCLSHywfENDamKBBg7HgM7MfYgA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

