---
title: Android源码分析 - Binder概述
date: 2022-02-01 13:03:00
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇无源码分析，只对Binder做通信过程和基础架构的介绍**

`Binder`是`Android`中最重要的一种进程间通信机制，基于开源的`OpenBinder`

**George Hoffman**当时任**Be**公司的工程师，他启动了一个名为`OpenBinder`的项目，在**Be**公司被**ParmSource**公司收购后，`OpenBinder`由**Dinnie Hackborn**继续开发，后来成为管理`ParmOS6 Cobalt OS`的进程的基础。在**Hackborn**加入谷歌后，他在`OpenBinder`的基础上开发出了`Android Binder`(以下简称`Binder`)，用来完成`Android`的进程通信。

# 为什么需要学习Binder

作为一名`Android`开发，我们每天都在和`Binder`打交道，虽然可能有的时候不会注意到，譬如：

- `startActivity`的时候，会获取AMS服务，调用AMS服务的`startActivity`方法
- `startActivity`传递的对象为什么需要序列化
- `bindService`为什么回调的是一个`Ibinder`对象
- 多进程应用，各个进程之间如何通信
- `AIDL`的使用
- ...

它们都和`Binder`有着莫切关系，当碰到上面的场景，或者一些疑难问题的时候，理解`Binder`机制是非常有必要的

# 为什么Android选择Binder

这就要从进程间通信开始说起了，我们先看看比较常见的几种进程间通信方式

## 常见进程间通信

### 共享内存

共享内存是进程间通信中最简单的方式之一，共享内存允许两个或更多进程访问同一块内存，当一个进程改变了这块地址中的内容的时候，其它进程都会察觉到这个更改，它的原理如下图所示：

![共享内存](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cb8748d544c4f78999232eee81544af~tplv-k3u1fbpfcp-watermark.image?)

因为共享内存是访问同一块内存，所以数据不需要进行任何复制，是IPC几种方式中最快，性能最好的方式。但相对应的，共享内存未提供同步机制，需要我们手动控制内存间的互斥操作，较容易发生问题。同时共享内存由于能任意的访问和修改内存中的数据，如果有恶意程序去针对某个程序设计代码，很可能导致隐私泄漏或者程序崩溃，所以安全性较差。

### 管道

管道分为命名管道和无名管道，它是以一种特殊的文件作为中间介质，我们称为管道文件，它具有固定的读端和写端，写进程通过写段向管道文件里写入数据，读进程通过读段从读进程中读出数据，构成一条数据传递的流水线，它的原理如下图所示：

![管道](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82253e92afc34a5580e9839ef46a3f9a~tplv-k3u1fbpfcp-watermark.image?)

管道一次通信需要经历2次数据复制（进程A -> 管道文件，管道文件 -> 进程B）。管道的读写分阻塞和非阻塞，管道创建会分配一个缓冲区，而这个缓冲区是有限的，如果传输的数据大小超过缓冲区上限，或者在阻塞模式下没有安排好数据的读写，会出现阻塞的情况。管道所传送的是无格式字节流，这就要求管道的读出方和写入方必须事先约定好数据的格式。

### 消息队列

消息队列是存放在内核中的消息链表，每个消息队列由消息队列标识符表示。消息队列允许多个进程同时读写消息，发送方与接收方要约定好，消息体的数据类型与大小。消息队列克服了信号承载信息量少、管道只能承载无格式字节流等缺点，消息队列一次通信同样需要经历2次数据复制（进程A -> 消息队列，消息队列 -> 进程B），它的原理如下图所示：

![消息队列](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5712f1678b4468ba0f30e8d3f788a96~tplv-k3u1fbpfcp-watermark.image?)

### Socket

`Socket`原本是为了网络设计的，但也可以通过本地回环地址 (`127.0.0.1`) 进行进程间通信，后来在`Socket`的框架上更是发展出一种IPC机制，名叫`UNIX Domain Socket`。`Socket`是一种典型的`C/S`架构，一个`Socket`会拥有两个缓冲区，一读一写，由于发送/接收消息需要将一个`Socket`缓冲区中的内容拷贝至另一个`Socket`缓冲区，所以`Socket`一次通信也是需要经历2次数据复制，它的原理如下图所示：

![Socket](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8affe516eb524fc987bea572f00e2c15~tplv-k3u1fbpfcp-watermark.image?)

## Binder

了解了常见进程间通信的方式，我们再来看一下`Binder`的原理

`Binder`是基于内存映射`mmap`设计实现的，我们需要先了解一下`mmap`的概念

### mmap

`mmap`是一种内存映射的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用`read`,`write`等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。

`Linux`内核不会主动将`mmap`修改后的内容同步到磁盘文件中，有4个时机会触发`mmap`映射同步到磁盘：

- 调用 `msync` 函数主动进行数据同步（主动）
- 调用 `munmap` 函数对文件进行解除映射关系时（主动）
- 进程退出时（被动）
- 系统关机时（被动）

通过这种方式，直接操作映射的这一部分内存，可以避免一些数据复制，从而获得更好的性能

### 原理

一次`Binder` IPC通信的过程分为以下几个步骤：

1. 首先，`Binder`驱动在内核空间中开辟出一个`数据接收缓冲区`
2. 接着，在内核空间中开辟出一个`内核缓冲区`
3. 将`内核缓冲区`与`数据接收缓冲区`建立映射关系
4. 将`数据接收缓冲区`与`接收进程的用户空间地址`建立映射关系
5. 发送方进程通过`copy_from_user`将数据从用户空间复制到`内核缓冲区`
6. 由于`内核缓冲区`与`数据接收缓冲区`有映射关系，同时`数据接收缓冲区`与`接收进程的用户空间地址`有映射关系，所以在接收进程中可以直接获取到这段数据

这样便完成了一次`Binder` IPC通信，它的原理如下图所示：

![Binder](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c882d8e2ebd469786474c8897e4fba9~tplv-k3u1fbpfcp-watermark.image?)

可以看到，通过`mmap`，`Binder`通信时，只需要经历一次数据复制，性能要优于管道/消息队列/socket等方式，在安全性，易用性方面又优于共享内存。鉴于上述原因，`Android`选择了这种折中的IPC方式，来满足系统对稳定性、传输性能和安全性方面的要求

# Binder架构

`Binder`也是一种`C/S`架构，分为`BpBinder`（客户端）和`BBinder`（服务端），他们都派生自`IBinder`。其中`BpBinder`中的p表示proxy，即代理。`BpBinder`通过`transact`来发送事务请求，`BBinder`通过`onTransact`来接收相应的事务

![Ibinder](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e346a6cdb89f4ad5abf61e0a1e0a8046~tplv-k3u1fbpfcp-watermark.image?)

Binder一次通信的时序图如下：

![Binder通信](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d75fba62f67e46bca5ef8c3c73882625~tplv-k3u1fbpfcp-watermark.image?)

Binder采用分层架构设计

![Binder分层架构](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0664b95fe84f40f19021cb9d4ce0ea82~tplv-k3u1fbpfcp-watermark.image?)

# 总结

至此，我们大概了解了一下`Android`选用`Binder`的原因，以及`Binder`的基本结构和通信过程，为之后深入源码层分析`Binder`做了准备

# 参考文献

- [写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)
- [Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)
- [Android Binder原理图解](https://bbs.huaweicloud.com/blogs/308646)
- [Binder和AIDL实例及原理解析](https://www.jianshu.com/p/45563980bf61)
