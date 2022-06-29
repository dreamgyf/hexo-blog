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

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E6%AD%A3%E5%B8%B8mock.png)

而在kotlin单元测试中，我们却无法找到这个class

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E9%94%99%E8%AF%AFmock.png)

# 确定路线

我们先建立一个文件来写一个顶层函数，再建立一个单元测试类去测试它: 

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E5%BB%BA%E7%AB%8B%E6%B5%8B%E8%AF%95%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0.png)

其实从上文中已经可以看出，kotlin的顶层函数在编译之后实际上就变成了一个被class包起来的static方法。对此，我们可以简单验证一下:

在Android Studio中点击菜单中的Tools->Kotlin->Show Kotlin ByteCode，会弹出对应类的字节码，再点击Decompile按钮，我们会看到确实被编译成了一个类中的静态方法

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E5%8F%8D%E7%BC%96%E8%AF%91%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0.png)

确定了这一点后，我们只需要在kotlin中拿到这个顶层函数的所属类，就可以像java里一样使用mockStatic来模拟了。

# 分析过程

既然涉及到了运行时类型分析，自然而然就想到了**反射**，我们先引入kotlin的反射库

`implementation "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"`

其实我对kotlin的反射并不熟悉，去文档里查阅了一下发现了`::sampleTopFun`这种写法，它的返回值为一个叫`KFunction`的接口类，我们先看看它有哪些方法可以供我们调用

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_kFunction%E7%9A%84%E6%96%B9%E6%B3%95.png)

从字面上看好像没有什么方法和我们的需求有关，怎么办呢？那我们再看一下它的实现类吧，说不定会有一些私有变量保存了我们需要的信息。

那么怎么找到它的实现类呢？直接分析源码错综复杂的关系是很耗时且低效的，这里我采取了了一种取巧的方法，利用Android Studio的Debug功能:

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_kFunction%E5%AE%9E%E7%8E%B0%E7%B1%BB.png)

和预料的不同，为什么这里拿到的类型是这么个奇葩玩意儿呢？我们看一下这个文件的字节码

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_kFunction%E5%AD%97%E8%8A%82%E7%A0%81.png)

再往下看

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_kFunction%E5%AD%97%E8%8A%82%E7%A0%812.png)

我们发现，这个奇葩的类型是在kotlin编译后自动生成的，它继承自`FunctionReference`，同时，在Debugger里，我们获得了一个重要的信息: `KFunctionImpl`

根据名字猜测，它应该才是`KFunction`真正功能实现的地方，我们将它的信息展开

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_kFunction%E7%9C%9F%E6%AD%A3%E5%AE%9E%E7%8E%B0.png)

可以发现，我们已经找到我们想要的那个类了，只要拿到它，后续的mock工作就很简单了~

# 开始Mock

根据上文，我们已经得知了我们需要获取的`jClass`的路径

我们先从`FunctionReference`去获取被`reflected`引用的`KFunctionImpl`，这个`reflected`实际是被`FunctionReference`继承的`CallableReference`中的一个变量，在`FunctionReference`提供了一个`getReflected`方法，我们通过反射调用这个方法即可得到这个对象，当然，我们也可以通过反射Field获得它，但注意到`getReflected`方法处理了一些空对象的情况，为了保险起见，我们还是采取反射调用`getReflected`的方法获取`KFunctionImpl`

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E8%8E%B7%E5%8F%96kFunction.png)

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E8%8E%B7%E5%8F%96kFunction2.png)

反射调用`getReflected`方法获取`KFunctionImpl`

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E8%8E%B7%E5%8F%96kFunction3.png)

第一步没问题，接下来开始反射获取`container`

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E8%8E%B7%E5%8F%96container.png)

第二步也没什么问题，接下来就是反射获取`jClass`了

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_%E8%8E%B7%E5%8F%96jclass.png)

ok，一切正常，接下来和java一样，让我们试试用这个我们获取到的类mockStatic吧

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android-Kotlin%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%A6%82%E4%BD%95%E9%85%8D%E5%90%88Mockito%E6%A8%A1%E6%8B%9F%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0_mockStatic.png)

可以看到，测试成功通过，至此，我们成功解决了Mockito模拟顶层函数的问题。为了方便使用，可以将以上代码封装成一个函数，这里就不再赘述了。