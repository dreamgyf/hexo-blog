---
title: B站Android面试小记
date: 2022-03-22 19:09:00
tags: 
- 面试
categories: 
- [面试]
- [Android, 面试]
---

# 起因

看着我同学最近也在到处投简历，我想着我也投一下看看行情，于是在**2022-02-28**号，我向B站投出了第一封简历，说实话当初只是想练练手，没想到最后接了B站的offer，也是造化弄人了

# 一面

技术面，45min左右，基本围绕你简历上写的亮点和你的工作经历展开

- 自我介绍

- 你在项目中负责什么

- 用过什么设计模式，或在`Android`中常常会碰见的设计模式

    单例模式，策略模式，责任链模式（问了一下使用场景），工厂模式等
    
    `Android`中的观察者模式，适配器模式等

- 有没有做过什么比较有难度的模块

    `camera2`，自定义照片裁剪`View`

- 你对自定义`View`有什么了解

    回答了一些`Path`绘制以及触摸事件的处理

- `Android`动画

    属性动画，`ObjectAnimator`

- 多线程并发（锁、信号量、`syncnorized`），`syncnorized`对象和class有什么区别

- `ConcurrentHashMap`线程安全的原理

    1.8之前用的分段式锁，1.8之后用的`synchronized`，至于具体的细节没有答上来，因为确实也没看过这边源码

- `jni`，如何定位`jni`崩溃

    这个我当时回答的是打log，因为项目中用到`jni`的地方确实不多，当然`jni`也是可以断点调试的

- 你所开发的应用有多进程吗？进程间是怎么通信的

    这个我当时只回答了`mmap`，稍微聊了一下`mmap`原理和`binder`性能对比，后来复盘想起来项目中用到的`Broadcast`和`aidl binder`通信都没有回答

- `Webview`和`native`怎么交互的

    - `onUrlLoading`拦截`Schema`
    
    - 注册js方法（`addJavascriptInterface`）

- `Android`编译打包过程

    aapt -> class -> dex -> 签名

- 插桩

    ASM插桩，字节码操作

- 性能监控

    因为我之前做过一个性能监控库，`cpu`和`mem`使用`TOP`命令解析，`Anr`通过给`MainLooper`设置`Printer`

- `LeakCanary`原理

    `WeakReference` + `ReferenceQueue`，加了一些改进点：`new`一个弱引用的`Object`，等这个`Object`确认被回收后再确认`Activity`是否正常被回收

- `Jetpack Compose`

    稍微谈了一下看法，是否在项目中用过

- 算法题：最长公共前缀

    LeetCode 14题，easy难度：https://leetcode-cn.com/problems/longest-common-prefix/
    
# 二面

一面结束后5min左右，B站HR就给我打电话过来约了二面

二面也是技术面，20min左右，因为是晚上8点面的，估计人家急着想下班（笑）

- 自我介绍

- 工作职责

- 工作中有什么亮点

    - 拍照裁剪业务

    - 单元测试库
    
    - 性能监控
    
    - 内存泄漏检测

- 单元测试的库是怎么做的

    基于`Mockito`和`Robolectric`:
    
    1. 封装了一个反射库用来方便测试
    2. 做了一个`AutoCloser`类用来自动关闭释放mock的资源，这里提到了使用`MockedStatic`，如果在使用完后没有释放，那在下一次使用到同一个类的`MockedStatic`的时候会报错，这里我自定义了一个注解`@MockedStatic`用来自动mock和释放资源
    3. 针对kotlin做了一些mock工具，比如说顶层函数的mock（这个在我以前的文章[Android-Kotlin单元测试之 如何配合Mockito模拟顶层函数](https://juejin.cn/post/6932738373522030600)中介绍过）
    
- 开发模式（流程规范）：

    开发规范参考了阿里的`Java`规范和`Android`规范，选取了一些比较重要的条例和一些自己长时间开发的经验做成了一篇文档
    
- 崩溃率的优化，做了哪些事情

    感觉这里没答好，有点答非所问的意思，我就说了说目前处理bug的一个流程，没有谈到怎么解决一个bug
    
- 数据打点是怎么做的

    我们用的是神策第三方服务
    
- 内存泄漏工具是怎么做的

    这部分同一面`LeakCanary`原理
    
- 看你之前做过一个`MQTT`协议的客户端，是出于个人兴趣吗

    是的，当时是想要做一个`IM`应用
    
- 在项目中有遇到需要3D渲染展示的内容吗

    目前没有
    
- 两个`Activity`跳转时方法执行的顺序

    一个`Activity`创建是：onCreate -> onStart -> onResume（之后便在屏幕上显示了）
    
    假设从`A Activity`跳转到`B Activity`：A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop
    
    从`B`返回到`A`：B.onPause -> A.onRestart -> A.onResume -> B.onStop -> B.onDestory
    
- 两个`Activity`传递数据可以通过什么方式

    - `Intent`
    
    - 如果是同一个进程的话，可以用全局变量或者单例等
    
    - `SharedPreference`
    
    - 文件
    
- 什么时候使用`Service`

    后台任务，比如说后台播放音乐等，这里提了一下`IntentService`是开了一个子线程的
    
- `Service`怎么启动，怎么停止

    - `startService` <---> `stopService`
    
    - `bindService` <---> `unbindService`
    
- 包体积优化

    清理资源（字体、图片、代码等）

# HR面

二面结束后过了2-3天，HR发微信过来恭喜我进入下一轮面试，我问她接下来是还有三面和HR面吗，她回答我说后面就直接是HR面了，说实话我还是挺惊讶的

HR面15min左右，大概就问了一下，为什么要从上家公司离职，我们是一个新部门，处于项目初期，有什么看法之类的，然后问了一下目前的薪资和期望薪资，over~

# 总结

说实话感觉这次面试太简单了，有点白瞎了我准备了那么多，还做了查漏补缺 ㄟ( ▔, ▔ )ㄏ ，最后祝大家都能找到心仪的工作 (๑•̀ㅂ•́)و✧