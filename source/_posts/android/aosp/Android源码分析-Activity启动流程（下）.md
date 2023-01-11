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

# 两种路径启动Activity

我们在 [Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999) 的末尾分析过，`Activity`的启动存在两条路径

## 启动进程，然后启动Activity

这条路径就是我们在上一篇文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128) 中分析了前半部分：启动进程

当进程启动后，会执行到`AMS.attachApplicationLocked`方法，在这个方法的最后，会有一段代码检查是否有`Activity`等待启动

```java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    ...

    // See if the top visible activity is waiting to run in this process...
    //检查是否有Activity等待启动
    if (normalMode) {
        try {
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
        } catch (Exception e) {
            badApp = true;
        }
    }

    ...

    return true;
}
```

然后调用`ActivityTaskManagerInternal.attachApplication`，`ActivityTaskManagerInternal`是一个抽象类，被`ATMS`的内部类`LocalService`实现

```java
public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
    synchronized (mGlobalLockWithoutBoost) {
        try {
            return mRootWindowContainer.attachApplication(wpc);
        } finally {
            ...
        }
    }
}
```

接着调用`RootWindowContainer.attachApplication`

```java
boolean attachApplication(WindowProcessController app) throws RemoteException {
    boolean didSomething = false;
    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        mTmpRemoteException = null;
        mTmpBoolean = false; // Set to true if an activity was started.

        final DisplayContent display = getChildAt(displayNdx);
        for (int areaNdx = display.getTaskDisplayAreaCount() - 1; areaNdx >= 0; --areaNdx) {
            final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(areaNdx);
            for (int taskNdx = taskDisplayArea.getStackCount() - 1; taskNdx >= 0; --taskNdx) {
                final ActivityStack rootTask = taskDisplayArea.getStackAt(taskNdx);
                if (rootTask.getVisibility(null /*starting*/) == STACK_VISIBILITY_INVISIBLE) {
                    break;
                }
                //遍历ActivityStack下的所有ActivityRecord，
                //以其为参数调用startActivityForAttachedApplicationIfNeeded方法
                final PooledFunction c = PooledLambda.obtainFunction(
                        RootWindowContainer::startActivityForAttachedApplicationIfNeeded, this,
                        PooledLambda.__(ActivityRecord.class), app,
                        rootTask.topRunningActivity());
                rootTask.forAllActivities(c);
                c.recycle();
                if (mTmpRemoteException != null) {
                    throw mTmpRemoteException;
                }
            }
        }
        didSomething |= mTmpBoolean;
    }
    if (!didSomething) {
        ensureActivitiesVisible(null, 0, false /* preserve_windows */);
    }
    return didSomething;
}
```

这里有两层`for`循环，可以看出来，这个方法遍历了所有可见的`ActivityStack`，然后再对每个可见的`ActivityStack`进行操作

其中，`PooledLambda`这个类采用了池化技术，用于构造可回收复用的匿名函数，这里`PooledLambda.obtainFunction`方法得到的结果是一个`Function<ActivityRecord, Boolean>`，`forAllActivities`方法被定义在父类`WindowContainer`中，就是遍历执行所有`child`的`forAllActivities`方法，而`ActivityRecord`中重写了这个方法，直接用自身执行这个`Function`

简单来说，可以将这部分代码看作以下伪代码：

```java
rootTask.forEachAllActivityRecord((activityRecord) -> {
    startActivityForAttachedApplicationIfNeeded(activityRecord, app, rootTask.topRunningActivity())
})
```

接着我们来观察`startActivityForAttachedApplicationIfNeeded`方法

```java
private boolean startActivityForAttachedApplicationIfNeeded(ActivityRecord r,
        WindowProcessController app, ActivityRecord top) {
    if (r.finishing || !r.okToShowLocked() || !r.visibleIgnoringKeyguard || r.app != null
            || app.mUid != r.info.applicationInfo.uid || !app.mName.equals(r.processName)) {
        return false;
    }

    try {
        if (mStackSupervisor.realStartActivityLocked(r, app,
                top == r && r.isFocusable() /*andResume*/, true /*checkConfig*/)) {
            mTmpBoolean = true;
        }
    } catch (RemoteException e) {
        ...
        return true;
    }
    return false;
}
```

## 已有进程，直接启动Activity