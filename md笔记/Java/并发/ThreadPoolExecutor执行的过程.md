# ThreadPoolExecutor执行的过程

探究Executor

核心线程数1

最大线程数1

阻塞队列1

调用方法为sleep一个小时

## 第一次execute

查看当前worker的数量是否小于核心线程数量。小于的话就addWorker(Runnable,core=true)

进入addWorker后查看当前缓冲池的状态，是否shutdown之类的。

再次判断核心线程数、最大线程数等一些列判断条件之后，如果可以增加Worker就调用

![1591869815953](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1591869815953.png)

CAS的增加一个Worker数量，保存当前状态，再退出循环

紧接着初始化一个worker

获取一个锁mainLock，接着再次判断线程池状态。

workers是一个Set里面包含现有的worker，把刚才的worker 加进去，更新最大线程池size

解锁，设workerAdded=true

如果workerAdded=true，就直接run这个任务

![1591870409222](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/11/181708-87152.png)

## 第二次exectue

查看当前worker数量不小于核心线程数量，然后判断下当前的状态直接往阻塞队列里仍，如果返回true则代表加入成功

![1591870869214](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/11/182109-210818.png)

接着无事发生，直接退出。

## 第三次execute

和上一步一模一样只不过由于队列慢了进不了if代码块

![1591871300549](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/11/182822-2768.png)

进入了addWorker里面，core=false

接着还是和第一次execute一样通过CAS更新pool的状态，跳出循环

紧接着初始化一个worker，加入到Set中，然后run该线程

## 第四次exectute

会经过一些状态判断调用reject(command)

然后调用

![1591871736302](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/11/183538-951016.png)

## 当1或3执行完后，取新任务

由于创建好的线程会调用run，里面有个while循环，第一行getTask会去取当前队列中的任务，如果为空就阻塞，如果不为空就取任务执行。

由此我们可以看出这个阻塞队列永远只有 取阻塞 没有 存阻塞 ，因为会根据状态判断执行reject

```java
while (task != null || (task = getTask()) != null) {
    w.lock();
    // If pool is stopping, ensure thread is interrupted;
    // if not, ensure thread is not interrupted.  This
    // requires a recheck in second case to deal with
    // shutdownNow race while clearing interrupt
    if ((runStateAtLeast(ctl.get(), STOP) ||
         (Thread.interrupted() &&
          runStateAtLeast(ctl.get(), STOP))) &&
        !wt.isInterrupted())
        wt.interrupt();
    try {
        beforeExecute(wt, task);
        try {
            task.run();
            afterExecute(task, null);
        } catch (Throwable ex) {
            afterExecute(task, ex);
            throw ex;
        }
    } finally {
        task = null;
        w.completedTasks++;
        w.unlock();
    }
}
```



## 前面太乱不看

1. 第一次执行：Runnable赋给worker.thread直接运行
2. 第二次执行：Runnable赋给worker.thread加入到阻塞队列
3. 第三次执行：Runnable赋给worker.thread直接运行
4. 第四次执行：调用拒绝策略

当线程启动后就不会停止，执行完后会马上去取新任务。