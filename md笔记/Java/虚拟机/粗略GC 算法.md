# GC 算法

## 判断对象是否去世

### 引用计数算法

### 可达性分析

GCroot：

1. 虚拟机栈中引用的对象
2. 方法去中类静态属性引用的对象
3. 方法区中常量引用的对象
4. JNI(本地方法栈)中引用的对象
5. Java 虚拟机内部的引用，基本数据类型的Class对象、常驻的异常对象
6. 所有被同步锁持有的对象
7. 反应Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。

1. 四种引用类型

   1. 强引用
   2. 软引用：内存溢出前可以进行回收
   3. 弱引用：只能活到下次gc之前
   4. 虚引用：是否有虚引用完全不影响gc、也无法通过引用调用对象。目的就是对象被收集器回收时收到系统通知

2. 自救

   第一次可达性分析到达不了，会进行一次标记，并放入F-Queue中，进行执行finalize()，这时可自救，智能逃脱一次。稍后会对这些对象进行第二次标记，进行回收。

## 方法区回收

对象->废弃的常量和不再使用的类型

不再被使用的类：

1. 所有实例都被回收
2. 加载该类的类加载器被回收
3. Class对象没被引用

在大量反射、动态代理、cglib场景、动态生成jsp通常需要这种能力

## 垃圾收集算法

### 3.1 分代收集理论

![1590683198356](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/29/002638-250018.png)

### 3.2 标记-清除算法

![1590683287630](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/29/002812-777899.png)

### 3.3 标记-复制算法

![1590683318972](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/29/002840-616627.png)

HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1

### 3.4 标记-整理算法

![1590683398492](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/29/002958-373121.png)

### 3.5 卡表

![1590687189394](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/29/013310-102702.png)