---
title: Android源码分析 - Binder驱动（中）
date: 2022-02-28 16:33:00
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以aosp分支`android-11.0.0_r25`，kernel分支`android-msm-wahoo-4.4-android11`作为基础解析**

上一篇文章[Android源码分析 - Binder驱动（上）](https://juejin.cn/post/7059601252367204365 "https://juejin.cn/post/7059601252367204365")，我们已经了解了`binder`驱动设备是如何注册的，并且分析了`binder_open`和`binder_mmap`操作函数，接下来我们继续分析`binder`驱动中最重要的部分`binder_ioctl`

# ioctl

我们先简单介绍一下`ioctl`函数，这个函数是用来控制设备的，函数原型如下：

```c
int ioctl(int fd , unsigned long cmd , .../* args */); 
```

第一个参数`fd`为设备的文件描述符

第二个参数`cmd`为命令码，它由驱动方自定义，用户通过命令码告诉设备驱动想要它做什么

后面为可选参数，具体内容和`cmd`有关，是传入驱动层的参数

## 命令码

`Linux`内核是这么定义一个命令码的

|设备类型 | 序列号 | 方向 |数据尺寸|
|----------|--------|------|--------|
| 8 bit | 8 bit |2 bit |8~14 bit|

这样，一个命令就变成了一个整数形式的命令码了，为了使用起来方便，`Linux`定义了一些生成命令码的宏：

```c
_IO(type,nr)        //没有参数的命令
_IOR(type,nr,size)  //从驱动中读数据
_IOW(type,nr,size)  //写数据到驱动中
_IOWR(type,nr,size) //双向读写
```

# binder驱动命令码

了解了`ioctl`和它的命令码后，我们来看看`binder`驱动定义了哪些命令码，以及它们分别有什么作用

`binder`驱动命令码被定义在`include/uapi/linux/android/binder.h`中，其中有几个貌似未使用，我就不列出来了

```c
#define BINDER_WRITE_READ		_IOWR('b', 1, struct binder_write_read)
#define BINDER_SET_MAX_THREADS		_IOW('b', 5, __u32)
#define BINDER_SET_CONTEXT_MGR		_IOW('b', 7, __s32)
#define BINDER_THREAD_EXIT		_IOW('b', 8, __s32)
#define BINDER_VERSION			_IOWR('b', 9, struct binder_version)
#define BINDER_GET_NODE_DEBUG_INFO	_IOWR('b', 11, struct binder_node_debug_info)
#define BINDER_GET_NODE_INFO_FOR_REF	_IOWR('b', 12, struct binder_node_info_for_ref)
#define BINDER_SET_CONTEXT_MGR_EXT	_IOW('b', 13, struct flat_binder_object)
```

- `BINDER_WRITE_READ`：读写命令，用于数据传输，`binder IPC`通信中的核心
- `BINDER_SET_MAX_THREADS`：设置最大线程数
- `BINDER_SET_CONTEXT_MGR`：设置成为`binder`上下文管理者
- `BINDER_THREAD_EXIT`：`binder`线程退出命令，释放相关资源
- `BINDER_VERSION`：获取`binder`驱动版本号
- `BINDER_GET_NODE_DEBUG_INFO`：获得`binder`节点的`debug`信息
- `BINDER_GET_NODE_INFO_FOR_REF`：从`binder`引用获得`binder`节点信息
- `BINDER_SET_CONTEXT_MGR_EXT`：和`BINDER_SET_CONTEXT_MGR`作用相同，携带额外参数

了解了这些`binder`驱动命令码，我们就可以开始正式分析`binder_ioctl`

# binder_ioctl

这个函数位于`drivers/android/binder.c`文件中

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    //从命令参数中解析出用户数据大小
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    ...
    //进入休眠状态，等待被唤醒
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;
    //根据请求系统调用的线程的pid，查找对应的binder_thread，没有则新建一个
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    case BINDER_WRITE_READ:
        ...
        break;
    case BINDER_SET_MAX_THREADS: {
        ...
        break;
    }
    case BINDER_SET_CONTEXT_MGR_EXT: {
        ...
        break;
    }
    case BINDER_SET_CONTEXT_MGR:
        ...
        break;
    case BINDER_VERSION: {
        ...
        break;
    }
    case BINDER_GET_NODE_INFO_FOR_REF: {
        ...
        break;
    }
    case BINDER_GET_NODE_DEBUG_INFO: {
        ...
        break;
    }
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    ...
    return ret;
}
```

从整体上来看还是比较清晰的，我们对一些点做一下详解

## __user

`__user`是一个宏，它告诉编译器不应该解除这个指针的引用（因为在当前地址空间中它是没有意义的），`(void __user *)arg`表示`arg`是一个用户空间的地址，不能直接进行拷贝等，要使用`copy_from_user`，`copy_to_user`等函数。

## wait_event_interruptible

`wait_event_interruptible(wq, condition)`是一个宏，它是用来挂起进程直到满足判断条件的

`binder_stop_on_user_error`是一个全局变量，它的初始值为0，`binder_user_error_wait`是一个等待队列

在正常情况下，`binder_stop_on_user_error < 2`这个条件是成立的，所以不会进入挂起状态，而当`binder`因为错误而停止后，调用`binder_ioctl`，则会挂起进程，直到其他进程通过`wake_up_interruptible`来唤醒`binder_user_error_wait`队列，并且满足`binder_stop_on_user_error < 2`这个条件，`binder_ioctl`才会继续往后运行

## binder_thread结构体

我们需要关注一个重要的结构体`binder_thread`，它在后续的代码中会频繁的出现，这个结构体描述了进程中的工作线程

```c
struct binder_thread {
    //binder线程所属的进程
    struct binder_proc *proc;
    //红黑树节点
    struct rb_node rb_node;
    //链表节点
    struct list_head waiting_thread_node;
    //进程pid
    int pid;
    //描述了线程当前的状态
    int looper;              /* only modified by this thread */
    bool looper_need_return; /* can be written by other thread */
    //binder事务栈（链表形式，内部存在前后节点）
    struct binder_transaction *transaction_stack;
    //todo队列，为需要处理的工作的链表
    struct list_head todo;
    //binder_thread_write后是否立即执行完成binder_thread_read
    //false的情况下会在binder_thread_read中休眠，延迟执行BINDER_WORK_TRANSACTION_COMPLETE
    bool process_todo;
    struct binder_error return_error;
    struct binder_error reply_error;
    //等待队列，当处理binder事务需要依赖别的binder事务的时候，则会以此等待队列睡眠
    //直到它所依赖的binder事务完成后唤醒
    wait_queue_head_t wait;
    //统计信息
    struct binder_stats stats;
    //临时引用计数
    atomic_t tmp_ref;
    //是否死亡
    bool is_dead;
    //线程信息结构体
    struct task_struct *task;
};
```

## binder_get_thread

接下来我们看一下`binder_ioctl`是怎么获得`binder_thread`的

```c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread;
    struct binder_thread *new_thread;

    binder_inner_proc_lock(proc);
    thread = binder_get_thread_ilocked(proc, NULL);
    binder_inner_proc_unlock(proc);
    if (!thread) {
        new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (new_thread == NULL)
            return NULL;
        binder_inner_proc_lock(proc);
        thread = binder_get_thread_ilocked(proc, new_thread);
        binder_inner_proc_unlock(proc);
        if (thread != new_thread)
            kfree(new_thread);
    }
    return thread;
}
```

我们可以看到里面有锁操作，使用的就是上一章[Android源码分析 - Binder驱动（上）](https://juejin.cn/post/7062654742329032740#heading-10)中所介绍过的`spinlock`，使用的是`binder_proc`结构体中的`inner_lock`

简单浏览一下代码我们就可以知道，`binder_get_thread`首先试着从`binder_proc`获得`binder_thread`，如果没能获得，就新建一个，这两种情况都调用了`binder_get_thread_ilocked`函数

```c
static struct binder_thread *binder_get_thread_ilocked(
        struct binder_proc *proc, struct binder_thread *new_thread)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;
    struct rb_node **p = &proc->threads.rb_node;

    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);

        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            return thread;
    }
    if (!new_thread)
        return NULL;
    thread = new_thread;
    //binder_thread对象创建计数加1
    binder_stats_created(BINDER_STAT_THREAD);
    thread->proc = proc;
    thread->pid = current->pid;
    //引用计数加1
    get_task_struct(current);
    thread->task = current;
    atomic_set(&thread->tmp_ref, 0);
    init_waitqueue_head(&thread->wait);
    //初始化todo队列
    INIT_LIST_HEAD(&thread->todo);
    //插入红黑树
    rb_link_node(&thread->rb_node, parent, p);
    rb_insert_color(&thread->rb_node, &proc->threads);
    thread->looper_need_return = true;
    thread->return_error.work.type = BINDER_WORK_RETURN_ERROR;
    thread->return_error.cmd = BR_OK;
    thread->reply_error.work.type = BINDER_WORK_RETURN_ERROR;
    thread->reply_error.cmd = BR_OK;
    INIT_LIST_HEAD(&new_thread->waiting_thread_node);
    return thread;
}
```

这个函数分为前后两个部分，前半部分通过`binder_proc->threads`这个红黑树查找当前系统调用进程`pid`所对应的`binder_thread`，后半部分初始化了传入的`new_thread`，并将其插入到红黑树中（`binder_proc->threads`）

--- 

接下来就是判断命令码`cmd`，来执行相应的工作了，我们只分析比较重要的几个命令码

# BINDER_WRITE_READ

`binder`驱动中最重要的命令码肯定非`BINDER_WRITE_READ`莫属了，这个命令用来进行`binder`读写交互

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ...
    switch (cmd) {
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
        if (ret)
            goto err;
        break;
    ...
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    ...
    return ret;
}
```

switch case命令码后，直接调用了`binder_ioctl_write_read`函数

```c
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    //校验用户传入arg数据大小
    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    //将用户空间ubuf拷贝至内核空间bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
    ...
    //当写缓存中有数据，执行binder写操作
    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            //有错误发生，将已读数据大小设为0
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    //当读缓存中有数据，执行binder读操作
    if (bwr.read_size > 0) {
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        //如果todo队列中有未处理的任务，唤醒等待状态下的线程
        binder_inner_proc_lock(proc);
        if (!binder_worklist_empty_ilocked(&proc->todo))
            binder_wakeup_proc_ilocked(proc);
        binder_inner_proc_unlock(proc);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    ...
    //将内核空间修改后的bwr拷贝至用户空间ubuf
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

## binder_write_read结构体

`BINDER_WRITE_READ`命令码所接受的参数为一个`binder_write_read`结构体，我们先来了解一下它

```c
struct binder_write_read {
    binder_size_t        write_size;        /* bytes to write */
    binder_size_t        write_consumed;    /* bytes consumed by driver */
    binder_uintptr_t     write_buffer;
    binder_size_t        read_size;         /* bytes to read */
    binder_size_t        read_consumed;     /* bytes consumed by driver */
    binder_uintptr_t     read_buffer;
};
```

- `write_size`：写数据的总大小
- `write_consumed`：已写数据大小
- `write_buffer`：写数据的虚拟内存地址
- `read_size`：读数据的总大小
- `read_consumed`：已读数据大小
- `read_buffer`：读数据的虚拟内存地址

---

整个`binder_ioctl_write_read`函数结构是比较简单的，首先校验了一下用户空间所传的参数`arg`为`binder_write_read`结构体，接着将其从用户空间拷贝至内核空间`bwr`，接下来便是分别检查写缓存读缓存中是否有数据，有的话则执行相应的写读操作。这里需要注意的是，读写操作所传入的`write_consumed`和`read_consumed`是以地址的形式，即会对这两个值进行修改，不管读写操作是否执行，成功或者失败，最后都会调用`copy_to_user`将`bwr`从内核空间复制到用户空间`ubuf`

看到这里，可能有些同学会觉得有些奇怪，说好`binder`只进行一次复制的呢？其实是这样的没错，这里的`copy_from_user`或者`copy_to_user`只是复制了`binder_write_read`结构体，得到了需要IPC数据的虚拟内存地址而已，真正的复制操作是在`binder`读写操作中进行的

## binder_thread_write

先看`binder`写操作，这个函数首先从传入的参数中，计算出待写的起始地址和结束地址，因为可能数据中含有多个命令和对应数据要处理，所以这里开了一个循环，在循环中，首先调用`get_user`，从用户空间读取一个值到内核空间中来，这个值就是`binder`请求码，然后将指针向后移动32位，使其指向对应请求码的数据头，接着根据`binder`请求码去完成不同的工作，处理完后修改已写数据大小

```c
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    struct binder_context *context = proc->context;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    //可能含有多个命令和对应数据要处理
    while (ptr < end && thread->return_error.cmd == BR_OK) {
        int ret;

        //获得binder请求码
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        //使指针指向数据头
        ptr += sizeof(uint32_t);
        trace_binder_command(cmd);
        //记录binder数据信息
        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
            atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
            atomic_inc(&proc->stats.bc[_IOC_NR(cmd)]);
            atomic_inc(&thread->stats.bc[_IOC_NR(cmd)]);
        }
        //根据binder请求码，执行不同工作
        switch (cmd) {
        case BC_INCREFS:
        case BC_ACQUIRE:
        case BC_RELEASE:
        case BC_DECREFS: {
            ...
            break;
        }
        case BC_INCREFS_DONE:
        case BC_ACQUIRE_DONE: {
            ...
            break;
        }
        case BC_ATTEMPT_ACQUIRE:
            pr_err("BC_ATTEMPT_ACQUIRE not supported\n");
            return -EINVAL;
        case BC_ACQUIRE_RESULT:
            pr_err("BC_ACQUIRE_RESULT not supported\n");
            return -EINVAL;

        case BC_FREE_BUFFER: {
            ...
            break;
        }

        case BC_TRANSACTION_SG:
        case BC_REPLY_SG: {
            ...
            break;
        }
        case BC_TRANSACTION:
        case BC_REPLY: {
            ...
            break;
        }

        case BC_REGISTER_LOOPER:
            ...
            break;
        case BC_ENTER_LOOPER:
            ...
            break;
        case BC_EXIT_LOOPER:
            ...
            break;

        case BC_REQUEST_DEATH_NOTIFICATION:
        case BC_CLEAR_DEATH_NOTIFICATION: {
            ...
        } break;
        case BC_DEAD_BINDER_DONE: {
            ...
        } break;

        default:
            pr_err("%d:%d unknown command %d\n",
                   proc->pid, thread->pid, cmd);
            return -EINVAL;
        }
        //设置已写数据大小
        *consumed = ptr - buffer;
    }
    return 0;
}
```

### binder请求码

`binder`请求码用于用户空间程序向`binder`驱动发送请求消息，以`BC`开头，被定义在`enum` `binder_driver_command_protocol`中（`include/uapi/linux/android/binder.h`）

命令                            | 说明                           | 参数类型                    |
| ----------------------------- | ---------------------------- | -------------------- |
| BC_TRANSACTION                | Binder事务，即：Client对于Server的请求 | binder_transaction_data |
| BC_REPLY                      | 事务的应答，即：Server对于Client的回复    | binder_transaction_data |
| BC_FREE_BUFFER                | 通知驱动释放Buffer                 | binder_uintptr_t |
| BC_ACQUIRE                    | 强引用计数+1                      | __u32             |
| BC_RELEASE                    | 强引用计数-1                      | __u32             |
| BC_INCREFS                    | 弱引用计数+1                      | __u32             |
| BC_DECREFS                    | 弱引用计数-1                      | __u32             |
| BC_ACQUIRE_DONE               | BR_ACQUIRE的回复                | binder_ptr_cookie  |
| BC_INCREFS_DONE               | BR_INCREFS的回复                | binder_ptr_cookie  |
| BC_ENTER_LOOPER               | 通知驱动主线程ready                 | void            |
| BC_REGISTER_LOOPER            | 通知驱动子线程ready                 | void            |
| BC_EXIT_LOOPER                | 通知驱动线程已经退出                   | void          |
| BC_REQUEST_DEATH_NOTIFICATION | 请求接收死亡通知             | binder_handle_cookie    |
| BC_CLEAR_DEATH_NOTIFICATION   | 去除接收死亡通知             | binder_handle_cookie    |
| BC_DEAD_BINDER_DONE           | 已经处理完死亡通知            | binder_uintptr_t       |
| BC_ATTEMPT_ACQUIRE            | 暂不支持                         | -                 |
| BC_ACQUIRE_RESULT             | 暂不支持                         | -                 |

其中，最重要且最频繁的操作为`BC_TRANSACTION`/`BC_REPLY`，我们就只分析一下这两个请求码做了什么事

### binder_transaction

```c
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{   
    ...
    while (...) {
        ...
        switch (cmd) {
        ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr,
                       cmd == BC_REPLY, 0);
            break;
        }
        ...
    }
    ...
}
```

对于这两个请求码，首先从用户空间中复制了一份`binder_transaction_data`到内核空间，接着就调用`binder_transaction`函数继续处理

#### binder_transaction_data结构体

在分析`binder_transaction`函数前，我们需要先了解一些结构体

`binder_transaction_data`结构体就是`BC_TRANSACTION`/`BC_REPLY`所对应的参数类型，它被定义在`include/uapi/linux/android/binder.h`中

```c
struct binder_transaction_data {
    union {
        //当BINDER_WRITE_READ命令的目标对象非本地binder实体时，用handle表示对目标binder的引用
        __u32	handle;
        //当BINDER_WRITE_READ命令的目标对象是本地binder实体时，用此域成员变量表示这个对象在本进程中的内存地址
        binder_uintptr_t ptr;
    } target;
    //目标binder实体所带的附加数据
    binder_uintptr_t	cookie;
    //表示要对目标对象请求的命令代码
    __u32		code;

    //事务标志，详见enum transaction_flags
    __u32	        flags;
    //发起请求的进程pid
    pid_t		sender_pid;
    //发起请求的进程uid
    uid_t		sender_euid;
    //真正要传输的数据的大小
    binder_size_t	data_size;
    //偏移数组大小，这个偏移数组是用来描述数据区中，每一个binder对象的位置的
    binder_size_t	offsets_size;

    union {
        struct {
            //数据区的首地址
            binder_uintptr_t	buffer;
            //偏移数组的首地址，这个偏移数组是用来描述数据区中，每一个binder对象的位置的
            //数组的每一项为一个binder_size_t，这个值对应着每一个binder对象相对buffer首地址的偏移
            binder_uintptr_t	offsets;
        } ptr;
        //数据较小的时候可以直接装在这个数组里
        __u8	buf[8];
    } data;
};
```

可以看到，真正需要拷贝的数据的地址是保存在`data`域中的，可能文字描述的`data`结构不是特别清晰，可以结合下图理解：

![data结构](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Binder%E9%A9%B1%E5%8A%A8%EF%BC%88%E4%B8%AD%EF%BC%89_binder_transaction_data.png)

这里我用一个例子来解释一下`binder_transaction_data`传输的数据是什么样子的

小伙伴们应该都了解`Parcel`吧，它是一个存放读取数据的容器，我们`binder_transaction_data`中实际传输的数据就是通过它组合而成的，它可以传输基本数据类型，`Parcelable`类型和`binder`类型

其中基本数据类型就不用说了，每种基本类型所占用的大小是固定的，`Parcelable`类型实际上也是传输基本数据类型，它是通过实现`Parcelable`接口将一个复杂对象中的成员序列化成了一个个基本数据类型传输，而`binder`类型的传输有点特别，它会将这个`binder`对象 "压扁" 成一个`flat_binder_object`结构体传输

假设我们有一个客户端`client`，一个服务端`server`，`client`想要向`binder`驱动发起一个事物，调用`server`的某个方法，我们该怎么构建`binder_transaction_data`的数据区呢？

一般来说，我们需要先写一个`token`，这个`token`是为了进行校验的，两端需要保持一致。接着我们需要按顺序依次写入参数，假设我们想要调用`server`的`callMe(int, Parcelable, IBinder)`函数，那我们就需要先写入一个`int`，再写入一个`Parcelable`，最后再将`IBinder` "压扁" 成一个`flat_binder_object`写入。

此时数据布局如下图所示：

![data结构](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Binder%E9%A9%B1%E5%8A%A8%EF%BC%88%E4%B8%AD%EF%BC%89_binder_transaction_data2.png)

从图中我们可以看出来，`offsets`指示出了`buffer`中传输的`binder`对象的位置，有几个`binder`对象，就会有几个`offset`与之对应

##### transaction_flags

我们再看一下有哪些事务标志，他们分别代表什么意思

```c
enum transaction_flags {
    //单向调用，异步操作，无返回
    TF_ONE_WAY	        = 0x01,
    //reply内容是一个组件的根对象，对应类型为本地binder
    TF_ROOT_OBJECT	= 0x04,
    //reply内容是一个32位的状态码，对应类型为远程binder引用的句柄
    TF_STATUS_CODE	= 0x08,
    //可以接收一个文件描述符，对应的类型为文件（BINDER_TYPE_FD），即handle中存储的为文件描述符
    TF_ACCEPT_FDS	= 0x10,
};
```

#### binder_transaction结构体

`binder_transaction`结构体用来描述进程间通信过程（事务），它被定义在`drivers/android/binder.c`中

```c
struct binder_transaction {
    int debug_id;
    //用来描述需要处理的工作事项
    struct binder_work work;
    //发起事务的线程
    struct binder_thread *from;
    //事务所依赖的另一个事务
    struct binder_transaction *from_parent;
    //处理该事务的进程
    struct binder_proc *to_proc;
    //处理该事务的线程
    struct binder_thread *to_thread;
    //目标线程下一个需要处理的事务
    struct binder_transaction *to_parent;
    //1: 表示同步事务，需要等待对方回复
    //0: 表示异步事务
    unsigned need_reply:1;
    
    //指向为该事务分配的内核缓冲区
    struct binder_buffer *buffer;
    unsigned int	code;
    unsigned int	flags;
    //发起事务线程的优先级
    struct binder_priority	priority;
    //线程在处理事务时，驱动会修改它的优先级以满足源线程和目标Service组建的要求
    //在修改之前，会将它原来的线程优先级保存在该成员中，以便线程处理完该事务后可以恢复原来的优先级
    struct binder_priority	saved_priority;
    bool    set_priority_called;
    kuid_t	sender_euid;
    binder_uintptr_t security_ctx;

    spinlock_t lock;
};
```

#### binder_work结构体

`binder_work`结构体用来描述需要处理的工作事项，它被定义在`drivers/android/binder.c`中

```c
struct binder_work {
    //双向链表中的一个节点，这个链表储存了所有的binder_work
    struct list_head entry;

    //工作项类型
    enum binder_work_type {
        BINDER_WORK_TRANSACTION = 1,
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_RETURN_ERROR,
        BINDER_WORK_NODE,
        BINDER_WORK_DEAD_BINDER,
        BINDER_WORK_DEAD_BINDER_AND_CLEAR,
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;
};
```

---

简单看完了一些必要的结构体后，我们把目光转回`binder_transaction`函数上

`binder_transaction`函数的代码很长，我们精简一下，然后再分段来看，从整体上，我们可以将它分为几个部分：

1. 获得目标进程/线程信息
2. 将数据拷贝到目标进程所映射的内存中（此时会建立实际的映射关系）
3. 将待处理的任务加入`todo`队列，唤醒目标线程

#### 第一部分：获得目标进程/线程信息

这里根据是否为`reply`，分成了两种情况

##### BC_TRANSACTION

我们先看`BC_TRANSACTION`的情况

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    struct binder_proc *target_proc = NULL;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    uint32_t return_error = 0;
    struct binder_context *context = proc->context;

    if (reply) {
        ...
    } else {
        if (tr->target.handle) {
            struct binder_ref *ref;

            binder_proc_lock(proc);
            //查找binder引用
            ref = binder_get_ref_olocked(proc, tr->target.handle, true);
            //通过目标binder实体获取目标进程信息
            target_node = binder_get_node_refs_for_txn(
                    ref->node, &target_proc,
                    &return_error);
            binder_proc_unlock(proc);
        } else {    //handle为0代表目标target是ServiceManager
            mutex_lock(&context->context_mgr_node_lock);
            //ServiceManager为binder驱动的context，所以可以直接从context中获取binder实体
            target_node = context->binder_context_mgr_node;
            if (target_node)
                //通过目标binder实体获取目标进程信息
                target_node = binder_get_node_refs_for_txn(
                        target_node, &target_proc,
                        &return_error);
            else
                return_error = BR_DEAD_REPLY;
            mutex_unlock(&context->context_mgr_node_lock);
            if (target_node && target_proc == proc) {
                ... //error
            }
        }
        ...
        //使用LSM进行安全检查
        if (security_binder_transaction(proc->tsk,
                        target_proc->tsk) < 0) {
            ... //error
        }
        binder_inner_proc_lock(proc);
        //flags不带TF_ONE_WAY（即需要reply）并且当前线程存在binder事务栈
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
            struct binder_transaction *tmp;

            tmp = thread->transaction_stack;
            if (tmp->to_thread != thread) {
                ... //error
            }
            //寻找一个合适的目标binder线程
            while (tmp) {
                struct binder_thread *from;

                spin_lock(&tmp->lock);
                from = tmp->from;
                if (from && from->proc == target_proc) {
                    atomic_inc(&from->tmp_ref);
                    target_thread = from;
                    spin_unlock(&tmp->lock);
                    break;
                }
                spin_unlock(&tmp->lock);
                tmp = tmp->from_parent;
            }
        }
        binder_inner_proc_unlock(proc);
    }
    ...
}
```

可以看到，虽然整个函数很长很复杂，但经过我们的拆分精简，逻辑就清晰很多了

`binder_transaction_data.target.handle`用一个`int`值表示目标`binder`引用，当它不为0时，调用`binder_get_ref_olocked`函数查找`binder_ref`

###### binder_get_ref_olocked

```c
static struct binder_ref *binder_get_ref_olocked(struct binder_proc *proc,
                         u32 desc, bool need_strong_ref)
{
    struct rb_node *n = proc->refs_by_desc.rb_node;
    struct binder_ref *ref;

    while (n) {
        ref = rb_entry(n, struct binder_ref, rb_node_desc);

        if (desc < ref->data.desc) {
            n = n->rb_left;
        } else if (desc > ref->data.desc) {
            n = n->rb_right;
        } else if (need_strong_ref && !ref->data.strong) {
            binder_user_error("tried to use weak ref as strong ref\n");
            return NULL;
        } else {
            return ref;
        }
    }
    return NULL;
}
```

可以看到，这个函数就是从`binder_proc.refs_by_desc`这个红黑树中，通过`desc`句柄查找到对应的`binder`引用`binder_ref`，这样就可以通过`binder_ref.node`获得到`binder`实体`binder_node`

接着再调用`binder_get_node_refs_for_txn`函数通过目标`binder`实体获取目标进程信息

###### binder_get_node_refs_for_txn

```c
static struct binder_node *binder_get_node_refs_for_txn(
        struct binder_node *node,
        struct binder_proc **procp,
        uint32_t *error)
{
    struct binder_node *target_node = NULL;

    binder_node_inner_lock(node);
    if (node->proc) {
        target_node = node;
        //binder_node强引用计数加1
        binder_inc_node_nilocked(node, 1, 0, NULL);
        //binder_node临时引用计数加1
        binder_inc_node_tmpref_ilocked(node);
        //binder_proc临时引用计数加1
        atomic_inc(&node->proc->tmp_ref);
        //使外部传入的proc指针指向binder_proc地址
        *procp = node->proc;
    } else
        *error = BR_DEAD_REPLY;
    binder_node_inner_unlock(node);

    return target_node;
}
```

这个函数第二个参数接受一个`binder_proc **`类型，即指向指针的指针，调用方对`proc`取地址，即指向`proc`指针分配在栈上的地址，这样函数中对`procp`解引用就得到了`proc`指针本身的地址，即可使`proc`指针指向`binder_proc`的地址

当`binder_transaction_data.target.handle`为0时，表示目标是`ServiceManager`，而`ServiceManager`是`binder`驱动的`context`，所以可以直接从`context`中获取`binder`实体，关于`ServiceManager`是怎么成为`binder`驱动的`context`的，我们会在后面的章节进行分析

接下来做一下安全检查，当`flags`不带`TF_ONE_WAY`（即需要`reply`）并且当前线程存在`binder`事务栈时，寻找一个合适的目标`binder`工作线程用来处理此事务（线程复用）

这里`client`端可能是第一次请求服务，此时`binder_thread`里是不存在`binder`事务栈，所以是没法找到目标`binder`线程的

##### BC_REPLY

接着，我们再看`BC_REPLY`的情况

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    struct binder_proc *target_proc = NULL;
    struct binder_thread *target_thread = NULL;
    struct binder_transaction *in_reply_to = NULL;

    if (reply) {
        binder_inner_proc_lock(proc);
        //这个事务是发起事务，也就是说我们需要对这个事务做应答
        in_reply_to = thread->transaction_stack;
        if (in_reply_to == NULL) {
            ... //error
        }
        if (in_reply_to->to_thread != thread) {
            ... //error
        }
        //改指向下一个需要处理的事务，即将这个事务移出链表
        thread->transaction_stack = in_reply_to->to_parent;
        binder_inner_proc_unlock(proc);
        //目标线程即为需要回应的事务的发起线程
        target_thread = binder_get_txn_from_and_acq_inner(in_reply_to);
        if (target_thread->transaction_stack != in_reply_to) {
            ... //error
        }
        //通过binder_thread获得binder_proc
        target_proc = target_thread->proc;
        atomic_inc(&target_proc->tmp_ref);
        binder_inner_proc_unlock(target_thread->proc);
    } else {
        ...
    }
    ...
}
```

```c
static struct binder_thread *binder_get_txn_from_and_acq_inner(
        struct binder_transaction *t)
{
    struct binder_thread *from;

    //相当于 from = t->from; 内部加了锁和引用计数操作
    from = binder_get_txn_from(t);
    if (!from)
        return NULL;
    binder_inner_proc_lock(from->proc);
    if (t->from) {
        BUG_ON(from != t->from);
        return from;
    }
    binder_inner_proc_unlock(from->proc);
    binder_thread_dec_tmpref(from);
    return NULL;
}
```

`BC_REPLY`获取目标进程/线程信息就更简单了，`BC_TRANSACTION`中我们还需要根据`binder`句柄来获取各种信息，`BC_REPLY`我们只需要找到需要回应的那个事务，那个事务所在的线程和进程即为`reply`事务的目标线程和目标进程

#### 第二部分：数据拷贝，建立映射

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    int ret;
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    binder_size_t *offp, *off_end, *off_start;
    binder_size_t off_min;
    u8 *sg_bufp, *sg_buf_end;
    struct binder_proc *target_proc = NULL;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL
    u32 secctx_sz = 0;

    ...

    //为目标进程binder事务分配空间（后续会加到目标进程/线程的todo队列中，由目标进程/线程处理这个事务）
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    spin_lock_init(&t->lock);

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    t->debug_id = t_debug_id;
    //设置事务发起线程
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    t->sender_euid = task_euid(proc->tsk);
    //设置事务处理进程
    t->to_proc = target_proc;
    //设置事务处理线程
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    //设置优先级
    if (!(t->flags & TF_ONE_WAY) &&
        binder_supported_policy(current->policy)) {
        /* Inherit supported policies for synchronous transactions */
        t->priority.sched_policy = current->policy;
        t->priority.prio = current->normal_prio;
    } else {
        /* Otherwise, fall back to the default priority */
        t->priority = target_proc->default_priority;
    }

    //安全相关
    if (target_node && target_node->txn_security_ctx) {
        ...
    }
    
    //分配缓存，建立映射
    t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
        tr->offsets_size, extra_buffers_size,
        !reply && (t->flags & TF_ONE_WAY));
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    
    off_start = (binder_size_t *)(t->buffer->data +
                      ALIGN(tr->data_size, sizeof(void *)));
    offp = off_start;

    //这里就是真正的一次复制
    copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
               tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp, (const void __user *)(uintptr_t)
               tr->data.ptr.offsets, tr->offsets_size);

    //检查数据对齐
    if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
        ... //error
    }
    if (!IS_ALIGNED(extra_buffers_size, sizeof(u64))) {
        ... //error
    }
    off_end = (void *)off_start + tr->offsets_size;
    sg_bufp = (u8 *)(PTR_ALIGN(off_end, sizeof(void *)));
    sg_buf_end = sg_bufp + extra_buffers_size -
        ALIGN(secctx_sz, sizeof(u64));
    off_min = 0;
    //循环遍历每一个binder对象
    for (; offp < off_end; offp++) {
        struct binder_object_header *hdr;
        size_t object_size = binder_validate_object(t->buffer, *offp);

        if (object_size == 0 || *offp < off_min) {
            ... //error
        }

        hdr = (struct binder_object_header *)(t->buffer->data + *offp);
        off_min = *offp + object_size;
        switch (hdr->type) {
        //需要对binder类型进行转换
        //因为在A进程中为本地binder对象，对于B进程则为远程binder对象，反之亦然
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct flat_binder_object *fp;

            fp = to_flat_binder_object(hdr);
            ret = binder_translate_binder(fp, t, thread);
        } break;
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            struct flat_binder_object *fp;

            fp = to_flat_binder_object(hdr);
            ret = binder_translate_handle(fp, t, thread);
        } break;

        case BINDER_TYPE_FD: {
            ...
        } break;
        case BINDER_TYPE_FDA: {
            ...
        } break;
        case BINDER_TYPE_PTR: {
            ...
        } break;
        default:
            ... //error
        }
    }
    //设置工作类型
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    //设置目标进程的事务类型
    t->work.type = BINDER_WORK_TRANSACTION;
    ...
}
```

我们可以将这一部分再细分成几个部分：

1. 分配缓存，建立映射
2. 数据拷贝
3. `binder`类型转换

##### 分配缓存，建立映射

我们首先看分配缓存，建立映射是怎么做的，它调用了`binder_alloc_new_buf`函数，这个函数定义在`drivers/android/binder_alloc.c`中，内部加了锁后调用了`binder_alloc_new_buf_locked`函数

```c
static struct binder_buffer *binder_alloc_new_buf_locked(
                struct binder_alloc *alloc,
                size_t data_size,
                size_t offsets_size,
                size_t extra_buffers_size,
                int is_async)
{
    struct rb_node *n = alloc->free_buffers.rb_node;
    struct binder_buffer *buffer;
    size_t buffer_size;
    struct rb_node *best_fit = NULL;
    void *has_page_addr;
    void *end_page_addr;
    size_t size, data_offsets_size;
    int ret;

    if (alloc->vma == NULL) {
        ... //error
    }

    //计算需要的缓冲区大小
    //这里需要将size对齐void *（32位下占用4字节，64位下占用8字节）
    data_offsets_size = ALIGN(data_size, sizeof(void *)) +
        ALIGN(offsets_size, sizeof(void *));
    size = data_offsets_size + ALIGN(extra_buffers_size, sizeof(void *));
    size = max(size, sizeof(void *));

    //从binder_alloc的空闲缓冲红黑树中找到一个大小最合适的binder_buffer
    while (n) {
        //当找到一个需求大小和缓存区大小刚好相同的空闲缓存区时
        //此时buffer就正好指向这个空闲缓存区
        buffer = rb_entry(n, struct binder_buffer, rb_node);
        BUG_ON(!buffer->free);
        buffer_size = binder_alloc_buffer_size(alloc, buffer);

        //当只找到一个比需求大小稍大一点的空闲缓存区时
        //此时buffer指向的是这个空闲缓存区所在节点的父节点
        //然后n指向NULL
        if (size < buffer_size) {
            best_fit = n;
            n = n->rb_left;
        } else if (size > buffer_size)
            n = n->rb_right;
        else {
            best_fit = n;
            break;
        }
    }
    if (best_fit == NULL) {
        ... //error
    }
    //此时buffer指向的是所需求的空闲缓存区所在红黑树节点的父节点
    //需要让其指向真正需求的那个空闲缓存区
    if (n == NULL) {
        buffer = rb_entry(best_fit, struct binder_buffer, rb_node);
        buffer_size = binder_alloc_buffer_size(alloc, buffer);
    }

    //计算出buffer的终点，向下对齐（不能超过可用的buffer_size）
    has_page_addr =
        (void *)(((uintptr_t)buffer->data + buffer_size) & PAGE_MASK);
    WARN_ON(n && buffer_size != size);
    //计算出实际上我们接收数据需要的空间的终点，向上映射
    end_page_addr =
        (void *)PAGE_ALIGN((uintptr_t)buffer->data + size);
    //如果超出了可用的buffer_size，恢复到正常可用的结束地址
    if (end_page_addr > has_page_addr)
        end_page_addr = has_page_addr;
    //分配物理页，建立映射
    ret = binder_update_page_range(alloc, 1,
        (void *)PAGE_ALIGN((uintptr_t)buffer->data), end_page_addr);
    if (ret)
        return ERR_PTR(ret);

    //有空余空间的话，分隔这个buffer，剩余的buffer加入到空闲缓存区红黑树中（合理利用空间）
    if (buffer_size != size) {
        struct binder_buffer *new_buffer;

        new_buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
        new_buffer->data = (u8 *)buffer->data + size;
        list_add(&new_buffer->entry, &buffer->entry);
        new_buffer->free = 1;
        binder_insert_free_buffer(alloc, new_buffer);
    }

    //我们已经使用了这个buffer，要将其从空闲缓存区红黑树中移除
    rb_erase(best_fit, &alloc->free_buffers);
    //标记为非空闲
    buffer->free = 0;
    buffer->allow_user_free = 0;
    //插入到已分配缓存区红黑树中
    binder_insert_allocated_buffer_locked(alloc, buffer);
    buffer->data_size = data_size;
    buffer->offsets_size = offsets_size;
    buffer->async_transaction = is_async;
    buffer->extra_buffers_size = extra_buffers_size;
    //如果是异步事件, 那么更新binder_alloc的异步事件空闲buffer
    if (is_async) {
        alloc->free_async_space -= size + sizeof(struct binder_buffer);
    }
    return buffer;
    ...
}
```

这个函数的整体逻辑分为三个部分：

1. 找到可用的空闲内核缓存区，计算我们需要分配的大小
2. 分配物理页，建立映射
3. 初始化新分配的`buffer`

其中1、3部分已经用注释标出来了，应该还是比较好理解的，我们终点看一下第2部分：怎么分配物理页，建立映射

我们在上一章[Android源码分析 - Binder驱动（上）](https://juejin.cn/post/7062654742329032740#heading-16)中说到，`binder_mmap`并没有立即将内核虚拟内存和进程虚拟内存与物理内存做映射，实际上这个映射操作是在`binder_update_page_range`这里做的

```c
static int binder_update_page_range(struct binder_alloc *alloc, int allocate,
                    void *start, void *end)
{
    void *page_addr;
    unsigned long user_page_addr;
    struct binder_lru_page *page;
    struct vm_area_struct *vma = NULL;
    struct mm_struct *mm = NULL;
    bool need_mm = false;

    if (end <= start)
        return 0;

    if (allocate == 0)
        goto free_range;

    //检查是否有页框需要分配
    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        page = &alloc->pages[(page_addr - alloc->buffer) / PAGE_SIZE];
        if (!page->page_ptr) {
            need_mm = true;
            break;
        }
    }

    //指向目标用户进程的内存空间描述体
    if (need_mm && atomic_inc_not_zero(&alloc->vma_vm_mm->mm_users))
        mm = alloc->vma_vm_mm;

    if (mm) {
        //获取mm_struct的读信号量
        down_read(&mm->mmap_sem);
        //检查mm是否有效
        if (!mmget_still_valid(mm)) {
            //释放
            if (allocate == 0)
                goto free_range;
            //错误
            goto err_no_vma;
        }
        vma = alloc->vma;
    }

    if (!vma && need_mm) {
        ... //error
    }

    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        int ret;
        bool on_lru;
        size_t index;

        //指向对应页框地址，为后面赋值做准备
        index = (page_addr - alloc->buffer) / PAGE_SIZE;
        page = &alloc->pages[index];

        //page->page_ptr不为NULL说明之前已经分配并映射过了
        if (page->page_ptr) {
            on_lru = list_lru_del(&binder_alloc_lru, &page->lru);
            continue;
        }

        //分配一个页的物理内存
        page->page_ptr = alloc_page(GFP_KERNEL |
                        __GFP_HIGHMEM |
                        __GFP_ZERO);
        //未分配成功
        if (!page->page_ptr) {
            ... //error
        }
        page->alloc = alloc;
        INIT_LIST_HEAD(&page->lru);

        //将物理内存空间映射到内核虚拟内存空间
        ret = map_kernel_range_noflush((unsigned long)page_addr,
                           PAGE_SIZE, PAGE_KERNEL,
                           &page->page_ptr);
        flush_cache_vmap((unsigned long)page_addr,
                (unsigned long)page_addr + PAGE_SIZE);

        //根据之前计算的user_buffer_offset可以直接得到目标用户空间进程虚拟内存地址
        user_page_addr =
            (uintptr_t)page_addr + alloc->user_buffer_offset;
        //将物理内存空间映射到目标用户进程虚拟内存空间
        ret = vm_insert_page(vma, user_page_addr, page[0].page_ptr);

        if (index + 1 > alloc->pages_high)
            alloc->pages_high = index + 1;
    }
    if (mm) {
        //释放mm_struct的读信号量
        up_read(&mm->mmap_sem);
        mmput(mm);
    }
    return 0;
    ... //错误处理
}
```

代码中的注释写的应该比较清楚了，总之就是先分配`物理内存`，再将这块`物理内存`分别映射到`内核虚拟空间`和`用户进程虚拟空间`，这样`内核虚拟空间`与`用户进程虚拟空间`相当于也间接的建立了映射关系

关于物理内存的分配以及映射，就是`Linux`内核层的事情了，感兴趣的同学可以再深入往里看看，这里就不再多说了

##### 数据拷贝

关于数据拷贝这部分就不用多说了，物理内存已经分配好了，映射也建立了，接下来直接调用`copy_from_user`将数据从用户空间拷贝至映射的那块内存就可以了

##### binder类型转换

最后循环遍历每一个`binder`对象，对其中每一个`binder`对象类型做转换，因为在一个进程中为本地`binder`对象，对于另一个进程则为远程`binder`对象，反之亦然

###### flat_binder_object结构体

这里就是我们之前提到的，`binder`对象在传输过程中会被 "压扁" 的结构

```c
struct flat_binder_object {
    //描述了binder对象的类型
    struct binder_object_header	hdr;
    //和binder_transaction_data中flags含义相同
    __u32				flags;

    /* 8 bytes of data. */
    union {
        //当hdr.type == BINDER_TYPE_BINDER时，表示是一个binder实体对象，指向binder实体在用户空间的地址
        binder_uintptr_t	binder;	/* local object */
        //当hdr.type == BINDER_TYPE_HANDLE，表示是一个binder引用句柄
        __u32			handle;	/* remote object */
    };

    ////当hdr.type == BINDER_TYPE_BINDER时才有值，表示携带的额外数据
    /* extra data associated with local object */
    binder_uintptr_t	cookie;
};
```

###### binder_translate_binder

`BINDER_TYPE_BINDER`表示是一个`binder`实体对象，需要将它转换成`binder`引用句柄

```c
static int binder_translate_binder(struct flat_binder_object *fp,
                   struct binder_transaction *t,
                   struct binder_thread *thread)
{
    struct binder_node *node;
    struct binder_proc *proc = thread->proc;
    struct binder_proc *target_proc = t->to_proc;
    struct binder_ref_data rdata;
    int ret = 0;

    //通过proc->nodes.rb_node红黑树查找binder_node
    node = binder_get_node(proc, fp->binder);
    //如果没有找到，新建一个binder_node并将其插入红黑树
    if (!node) {
        node = binder_new_node(proc, fp);
        if (!node)
            return -ENOMEM;
    }
    
    if (fp->cookie != node->cookie) {
        ... //error
    }
    
    //安全检查
    if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
        ret = -EPERM;
        goto done;
    }

    //查找binder_ref并将其引用计数加1，如果没有查找到则创建一个，并将其插入红黑树
    ret = binder_inc_ref_for_node(target_proc, node,
            fp->hdr.type == BINDER_TYPE_BINDER,
            &thread->todo, &rdata);
    if (ret)
        goto done;
    
    //转换binder类型
    if (fp->hdr.type == BINDER_TYPE_BINDER)
        fp->hdr.type = BINDER_TYPE_HANDLE;
    else
        fp->hdr.type = BINDER_TYPE_WEAK_HANDLE;
    fp->binder = 0;
    //binder引用句柄赋值
    fp->handle = rdata.desc;
    fp->cookie = 0;

done:
    //binder_node临时引用计数减1
    binder_put_node(node);
    return ret;
}
```

###### binder_translate_handle

`BINDER_TYPE_HANDLE`表示是一个`binder`引用句柄，，需要将它转换成`binder`实体对象

```c
static int binder_translate_handle(struct flat_binder_object *fp,
                   struct binder_transaction *t,
                   struct binder_thread *thread)
{
    struct binder_proc *proc = thread->proc;
    struct binder_proc *target_proc = t->to_proc;
    struct binder_node *node;
    struct binder_ref_data src_rdata;
    int ret = 0;

    //从proc->refs_by_desc.rb_node红黑树中查找binder_node，并将其临时引用计数加1
    node = binder_get_node_from_ref(proc, fp->handle,
            fp->hdr.type == BINDER_TYPE_HANDLE, &src_rdata);
    
    //安全检查
    if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
        ret = -EPERM;
        goto done;
    }

    binder_node_lock(node);
    //如果binder实体所在的进程为事务处理进程
    if (node->proc == target_proc) {
        //binder类型转换
        if (fp->hdr.type == BINDER_TYPE_HANDLE)
            fp->hdr.type = BINDER_TYPE_BINDER;
        else
            fp->hdr.type = BINDER_TYPE_WEAK_BINDER;
        fp->binder = node->ptr;
        fp->cookie = node->cookie;
        if (node->proc)
            binder_inner_proc_lock(node->proc);
        //binder强引用计数加1
        binder_inc_node_nilocked(node,
                     fp->hdr.type == BINDER_TYPE_BINDER,
                     0, NULL);
        if (node->proc)
            binder_inner_proc_unlock(node->proc);
        binder_node_unlock(node);
    } else {
        //重新查找binder_ref
        struct binder_ref_data dest_rdata;

        binder_node_unlock(node);
        ret = binder_inc_ref_for_node(target_proc, node,
                fp->hdr.type == BINDER_TYPE_HANDLE,
                NULL, &dest_rdata);
        if (ret)
            goto done;

        fp->binder = 0;
        fp->handle = dest_rdata.desc;
        fp->cookie = 0;
    }
done:
    //binder_node临时引用计数减1
    binder_put_node(node);
    return ret;
}
```

#### 第三部分：加入todo队列，唤醒目标线程

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    struct binder_thread *target_thread = NULL;
    struct binder_transaction *in_reply_to = NULL;

    ...

    if (reply) {    //如果请求码为BC_REPLY
        //将tcomplete插入到事务发起binder线程的todo队列中
        binder_enqueue_thread_work(thread, tcomplete);
        binder_inner_proc_lock(target_proc);
        if (target_thread->is_dead) {
            binder_inner_proc_unlock(target_proc);
            goto err_dead_proc_or_thread;
        }
        //将发起事务从目标binder线程的事务链表中移除
        binder_pop_transaction_ilocked(target_thread, in_reply_to);
        //将t->work插入到目标binder线程的todo队列中
        binder_enqueue_thread_work_ilocked(target_thread, &t->work);
        binder_inner_proc_unlock(target_proc);
        //唤醒目标binder线程的等待队列
        wake_up_interruptible_sync(&target_thread->wait);
        //恢复发起事务的优先级
        binder_restore_priority(current, in_reply_to->saved_priority);
        //释放发起事务
        binder_free_transaction(in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {    //如果请求码为BC_TRANSACTION并且不为异步操作，需要返回
        binder_inner_proc_lock(proc);
        //将tcomplete插入到事务发起binder线程的todo队列中（这里会延迟执行BINDER_WORK_TRANSACTION_COMPLETE）
        binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
        //设置为需要回应
        t->need_reply = 1;
        //插入事务链表中
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
        binder_inner_proc_unlock(proc);
        //将t->work插入目标线程的todo队列中并唤醒目标进程
        if (!binder_proc_transaction(t, target_proc, target_thread)) {
            binder_inner_proc_lock(proc);
            //出错后，移除该事务
            binder_pop_transaction_ilocked(thread, t);
            binder_inner_proc_unlock(proc);
            goto err_dead_proc_or_thread;
        }
    } else {    //如果请求码为BC_TRANSACTION并且为异步操作，不需要返回
        //将tcomplete插入到事务发起binder线程的todo队列中
        binder_enqueue_thread_work(thread, tcomplete);
        //将t->work插入目标进程的某个线程（或目标进程）的todo队列中并唤醒目标进程
        if (!binder_proc_transaction(t, target_proc, NULL))
            goto err_dead_proc_or_thread;
    }

    //减临时引用计数
    if (target_thread)
        binder_thread_dec_tmpref(target_thread);
    binder_proc_dec_tmpref(target_proc);
    if (target_node)
        binder_dec_node_tmpref(target_node);

    return;
    ... //错误处理
}
```

这一块的代码基本上格式都是一样的，都是将`tcomplete`插入到事务发起`binder`线程的`todo`队列中，`t->work`插入到目标`binder`线程的`todo`队列中，最后唤醒目标进程

这里需要注意的是，在`BC_TRANSACTION`的情况下，需要区分事务的`flags`中是否包含`TF_ONE_WAY`，这意味着这个事务是否需要回应

在没有`TF_ONE_WAY`的情况下，会使用`binder_enqueue_deferred_thread_work_ilocked`函数将`tcomplete`插入到事务发起`binder`线程的`todo`队列中，这个函数区别于`binder_enqueue_thread_work_ilocked`函数，它没有将`thread->process_todo`设为`true`，这个标记在之前介绍`binder_thread`结构体的时候提到了，当其为`false`的情况下会在`binder_thread_read`中休眠，延迟执行`BINDER_WORK_TRANSACTION_COMPLETE`，具体是怎么操作的，我们会在后续的`binder_thread_read`函数中进行分析

在`TF_ONE_WAY`的情况下，我们是没有去寻找合适的目标处理`binder`线程的，关于这一点，我们需要看一下`binder_proc_transaction`函数是怎么处理没有传入`binder_thread`的情况的

```c
static bool binder_proc_transaction(struct binder_transaction *t,
                    struct binder_proc *proc,
                    struct binder_thread *thread)
{
    struct binder_node *node = t->buffer->target_node;
    struct binder_priority node_prio;
    bool oneway = !!(t->flags & TF_ONE_WAY);
    bool pending_async = false;

    binder_node_lock(node);
    node_prio.prio = node->min_priority;
    node_prio.sched_policy = node->sched_policy;

    //如果设置了TF_ONE_WAY标志
    if (oneway) {
        if (node->has_async_transaction) {
            //如果binder实体对象正在处理一个异步事务，做一个标记
            pending_async = true;
        } else {
            //如果binder实体对象没有正在处理一个异步事务，将has_async_transaction置为true，表示接下来要处理一个异步任务
            node->has_async_transaction = true;
        }
    }

    binder_inner_proc_lock(proc);

    //如果目标进程死亡或者目标线程不为NULL且死亡
    if (proc->is_dead || (thread && thread->is_dead)) {
        binder_inner_proc_unlock(proc);
        binder_node_unlock(node);
        return false;
    }

    //如果没有传入目标线程，且目标binder实体对象没有正在处理一个异步事务
    if (!thread && !pending_async)
        //从proc->waiting_threads链表中取出第一个节点元素（没有的话则为NULL）
        thread = binder_select_thread_ilocked(proc);

    if (thread) {    //当找到了合适的binder线程
        //设置事务优先级
        binder_transaction_priority(thread->task, t, node_prio,
                        node->inherit_rt);
        //将t->work插入到目标binder线程的todo队列中
        binder_enqueue_thread_work_ilocked(thread, &t->work);
    } else if (!pending_async) {    //没有找到合适的binder线程，且目标binder实体对象没有正在处理一个异步事务
        //将t->work加入到目标binder进程的todo队列中
        binder_enqueue_work_ilocked(&t->work, &proc->todo);
    } else {    //没有找到合适的binder线程，且目标binder实体对象正在处理一个异步事务
        //将t->work加入到目标binder实体的async_todo队列中
        binder_enqueue_work_ilocked(&t->work, &node->async_todo);
    }

    //目标binder实体对象没有正在处理一个异步事务
    if (!pending_async)
        //唤醒目标binder线程的等待队列
        binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

    binder_inner_proc_unlock(proc);
    binder_node_unlock(node);

    return true;
}
```

当没有传入目标`binder`线程时，从目标进程的等待线程链表中取出第一个`binder_thread`作为处理线程处理该事务，如果没找到合适的空闲线程，分为两种情况：

1. 目标`binder`实体对象正在处理一个异步事务：将相应的`binder_work`插入到目标`binder`实体的`async_todo`队列中
2. 目标`binder`实体对象没有正在处理一个异步事务：将相应的`binder_work`插入到目标`binder`进程的`todo`队列中

关于`binder`驱动是怎么从这些`todo`队列取出`binder_work`并处理的，我们马上在后面`binder_thread_read`里分析，这里我们最后再看一下如何唤醒目标`binder`线程的等待队列

```c
static void binder_wakeup_thread_ilocked(struct binder_proc *proc,
                     struct binder_thread *thread,
                     bool sync)
{
    assert_spin_locked(&proc->inner_lock);

    if (thread) {
        if (sync)
            wake_up_interruptible_sync(&thread->wait);
        else
            wake_up_interruptible(&thread->wait);
        return;
    }
    
    //没有找到一个可用的等待线程，可能在两种情况下发生：
    //1. 所有线程都忙于处理事务
    //在这种情况下，这些线程中的一个应该很快回调到内核驱动程序并执行这项工作
    //2. 线程正在使用epoll轮询，在这种情况下，它们可能在没有被添加到waiting_threads的情况下被阻塞在等待队列上
    //对于这种情况，我们只循环获取所有不处理事务工作的线程，并将它们全部唤醒
    binder_wakeup_poll_threads_ilocked(proc, sync);
}
```

这个函数也有可能`binder_thread`参数传入NULL，在这种情况下，我们需要循环获取目标进程下的所有`binder`线程，对所有不处理事务工作的线程全部执行唤醒操作

```c
static void binder_wakeup_poll_threads_ilocked(struct binder_proc *proc,
                           bool sync)
{
    struct rb_node *n;
    struct binder_thread *thread;

    for (n = rb_first(&proc->threads); n != NULL; n = rb_next(n)) {
        thread = rb_entry(n, struct binder_thread, rb_node);
        if (thread->looper & BINDER_LOOPER_STATE_POLL &&
            binder_available_for_proc_work_ilocked(thread)) {
            if (sync)
                wake_up_interruptible_sync(&thread->wait);
            else
                wake_up_interruptible(&thread->wait);
        }
    }
}
```

# 总结

到这里，我们已经分析了`binder_ioctl`函数的一半`binder_thread_write`，了解了一些相关的数据结构，并且补充了`binder_mmap`篇未完成的内存映射的分析，大家应该对`binder`请求的发起与调度有了一个初步的认识了

本来这一篇是打算把整个`binder_ioctl`分析完的，但没想到写到后面内容这么多，只好再分一篇，下一篇我们将分析`binder_thread_read`，将`binder`驱动篇完结