# Linux

## 用户空间和内核空间

* 进程要么在用户空间执行要么在内核空间执行，取决于特权、地址空间和地址。
* 在用户空间执行的话，会有很多操作没权限，内核空间就可以 do anything
* 进程通过系统调用切换用户空间和内核空间

## 内存分配

* malloc 是一个 c 的库方法，不是系统调用

* Linux 操作系统分配空间通过 `brk()`,或者 映射 /dev/zero 到匿名空间，两种方法的结果相同
* 内核为应用分配虚拟空间，只用访问物理空间时才分配物理空间。

## 进程和线程

* Linux 中进程和线程基本一样，只是线程共享同一份虚拟内存地址空间（我的理解：指向同一片内存空间）
* 底层接口创建线程使用的 `clone()`，高级接口为`pthread_create()`。
* LInux 中切换进程上下文很快、线程更快

进程的创建

1. sys_fork()

```c
asmlinkage long sys_fork(struct pt_regs regs){
    return do_fork(SIGCHLD, regs.rsp, &regs, 0);
}
```

SIGCHLD 表示在子进程终止后将发送信号 SIGCHLD 信号通知父进程

2. sys_vfork()

```c

asmlinkage long sys_vfork(struct pt_regs regs)
{
    return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs.rsp, &regs, 0);
}
```

3. sys_clone()

```
casmlinkage int sys_clone(struct pt_regs regs){
    unsigned long clone_flags;
    unsigned long newsp;

    clone_flags = regs.ebx;
    newsp = regs.ecx;
    if (!newsp) 
        newsp = regs.esp;
    return do_fork(clone_flags, newsp, &regs, 0);
}
```



用户态中，多线程会作为一个进程看待，在内核中，会被抽象为“线程组”。线程组中的task_struct有不同的pid字段，但是会有相同的tgid(threadgroup id)字段，这个tgid作为用户态进程的pid给上层使用。所以，在一个进程下的每个线程中获取到的pid是相同的，用户态的pid与内核态的pid不是同一个东西，想获取线程在内核态的pid可以通过系统调用（gettid）来获取。线程组中的task_struct会通过一个链表（thread_group）链接起来，后面创建的线程的task_struct中的group_leader字段会指向leader。通过共享mm_struct、fs、signal相关的数据，再通过thread_group相关的设置，线程的概念基本就实现了
