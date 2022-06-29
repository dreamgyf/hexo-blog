---
title: Android源码分析 - Binder驱动（下）
date: 2022-03-11 18:13:00
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以aosp分支`android-11.0.0_r25`，kernel分支`android-msm-wahoo-4.4-android11`作为基础解析**

上一篇文章[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391 "https://juejin.cn/post/7069675794028560391")，我们分析了`binder_ioctl`中的写操作`binder_thread_write`部分，了解了`binder`请求的发起与调度，接下来我们就进行`binder`驱动的最后一部分分析，`binder_thread_read`

# binder_ioctl_write_read

我们还是先从`binder_ioctl`后的`BINDER_WRITE_READ`命令码开始

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
    ...
    //将用户空间ubuf拷贝至内核空间bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
    ...
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
out:
    return ret;
}
```

# binder_thread_read

这是进行`binder`读操作的函数，这个函数也是比较长，我们同样将它分成几个部分：

1. 等待可用的`binder_work`
2. 循环获取`todo`队列中的`binder_work`，并根据`binder_work`的`type`，执行一定的处理
3. 处理`binder_transaction`以及`binder_transaction_data`，并将`binder_transaction_data`拷贝回用户空间

## 第一部分：等待工作

### binder_thread.looper

在此之前我们需要先看一下之前提到的，在`binder_thread`中的域成员`looper`，前面我们只是注释了这个域表示线程状态，这里我们介绍一下它有哪些取值：

- `BINDER_LOOPER_STATE_REGISTERED`：表示该`binder`线程是非主`binder`线程
- `BINDER_LOOPER_STATE_ENTERED`：表示该`binder`线程是主`binder`线程
- `BINDER_LOOPER_STATE_EXITED`：表示该`binder`线程马上就要退出了
- `BINDER_LOOPER_STATE_INVALID`：表示该`binder`线程是无效的（比如原来是`binder`主线程，后续用户又发送了一个`BC_REGISTER_LOOPER`请求）
- `BINDER_LOOPER_STATE_WAITING`：表示当前`binder`线程正在等待请求
- `BINDER_LOOPER_STATE_NEED_RETURN`：表示该`binder`线程在处理完`transaction`后需要返回到用户态

---

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    //用户空间传进来的需要将数据读到的地址
    //实际上只是传输一些命令码和一个binder_transaction_data_secctx结构体
    //真正的数据已经映射到用户虚拟内存空间中了，根据binder_transaction_data中所给的地址直接读就可以了
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    //起始地址 = 读数据的首地址 + 已读数据大小
    void __user *ptr = buffer + *consumed;
    //结束地址 = 读数据的首地址 + 读数据的总大小
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    if (*consumed == 0) {
        //向用户空间写一个binder响应码，该响应码不做任何操作
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    binder_inner_proc_lock(proc);
    //检查是否有可用的工作需要处理
    wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
    binder_inner_proc_unlock(proc);

    //将线程的状态置为等待中
    thread->looper |= BINDER_LOOPER_STATE_WAITING;

    //如果没有可用的工作，可以等待进程todo队列中的工作
    if (wait_for_proc_work) {
        //这个binder线程既不是主线程，也没有被注册成binder子线程
        //这里条件在binder_available_for_proc_work_ilocked中已经做了判断
        //似乎永远不会进入到这个case中？
        if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
                    BINDER_LOOPER_STATE_ENTERED))) {
            binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)\n",
                proc->pid, thread->pid, thread->looper);
            //进程进入休眠状态，等待唤醒
            wait_event_interruptible(binder_user_error_wait,
                         binder_stop_on_user_error < 2);
        }
        //恢复优先级
        binder_restore_priority(current, proc->default_priority);
    }

    if (non_block) {    //如果是非阻塞模式（这里似乎不会执行到）
        //线程和进程的todo队列中都没有工作
        if (!binder_has_work(thread, wait_for_proc_work))
            ret = -EAGAIN;
    } else {    //如果是阻塞模式
        //等待binder工作到来
        ret = binder_wait_for_work(thread, wait_for_proc_work);
    }

    //将线程的等待中状态解除
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret;
    ...
}
```

这一部分先检查是否有可用的`binder_work`待处理，如果有的话进入到下一部分，如果没有的话则需要等待

具体的，我们先看这个函数中的`wait_for_proc_work`变量，这个变量表示是否需要等待`binder_proc`中的工作，当在`binder_thread`中找不到事务栈并且`todo`队列为空时，此变量值为`true`

```c
static bool binder_available_for_proc_work_ilocked(struct binder_thread *thread)
{
    return !thread->transaction_stack &&
        binder_worklist_empty_ilocked(&thread->todo) &&
        (thread->looper & (BINDER_LOOPER_STATE_ENTERED |
                   BINDER_LOOPER_STATE_REGISTERED));    //这个binder线程既不是主线程，也没有被注册成binder子线程
}
```

在上一篇文章[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391#heading-30)，我们分析过是有分支会将`binder_work`直接添加到`binder_proc`的`todo`队列中的，当`binder_thread`中找不到工作的话，可能就需要从`binder_proc`中找了

然后会将当前线程的状态置为等待中，等到有可处理的`binder_work`后再解除这个状态

之后会判断传入的参数是否为阻塞模式，这个是由`framework`层执行`ioctl`时传入的`flags`所决定的，根据`framework`中的`binder`代码，这里应该恒为阻塞模式，在阻塞模式下，会调用`binder_wait_for_work`函数，等待存在可用的`binder_work`

```c
static int binder_wait_for_work(struct binder_thread *thread,
                bool do_proc_work)
{
    //定义一个等待队列项
    DEFINE_WAIT(wait);
    struct binder_proc *proc = thread->proc;
    int ret = 0;

    freezer_do_not_count();
    binder_inner_proc_lock(proc);
    for (;;) {
        //准备睡眠等待
        prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE);
        //检查确认是否有binder_work可以处理
        if (binder_has_work_ilocked(thread, do_proc_work))
            break;
        //可以处理binder_proc.todo中的工作的话
        if (do_proc_work)
            //将该binder线程加入到binder_proc中的等待线程中
            list_add(&thread->waiting_thread_node,
                 &proc->waiting_threads);
        binder_inner_proc_unlock(proc);
        //睡眠
        schedule();
        binder_inner_proc_lock(proc);
        //将该binder线程从binder_proc中的等待线程中移除
        list_del_init(&thread->waiting_thread_node);
        //检查当前系统调用进程是否有信号处理
        if (signal_pending(current)) {
            ret = -ERESTARTSYS;
            break;
        }
    }
    //结束等待
    finish_wait(&thread->wait, &wait);
    binder_inner_proc_unlock(proc);
    freezer_count();

    return ret;
}

static bool binder_has_work_ilocked(struct binder_thread *thread,
                    bool do_proc_work)
{
    //thread->process_todo为false时会延时执行
    return thread->process_todo ||
        thread->looper_need_return ||
        (do_proc_work &&
         !binder_worklist_empty_ilocked(&thread->proc->todo));
}
```

这个函数的结构和`Linux`函数`wait_event_interruptible`非常相似，我们先来看看它是怎么做到阻塞等待的

---

### Linux进程调度

#### DEFINE_WAIT

这是一个宏，它定义了一个`wait_queue_t`结构体

```c
#define DEFINE_WAIT_FUNC(name, function)				\
	wait_queue_t name = {						\
		.private	= current,				\
		.func		= function,				\
		.task_list	= LIST_HEAD_INIT((name).task_list),	\
	}

#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)
```

这个结构体中的`private`域指向的即是当前执行系统调用所在进程的描述结构体，`func`域指向唤醒这个队列项进程所执行的函数

#### prepare_to_wait

这个函数将我们刚刚定义的`wait`队列项加入到一个等待队列中（在`binder`中即是加入到`thread->wait`中），然后将进程的状态设置为我们指定的状态，在这里为`TASK_INTERRUPTIBLE`（可中断的睡眠状态）

#### schedule

`schedule`是真正执行进程调度的地方，由于之前进程状态已经被设置成`TASK_INTERRUPTIBLE`状态，在调用`schedule`后，该进程就会让出`CPU`，不再调度运行，直到该进程恢复`TASK_RUNNING`状态

#### wake_up_interruptible

我们在上一篇文章[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391#heading-30)中提到过，当`binder_transaction`工作处理完后，会调用`wake_up_interruptible`函数唤醒目标`binder`线程的等待队列

这个函数会唤醒`TASK_INTERRUPTIBLE`状态下的进程，它会循环遍历等待队列中的每个元素，分别执行其唤醒函数，也就对应着我们`DEFINE_WAIT`定义出来的结构体中的`func`域，即`autoremove_wake_function`，它最终会调用`try_to_wake_up`函数将进程置为`TASK_RUNNING`状态，这样后面的进程调度便会调度到该进程，从而唤醒该进程继续执行

#### signal_pending

这个函数是用来检查当前系统调用进程是否有信号需要处理的，当一个进程陷入系统调用并处于等待状态时，如果此时产生了一个信号，仅仅是在该进程的`thread_info`中标识了一下，所以我们唤醒进程后需要检查一下是否有信号需要处理，如果有的话，返回`-ERESTARTSYS`，先处理信号，后续`Linux`上层库函数会根据`-ERESTARTSYS`此返回值重新执行这个系统调用

#### finish_wait

最后一步，当进程被唤醒后，调用`finish_wait`函数执行清理工作，将当前进程置为`TASK_RUNNING`状态，并把当前`wait`队列项从等待队列中移除

---

了解了上面这些知识，我们再看`binder_wait_for_work`函数应该比较清晰了，这里有一点需要注意，我们在上一篇文章[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391#heading-30)提到，在没有`TF_ONE_WAY`标志的情况下，会使用`binder_enqueue_deferred_thread_work_ilocked`函数将`tcomplete`插入到事务发起`binder`线程的`todo`队列中，这个函数区别于`binder_enqueue_thread_work_ilocked`函数，它没有将`thread->process_todo`设为`true`，所以结合着`binder_has_work_ilocked`函数看，我们可以发现，当`thread->process_todo`为`false`时，整个`binder_has_work_ilocked`返回`false`，即会进入到睡眠状态，延迟执行`BINDER_WORK_TRANSACTION_COMPLETE`，这么设计可以让`binder`接收端优先处理事务，提高了性能

## 第二部分：获取工作，根据type做一定的处理

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    //用户空间传进来的需要将数据读到的地址
    //实际上只是传输一些命令码和一个binder_transaction_data_secctx结构体
    //真正的数据已经映射到用户虚拟内存空间中了，根据binder_transaction_data中所给的地址直接读就可以了
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    //起始地址 = 读数据的首地址 + 已读数据大小
    void __user *ptr = buffer + *consumed;
    //结束地址 = 读数据的首地址 + 读数据的总大小
    void __user *end = buffer + size;
    ...
retry:
    ...
    //循环处理todo队列中的工作
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data_secctx tr;
        struct binder_transaction_data *trd = &tr.transaction_data;
        struct binder_work *w = NULL;
        struct list_head *list = NULL;
        struct binder_transaction *t = NULL;
        struct binder_thread *t_from;
        size_t trsize = sizeof(*trd);

        binder_inner_proc_lock(proc);
        //找到需要处理的todo队列
        if (!binder_worklist_empty_ilocked(&thread->todo))
            list = &thread->todo;
        else if (!binder_worklist_empty_ilocked(&proc->todo) &&
               wait_for_proc_work)
            list = &proc->todo;
        else {
            binder_inner_proc_unlock(proc);

            /* no data added */
            //只跳过了数据头部的命令码，没有读取任何数据
            if (ptr - buffer == 4 && !thread->looper_need_return)
                goto retry;
            break;
        }

        //传输过来的数据大小不符合
        if (end - ptr < sizeof(tr) + 4) {
            binder_inner_proc_unlock(proc);
            break;
        }
        //从todo队列中出队一项binder_work
        w = binder_dequeue_work_head_ilocked(list);
        if (binder_worklist_empty_ilocked(&thread->todo))
            thread->process_todo = false;

        switch (w->type) {
            case BINDER_WORK_TRANSACTION: {
                binder_inner_proc_unlock(proc);
                //根据binder_work找到binder_transaction结构
                t = container_of(w, struct binder_transaction, work);
            } break;
            case BINDER_WORK_RETURN_ERROR: {
                ...
            } break;
            case BINDER_WORK_TRANSACTION_COMPLETE: {
                binder_inner_proc_unlock(proc);
                cmd = BR_TRANSACTION_COMPLETE;
                //回复给用户进程BR_TRANSACTION_COMPLETE响应码
                if (put_user(cmd, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);

                //更新统计数据
                binder_stat_br(proc, thread, cmd);
                //释放
                kfree(w);
                binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
            } break;
            case BINDER_WORK_NODE: {
                ...
            } break;
            case BINDER_WORK_DEAD_BINDER:
            case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
            case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: {
                ...
            } break;
        }
        ...
    }
done:
    ...
}
```

这里先创建了一个`binder_transaction_data_secctx`结构体，后续会将它拷贝到用户空间去，然后创建了一个指针`trd`指向`tr.transaction_data`的地址，这样后续操作`trd`就相当于操作`tr.transaction_data`了

当进程从睡眠中唤醒，意味着有可用的`binder_work`了，这时候理论上来说，`binder_thread`和`binder_proc`其中总有一个`todo`队列不为空，这里优先处理`binder_thread`的`todo`队列，如果两者都为空，且还未读取过任何数据，重新`goto`到`retry`处等待

接着就是将`binder_work`从相应的`todo`队列中出队，再根据其类型执行不同的处理操作，这里我们只针对`BINDER_WORK_TRANSACTION`和`BINDER_WORK_TRANSACTION_COMPLETE`这两种最重要的类型分析

当类型为`BINDER_WORK_TRANSACTION`时，表示是别的进程向自己发起`binder`请求，此时，我们根据`binder_work`找到对应的`binder_transaction`结构

当类型为`BINDER_WORK_TRANSACTION_COMPLETE`时，表示发起的请求`BC_TRANSACTION`已经完成了，此时将回复给用户空间`BR_TRANSACTION_COMPLETE`响应码，然后更新统计数据，释放资源

## 第三部分：处理`binder_transaction`，拷贝回用户空间

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    //用户空间传进来的需要将数据读到的地址
    //实际上只是传输一些命令码和一个binder_transaction_data_secctx结构体
    //真正的数据已经映射到用户虚拟内存空间中了，根据binder_transaction_data中所给的地址直接读就可以了
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    //起始地址 = 读数据的首地址 + 已读数据大小
    void __user *ptr = buffer + *consumed;
    //结束地址 = 读数据的首地址 + 读数据的总大小
    void __user *end = buffer + size;
    ...
retry:
    ...
    //循环处理todo队列中的工作
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data_secctx tr;
        struct binder_transaction_data *trd = &tr.transaction_data;
        struct binder_work *w = NULL;
        struct list_head *list = NULL;
        struct binder_transaction *t = NULL;
        struct binder_thread *t_from;
        size_t trsize = sizeof(*trd);
        ...
        //只有在type == BINDER_WORK_TRANSACTION的情况下，t才会被赋值
        if (!t)
            continue;

        if (t->buffer->target_node) {    //binder实体不为NULL，对应着BC_TRANSACTION请求
            struct binder_node *target_node = t->buffer->target_node;
            struct binder_priority node_prio;

            //binder实体在用户空间中的地址
            trd->target.ptr = target_node->ptr;
            //携带的额外数据
            trd->cookie =  target_node->cookie;
            //优先级
            node_prio.sched_policy = target_node->sched_policy;
            node_prio.prio = target_node->min_priority;
            binder_transaction_priority(current, t, node_prio,
                            target_node->inherit_rt);
            //设置响应码
            cmd = BR_TRANSACTION;
        } else {    //binder实体为NULL，对应着BC_REPLY请求
            trd->target.ptr = 0;
            trd->cookie = 0;
            //设置响应码
            cmd = BR_REPLY;
        }
        //表示要对目标对象请求的命令代码
        trd->code = t->code;
        //事务标志，详见enum transaction_flags
        trd->flags = t->flags;
        //请求发起进程的uid
        trd->sender_euid = from_kuid(current_user_ns(), t->sender_euid);

        //获取发起请求的binder线程
        t_from = binder_get_txn_from(t);
        if (t_from) {
            struct task_struct *sender = t_from->proc->tsk;
            //设置发起请求的进程pid
            trd->sender_pid =
                task_tgid_nr_ns(sender,
                        task_active_pid_ns(current));
        } else {
            trd->sender_pid = 0;
        }

        //数据大小
        trd->data_size = t->buffer->data_size;
        //偏移数组大小
        trd->offsets_size = t->buffer->offsets_size;
        //设置数据区首地址（这里通过内核空间地址和user_buffer_offset计算得出用户空间地址）
        trd->data.ptr.buffer = (binder_uintptr_t)
            ((uintptr_t)t->buffer->data +
            binder_alloc_get_user_buffer_offset(&proc->alloc));
        //偏移数组紧挨着数据区，所以它的首地址就为数据区地址加上数据大小
        trd->data.ptr.offsets = trd->data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        tr.secctx = t->security_ctx;
        if (t->security_ctx) {
            cmd = BR_TRANSACTION_SEC_CTX;
            trsize = sizeof(tr);
        }
        
        //回复给用户进程对应的响应码
        if (put_user(cmd, (uint32_t __user *)ptr)) {
            ... //Error
        }
        ptr += sizeof(uint32_t);
        //将binder_transaction_data拷贝至用户空间
        if (copy_to_user(ptr, &tr, trsize)) {
            ... //Error
        }
        ptr += trsize;
        //更新数据统计
        binder_stat_br(proc, thread, cmd);

        //临时引用计数减1
        if (t_from)
            binder_thread_dec_tmpref(t_from);
        //允许释放这个buffer
        t->buffer->allow_user_free = 1;
        if (cmd != BR_REPLY && !(t->flags & TF_ONE_WAY)) {
            //非异步处理
            binder_inner_proc_lock(thread->proc);
            //将这个事务插入到事务栈中
            t->to_parent = thread->transaction_stack;
            //设置目标处理线程
            t->to_thread = thread;
            thread->transaction_stack = t;
            binder_inner_proc_unlock(thread->proc);
        } else {
            binder_free_transaction(t);
        }
        break;
    }

done:
    //更新已读数据大小
    *consumed = ptr - buffer;
    binder_inner_proc_lock(proc);
    //请求线程数为0且没有等待线程，已启动线程数小于最大线程数
    //且这个binder线程既不是主线程，也没有被注册成binder子线程
    if (proc->requested_threads == 0 &&
        list_empty(&thread->proc->waiting_threads) &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
         /*spawn a new thread if we leave this out */) {
        //向用户空间发送BR_SPAWN_LOOPER响应码，创建新binder线程
        proc->requested_threads++;
        binder_inner_proc_unlock(proc);
        if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
            return -EFAULT;
        //更新统计信息
        binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
    } else
        binder_inner_proc_unlock(proc);
    return 0;
}
```

只有当`type == BINDER_WORK_TRANSACTION`的情况下，才会走后面的这些逻辑，这一部分主要做的工作是，将处理事务所需要的信息（命令码、PID等）和数据（数据区首地址和偏移数组地址）准备好，拷贝到用户空间，交给用户空间处理这个事务，还有一些细节都在上面代码注释中标出来了，应该还是比较清晰的

# 结束

到这里，从`client`向`binder`驱动发起请求，到`binder`驱动找到`server`并将请求传递给`server`这一整个流程我们基本上是看完了，对于被`TF_ONE_WAY`标记的事务（无需返回），在`server`中处理完，这一整条`binder`进程通信流程也就结束了，而对于需要返回的事务，则还需要`server`向`binder`驱动发起`BC_REPLY`事务请求，进行一次进程间通信，将处理后的返回值传给`client`进程

# 总结

我们已经将`binder`驱动所承担的工作分析的七七八八了，但肯定还有很多地方大家云里雾里（实际上我也是这样的），这是因为`binder`进程间通信不仅需要`binder`驱动的处理，还需要`framework`层的协助，很多事情，比如说`binder`对象的压扁打包就是在`framework`层做的，我们在`binder`驱动层只关注了传输，但却不知道传输的是什么，结构是怎样的，当然会产生各种各样的疑惑，后面我们就将会对`framework`层对`binder`提供的支持接着做分析