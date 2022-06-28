---
title: Linux Capabilities机制
date: 2022-01-14 13:43:00
tags: Capabilities
categories: Linux
---

# 介绍

传统的Linux权限控制粒度太粗，以`passwd`命令为例，修改用户密码是需要root权限的，但普通用户应该是能够修改自己密码的才对，这时候Linux就使用了`SUID`、`EUID`机制，使`passwd`进程以它的所有者root权限运行这样就可以以root权限修改密码了

`SUID`机制是有安全隐患的，`passwd`进程只需要修改密码的就可以了，却在整个运行周期内获得了root权限，一旦出现漏洞，很有可能会被利用

所以，Linux内核在2.2后引入了`Capabilities`机制，细粒度化了权限控制，可以做到按需授权

这里是文档：https://man7.org/linux/man-pages/man7/capabilities.7.html

# 如何使用

首先，`Capabilities`是有一个集合的概念的，即一个进程或可执行文件，它可以拥有哪些特权的集合

## 可执行文件

可执行文件有三种Capabilities集合：

### Permitted

当文件执行时，这个集合的内容会被添加到进程的Permitted集合中

### Inheritable

当文件执行后，这个集合会与进程的`Inheritable`集合做位与操作(&)，以确定进程在执行`execve`函数后哪些`capabilites`可以被继承

### Effective

这不是一个集合，而是一个位(bit)，如果此bit设为1，则`Permitted`集合中新增的`capabilites`会在执行`execve`函数后添加到进程的`Effective`集合中

### 命令

- 设置`capabilites`

```shell
setcap [capability,capability,...]+[ep] [文件]

# or

setcap [capability+ep capability+ep ...] [文件]
```

`capability`就是某个特权值，+ep代表加入`Effective`和`Permitted`集合中

- 获取`capabilites`

```shell
getcap [文件]
```

## 线程（进程）

线程（进程）有五种Capabilities集合：

### Permitted

这个集合定义了线程所能够拥有的特权的上限，是`Inheritable`和`Effective`集合的的超集

### Inheritable

包含了当执行`execve` 函数时，能够被新的可执行文件继承的`capabilities`（执行`execve` 函数后会被添加到`Permitted`集合中）

### Effective

内核检查特权操作时，实际检查的集合（可以通过执行操作前增/删`Effective`中的`capabilities`，以达到临时开/关权限的功能）

### Bounding (内核2.6.25以后)

这个集合是 `Inheritable` 集合的超集，如果某个`capability`不在`Bounding`集合中，即使它在`Permitted`集合中，该线程也不能将该`capability`添加到它的`Inheritable`集合中，该集合在`execve`后不可再添加`capabilities`

### Ambient (内核4.3以后)

这个集合是`Permitted`和`Inheritable`的子集，当`Permitted`和`Inheritable`删除某个`capability`时，也会自动删除该集合中对应的`capability`，子进程会自动继承这个集合中的`capabilities`，子进程的`Permitted`、`Effective`和`Ambient`都会拥有这些`capabilities`

### 函数

#### capset

原型：

```c
int capset(cap_user_header_t hdrp, const cap_user_data_t datap);
```

文档：https://linux.die.net/man/2/capset

#### capget

原型：

```c
int capget(cap_user_header_t hdrp , cap_user_data_t datap);
```

文档：https://linux.die.net/man/2/capget

#### 结构体

```c
typedef struct __user_cap_header_struct {
    __u32 version;
    int pid;
 } *cap_user_header_t;

typedef struct __user_cap_data_struct {
    __u32 effective;
    __u32 permitted;
    __u32 inheritable;
 } *cap_user_data_t;
```

# 计算公式

我们用 `P` 代表执行 `execve()` 前线程的 capabilities，`P'` 代表执行 `execve()` 后线程的 `capabilities`，`F` 代表可执行文件的 `capabilities`，那么：

> P'(ambient) = (file is privileged) ? 0 : P(ambient)
>
> P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & P(bounding))) | P'(ambient)
>
> P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)
>
> P'(inheritable) = P(inheritable) [i.e., unchanged]
>
> P'(bounding) = P(bounding) [i.e., unchanged]

我们一条一条来解释：

- 如果用户是 root 用户，那么执行 `execve()` 后线程的 `Ambient` 集合是空集；如果是普通用户，那么执行 `execve()` 后线程的 `Ambient` 集合将会继承执行 `execve()` 前线程的 `Ambient` 集合。

- 执行 `execve()` 前线程的 `Inheritable` 集合与可执行文件的 `Inheritable` 集合取交集，会被添加到执行 `execve()` 后线程的 `Permitted` 集合；可执行文件的 capability bounding 集合与可执行文件的 `Permitted` 集合取交集，也会被添加到执行 `execve()` 后线程的 `Permitted` 集合；同时执行 `execve()` 后线程的 `Ambient` 集合中的 capabilities 会被自动添加到该线程的 `Permitted` 集合中。

- 如果可执行文件开启了 Effective 标志位，那么在执行完 `execve()` 后，线程 `Permitted` 集合中的 capabilities 会自动添加到它的 `Effective` 集合中。

- 执行 `execve()` 前线程的 `Inheritable` 集合会继承给执行 `execve()` 后线程的 `Inheritable` 集合。

这里有几点需要着重强调：

1.  上面的公式是针对系统调用 `execve()` 的，如果是 `fork()`，那么子线程的 capabilities 信息完全复制父进程的 capabilities 信息。

2.  可执行文件的 `Inheritable` 集合与线程的 `Inheritable` 集合并没有什么关系，可执行文件 `Inheritable` 集合中的 capabilities 不会被添加到执行 `execve()` 后线程的 `Inheritable` 集合中。如果想让新线程的 `Inheritable` 集合包含某个 capability，只能通过 `capset()` 将该 capability 添加到当前线程的 `Inheritable` 集合中（因为 P'(inheritable) = P(inheritable)）。

3.  如果想让当前线程 `Inheritable` 集合中的 capabilities 传递给新的可执行文件，该文件的 `Inheritable` 集合中也必须包含这些 capabilities（因为 P'(permitted)   = (P(inheritable) & F(inheritable))|…）。

4.  将当前线程的 capabilities 传递给新的可执行文件时，仅仅只是传递给新线程的 `Permitted` 集合。如果想让其生效，新线程必须通过 `capset()` 将 capabilities 添加到 `Effective` 集合中。或者开启新的可执行文件的 Effective 标志位（因为 P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)）。

5.  在没有 `Ambient` 集合之前，如果某个脚本不能调用 `capset()`，但想让脚本中的线程都能获得该脚本的 `Permitted` 集合中的 capabilities，只能将 `Permitted` 集合中的 capabilities 添加到 `Inheritable` 集合中（P'(permitted)  = P(inheritable) & F(inheritable)|…），同时开启 Effective 标志位（P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)）。有 有 `Ambient` 集合之后，事情就变得简单多了，后续的文章会详细解释。

6.  如果某个 UID 非零（普通用户）的线程执行了 `execve()`，那么 `Permitted` 和 `Effective` 集合中的 capabilities 都会被清空。

7.  从 root 用户切换到普通用户，那么 `Permitted` 和 `Effective` 集合中的 capabilities 都会被清空，除非设置了 SECBIT_KEEP_CAPS 或者更宽泛的 SECBIT_NO_SETUID_FIXUP。

# 附录

## Capabilities表

| Capability                   | 描述                                          |
| ---------------------------- | ------------------------------------------- |
| CAP_AUDIT_CONTROL            | 启用和禁用内核审计；改变审计过滤规则；检索审计状态和过滤规则   |
| CAP_AUDIT_READ               | 允许通过 multicast netlink 套接字读取审计日志            |
| CAP_AUDIT_WRITE              | 将记录写入内核审计日志                                  |
| CAP_BLOCK_SUSPEND            | 使用可以阻止系统挂起的特性                               |
| CAP_BPF (5.8)                | 从CAP_SYS_ADMIN分离一部分BFP功能，控制了一些BPF特定的操作，包括创建BPF maps、使用一些高级的BPF程序功能、访问BPF type format（BTF）数据等                   |
| CAP_CHECKPOINT_RESTORE (5.9) | 允许更新/proc/sys/kernel/ns_last_pid，使用set_tid特性，读其他进程的/proc/[pid]/map_files                                                        |
| CAP_CHOWN                    | 修改文件所有者的权限                                    |
| CAP_DAC_OVERRIDE             | 忽略文件的 DAC 访问限制                                 |
| CAP_DAC_READ_SEARCH          | 忽略文件读及目录搜索的 DAC 访问限制                       |
| CAP_FOWNER                   | 忽略文件属主 ID 必须和进程用户 ID 相匹配的限制             |
| CAP_FSETID                   | 允许设置文件的 setuid 位                                |
| CAP_IPC_LOCK                 | 允许锁定共享内存片段                                    |
| CAP_IPC_OWNER                | 忽略 IPC 所有权检查                                    |
| CAP_KILL                     | 允许对不属于自己的进程发送信号                            |
| CAP_LEASE                    | 允许修改文件锁的 FL_LEASE 标志                          |
| CAP_LINUX_IMMUTABLE          | 允许修改文件的 IMMUTABLE 和 APPEND 属性标志              |
| CAP_MAC_ADMIN                | 允许 MAC 配置或状态更改                                 |
| CAP_MAC_OVERRIDE             | 忽略文件的 DAC 访问限制                                 |
| CAP_MKNOD                    | 允许使用 mknod() 系统调用                               |
| CAP_NET_ADMIN                | 允许执行网络管理任务                                    |
| CAP_NET_BIND_SERVICE         | 允许绑定到小于 1024 的端口                              |
| CAP_NET_BROADCAST            | 允许网络广播和多播访问                                   |
| CAP_NET_RAW                  | 允许使用原始套接字                                      |
| CAP_PERFMON (5.8)            | 管理性能监控task                                       |
| CAP_SETGID                   | 允许改变进程的 GID                                     |
| CAP_SETFCAP                  | 允许为文件设置任意的 capabilities                       |
| CAP_SETPCAP                  | 允许设置其他进程的 capabilities                         |
| CAP_SETUID                   | 允许改变进程的 UID                                     |
| CAP_SYS_ADMIN                | 允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等     |
| CAP_SYS_BOOT                 | 允许重新启动系统                                        |
| CAP_SYS_CHROOT               | 允许使用 chroot() 系统调用                              |
| CAP_SYS_MODULE               | 允许插入和删除内核模块                                   |
| CAP_SYS_NICE                 | 允许提升优先级及设置其他进程的优先级                       |
| CAP_SYS_PACCT                | 允许执行进程的 BSD 式审计                               |
| CAP_SYS_PTRACE               | 允许跟踪任何进程                                        |
| CAP_SYS_RAWIO                | 允许直接访问 /devport、/dev/mem、/dev/kmem 及原始块设备   |
| CAP_SYS_RESOURCE             | 忽略资源限制                                           |
| CAP_SYS_TIME                 | 允许改变系统时钟                                        |
| CAP_SYS_TTY_CONFIG           | 允许配置 TTY 设备                                      |
| CAP_SYSLOG                   | 允许使用 syslog() 系统调用                              |
| CAP_WAKE_ALARM               | 允许触发一些能唤醒系统的东西(比如 CLOCK_BOOTTIME_ALARM 计时器)|

## 参考文献

[Linux Capabilities 简介](https://www.cnblogs.com/sparkdev/p/11417781.html)

[Linux的capabilities机制](http://rk700.github.io/2016/10/26/linux-capabilities/)

[Linux Capabilities 入门教程：概念篇](https://fuckcloudnative.io/posts/linux-capabilities-why-they-exist-and-how-they-work/)