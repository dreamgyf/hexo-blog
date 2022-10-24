---
title: Android源码分析 - Activity启动流程（下）
date: 2022-10-24 14:49:22
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

上一篇文章 [Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999) 中，我们分析了`Activity`启动流程中的一小部分，基本上可以算是`Activity`启动的前置准备工作，这篇文章我们将会分析`Activity`启动的主要流程

# 启动App进程

我们从App进程的启动开始说起，上篇文章中我们说过了，如果App尚未启动，则会调用`ATMS`的`startProcessAsync`方法去启动App进程

```java
void startProcessAsync(ActivityRecord activity, boolean knownToBeDead, boolean isTop,
        String hostingType) {
    try {
        ...
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess,
                mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                isTop, hostingType, activity.intent.getComponent());
        mH.sendMessage(m);
    } finally {
        ...
    }
}
```

这个方法实际上是通过`Hander`调用了`ActivityManagerInternal (AMS.LocalService)`的`startProcess`方法

```java
@Override
public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead,
    boolean isTop, String hostingType, ComponentName hostingName) {
try {
    ...
    synchronized (ActivityManagerService.this) {
        // If the process is known as top app, set a hint so when the process is
        // started, the top priority can be applied immediately to avoid cpu being
        // preempted by other processes before attaching the process of top app.
        startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
                new HostingRecord(hostingType, hostingName, isTop),
                ZYGOTE_POLICY_FLAG_LATENCY_SENSITIVE, false /* allowWhileBooting */,
                false /* isolated */, true /* keepIfLarge */);
    }
} finally {
    ...
}
}
```

这里将进程启动的一些信息封装到了`HostingRecord`类中

```java
final ProcessRecord startProcessLocked(String processName,
    ApplicationInfo info, boolean knownToBeDead, int intentFlags,
    HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
    boolean isolated, boolean keepIfLarge) {
return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
        hostingRecord, zygotePolicyFlags, allowWhileBooting, isolated, 0 /* isolatedUid */,
        keepIfLarge, null /* ABI override */, null /* entryPoint */,
        null /* entryPointArgs */, null /* crashHandler */);
}
```

`AMS`将启动进程的任务转交给了`ProcessList`，这个类的职责是管理进程，包括管理进程优先级(Adj)、进程OOM等

```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean allowWhileBooting, boolean isolated, int isolatedUid,
        boolean keepIfLarge, String abiOverride, String entryPoint, String[] entryPointArgs,
        Runnable crashHandler) {
    long startTime = SystemClock.uptimeMillis();
    ProcessRecord app;
    if (!isolated) {
        //先通过进程名和uid查找相应App的ProcessRecord
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);

        //如果是由后台进程发起的 startProcess
        //判断启动进程是否为 bad process，如果是，直接启动失败返回
        //这里 bad process 的定义为：短时间内连续崩溃两次以上的进程
        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            // If we are in the background, then check to see if this process
            // is bad.  If so, we will just silently fail.
            if (mService.mAppErrors.isBadProcessLocked(info)) {
                return null;
            }
        } else {
            //如果是用户显式的要求启动进程，则会清空启动进程的崩溃次数，将启动进程从 bad process 列表中移除
            // When the user is explicitly starting a process, then clear its
            // crash count so that we won't make it bad until they see at
            // least one crash dialog again, and make the process good again
            // if it had been bad.
            mService.mAppErrors.resetProcessCrashTimeLocked(info);
            if (mService.mAppErrors.isBadProcessLocked(info)) {
                EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                        UserHandle.getUserId(info.uid), info.uid,
                        info.processName);
                mService.mAppErrors.clearBadProcessLocked(info);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        // If this is an isolated process, it can't re-use an existing process.
        app = null;
    }

    // We don't have to do anything more if:
    // (1) There is an existing application record; and
    // (2) The caller doesn't think it is dead, OR there is no thread
    //     object attached to it so we know it couldn't have crashed; and
    // (3) There is a pid assigned to it, so it is either starting or
    //     already running.
    ProcessRecord precedence = null;
    //如果已经存在了对应App的ProcessRecord，并且分配了pid
    if (app != null && app.pid > 0) {
        //如果进程没有死亡或者进程还未绑定binder线程，说明进程是正常运行状态或正在启动中
        if ((!knownToBeDead && !app.killed) || app.thread == null) {
            // We already have the app running, or are waiting for it to
            // come up (we have a pid but not yet its thread), so keep it.
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
            return app;
        }

        // An application record is attached to a previous process,
        // clean it up now.
        ProcessList.killProcessGroup(app.uid, app.pid);

        Slog.wtf(TAG_PROCESSES, app.toString() + " is attached to a previous process");
        // We are not going to re-use the ProcessRecord, as we haven't dealt with the cleanup
        // routine of it yet, but we'd set it as the precedence of the new process.
        precedence = app;
        app = null;
    }

    if (app == null) {
        checkSlow(startTime, "startProcess: creating new process record");
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid, hostingRecord);
        if (app == null) {
            Slog.w(TAG, "Failed making new process record for "
                    + processName + "/" + info.uid + " isolated=" + isolated);
            return null;
        }
        app.crashHandler = crashHandler;
        app.isolatedEntryPoint = entryPoint;
        app.isolatedEntryPointArgs = entryPointArgs;
        if (precedence != null) {
            app.mPrecedence = precedence;
            precedence.mSuccessor = app;
        }
        checkSlow(startTime, "startProcess: done creating new process record");
    } else {
        // If this is a new package in the process, add the package to the list
        app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
        checkSlow(startTime, "startProcess: added package to existing proc");
    }

    // If the system is not ready yet, then hold off on starting this
    // process until it is.
    if (!mService.mProcessesReady
            && !mService.isAllowedWhileBooting(info)
            && !allowWhileBooting) {
        if (!mService.mProcessesOnHold.contains(app)) {
            mService.mProcessesOnHold.add(app);
        }
        if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES,
                "System not ready, putting on hold: " + app);
        checkSlow(startTime, "startProcess: returning with proc on hold");
        return app;
    }

    checkSlow(startTime, "startProcess: stepping in to startProcess");
    final boolean success =
            startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride);
    checkSlow(startTime, "startProcess: done starting proc!");
    return success ? app : null;
}
```