# CMU15-445 Buffer Pool & Project1-Fall2019版

### 1. Buffer Pool 是什么

Buffer Pool 是暂时存放再内存中的一块数据，数据内容为从磁盘中读取到的页的信息。

### 2. 为什么要用 Buffer Pool

简单的说是为了题速，由于读取速度从CPU->磁盘依次减慢

![1589886039471](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/19/190042-875268.png)

我们需要想办法把磁盘中的数据暂时存放在内存中，buffer pool 是我们用来存放这些信息的地方。

### 3. Buffer Pool 的结构

Buffer Pool 可以设计为一个对象数组，用来存放一个个的 Page 对象，由 Page_Id 来做区分读取和存放数据。

一些 metadata：

1. isDirty 该page 是否被修改
2. pinCount 该 page 正在被几个线程访问
3. pageId 用来标记唯一 page
4. data 用来存储 page 的真实数据

### 4. Lock VS Latches

 在数据库中 Lock 并不是像 OS 中的Lock 一样，这里的 Lock 更像是在事务中开始定义的标志，在发生错误的时候支持回滚

而 Latches 和 OS 中的 Lock 相似。稍作区分，叫法不同影响不大

### 5. Page Directory 和 Page Table

page directory：pageId -> location in file

page table ： pageId -> frameId

如果我们要找的 page 在 buffer pool 中，我们可以直接根据 page table 找到 frameId 然后根据 frameId 在 Page 数组中找到想要的数据

如果不在 buffer pool 中，我们需要先在通过 page directory 读取那一个 page 的数据，然后存入 buffer pool 中再进行读取数据。

### 6. 一些优化

#### Clock 策略

这个是要在 project 中实现的机制，是一种优化的 LRU ，构建一个环形结构用来存放 page 的信息，一个指针在上面移动，当没有进程引用该page时，当ref=1，则ref=0，若 ref=0，则移出 buffer pool

![1589887963039](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202005/19/193243-704085.png)

#### LRU-K

LRU 和 CLOCK 可能会引发[顺序泛洪](https://stackoverflow.com/questions/20464198/what-is-sequential-flooding)，所以更好的办法是采用 MRC，这里不再赘述。

## Project1

### 1. 实现 CLOCK 策略

其实实现 CLOCK 策略很简单，最大的问题是不知道各个方面的意思，这也是开始最难的，我这里解释一个各个metadata的意思。

```C++
clockHand = -1;
    pageSize = num_pages;
    //inClock[i] = true 代表没有人占用这个frame，所以把这个 frame 放在 Clock 中
    //false 就代表有进程正在使用这个 frame，所以在找 victim 时直接跳过它
    inClock = std::vector<bool>(num_pages,0);
    //就简单的记录，能访问到时，true->false  or  false->return value
    refBit = std::vector<bool>(num_pages,0);
```

注意 inClock 的意思并不是 page 在不在 buffer pool 中，而是buffer pool 的那一个 frame 的状态。index 也是代表的 frameId。

inClock 完全不知道在这个 frame 上的是哪一个page，它只需要知到当前的 frame 上的 page 是否被 pin，这也是inClock中记录的信息，当这块 frame 被使用时，它的 inClock[i] = false，当没人使用 这块 frame 时，就是 inClock = true。

熟悉熟悉C++，实现起来还是很快的。

### 2. Buffer Pool

这里主要是实现 Buffer Pool Manager，主要是实现它的各种行为，包括调用 Clock 的各种方法。

```C++
NewPageImpl(page_id_t *page_id){//}
//调用DiskManager::AllocatePage来获取磁盘上的一个页，并且通过 Clock 机制选择放入 Buffer Pool 中的位置，做好判重。
```

```C++
FetchPageImpl(page_id_t page_id){//}
//用来获取这个页，如果刚好在buffer pool 中就直接返回
//如果不在 buffer pool 中就在磁盘中读取该页然后放入 Buffer Pool中，通过 Clock 机制选择的frame位置
```

```C++
UnpinPageImpl(page_id_t page_id){//}
//某一个线程访问结束，pinCount-1，在这里有机会调用Clock的unpin（），丢弃时如果page被修改了就Flush
```

```C++
FlushPageImpl(page_id_t page_id)
//强制写入磁盘中
```

具体实现这里不多说了，现在课程已经结束可以把代码放 GitHub上了。想看的可以在页脚找到我的链接。