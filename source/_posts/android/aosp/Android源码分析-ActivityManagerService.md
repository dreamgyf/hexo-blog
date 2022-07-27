---
title: Android源码分析-ActivityManagerService
date: 2022-07-26 18:03:08
tags: 
- Android源码
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

作为一名`Android`开发，我们最熟悉并且最常打交道的当然非四大组件中的`Activity`莫属，这次我们就来讲讲`ActivityManagerService`这个`Android`系统核心服务究竟扮演了一个怎样的角色，提供了哪些功能

# 简介

