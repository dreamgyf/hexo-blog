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

我们在上一片文章[Android源码分析 - Framework层的Binder（客户端篇）](https://juejin.cn/post/7113760814409973790)中，分析了客户端是怎么向服务端通过`binder`驱动发起请求，然后再接收服务端的返回的，本篇文章，我们将会以服务端的视角，分析服务端是怎么通过`binder`驱动接收客户端的请求，处理，然后再返回给客户端的