---
title: Android源码分析 - Activity启动流程（上）
date: 2022-12-19 21:49:36
tags: 
- Android源码
- ActivityThread
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, ActivityThread]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

上一篇文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128) 中，我们分析了`App`进程的启动过程，包括`Application`是怎么创建并执行`onCreate`方法的，本篇文章我们将会继续分析`App`进程启动、`Application`创建完成后，`Activity`是如何启动的
