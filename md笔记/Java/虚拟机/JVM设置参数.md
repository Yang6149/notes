# 设置参数

不断补充中...

## 设置堆大小

```
-Xmx10m //设置堆最大10M
-Xms10m //设置堆最小10M
-Xmm5m  //设置年轻代5M
```

## 切换GC

```
-XX:+UseSerialGC			Serial+Serial Old
-XX:UseParallelGC			Parallel scavenge+Serial Old
-XX:UseParallelOldGC		Parallel scavenge+Parallel Old
-XX:UseConcMarkSweepGC		Serial+CMS  可通过(-XX+UseParNewGC)替换YoungGC
-XX:UseG1GC					G1
```

## oom时自动 dump 堆

```
-XX:+HeapDumpOnOutOfMemoryError
```

## 设置metaspace

```
-XX:MetaspaceSize=N			设置初始化大小
-XX:MaxMetaspaceSize=N		设置最大大小
```

## 其他

当老年代的连续空间小于新生代所有空间时，取true会判断历次MinorGC大小与老年代比较（有风险），仍然小于就FullGC。取false直接FullGC

```
-XX：HandlePromotionFailure=false 
```





# 工具

## 查看 java进程 jps

![1592637876279](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/160601-778787.png)

## 图形化界面 jconsole

![1592638233411](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/153034-757735.png)

可以方便的查看进程的各种信息。

## jstat 

![1592638423309](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/153343-608.png)

后面参数可以跟 pid 以及打印间隔

可以根据一些参数如

-gc、-gcnew、-gcold来查看具体的值



## jstack

通过这个指令可以打印一些栈的信息

和jconsole中打印出的一样如下

![1592638897823](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/154138-322891.png)

说明 jconsole也在调用jstack或使用同一个方法  

## jmp

```
jmap -dump:file=a 10264
```

这个语句把堆信息打印到文件中。

还可以直接打印出堆的一些信息

```
jmap -heap pid
```

这里windows出了点问题，使用linux打印

![1592639372776](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/154933-634721.png)

## jvisualvm

我通过这个指令并没有出现该程序，可能是版本问题。去下载了一个

打开后界面是这样的

![1592639872942](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202006/20/155753-660077.png)

相比于 jconsole ，界面更加美观了，而且点一点就可以主动触发GC，并且触发dump。这里我点了一下GC

并且提供了查看的工具

![1592640070851](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1592640070851.png)