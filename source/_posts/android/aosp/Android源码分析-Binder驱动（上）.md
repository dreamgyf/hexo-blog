---
title: Android源码分析 - Binder驱动（上）
date: 2022-02-09 18:29:00
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以aosp分支`android-11.0.0_r25`，kernel分支`android-msm-wahoo-4.4-android11`作为基础解析**

上一篇文章[Android源码分析 - Binder概述](https://juejin.cn/post/7059601252367204365 "Android源码分析 - Binder概述")我们大概了解了一下`Android`选用`Binder`的原因，以及`Binder`的基本结构和通信过程。今天，我们便开始从`Binder`驱动层代码开始分析`Binder`的机制

## 提示

`Binder`驱动部分代码不在`AOSP`项目中，所以我们需要单独`clone`一份驱动代码

由于我的开发设备是pixel2，查了`Linux`内核版本号为`4.4.223`，对应的分支为`android-msm-wahoo-4.4-android11`，所以今天的分析我们也是基于此分支

我是从清华大学镜像站`clone`的代码，高通的设备，所以地址为：https://aosp.tuna.tsinghua.edu.cn/android/kernel/msm.git

# 初始化

`binder`驱动的源码位于`drivers/android`目录下，我们从`binder.c`文件看起

## Linux initcall机制

在`binder.c`的最底下，我们可以看到这一行代码

```c
device_initcall(binder_init);
```

在`Linux`内核中，驱动程序通常是用`xxx_initcall(fn)`启动的，这实际上是一个宏定义，被定义在平台对应的`init.h`文件中

```c
#define early_initcall(fn) __define_initcall(fn, early)
#define pure_initcall(fn) __define_initcall(fn, 0) 
#define core_initcall(fn) __define_initcall(fn, 1) 
#define core_initcall_sync(fn) __define_initcall(fn, 1s) 
#define postcore_initcall(fn) __define_initcall(fn, 2) 
#define postcore_initcall_sync(fn) __define_initcall(fn, 2s) 
#define arch_initcall(fn) __define_initcall(fn, 3) 
#define arch_initcall_sync(fn) __define_initcall(fn, 3s) 
#define subsys_initcall(fn) __define_initcall(fn, 4)
#define subsys_initcall_sync(fn) __define_initcall(fn, 4s) 
#define fs_initcall(fn) __define_initcall(fn, 5) 
#define fs_initcall_sync(fn) __define_initcall(fn, 5s) 
#define rootfs_initcall(fn) __define_initcall(fn, rootfs) 
#define device_initcall(fn) __define_initcall(fn, 6) 
#define device_initcall_sync(fn) __define_initcall(fn, 6s) 
#define late_initcall(fn) __define_initcall(fn, 7) 
#define late_initcall_sync(fn) __define_initcall(fn, 7s)
```

可以看到，实际上调用的是`__define_initcall()`函数，这个函数的第二个参数表示优先级，数字越小，优先级越高，带s的优先级低于不带s的优先级

在Linux内核启动过程中，需要调用各种函数，在底层实现是通过在内核镜像文件中，自定义一个段，这个段里面专门用来存放这些初始化函数的地址，内核启动时，只需要在这个段地址处取出函数指针，一个个执行即可，而`__define_initcall()`函数，就是将自定义的init函数添加到上述段中

## binder_init

了解了以上函数定义后，我们再回头看`device_initcall(binder_init)`就可以知道，在`Linux`内核启动时，会调用`binder_init`这么一个函数

```c
static int __init binder_init(void)
{
    int ret;
    char *device_name, *device_names, *device_tmp;
    struct binder_device *device;
    struct hlist_node *tmp;

    //初始化binder内存回收
    ret = binder_alloc_shrinker_init();
    if (ret)
        return ret;

    ...
    //创建一个单线程工作队列，用于处理异步任务
    binder_deferred_workqueue = create_singlethread_workqueue("binder");
    if (!binder_deferred_workqueue)
        return -ENOMEM;
    
    //创建binder/proc目录
    binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
    if (binder_debugfs_dir_entry_root)
        binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
                         binder_debugfs_dir_entry_root);
    //在binder目录下创建5个文件
    if (binder_debugfs_dir_entry_root) {
        debugfs_create_file("state",
                    0444,
                    binder_debugfs_dir_entry_root,
                    NULL,
                    &binder_state_fops);
        debugfs_create_file("stats",
                    0444,
                    binder_debugfs_dir_entry_root,
                    NULL,
                    &binder_stats_fops);
        debugfs_create_file("transactions",
                    0444,
                    binder_debugfs_dir_entry_root,
                    NULL,
                    &binder_transactions_fops);
        debugfs_create_file("transaction_log",
                    0444,
                    binder_debugfs_dir_entry_root,
                    &binder_transaction_log,
                    &binder_transaction_log_fops);
        debugfs_create_file("failed_transaction_log",
                    0444,
                    binder_debugfs_dir_entry_root,
                    &binder_transaction_log_failed,
                    &binder_transaction_log_fops);
    }

    //"binder,hwbinder,vndbinder"
    device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
    if (!device_names) {
        ret = -ENOMEM;
        goto err_alloc_device_names_failed;
    }
    strcpy(device_names, binder_devices_param);

    device_tmp = device_names;
    //用binder,hwbinder,vndbinder分别调用init_binder_device函数
    while ((device_name = strsep(&device_tmp, ","))) {
        ret = init_binder_device(device_name);
        if (ret)
            goto err_init_binder_device_failed;
    }

    return ret;

err_init_binder_device_failed:
    ...

err_alloc_device_names_failed:
    ...
}
```

我们将重点放在`init_binder_device`函数上

## init_binder_device

```c
static int __init init_binder_device(const char *name)
{
    int ret;
    struct binder_device *binder_device;

    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
    if (!binder_device)
        return -ENOMEM;

    //binder注册虚拟字符设备所对应的file_operations
    binder_device->miscdev.fops = &binder_fops;
    //动态分配次设备号
    binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
    binder_device->miscdev.name = name;

    binder_device->context.binder_context_mgr_uid = INVALID_UID;
    binder_device->context.name = name;
    //初始化互斥锁
    mutex_init(&binder_device->context.context_mgr_node_lock);
    //注册misc设备
    ret = misc_register(&binder_device->miscdev);
    if (ret < 0) {
        kfree(binder_device);
        return ret;
    }
    //将binder设备加入链表（头插法）
    hlist_add_head(&binder_device->hlist, &binder_devices);

    return ret;
}
```

先构造了一个结构体用来存放`binder`参数，然后通过`misc_register`函数，以`misc`设备进行注册`binder`，作为虚拟字符设备

### 注册misc设备

我们先学习一下在`Linux`中如何注册一个`misc`设备

在Linux驱动中把无法归类的五花八门的设备定义为`misc`设备，`Linux`内核所提供的`misc`设备有很强的包容性，各种无法归结为标准字符设备的类型都可以定义为`misc`设备，譬如NVRAM，看门狗，实时时钟，字符LCD等

在`Linux`内核里把所有的`misc`设备组织在一起，构成了一个子系统(`subsys`)，统一进行管理。在这个子系统里的所有`miscdevice`类型的设备共享一个主设备号`MISC_MAJOR`(10)，但次设备号不同

在内核中用`miscdevice`结构体表示`misc`设备，具体的定义在`include/linux/miscdevice.h`中

```c
struct miscdevice  {
    int minor;
    const char *name;
    const struct file_operations *fops;
    struct list_head list;
    struct device *parent;
    struct device *this_device;
    const struct attribute_group **groups;
    const char *nodename;
    umode_t mode;
};
```

我们自己注册`misc`设备时只需要填入前3项即可：

- `minor`：次设备号，如果填充`MISC_DYNAMIC_MINOR`，则由内核动态分配次设备号
- `name`：设备名
- `fops`：`file_operations`结构体，用于定义自己`misc`设备的文件操作函数，如果不填此项则会使用默认的`misc_fops`

`file_operations`结构体被定义在`include/linux/fs.h`中

```c
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    int (*iterate) (struct file *, struct dir_context *);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
    int (*aio_fsync) (struct kiocb *, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
    ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock) (struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    int (*setlease)(struct file *, long, struct file_lock **, void **);
    long (*fallocate)(struct file *file, int mode, loff_t offset,
              loff_t len);
    void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
    unsigned (*mmap_capabilities)(struct file *);
#endif
};
```

`file_operation`是把系统调用和驱动程序关联起来的关键结构，这个结构的每一个成员都对应着一个系统调用，`Linux`系统调用通过读取`file_operation`中相应的函数指针，接着把控制权转交给函数，从而完成`Linux`设备驱动程序的工作

最后调用`misc_register`函数注册`misc`设备，函数原型如下：

```c
//注册misc设备
extern int misc_register(struct miscdevice *misc);
//卸载misc设备
extern void misc_deregister(struct miscdevice *misc);
```

### 注册binder设备

了解了`misc`设备的注册，我们就可以看一下`binder`的注册过程了，代码中先构建了一个`binder_device`结构体，我们先观察一下这个结构体长什么样子

```c
struct binder_device {
    struct hlist_node hlist;
    struct miscdevice miscdev;
    struct binder_context context;
};
```

其中的`hlist_node`是链表中的一个节点，`miscdevice`就是上文所描述的注册`misc`所必要的结构体参数，`binder_context`用于保存`binder`上下文管理者的信息

回到代码中，首先给`miscdevice`赋了值，指定了`file_operation`，设置了`minor`动态分配次设备号，`binder_context`则是简单初始化了一下，然后便调用`misc_register`函数注册`misc`设备，最后将这个`binder`设备使用头插法加入到一个全局链表中

我们看一下它指定的`file_operation`

```c
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};
```

可以看到，`binder`驱动支持以上7种系统调用，接下来，我们就逐一分析这些系统调用

# binder_proc

在分析这些系统调用前，我们有必要先了解一下在`binder`中非常重要的结构体`binder_proc`，它是用来描述进程上下文信息以及管理IPC的一个结构体，被定义在`drivers/android/binder.c`中，是一个私有的结构体

```c
struct binder_proc {
    //hash链表中的一个节点
    struct hlist_node proc_node;
    //处理用户请求的线程组成的红黑树
    struct rb_root threads;
    //binder实体组成的红黑树
    struct rb_root nodes;
    //binder引用组成的红黑树，以句柄来排序
    struct rb_root refs_by_desc;
    //binder引用组成的红黑树，以它对应的binder实体的地址来排序
    struct rb_root refs_by_node;
    struct list_head waiting_threads;
    //进程id
    int pid;
    //进程描述符
    struct task_struct *tsk;
    //进程打开的所有文件数据
    struct files_struct *files;
    struct mutex files_lock;
    struct hlist_node deferred_work_node;
    int deferred_work;
    bool is_dead;
    //待处理事件队列
    struct list_head todo;
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;
    int requested_threads;
    int requested_threads_started;
    atomic_t tmp_ref;
    struct binder_priority default_priority;
    struct dentry *debugfs_entry;
    //用来记录mmap分配的用户虚拟地址空间和内核虚拟地址空间等信息
    struct binder_alloc alloc;
    struct binder_context *context;
    spinlock_t inner_lock;
    spinlock_t outer_lock;
};
```

# binder_open

我们先从打开`binder`驱动设备开始

```c
static int binder_open(struct inode *nodp, struct file *filp)
{
    //管理IPC和保存进程信息的结构体
    struct binder_proc *proc;
    struct binder_device *binder_dev;
    ...
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
        
    //初始化内核同步自旋锁
    spin_lock_init(&proc->inner_lock);
    spin_lock_init(&proc->outer_lock);
    //原子操作赋值
    atomic_set(&proc->tmp_ref, 0);
    //使执行当前系统调用进程的task_struct.usage加1
    get_task_struct(current->group_leader);
    //使binder_proc中的tsk指向执行当前系统调用的进程
    proc->tsk = current->group_leader;
    //初始化文件锁
    mutex_init(&proc->files_lock);
    //初始化todo列表
    INIT_LIST_HEAD(&proc->todo);
    //设置优先级
    if (binder_supported_policy(current->policy)) {
        proc->default_priority.sched_policy = current->policy;
        proc->default_priority.prio = current->normal_prio;
    } else {
        proc->default_priority.sched_policy = SCHED_NORMAL;
        proc->default_priority.prio = NICE_TO_PRIO(0);
    }
    //找到binder_device结构体的首地址
    binder_dev = container_of(filp->private_data, struct binder_device,
                  miscdev);
    //使binder_proc的上下文指向binder_device的上下文
    proc->context = &binder_dev->context;
    //初始化binder缓冲区
    binder_alloc_init(&proc->alloc);
    //全局binder_stats结构体中，BINDER_STAT_PROC类型的对象创建数量加1
    binder_stats_created(BINDER_STAT_PROC);
    //设置当前进程id
    proc->pid = current->group_leader->pid;
    //初始化已分发的死亡通知列表
    INIT_LIST_HEAD(&proc->delivered_death);
    //初始化等待线程列表
    INIT_LIST_HEAD(&proc->waiting_threads);
    //保存binder_proc数据
    filp->private_data = proc;

    //因为binder支持多线程，所以需要加锁
    mutex_lock(&binder_procs_lock);
    //将binder_proc添加到binder_procs全局链表中
    hlist_add_head(&proc->proc_node, &binder_procs);
    //释放锁
    mutex_unlock(&binder_procs_lock);

    //在binder/proc目录下创建文件，以执行当前系统调用的进程id为名
    if (binder_debugfs_dir_entry_proc) {
        char strbuf[11];
        snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
        proc->debugfs_entry = debugfs_create_file(strbuf, 0444,
            binder_debugfs_dir_entry_proc,
            (void *)(unsigned long)proc->pid,
            &binder_proc_fops);
    }

    return 0;
}
```

`binder_open`函数创建了`binder_proc`结构体，并把初始化并将当前进程等信息保存到`binder_proc`结构体中，然后将`binder_proc`结构体保存到文件指针`filp`的`private_data`中，再将`binder_proc`加入到全局链表`binder_procs`中

这里面有一些关于`Linux`的知识需要解释一下

## spinlock

`spinlock`是内核中提供的一种自旋锁机制。在`Linux`内核实现中，常常会碰到共享数据被中断上下文和进程上下文访问的场景，如果只有进程上下文的话，我们可以使用互斥锁或者信号量解决，将未获得锁的进程置为睡眠状态等待，但由于中断上下文不是一个进程，它不存在`task_struct`，所以不可被调度，当然也就不可睡眠，这时候就可以通过`spinlock`自旋锁的忙等待机制来达成睡眠同样的效果

## current

在`Linux`内核中，定义了一个叫`current`的宏，它被定义在`asm/current.h`中

```c
static inline struct task_struct *get_current(void)
{
	return(current_thread_info()->task);
}

#define	current	get_current()
```

它返回一个`task_struct`指针，指向执行当前这段内核代码的进程

## container_of

`container_of`也是`Linux`中定义的一个宏，它的作用是根据一个结构体变量中的一个域成员变量的指针来获取指向整个结构体变量的指针

```c
#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE *)0)->MEMBER)

#define container_of(ptr, type, member) ({              \         
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \         
    (type *)( (char *)__mptr - offsetof(type,member) );})
```


## fd&filp

`filp->private_data`保存了`binder_proc`结构体，当进程调用`open`系统函数时，内核会返回一个文件描述符`fd`，这个`fd`指向文件指针`filp`，在后续调用`mmap`，`ioctl`等函数与`binder`驱动交互时，会传入这个`fd`，内核就会以这个`fd`指向文件指针`filp`作为参数调用`binder_mmap`，`binder_ioctl`等函数，这样这些函数就可以通过`filp->private_data`取出`binder_proc`结构体

# binder_mmap

## vm_area_struct

在分析`mmap`前，我们需要先了解一下`vm_area_struct`这个结构体，它被定义在`include/linux/mm_types.h`中

```c
struct vm_area_struct {
    //当前vma的首地址
    unsigned long vm_start;
    //当前vma的末地址后第一个字节的地址
    unsigned long vm_end;
    
    //链表
    struct vm_area_struct *vm_next, *vm_prev;
    //红黑树中对应节点
    struct rb_node vm_rb;

    //当前vma前面还有多少空闲空间
    unsigned long rb_subtree_gap;

    //当前vma所属的内存地址空间
    struct mm_struct *vm_mm;
    //访问权限
    pgprot_t vm_page_prot;
    //vma标识集，定义在 include/linux/mm.h 中
    unsigned long vm_flags;

    union {
        struct {
            struct rb_node rb;
            unsigned long rb_subtree_last;
        } shared;
        const char __user *anon_name;
    };

    struct list_head anon_vma_chain;
    struct anon_vma *anon_vma;

    //当前vma操作函数集指针
    const struct vm_operations_struct *vm_ops;

    //当前vma起始地址在vm_file中的文件偏移，单位为物理页面PAGE_SIZE
    unsigned long vm_pgoff;
    //被映射的文件（如果使用文件映射）
    struct file * vm_file;
    void * vm_private_data;

#ifndef CONFIG_MMU
    struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
    struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
    struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
};
```

`vm_area_struct`结构体描述了一段虚拟内存空间，通常，进程所使用到的虚拟内存空间不连续，且各部分虚存空间的访问属性也可能不同，所以一个进程的虚拟内存空间需要多个`vm_area_struct`结构来描述（后面简称`vma`）

每个进程都有一个对应的`task_struct`结构描述，这个`task_struct`结构中有一个`mm_struct`结构用于描述进程的内存空间，`mm_struct`结构中有两个域成员变量分别指向了`vma`链表头和红黑树根

`vma`所描述的虚拟内存空间范围由`vm_start`和`vm_end`表示，`vm_start`代表当前`vma`的首地址，`vm_end`代表当前`vma`的末地址后第一个字节的地址，即虚拟内存空间范围为`[vm_start, vm_end)`

`vm_operations_struct`和上文中的`file_operations`类似，用来定义虚拟内存的操作函数

--- 

介绍完`vma`，接下来我们便看一下`binder_mmap`函数

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;

    //校验进程信息
    if (proc->tsk != current->group_leader)
        return -EINVAL;

    //将虚拟内存地址大小限制在4M
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;
    ...
    //检查用户空间是否可写（FORBIDDEN_MMAP_FLAGS == VM_WRITE）
    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        ret = -EPERM;
        failure_string = "bad vm_flags";
        goto err_bad_arg;
    }
    //VM_DONTCOPY表示此vma不可被fork所复制
    vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
    //用户空间不可设置该vma的VM_WRITE标志
    vma->vm_flags &= ~VM_MAYWRITE;
    //设置此vma操作函数集
    vma->vm_ops = &binder_vm_ops;
    //指向binder_proc
    vma->vm_private_data = proc;

    //处理进程虚拟内存空间与内核虚拟地址空间的映射关系
    ret = binder_alloc_mmap_handler(&proc->alloc, vma);
    if (ret)
        return ret;
    mutex_lock(&proc->files_lock);
    //获取进程的打开文件信息结构体files_struct，并将引用计数加1
    proc->files = get_files_struct(current);
    mutex_unlock(&proc->files_lock);
    return 0;

err_bad_arg:
    pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
           proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
    return ret;
}
```

1. 首先从`filp`中获取对应的`binder_proc`信息
2. 将它的进程`task_struct`和执行当前这段内核代码的进程`task_struct`对比校验
3. 限制了用户空间虚拟内存的大小在4M以内
4. 检查用户空间是否可写（`binder`驱动为进程分配的缓冲区在用户空间中只可以读，不可以写）
5. 设置`vm_flags`，令`vma`不可写，不可复制
6. 设置`vma`的操作函数集
7. 将`vm_area_struct`中的成员变量`vm_private_data`指向`binder_proc`，使得`vma`设置的操作函数中可以拿到`binder_proc`
8. 处理进程虚拟内存空间与内核虚拟地址空间的映射关系
9. 获取进程的打开文件信息结构体`files_struct`，令`binder_proc`的`files`指向它，并将引用计数加1

## binder_alloc_mmap_handler

`binder_alloc_mmap_handler`将进程虚拟内存空间与内核虚拟地址空间做映射，它被实现在`drivers/android/binder_alloc.c`中

这里先介绍一下`vm_struct`，之前我们已经了解了`vm_area_struct`表示用户进程中的虚拟地址空间，而相对应的，`vm_struct`则表示内核中的虚拟地址空间

```c
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
			      struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area;
	const char *failure_string;
	struct binder_buffer *buffer;

	mutex_lock(&binder_alloc_mmap_lock);
        //检查是否已经分配过内核缓冲区
	if (alloc->buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}
        //获得一个内核虚拟空间
	area = get_vm_area(vma->vm_end - vma->vm_start, VM_ALLOC);
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
        //alloc->buffer指向内核虚拟内存空间地址
	alloc->buffer = area->addr;
        //计算出用户虚拟空间线性地址到内核虚拟空间线性地址的偏移量
	alloc->user_buffer_offset =
		vma->vm_start - (uintptr_t)alloc->buffer;
	mutex_unlock(&binder_alloc_mmap_lock);
        ...
        //申请内存
	alloc->pages = kzalloc(sizeof(alloc->pages[0]) *
				   ((vma->vm_end - vma->vm_start) / PAGE_SIZE),
			       GFP_KERNEL);
	if (alloc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
        //buffer大小等于vma大小
	alloc->buffer_size = vma->vm_end - vma->vm_start;

	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}
        //指向内核虚拟空间地址
	buffer->data = alloc->buffer;
        //将buffer添加到链表中
	list_add(&buffer->entry, &alloc->buffers);
	buffer->free = 1;
        //将此内核缓冲区加入到binder_alloc的空闲缓冲红黑树中
	binder_insert_free_buffer(alloc, buffer);
        //设置进程最大可用异步事务缓冲区大小（防止异步事务消耗过多内核缓冲区，影响同步事务）
	alloc->free_async_space = alloc->buffer_size / 2;
        //内存屏障，保证指令顺序执行
	barrier();
        //设置binder_alloc
	alloc->vma = vma;
	alloc->vma_vm_mm = vma->vm_mm;
	//引用计数+1
	atomic_inc(&alloc->vma_vm_mm->mm_count);

	return 0;
        ... //错误处理
}
```

1. 检查是否已经分配过内核缓冲区
2. 从内核中寻找一块可用的虚拟内存地址
3. 将此内核虚拟内存空间地址保存至`binder_alloc`
4. 计算出用户虚拟空间线性地址到内核虚拟空间线性地址的偏移量（这样就可以非常方便的在用户虚拟内存空间与内核虚拟内存空间间切换）
5. 为`alloc->pages`数组申请内存，申请的大小等于`vma`能分配多少个页框
6. 设置`buffer`大小等于`vma`大小
7. 为`binder_buffer`申请内存，填充参数，使其指向内核虚拟空间地址，并将其添加到链表和红黑树中
8. 设置`binder_alloc`其他参数

这里要注意，虽然我们计算出了用户虚拟空间线性地址到内核虚拟空间线性地址的偏移量，但并没有建立映射关系。在旧版内核中，这里会调用`binder_update_page_range`函数分别将内核虚拟内存和进程虚拟内存与物理内存做映射，这样内核虚拟内存和进程虚拟内存也相当于间接建立了映射关系，而在`4.4.223`中，这件事将会延迟到`binder_ioctl`后

当完成物理内存的映射后，以32位系统，缓冲区大小4M为例，效果应该如下图所示：

![binder_mmap](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Binder%E9%A9%B1%E5%8A%A8%EF%BC%88%E4%B8%8A%EF%BC%89_mmap.png)

# 总结

到这里，我们已经了解了`binder`驱动设备是如何注册的，并且分析了`binder_open`和`binder_mmap`操作函数，了解了一些重要的结构体，明白了`mmap`是如何映射用户空间和内核空间的，由于篇幅原因，下一章我们会分析`binder`驱动中最重要的部分`binder_ioctl`

# 参考文献

- [linux中的misc设备](http://unicornx.github.io/2016/02/14/20160214-lk-drv-miscdevice/)
- [内存映射与VMA](https://blog.csdn.net/u012142460/article/details/90344951)
- [Android 重学系列 Binder驱动的初始化 映射原理(二)](https://www.jianshu.com/p/4399aedb4d42)
- [Binder系列1—Binder Driver初探](http://gityuan.com/2015/11/01/binder-driver/)
- [Linux 4.16 Binder驱动学习笔记--------接口简析](https://segmentfault.com/a/1190000014643994)