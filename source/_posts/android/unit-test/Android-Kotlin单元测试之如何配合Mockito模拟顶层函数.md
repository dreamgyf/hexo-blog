---
title: Android-Kotlin单元测试之 如何配合Mockito模拟顶层函数
date: 2021-02-24 16:05:00
tags: 单元测试
categories: 
- [Android, 单元测试]
---

# 简介

随着Kotlin语言在Android开发中越来越流行，自然也会遇到各种各样的问题。

本篇主要是针对我个人在Android单元测试Kotlin类时遇到的一些问题的思考和解决方案。

# 遇到的问题

我们都知道Kotlin给开发者提供了很多语法糖，其中之一就是**顶层函数**，我们可以直接把函数放在代码文件的顶层，让它不从属于任何类。

它的使用很简单，直接在kotlin代码的任意位置直接当作一个普通函数调用就行了，而在java中，需要像使用静态方法一样，以**文件名+Kt**为类名调用 (默认配置)

在java单元测试中，如果想mock这个顶层函数，只需要像对待一个静态方法一样，使用mockStatic方法即可

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e58aa52258d845818fa65ee8b192ab4a~tplv-k3u1fbpfcp-watermark.image)

而在kotlin单元测试中，我们却无法找到这个class

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/996122adcf9148e598cd9eb18ae666b8~tplv-k3u1fbpfcp-watermark.image)

# 确定路线

我们先建立一个文件来写一个顶层函数，再建立一个单元测试类去测试它: 

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87f9a4f739664cfc8a20321c77a0056d~tplv-k3u1fbpfcp-watermark.image)

其实从上文中已经可以看出，kotlin的顶层函数在编译之后实际上就变成了一个被class包起来的static方法。对此，我们可以简单验证一下:

在Android Studio中点击菜单中的Tools->Kotlin->Show Kotlin ByteCode，会弹出对应类的字节码，再点击Decompile按钮，我们会看到确实被编译成了一个类中的静态方法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98636e1fb7494a4d9c7785f55fb3112a~tplv-k3u1fbpfcp-watermark.image)

确定了这一点后，我们只需要在kotlin中拿到这个顶层函数的所属类，就可以像java里一样使用mockStatic来模拟了。

# 分析过程

既然涉及到了运行时类型分析，自然而然就想到了**反射**，我们先引入kotlin的反射库

`implementation "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"`

其实我对kotlin的反射并不熟悉，去文档里查阅了一下发现了`::sampleTopFun`这种写法，它的返回值为一个叫`KFunction`的接口类，我们先看看它有哪些方法可以供我们调用

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a8cdc54fe5948a2a460aab7214c946f~tplv-k3u1fbpfcp-watermark.image)

从字面上看好像没有什么方法和我们的需求有关，怎么办呢？那我们再看一下它的实现类吧，说不定会有一些私有变量保存了我们需要的信息。

那么怎么找到它的实现类呢？直接分析源码错综复杂的关系是很耗时且低效的，这里我采取了了一种取巧的方法，利用Android Studio的Debug功能:

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e463b2d5f8114a0ca563c2038f5c82a8~tplv-k3u1fbpfcp-watermark.image)

和预料的不同，为什么这里拿到的类型是这么个奇葩玩意儿呢？我们看一下这个文件的字节码

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4ed07e6a0b54c4382d05b52eda7dfaf~tplv-k3u1fbpfcp-watermark.image)

再往下看

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2c7b9d2f5c64371b5707235ad950909~tplv-k3u1fbpfcp-watermark.image)

我们发现，这个奇葩的类型是在kotlin编译后自动生成的，它继承自`FunctionReference`，同时，在Debugger里，我们获得了一个重要的信息: `KFunctionImpl`

根据名字猜测，它应该才是`KFunction`真正功能实现的地方，我们将它的信息展开

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fdb9a55a88149158b9626a1ad0d6eac~tplv-k3u1fbpfcp-watermark.image)

可以发现，我们已经找到我们想要的那个类了，只要拿到它，后续的mock工作就很简单了~

# 开始Mock

根据上文，我们已经得知了我们需要获取的`jClass`的路径

我们先从`FunctionReference`去获取被`reflected`引用的`KFunctionImpl`，这个`reflected`实际是被`FunctionReference`继承的`CallableReference`中的一个变量，在`FunctionReference`提供了一个`getReflected`方法，我们通过反射调用这个方法即可得到这个对象，当然，我们也可以通过反射Field获得它，但注意到`getReflected`方法处理了一些空对象的情况，为了保险起见，我们还是采取反射调用`getReflected`的方法获取`KFunctionImpl`

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e6a8d534d264f9fb035723561a33871~tplv-k3u1fbpfcp-watermark.image)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a562f4767a654c3c8d1e341e77366501~tplv-k3u1fbpfcp-watermark.image)

反射调用`getReflected`方法获取`KFunctionImpl`

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/201ebeb8cd6143afb8a2986cde22d66c~tplv-k3u1fbpfcp-watermark.image)

第一步没问题，接下来开始反射获取`container`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f0a7de899674827a890bf68992cb549~tplv-k3u1fbpfcp-watermark.image)

第二步也没什么问题，接下来就是反射获取`jClass`了

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dcce20dc0d240af831842e9703abda1~tplv-k3u1fbpfcp-watermark.image)

ok，一切正常，接下来和java一样，让我们试试用这个我们获取到的类mockStatic吧

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b32d0b0659f4129963bfde323bbda19~tplv-k3u1fbpfcp-watermark.image)

可以看到，测试成功通过，至此，我们成功解决了Mockito模拟顶层函数的问题。为了方便使用，可以将以上代码封装成一个函数，这里就不再赘述了。