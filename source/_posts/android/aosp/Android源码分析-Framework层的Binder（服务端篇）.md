---
title: Android源码分析 - Framework层的Binder（服务端篇）
date: 2022-07-01 18:48:28
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以`aosp`分支`android-11.0.0_r25`作为基础解析**

我们在上一片文章[Android源码分析 - Framework层的Binder（客户端篇）](https://juejin.cn/post/7113760814409973790)中，分析了客户端是怎么向服务端通过`binder`驱动发起请求，然后再接收服务端的返回的。本篇文章，我们将会以服务端的视角，分析服务端是怎么通过`binder`驱动接收客户端的请求，处理，然后再返回给客户端的。

# ServiceManager

上篇文章我们是以`ServiceManager`作为服务端分析的，本篇文章我们还是围绕着它来做分析，它也是一个比较特殊的服务端，我们正好可以顺便分析一下它是怎么成为`binder`驱动的`context_manager`的

## 进程启动

`ServiceManager`是在独立的进程中运行的，它是由`init`进程从`rc`文件中解析并启动的，路径为`frameworks/native/cmds/servicemanager/servicemanager.rc`

```
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    onrestart restart thermalservice
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```