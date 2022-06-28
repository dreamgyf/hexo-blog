---
title: AOSP的编译及刷机
date: 2021-12-18 14:13:00
tags: AOSP
categories: 
- [Android, AOSP]
---

# 简介

众所周知，Android是开源的，AOSP（Android Open Source Project）为Android开源项目的缩写。作为一名Android开发，掌握Android系统的工作机制是技术成长中的必经之路，第一步就是自己编译Android系统。

# 准备工作

- 一台可以解BL锁（BootLoader），并且厂商提供了硬件驱动的设备，这里推荐使用Google亲儿子手机（Nexus、Pixel系列），可以解BL锁，Google官方会提供硬件驱动，并且AOSP里会提供对应机型的配置
- 一块剩余空间至少大于300GB的硬盘（Android11源码-150GB左右，编译产物-150GB左右）
- 系统最好为Linux，MacOS也可（Windows可以用WSL）
- 内存至少要8GB，过小的内存会导致生成build.ninja文件失败（官方要求至少16GB，我实测8GB也可）

这里是Google官方的推荐要求：<https://source.android.com/setup/build/requirements?hl=zh-cN>

# 环境搭建

参考文档：<https://source.android.com/source/initializing?hl=zh-cn>

主要就是下载各种编译工具，像jdk，gcc，g++等，还有各种动态库以及辅助工具

注：此文档中部分环境安装有误，缺失了一些必要的库安装，可能会编译中途报错，可以参考下文的环境安装，如果编译还是出现了依赖缺失，安装好继续编译即可

## 安装JDK

以Ubuntu系统为例：

```shell
sudo apt-get update
sudo apt-get install openjdk-11-jdk
```

注：现在AOSP编译要求JDK版本>=9

## 安装其他程序包

```shell
sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip libncurses5
```

注：官方文档中缺失了libncurses5，会导致编译中途找不到libncurses.so.5库

# 下载源码

Android源码是由非常多的Git仓库组成的，为了可以统一管理这么多个Git仓库，Google出了一款工具，叫Repo

参考文档：<https://source.android.com/source/downloading?hl=zh-cn>

因为Google在国内访问的问题，建议使用镜像下载源码，下面提供几个镜像地址：

-   清华大学

https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest

-   中科大

git://mirrors.ustc.edu.cn/aosp/platform/manifest

repo init的时候可以指定分支：<https://source.android.com/setup/start/build-numbers?hl=zh-cn#source-code-tags-and-builds> 在这里可以找到对应系统分支所支持的设备，比如说我的设备是Pixel2，在这张表上可以看到android-11.0.0_r25这个分支下的代码支持我的设备，所以可以执行以下命令：

```shell
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-11.0.0_r25
```

然后开始进行同步：

```shell
repo sync -j8 #j8代表使用8个线程
```

AOSP代码下载是个漫长的过程，需要耐心等待

# 下载驱动

在<https://developers.google.com/android/drivers?hl=zh-cn>这个网站可以找到Nexus、Pixel系列的驱动，要注意每个驱动后面会有一串代号，需要和你下载的AOSP源码的build号相对应

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42ca0ef81bc34673ae00b33fe63e1ee6~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b485d8024e9474388baa05de3424e85~tplv-k3u1fbpfcp-zoom-1.image)

将他们解压后会得到两个shell文件

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46973e3cfcb6498f98133564e4273b2e~tplv-k3u1fbpfcp-zoom-1.image)

将他们复制到下载好的aosp源码的根目录

注：网上很多教程说终端要选用bash不要使用zsh，我亲测使用zsh没有问题，如果在编译过程中出现问题，可以尝试切换shell

1. 先将shell切换到aosp源码根目录
2. 执行两个解压出来的驱动shell，记得要同意License

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e753f0f11a4c4ed7bae08f2f3daaef96~tplv-k3u1fbpfcp-zoom-1.image)

3. 执行source build/envsetup.sh，这会向shell中写入一些环境变量
4. 先make clean一下
5. 使用lunch命令选择构建目标

这里是该命令的规则：<https://source.android.com/setup/build/building?hl=zh-cn#choose-a-target>

```shell
lunch aosp_walleye-userdebug
```

后面跟随的的参数可以在这里找到：<https://source.android.com/setup/build/running?hl=zh-cn#selecting-device-build>![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6686328101e24a0291ee12093e3874df~tplv-k3u1fbpfcp-zoom-1.image)

你也可以在lunch后不加参数，这样会弹出一个菜单提示您选择目标

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/941189e204764e22bf33010b737dce75~tplv-k3u1fbpfcp-zoom-1.image)

指定完成后会弹出这样一个信息提示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d525adf44b446cd87fba0c6e759c588~tplv-k3u1fbpfcp-zoom-1.image)

# 开始编译

构建部分的文档在这里：<https://source.android.com/setup/build/building?hl=zh-cn#build-the-code>
如果是初次编译，我们就直接使用`m`命令就可以了

```shell
m -j8 #开启8线程编译
```

注意事项：

- 现在直接使用`make`命令会提示`Calling make directly is no longer supported`然后退出编译，所以使用`m`命令替代`make`
- 不能使用root账号编译

# 刷机

1. 先将手机的BL锁解开（每个机型都不同，网上会有对应的教程），进入fastboot模式\
2. 配置fastboot工具（现在Google好像推出了在线刷写工具<https://flash.android.com/>，可以尝试使用），可以在aosp目录下通过make fastboot命令编译出来，也可以直接从网上下载：<https://developer.android.com/studio/releases/platform-tools>
3. 进入编译后产生的镜像的目录..../aosp/out/target/product/walleye(这个是你机型的代号，每种机器都不一样)
4. 执行命令

```shell
fastboot flashall -w
```

5. 重启即可看到，我们编译的Android系统已经运行到了手机上

```shell
fastboot reboot #重启命令
```

# 常见问题

## MacOS上找不到SDK

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99828f96b9e146db9c8eef93de19ce8f~tplv-k3u1fbpfcp-zoom-1.image)

去这里<https://github.com/phracker/MacOSX-SDKs/releases>下载对应版本的sdk，然后将它放到/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs目录下，然后重新编译

除此之外，也可以在Finder中查看

/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs

这个目录下存在哪个版本的sdk，确定后去修改..../aosp/build/soong/cc/config/x86_darwin_host.go文件，在darwinSupportedSdkVersions这个数组中加上你使用的sdk的版本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b622d2da89c94494a01022ec9c400a4b~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ee9f917b2724da79f35bd097169d434~tplv-k3u1fbpfcp-zoom-1.image)

保存后重新编译，这个方式可能当前编译脚本不支持你所用的sdk，可能会编译报错，所以还是推荐使用第一种方式

## too many open files

在Linux系统下有打开文件数的限制，可以使用以下命令设置最大可打开文件数

```shell
# ulimit -a 可以查看当前限制
ulimit -n 2048
```