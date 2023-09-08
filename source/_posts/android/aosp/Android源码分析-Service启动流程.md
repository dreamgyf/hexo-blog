---
title: Android源码分析 - Service启动流程
date: 2023-08-24 16:10:53
tags: 
- Android源码
- Service
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, Service]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

在之前的文章中，我们已经分析过了四大组件中`Activity`和`ContentProvider`的启动流程，这次我们就来讲讲四大组件之一的`Service`是如何启动和绑定的

# 流程图

在查阅资料的过程中，我发现有些博主会将梳理好的流程图贴在开头，我觉得这样有助于从宏观上去理解源码的整个流程和设计理念，所以以后的文章我都会尽量将源码梳理成流程图，以便大家理解

![startService流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B_startService%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

![bindService流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B_bindService%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

# 入口

启动`Service`有两种方式，一是`startService`，一是`bindService`，它们最终的实现都在`ContextImpl`中

## Context.startService

当`Service`通过这种方式启动后，会一直运行下去，直到外部调用了`stopService`或内部调用`stopSelf`

```java
//frameworks/base/core/java/android/app/ContextImpl.java
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, false, mUser);
}

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        //确保Intent有效
        validateServiceIntent(service);
        //跨进程准备
        service.prepareToLeaveProcess(this);
        //调用AMS.startService
        ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                getOpPackageName(), getAttributionTag(), user.getIdentifier());
        //通过AMS层返回的ComponentName.packageName来判断是否出错以及错误类型
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException(
                        "Unable to start service " + service
                        + ": " + cn.getClassName());
            } else if (cn.getPackageName().equals("?")) {
                throw new IllegalStateException(
                        "Not allowed to start service " + service + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

从代码可以看出，这里就是做了一下简单的校验，然后便调用了`AMS.startService`启动`Service`，最终通过返回的`ComponentName`中的`packageName`来判断是否出错以及错误类型

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, boolean requireForeground, String callingPackage,
        String callingFeatureId, int userId)
        throws TransactionTooLargeException {
    enforceNotIsolatedCaller("startService");
    // Refuse possible leaked file descriptors
    //校验Intent，不允许其携带fd
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    //调用方包名不能为空
    if (callingPackage == null) {
        throw new IllegalArgumentException("callingPackage cannot be null");
    }

    synchronized(this) {
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res;
        try {
            //调用ActiveServices.startServiceLocked方法
            res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid,
                    requireForeground, callingPackage, callingFeatureId, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
        return res;
    }
}
```

同样，这里做了一些简单的检查，然后调用`ActiveServices.startServiceLocked`方法，`ActiveServices`是一个辅助`AMS`进行`Service`管理的类，包括`Service`的启动、绑定和停止等

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
        @Nullable String callingFeatureId, final int userId)
        throws TransactionTooLargeException {
    return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, fgRequired,
            callingPackage, callingFeatureId, userId, false);
}

ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
        @Nullable String callingFeatureId, final int userId,
        boolean allowBackgroundActivityStarts) throws TransactionTooLargeException {
    //判断调用方是否为前台
    final boolean callerFg;
    if (caller != null) {
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + callingPid
                    + ") when starting service " + service);
        }
        callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
    } else {
        callerFg = true;
    }

    //查找待启动Service
    ServiceLookupResult res =
        retrieveServiceLocked(service, null, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false, false);
    //如果找不到待启动Service，直接返回null
    if (res == null) {
        return null;
    }
    //如果待启动的Service所在package和uid无法与调用方package和uid建立关联，则无法启动Service
    //返回异常ComponentName，由上层抛出SecurityException异常
    if (res.record == null) {
        return new ComponentName("!", res.permission != null
                ? res.permission : "private to package");
    }

    ServiceRecord r = res.record;

    //试图用一个不存在的用户启动Service
    if (!mAm.mUserController.exists(r.userId)) {
        Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
        return null;
    }

    // If we're starting indirectly (e.g. from PendingIntent), figure out whether
    // we're launching into an app in a background state.  This keys off of the same
    // idleness state tracking as e.g. O+ background service start policy.
    //Service所在应用未启动或处在后台
    final boolean bgLaunch = !mAm.isUidActiveLocked(r.appInfo.uid);

    // If the app has strict background restrictions, we treat any bg service
    // start analogously to the legacy-app forced-restrictions case, regardless
    // of its target SDK version.
    //检查Service所在应用后台启动限制
    boolean forcedStandby = false;
    if (bgLaunch && appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
        forcedStandby = true;
    }

    // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
    boolean forceSilentAbort = false;
    if (fgRequired) { //作为前台服务启动
        //权限检查
        final int mode = mAm.getAppOpsManager().checkOpNoThrow(
                AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
        switch (mode) {
            //默认和允许都可以作为前台服务启动
            case AppOpsManager.MODE_ALLOWED:
            case AppOpsManager.MODE_DEFAULT:
                // All okay.
                break;
            //不允许的话，回退到作为普通后台服务启动
            case AppOpsManager.MODE_IGNORED:
                // Not allowed, fall back to normal start service, failing siliently
                // if background check restricts that.
                Slog.w(TAG, "startForegroundService not allowed due to app op: service "
                        + service + " to " + r.shortInstanceName
                        + " from pid=" + callingPid + " uid=" + callingUid
                        + " pkg=" + callingPackage);
                fgRequired = false;
                forceSilentAbort = true;
                break;
            //错误的话直接返回，由上层抛出SecurityException异常
            default:
                return new ComponentName("!!", "foreground not allowed as per app op");
        }
    }

    // If this isn't a direct-to-foreground start, check our ability to kick off an
    // arbitrary service
    //如果不是从前台启动
    //startRequested表示Service是否由startService方式所启动，fgRequired表示作为前台服务启动
    if (forcedStandby || (!r.startRequested && !fgRequired)) {
        // Before going further -- if this app is not allowed to start services in the
        // background, then at this point we aren't going to let it period.
        //服务是否允许在后台启动
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
        //如果不允许，则无法启动服务
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            //静默的停止启动
            if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                // In this case we are silently disabling the app, to disrupt as
                // little as possible existing apps.
                return null;
            }
            if (forcedStandby) {
                // This is an O+ app, but we might be here because the user has placed
                // it under strict background restrictions.  Don't punish the app if it's
                // trying to do the right thing but we're denying it for that reason.
                if (fgRequired) {
                    return null;
                }
            }
            // This app knows it is in the new model where this operation is not
            // allowed, so tell it what has happened.
            //明确的告知不允许启动，上层抛出异常
            UidRecord uidRec = mAm.mProcessList.getUidRecordLocked(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
        }
    }

    // At this point we've applied allowed-to-start policy based on whether this was
    // an ordinary startService() or a startForegroundService().  Now, only require that
    // the app follow through on the startForegroundService() -> startForeground()
    // contract if it actually targets O+.
    //对于targetSdk 26以下（Android 8.0以下）的应用来说，不需要作为前台服务启动
    if (r.appInfo.targetSdkVersion < Build.VERSION_CODES.O && fgRequired) {
        fgRequired = false;
    }

    //检查通过Intent被临时授权的Uris
    NeededUriGrants neededGrants = mAm.mUgmInternal.checkGrantUriPermissionFromIntent(
            service, callingUid, r.packageName, r.userId);

    // If permissions need a review before any of the app components can run,
    // we do not start the service and launch a review activity if the calling app
    // is in the foreground passing it a pending intent to start the service when
    // review is completed.

    // XXX This is not dealing with fgRequired!
    //如果待启动的Service需要相应权限，则需要用户手动确认权限后，再进行启动
    if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage, callingFeatureId,
            callingUid, service, callerFg, userId)) {
        return null;
    }

    //取消之前的Service重启任务（如果有）
    if (unscheduleServiceRestartLocked(r, callingUid, false)) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
    }
    r.lastActivity = SystemClock.uptimeMillis();
    //表示Service是否由startService方式所启动的
    r.startRequested = true;
    r.delayedStop = false;
    //是否作为前台服务启动
    r.fgRequired = fgRequired;
    //构造启动参数
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants, callingUid));

    //作为前台服务启动
    if (fgRequired) {
        // We are now effectively running a foreground service.
        ... //使用ServiceState记录
        ... //通过AppOpsService监控
    }

    final ServiceMap smap = getServiceMapLocked(r.userId);
    boolean addToStarting = false;
    //对于后台启动的非前台服务，需要判断其是否需要延迟启动
    if (!callerFg && !fgRequired && r.app == null
            && mAm.mUserController.hasStartedUserState(r.userId)) {
        //获取Service所处进程信息
        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
        //没有对应进程或进程状态级别低于 [进程在后台运行Receiver]
        if (proc == null || proc.getCurProcState() > ActivityManager.PROCESS_STATE_RECEIVER) {
            // If this is not coming from a foreground caller, then we may want
            // to delay the start if there are already other background services
            // that are starting.  This is to avoid process start spam when lots
            // of applications are all handling things like connectivity broadcasts.
            // We only do this for cached processes, because otherwise an application
            // can have assumptions about calling startService() for a service to run
            // in its own process, and for that process to not be killed before the
            // service is started.  This is especially the case for receivers, which
            // may start a service in onReceive() to do some additional work and have
            // initialized some global state as part of that.
            //对于之前已经设置为延迟启动的服务，直接返回
            if (r.delayed) {
                // This service is already scheduled for a delayed start; just leave
                // it still waiting.
                return r.name;
            }
            //如果当前正在后台启动的Service数大于等于允许同时在后台启动的最大服务数
            //将这个Service设置为延迟启动
            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                // Something else is starting, delay!
                smap.mDelayedStartList.add(r);
                r.delayed = true;
                return r.name;
            }
            //添加到正在启动服务列表中
            addToStarting = true;
        } else if (proc.getCurProcState() >= ActivityManager.PROCESS_STATE_SERVICE) {
            //进程状态为 [正在运行Service的后台进程] 或 [正在运行Receiver的后台进程] 时
            // We slightly loosen when we will enqueue this new service as a background
            // starting service we are waiting for, to also include processes that are
            // currently running other services or receivers.
            //添加到正在启动服务列表中
            addToStarting = true;
        }
    }

    //如果允许Service后台启动Activity，则将其加入到白名单中
    if (allowBackgroundActivityStarts) {
        r.whitelistBgActivityStartsOnServiceStart();
    }
    //继续启动Service
    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);

    //检查是否允许前台服务使用while-in-use权限
    if (!r.mAllowWhileInUsePermissionInFgs) {
        r.mAllowWhileInUsePermissionInFgs =
                shouldAllowWhileInUsePermissionInFgsLocked(callingPackage, callingPid,
                        callingUid, service, r, allowBackgroundActivityStarts);
    }

    return cmp;
}
```
这个方法涉及到很多前后台判断，我想这里的前后台其实分为三个概念，一是调用方App是否在前台，二是`Service`方App是否在前台，三是`Service`是否作为前台服务启动，当然，大部分情况启动的都是App内的`Service`，即一二中的前后台状态是一致的，但也不排除启动其他App的`Service`这种情况，所以这里还是需要好好区分开来

这个方法看起来很长，但总之都是一些`Service`启动前的预处理工作，主要做了以下几点工作：

1. 判断调用方进程是否在前台（`callerFg`）：对于调用方在后台启动的`Service`，需要判断其是否需要延迟启动
2. 调用`retrieveServiceLocked`查找待启动`Service`信息（`ServiceRecord`）
3. 各种检查，一旦发现不满足启动条件就终止启动`Service`
4. 检查`Service`所在应用的前后台状态以及后台启动限制，不符合条件则终止启动`Service`
5. 判断是否可以作为前台服务启动
6. 如果待启动的`Service`需要相应权限，则需要用户手动确认权限后，再进行启动
7. 取消之前的`Service`重启任务（如果有）
8. 设置`ServiceRecord`状态，包括上次活动时间，是否由startService方式所启动的，是否作为前台服务启动等
9. 如果作为前台服务启动，则需要进行记录和监控
10. 对于后台启动的非前台服务，需要判断其是否需要延迟启动
11. 调用`startServiceInnerLocked`继续启动`Service`

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    ... //记录
    //启动前初始化
    r.callStart = false;
    ... //记录
    //拉起服务，如果服务未启动，则会启动服务并调用其onCreate和onStartCommand方法
    //如果服务已启动，由于之前构造了启动参数，则会直接调用其onStartCommand方法
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    if (r.startRequested && addToStarting) { //对于后台启动服务的情况
        //是否为第一个后台启动的服务
        boolean first = smap.mStartingBackground.size() == 0;
        //添加到正在后台启动服务列表中
        smap.mStartingBackground.add(r);
        //设置后台启动服务超时时间（默认15秒）
        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
        //如果为第一个后台启动的服务，则代表后面暂时没有正在后台启动的服务了
        //此时将之前设置为延迟启动的服务调度出来后台启动
        if (first) {
            smap.rescheduleDelayedStartsLocked();
        }
    } else if (callerFg || r.fgRequired) { //对于调用方进程为前台或作为前台服务启动的情况
        //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
        //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
        smap.ensureNotStartingBackgroundLocked(r);
    }

    return r.name;
}
```

在这个方法中，首先会调用`bringUpServiceLocked`方法拉起服务，然后根据服务是否为前台启动，分别调用`ServiceMap.rescheduleDelayedStartsLocked`和`ServiceMap.ensureNotStartingBackgroundLocked`方法从后台延迟启动服务列表`mDelayedStartList`中不断地调度启动服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
    //如果Service所在的进程存在，并且其IApplicationThread也存在
    //说明服务已启动（因为在启动服务时，会给ServiceRecord.app赋值，并且app.thread不为null说明进程没有被杀死）
    //此时直接拉起Service.onStartCommand方法
    if (r.app != null && r.app.thread != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }

    //如果服务正在重启中，则什么都不做，直接返回
    if (!whileRestarting && mRestartingServices.contains(r)) {
        // If waiting for a restart, then do nothing.
        return null;
    }

    // We are now bringing the service up, so no longer in the
    // restarting state.
    //Service马上启动，将其从重启中服务列表中移除，并清除其重启中状态
    if (mRestartingServices.remove(r)) {
        clearRestartingIfNeededLocked(r);
    }

    // Make sure this service is no longer considered delayed, we are starting it now.
    //走到这里，需要确保此服务不再被视为延迟启动，同时将其从延迟启动服务列表中移除
    if (r.delayed) {
        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    // Make sure that the user who owns this service is started.  If not,
    // we don't want to allow it to run.
    //确保Service所在的用户已启动
    if (!mAm.mUserController.hasStartedUserState(r.userId)) {
        //停止服务
        bringDownServiceLocked(r);
        return msg;
    }

    // Service is now being launched, its package can't be stopped.
    //Service即将启动，Service所属的App不该为stopped状态
    //将App状态置为unstopped，设置休眠状态为false
    AppGlobals.getPackageManager().setPackageStoppedState(
            r.packageName, false, r.userId);

    //服务所在进程是否为隔离进程，指服务是否在其自己的独立进程中运行
    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
    ProcessRecord app;

    if (!isolated) { //非隔离进程
        //获取进程的ProcessRecord对象
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
        if (app != null && app.thread != null) {
            //将App添加至进程中运行的包列表中
            app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
            //接着启动Service
            realStartServiceLocked(r, app, execInFg);
            return null;

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
    } else { //隔离进程
        // If this service runs in an isolated process, then each time
        // we call startProcessLocked() we will get a new isolated
        // process, starting another process if we are currently waiting
        // for a previous process to come up.  To deal with this, we store
        // in the service any current isolated process it is running in or
        // waiting to have come up.
        //获取服务之前所在的进程
        app = r.isolatedProc;
        //辅助zygote进程，用于创建isolated_app进程来渲染不可信的web内容，具有最为严格的安全限制
        if (WebViewZygote.isMultiprocessEnabled()
                && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
            hostingRecord = HostingRecord.byWebviewZygote(r.instanceName);
        }
        //应用zygote进程，与常规zygote创建的应用相比受到更多限制
        if ((r.serviceInfo.flags & ServiceInfo.FLAG_USE_APP_ZYGOTE) != 0) {
            hostingRecord = HostingRecord.byAppZygote(r.instanceName, r.definingPackageName,
                    r.definingUid);
        }
    }

    // Not running -- get it started, and enqueue this service record
    // to be executed when the app comes up.
    //如果Service所在进程尚未启动
    if (app == null && !permissionsReviewRequired) {
        // TODO (chriswailes): Change the Zygote policy flags based on if the launch-for-service
        //  was initiated from a notification tap or not.
        //启动App进程
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated, false)) == null) {
            //如果启动进程失败，停止服务
            bringDownServiceLocked(r);
            return msg;
        }
        if (isolated) {
            //如果是隔离进程，将这次启动的进程记录保存下来
            r.isolatedProc = app;
        }
    }

    //对于要启动的前台服务，加入到临时白名单，暂时绕过省电模式
    if (r.fgRequired) {
        mAm.tempWhitelistUidLocked(r.appInfo.uid,
                SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
    }

    //将启动的服务添加到mPendingServices列表中
    //如果服务进程尚未启动，进程在启动的过程中会检查此列表并启动需要启动的Service
    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }

    //Service被要求stop，停止服务
    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            stopServiceLocked(r);
        }
    }

    return null;
}
```

这个方法看起来长，其实做的事情并不多：

1. 如果`Service`已经启动，则调用`sendServiceArgsLocked`方法直接拉起`Service.onStartCommand`方法
2. 各种检查准备操作（待重启、用户是否启动等，具体见注释）
3. 如果`Service`所在进程已启动，调用`realStartServiceLocked`方法接着启动`Service`
4. 如果`Service`所在进程未启动，调用`AMS.startProcessLocked`方法启动进程
5. 将要启动的`Service`添加到`mPendingServices`列表中，对于`Service`所在进程未启动的这种情况，在进程的启动过程中会检查此列表并启动需要启动的`Service`（即此`Service`）

从这里可以看出来，`Service`的启动分为两个分支，一个是进程已启动，一个是进程未启动

### 进程未启动

在进程未启动的情况下，这里会调用`AMS.startProcessLocked`方法启动进程，接着等待进程启动完成后，会调用到`AMS.attachApplicationLocked`方法，在这个方法中有一段关于`Service`启动的代码：

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    ...
    // Find any services that should be running in this process...
    //检查是否有Services等待启动
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            badApp = true;
        }
    }
    ...
}
```

可以看到，在这里调用了`ActiveServices.attachApplicationLocked`方法去启动待启动的`Service`

关于`App`进程启动的流程详见我之前的文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128) ，这里就不赘述了

#### ActiveServices.attachApplicationLocked

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
boolean attachApplicationLocked(ProcessRecord proc, String processName)
        throws RemoteException {
    boolean didSomething = false;

    // Update the app background restriction of the caller
    //更新Service所在App后台限制
    proc.mState.setBackgroundRestricted(appRestrictedAnyInBackground(
            proc.uid, proc.info.packageName));

    // Collect any services that are waiting for this process to come up.
    //启动mPendingServices列表内，该进程下的所有Service
    if (mPendingServices.size() > 0) {
        ServiceRecord sr = null;
        try {
            for (int i=0; i<mPendingServices.size(); i++) {
                sr = mPendingServices.get(i);
                if (proc != sr.isolationHostProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }

                final IApplicationThread thread = proc.getThread();
                final int pid = proc.getPid();
                final UidRecord uidRecord = proc.getUidRecord();
                mPendingServices.remove(i);
                i--;
                //将App添加至进程中运行的包列表中
                proc.addPackage(sr.appInfo.packageName, sr.appInfo.longVersionCode,
                        mAm.mProcessStats);
                //启动Service
                realStartServiceLocked(sr, proc, thread, pid, uidRecord, sr.createdFromFg,
                        true);
                didSomething = true;
                //如果此Service不再需要了，则停止它
                //e.g. 通过bindService启动的服务，但此时调用bindService的Activity已死亡
                if (!isServiceNeededLocked(sr, false, false)) {
                    // We were waiting for this service to start, but it is actually no
                    // longer needed.  This could happen because bringDownServiceIfNeeded
                    // won't bring down a service that is pending...  so now the pending
                    // is done, so let's drop it.
                    bringDownServiceLocked(sr, true);
                }
                /* Will be a no-op if nothing pending */
                //更新进程优先级
                mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
            }
        } catch (RemoteException e) {
            throw e;
        }
    }
    // Also, if there are any services that are waiting to restart and
    // would run in this process, now is a good time to start them.  It would
    // be weird to bring up the process but arbitrarily not let the services
    // run at this point just because their restart time hasn't come up.
    //App被杀重启机制，后续文章再详细说明
    if (mRestartingServices.size() > 0) {
        ...
    }
    return didSomething;
}
```

这个方法会从`mPendingServices`列表内寻找该进程下的所有待启动`Service`，然后调用`ActiveServices.realStartServiceLocked`方法启动它

### 进程已启动

对于进程已启动的情况，我们通过`ActiveServices.bringUpServiceLocked`方法也可以得知，调用了`ActiveServices.realStartServiceLocked`方法，所以不管进程是否启动，最终都会殊途同归走到`ActiveServices.realStartServiceLocked`方法启动`Service`

### ActiveServices.realStartServiceLocked

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
/**
 * Note the name of this method should not be confused with the started services concept.
 * The "start" here means bring up the instance in the client, and this method is called
 * from bindService() as well.
 */
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    //IApplicationThread不存在则抛移除
    //即确保ActivityThread存在
    if (app.thread == null) {
        throw new RemoteException();
    }
    //为ServiceRecord设置所属进程
    r.setProcess(app);
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

    //在此进程中将Service记录为运行中
    //返回值为此Service是否之前未启动
    final boolean newService = app.startService(r);
    //记录Service执行操作并设置超时回调
    //前台服务超时时间为20s，后台服务超时时间为200s
    bumpServiceExecutingLocked(r, execInFg, "create");
    //更新进程优先级
    mAm.updateLruProcessLocked(app, false, null);
    updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
    mAm.updateOomAdjLocked(app, OomAdjuster.OOM_ADJ_REASON_START_SERVICE);

    boolean created = false;
    try {
        ... //记录
        //记录信息
        mAm.notifyPackageUse(r.serviceInfo.packageName,
                                PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
        //设置进程状态
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
        //回到App进程，调度创建Service
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                app.getReportedProcState());
        //显示前台服务通知
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        //杀死进程
        mAm.appDiedLocked(app, "Died when creating service");
        throw e;
    } finally {
        //如果没能成功创建Service
        if (!created) {
            // Keep the executeNesting count accurate.
            //保证executeNesting计数的准确
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            // Cleanup.
            //停止服务，清除信息
            if (newService) {
                app.stopService(r);
                r.setProcess(null);
            }

            // Retry.
            //重试
            if (!inDestroying) {
                scheduleServiceRestartLocked(r, false);
            }
        }
    }

    //允许管理白名单，如省电模式白名单
    if (r.whitelistManager) {
        app.whitelistManager = true;
    }

    //执行Service.onBind方法（通过bindService启动的情况下）
    requestServiceBindingsLocked(r, execInFg);

    //更新是否有与Service建立连接的Activity
    updateServiceClientActivitiesLocked(app, null, true);

    //添加绑定到Service所在进程的UID
    if (newService && created) {
        app.addBoundClientUidsOfNewService(r);
    }

    // If the service is in the started state, and there are no
    // pending arguments, then fake up one so its onStartCommand() will
    // be called.
    //如果Service已经启动，并且没有启动项，则构建一个假的启动参数供onStartCommand使用
    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                null, null, 0));
    }

    //拉起Service.onStartCommand方法
    sendServiceArgsLocked(r, execInFg, true);

    //走到这里，需要确保此服务不再被视为延迟启动，同时将其从延迟启动服务列表中移除
    if (r.delayed) {
        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    //Service被要求stop，停止服务
    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            stopServiceLocked(r);
        }
    }
}
```

### 创建Service

到了这一步，进程理应启动和初始化完成了，接下来就该实际的去创建`Service`并启动它了，首先创建`Service`这一块我们看`app.thread.scheduleCreateService`方法，这里的`app`是`ProcessRecord`，里面的`thread`是`IApplicationThread`，`ActivityThread`中的内部类

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public final void scheduleCreateService(IBinder token,
        ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
    //更新进程状态
    updateProcessState(processState, false);
    //将创建Service的必要信息包装
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;
    //通过Handler发送Message
    sendMessage(H.CREATE_SERVICE, s);
}
```

这里将创建`Service`的必要信息包装成`CreateServiceData`对象后，通过`Handler`发送`Message`处理服务创建

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public void handleMessage(Message msg) {
    switch (msg.what) {
        ...
        case CREATE_SERVICE:
            handleCreateService((CreateServiceData)msg.obj);
            break;
        ...
    }
   ...
}
```

`ActivityThread`的`Handler`在接收到`CREATE_SERVICE`消息后调用了`handleCreateService`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    //此时不要进行GC
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        //创建Context
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        //创建或获取Application（到了这里进程的初始化应该都完成了，所以是直接获取Application）
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        //通过AppComponentFactory反射创建Service实例
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
        // Service resources must be initialized with the same loaders as the application
        // context.
        //加载资源
        context.getResources().addLoaders(
                app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

        context.setOuterContext(service);
        //初始化
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        //执行onCreate回调
        service.onCreate();
        //保存运行中的Service
        mServices.put(data.token, service);
        try {
            //Service相关任务执行完成
            //这一步中会把之前的启动超时定时器取消
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to create service " + data.info.name
                + ": " + e.toString(), e);
        }
    }
}
```

可以看到，`Service`的创建和之前文章中所分析的`Activity`的创建流程基本一致，都是创建`Context`，通过`AppComponentFactory`反射实例化对象，然后加载资源，`attach`做绑定，最后执行`onCreate`回调

```java
//frameworks/base/core/java/android/app/AppComponentFactory.java
public @NonNull Service instantiateService(@NonNull ClassLoader cl,
        @NonNull String className, @Nullable Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Service) cl.loadClass(className).newInstance();
}
```

如果没有特别在`AndroidManifest.xml`中设置`android:appComponentFactory`的话，默认的实现就是这样，通过传进来的`ClassLoader`和`className`反射实例化`Service`对象

```java
//frameworks/base/core/java/android/app/Service.java
public final void attach(
        Context context,
        ActivityThread thread, String className, IBinder token,
        Application application, Object activityManager) {
    //绑定BaseContext
    attachBaseContext(context);
    mThread = thread;           // NOTE:  unused - remove?
    mClassName = className;
    //保存ServiceRecord
    mToken = token;
    mApplication = application;
    mActivityManager = (IActivityManager)activityManager;
    //启动兼容性设置
    mStartCompatibility = getApplicationInfo().targetSdkVersion
            < Build.VERSION_CODES.ECLAIR;
    //设置内容捕获功能
    setContentCaptureOptions(application.getContentCaptureOptions());
}
```

`attach`方法也很简单，做了一些绑定`Context`等基本操作

最后调用`onCreate`方法，这个方法默认是个空实现，让继承`Service`的类去实现这个方法

### 启动Service

`Service`创建完成后就该启动它了，这里对应着`ActiveServices.realStartServiceLocked`方法中调用的`sendServiceArgsLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
        boolean oomAdjusted) throws TransactionTooLargeException {
    //如果待启动项列表中没有内容则直接返回
    final int N = r.pendingStarts.size();
    if (N == 0) {
        return;
    }

    ArrayList<ServiceStartArgs> args = new ArrayList<>();

    //遍历待启动项
    while (r.pendingStarts.size() > 0) {
        ServiceRecord.StartItem si = r.pendingStarts.remove(0);
        //如果在多个启动项中有假启动项，则跳过假启动项
        //但如果这个假启动项是唯一的启动项则不要跳过它，这是为了支持onStartCommand(null)的情况
        if (si.intent == null && N > 1) {
            // If somehow we got a dummy null intent in the middle,
            // then skip it.  DO NOT skip a null intent when it is
            // the only one in the list -- this is to support the
            // onStartCommand(null) case.
            continue;
        }
        si.deliveredTime = SystemClock.uptimeMillis();
        r.deliveredStarts.add(si);
        si.deliveryCount++;
        //处理Uri权限
        if (si.neededGrants != null) {
            mAm.mUgmInternal.grantUriPermissionUncheckedFromIntent(si.neededGrants,
                    si.getUriPermissionsLocked());
        }
        //授权访问权限
        mAm.grantImplicitAccess(r.userId, si.intent, si.callingId,
                UserHandle.getAppId(r.appInfo.uid)
        );
        //记录Service执行操作并设置超时回调
        //前台服务超时时间为20s，后台服务超时时间为200s
        bumpServiceExecutingLocked(r, execInFg, "start");
        if (!oomAdjusted) {
            oomAdjusted = true;
            mAm.updateOomAdjLocked(r.app, true, OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
        }
        //如果是以前台服务的方式启动的Service（startForegroundService），并且之前没有设置启动前台服务超时回调
        if (r.fgRequired && !r.fgWaiting) {
            //如果当前服务还没成为前台服务，设置启动前台服务超时回调
            //在10s内需要调用Service.startForeground成为前台服务，否则停止服务
            //注：Android 11这个超时时间是10s，在后面的Android版本中这个时间有变化
            if (!r.isForeground) {
                scheduleServiceForegroundTransitionTimeoutLocked(r);
            } else {
                r.fgRequired = false;
            }
        }
        int flags = 0;
        if (si.deliveryCount > 1) {
            flags |= Service.START_FLAG_RETRY;
        }
        if (si.doneExecutingCount > 0) {
            flags |= Service.START_FLAG_REDELIVERY;
        }
        //添加启动项
        args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
    }

    //构建出一个支持Binder跨进程传输大量数据的列表来传输启动参数数据
    ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
    slice.setInlineCountLimit(4);
    try {
        //回到App进程，调度启动Service
        r.app.thread.scheduleServiceArgs(r, slice);
    } catch ...
}
```

这个方法中有几个比较重要的点需要注意：

1. `pendingStarts`中至少要有一个启动项才会执行`onStartCommand`，所以在前面的`ActiveServices.realStartServiceLocked`方法中才会有这样一段代码：如果`Service`已经启动，并且没有启动项，则构建一个假的启动参数供`onStartCommand`使用
2. 如果在多个启动项中有假启动项，则跳过假启动项，但如果这个假启动项是唯一的启动项则不要跳过它，这是为了支持`onStartCommand`方法的第一个参数`Intent`未`null`的情况
3. 如果服务是以前台服务的方式启动的（`startForegroundService`），如果当前服务还没成为前台服务，则需要设置一个启动前台服务的超时回调，如果在限制的时间范围内还没有成为前台服务（调用`Service.startForeground`方法），则会触发超时逻辑，停止服务，这个时间在`Android 11`上是10s，在后面的`Android`版本中有变化
4. 最后将所有的启动项放到一个支持`Binder`跨进程传输大量数据的列表中，然后调用`App`进程中的`ActivityThread$ApplicationThread.scheduleServiceArgs`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
    List<ServiceStartArgs> list = args.getList();

    for (int i = 0; i < list.size(); i++) {
        ServiceStartArgs ssa = list.get(i);
        ServiceArgsData s = new ServiceArgsData();
        s.token = token;
        s.taskRemoved = ssa.taskRemoved;
        s.startId = ssa.startId;
        s.flags = ssa.flags;
        s.args = ssa.args;

        sendMessage(H.SERVICE_ARGS, s);
    }
}
```

和前面一样，这里也是将启动`Service`的必要信息包装成一个个`ServiceStartArgs`对象后，通过`Handler`依次发送`Message`处理服务启动，这里最终调用的是`ActivityThread.handleServiceArgs`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            //Intent跨进程处理
            if (data.args != null) {
                data.args.setExtrasClassLoader(s.getClassLoader());
                data.args.prepareToEnterProcess();
            }
            int res;
            if (!data.taskRemoved) {
                //正常情况调用
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                //用户关闭Task栈时调用
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }

            //确保其他异步任务执行完成
            QueuedWork.waitToFinish();

            try {
                //Service相关任务执行完成
                //这一步会根据onStartCommand的返回值，调整Service死亡重建策略
                //同时会把之前的启动超时定时器取消
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to start service " + s
                        + " with " + data.args + ": " + e.toString(), e);
            }
        }
    }
}
```

这里首先判断`taskRemoved`标志，这个标志为`true`则代表用户之前从最近任务界面里划掉了这个任务栈或者在最近任务界面里点击了清理，此时会调用`Service.onTaskRemoved`方法（从最近任务界面关闭应用，进程不一定会被杀死，而且`Service`具有死亡重建机制），在正常情况下则是调用`Service.onStartCommand`处理服务启动

当任务执行完成后，会调用`AMS.serviceDoneExecuting`方法告知，在这个方法中会根据`onStartCommand`的返回值（或执行完`onTaskRemoved`被赋值为`START_TASK_REMOVED_COMPLETE`），调整`Service`的死亡重建策略，并且会把之前的启动超时定时器取消

`onStartCommand`可以有以下几种返回值：

- `START_STICKY_COMPATIBILITY`：`targetSdkVersion` < 5 (`Android 2.0`) 的App默认会返回这个，`Service`被杀后会被重建，但`onStartCommand`方法不会被执行
- `START_STICKY`：`targetSdkVersion` >= 5 (`Android 2.0`) 的App默认会返回这个，`Service`被杀后会被重建，`onStartCommand`方法也会被执行，但此时`onStartCommand`方法的第一个参数`Intent`为`null`
- `START_NOT_STICKY`：`Service`被杀后不会被重建
- `START_REDELIVER_INTENT`：`Service`被杀后会被重建，`onStartCommand`方法也会被执行，此时`onStartCommand`方法的第一个参数`Intent`为`Service`被杀死前最后一次调用`onStartCommand`方法时传递的`Intent`

在这里我们只摆出结论，暂时不分析原理，如果感兴趣的话我会在后面的文章中再去详细分析

## Context.bindService

到这里，`Service`通过`startService`路径的启动流程我们就基本分析完了，接着我们看另一条路径，`bindService`

```java
//frameworks/base/core/java/android/app/ContextImpl.java
public boolean bindService(Intent service, ServiceConnection conn, int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
            getUser());
}

private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
        String instanceName, Handler handler, Executor executor, UserHandle user) {
    // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
    //获取LoadedApk$ServiceDispatcher$IServiceConnection
    //这个类是用来后续连接建立完成后发布连接，回调ServiceConnection各种方法的
    IServiceConnection sd;
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (handler != null && executor != null) {
        throw new IllegalArgumentException("Handler and Executor both supplied");
    }
    if (mPackageInfo != null) {
        if (executor != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
        } else {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        }
    } else {
        throw new RuntimeException("Not supported in system context");
    }
    //确保Intent有效
    validateServiceIntent(service);
    try {
        //获取ActivityRecord的远程Binder对象
        IBinder token = getActivityToken();
        //targetSdkVersion < 14 (Android 4.0)的情况下，如果没有设置BIND_AUTO_CREATE
        //则该Service的优先级将会被视为等同于后台任务
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        //跨进程准备
        service.prepareToLeaveProcess(this);
        //跨进程使用AMS绑定Service
        int res = ActivityManager.getService().bindIsolatedService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

这个方法主要就做了两件事：

1. 创建或获取`IServiceConnection`：这里实际获取到的是`LoadedApk`中的内部类`ServiceDispatcher`中的内部类`InnerConnection`，这个类主要是用来后面建立或断开连接后，回调`ServiceConnection`接口的各个方法的
2. 跨进程调用`AMS.bindIsolatedService`方法绑定`Service`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags, String instanceName,
        String callingPackage, int userId) throws TransactionTooLargeException {
    enforceNotIsolatedCaller("bindService");

    // Refuse possible leaked file descriptors
    //校验Intent，不允许其携带fd
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    //校验调用方包名
    if (callingPackage == null) {
        throw new IllegalArgumentException("callingPackage cannot be null");
    }

    // Ensure that instanceName, which is caller provided, does not contain
    // unusual characters.
    if (instanceName != null) {
        for (int i = 0; i < instanceName.length(); ++i) {
            char c = instanceName.charAt(i);
            if (!((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')
                        || (c >= '0' && c <= '9') || c == '_' || c == '.')) {
                throw new IllegalArgumentException("Illegal instanceName");
            }
        }
    }

    synchronized(this) {
        return mServices.bindServiceLocked(caller, token, service,
                resolvedType, connection, flags, instanceName, callingPackage, userId);
    }
}
```

这里仅仅做了一些简单的校验，然后将绑定服务的任务转交给了`ActiveServices.bindServiceLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection, int flags,
        String instanceName, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    //获取调用方进程记录
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
    if (callerApp == null) {
        throw new SecurityException(
                "Unable to find app for caller " + caller
                + " (pid=" + Binder.getCallingPid()
                + ") when binding service " + service);
    }

    ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
    //token不为空表示是从Activity发起的，token实际为ActivityRecord的远程Binder对象
    if (token != null) {
        //获取Activity与Service的连接记录
        activity = mAm.mAtmInternal.getServiceConnectionsHolder(token);
        //ServiceConnectionsHolder为null说明调用方Activity不在栈中，直接异常返回
        if (activity == null) {
            return 0;
        }
    }

    int clientLabel = 0;
    PendingIntent clientIntent = null;
    final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

    //如果调用方为系统级应用
    if (isCallerSystem) {
        // Hacky kind of thing -- allow system stuff to tell us
        // what they are, so we can report this elsewhere for
        // others to know why certain services are running.
        service.setDefusable(true);
        clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);
        if (clientIntent != null) {
            clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
            if (clientLabel != 0) {
                // There are no useful extras in the intent, trash them.
                // System code calling with this stuff just needs to know
                // this will happen.
                service = service.cloneFilter();
            }
        }
    }

    //像对待Activity一样对待该Service
    //需要校验调用方应用是否具有MANAGE_ACTIVITY_STACKS权限
    if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
        mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                "BIND_TREAT_LIKE_ACTIVITY");
    }

    //此标志仅用于系统调整IMEs（以及与顶层App密切合作的其他跨进程的用户可见组件）的调度策略，仅限系统级App使用
    if ((flags & Context.BIND_SCHEDULE_LIKE_TOP_APP) != 0 && !isCallerSystem) {
        throw new SecurityException("Non-system caller (pid=" + Binder.getCallingPid()
                + ") set BIND_SCHEDULE_LIKE_TOP_APP when binding service " + service);
    }

    //允许绑定Service的应用程序管理白名单，仅限系统级App使用
    if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {
        throw new SecurityException(
                "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
    }

    //允许绑定到免安装应用提供的服务，仅限系统级App使用
    if ((flags & Context.BIND_ALLOW_INSTANT) != 0 && !isCallerSystem) {
        throw new SecurityException(
                "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                        + ") set BIND_ALLOW_INSTANT when binding service " + service);
    }

    //允许Service后台启动Activity
    //需要校验调用方应用是否具有START_ACTIVITIES_FROM_BACKGROUND权限
    if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
        mAm.enforceCallingPermission(
                android.Manifest.permission.START_ACTIVITIES_FROM_BACKGROUND,
                "BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS");
    }

    //判断调用方是否为前台
    final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
    final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;
    final boolean allowInstant = (flags & Context.BIND_ALLOW_INSTANT) != 0;

    //查找相应的Service
    ServiceLookupResult res =
        retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
                Binder.getCallingPid(), Binder.getCallingUid(), userId, true,
                callerFg, isBindExternal, allowInstant);
    if (res == null) {
        return 0;
    }
    if (res.record == null) {
        return -1;
    }
    ServiceRecord s = res.record;
    boolean permissionsReviewRequired = false;

    // If permissions need a review before any of the app components can run,
    // we schedule binding to the service but do not start its process, then
    // we launch a review activity to which is passed a callback to invoke
    // when done to start the bound service's process to completing the binding.
    //如果需要用户手动确认授权
    if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
            s.packageName, s.userId)) {

        permissionsReviewRequired = true;

        // Show a permission review UI only for binding from a foreground app
        //只有调用方进程在前台才可以显示授权弹窗
        if (!callerFg) {
            return 0;
        }

        final ServiceRecord serviceRecord = s;
        final Intent serviceIntent = service;

        //用户手动确认授权后执行的回调
        RemoteCallback callback = new RemoteCallback(
                new RemoteCallback.OnResultListener() {
            @Override
            public void onResult(Bundle result) {
                synchronized(mAm) {
                    final long identity = Binder.clearCallingIdentity();
                    try {
                        if (!mPendingServices.contains(serviceRecord)) {
                            return;
                        }
                        // If there is still a pending record, then the service
                        // binding request is still valid, so hook them up. We
                        // proceed only if the caller cleared the review requirement
                        // otherwise we unbind because the user didn't approve.
                        //二次检查权限
                        if (!mAm.getPackageManagerInternalLocked()
                                .isPermissionsReviewRequired(
                                        serviceRecord.packageName,
                                        serviceRecord.userId)) {
                            try {
                                //拉起服务，如果服务未创建，则会创建服务并调用其onCreate方法
                                //如果服务已创建则什么都不会做
                                bringUpServiceLocked(serviceRecord,
                                        serviceIntent.getFlags(),
                                        callerFg, false, false);
                            } catch (RemoteException e) {
                                /* ignore - local call */
                            }
                        } else {
                            //无相应权限则解绑Service
                            unbindServiceLocked(connection);
                        }
                    } finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                }
            }
        });

        final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_MULTIPLE_TASK
                | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
        intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
        intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);

        //弹出授权弹窗
        mAm.mHandler.post(new Runnable() {
            @Override
            public void run() {
                mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
            }
        });
    }

    final long origId = Binder.clearCallingIdentity();

    try {
        //取消之前的Service重启任务（如果有）
        if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: "
                    + s);
        }

        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            //如果是第一次绑定的话，设置跟踪器
            if (!s.hasAutoCreateConnections()) {
                // This is the first binding, let the tracker know.
                ServiceState stracker = s.getTracker();
                if (stracker != null) {
                    stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                            s.lastActivity);
                }
            }
        }

        //绑定的服务代表受保护的系统组件，因此必须对其应用关联做限制
        if ((flags & Context.BIND_RESTRICT_ASSOCIATIONS) != 0) {
            mAm.requireAllowedAssociationsLocked(s.appInfo.packageName);
        }

        //建立调用方与服务方之间的关联
        mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                callerApp.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
                s.instanceName, s.processName);
        // Once the apps have become associated, if one of them is caller is ephemeral
        // the target app should now be able to see the calling app
        mAm.grantImplicitAccess(callerApp.userId, service,
                callerApp.uid, UserHandle.getAppId(s.appInfo.uid));

        //查询App绑定信息
        AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
        //创建连接信息
        ConnectionRecord c = new ConnectionRecord(b, activity,
                connection, flags, clientLabel, clientIntent,
                callerApp.uid, callerApp.processName, callingPackage);

        IBinder binder = connection.asBinder();
        //添加连接
        s.addConnection(binder, c);
        b.connections.add(c);
        if (activity != null) {
            activity.addConnection(c);
        }
        b.client.connections.add(c);
        //建立关联
        c.startAssociationIfNeeded();
        //表示此服务比发起绑定的应用重要性更高
        if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
            b.client.hasAboveClient = true;
        }
        //允许绑定Service的应用程序管理白名单
        if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
            s.whitelistManager = true;
        }
        //允许Service后台启动Activity
        if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
            s.setHasBindingWhitelistingBgActivityStarts(true);
        }
        //更新是否有与Service建立连接的Activity
        if (s.app != null) {
            updateServiceClientActivitiesLocked(s.app, c, true);
        }
        //更新连接列表
        ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
        if (clist == null) {
            clist = new ArrayList<>();
            mServiceConnections.put(binder, clist);
        }
        clist.add(c);

        //绑定存在就会自动创建服务
        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            //拉起服务，如果服务未创建，则会创建服务并调用其onCreate方法
            //如果服务已创建则什么都不会做
            if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                    permissionsReviewRequired) != null) {
                return 0;
            }
        }

        //检查是否允许前台服务使用while-in-use权限
        if (!s.mAllowWhileInUsePermissionInFgs) {
            s.mAllowWhileInUsePermissionInFgs =
                    shouldAllowWhileInUsePermissionInFgsLocked(callingPackage,
                            Binder.getCallingPid(), Binder.getCallingUid(),
                            service, s, false);
        }

        //更新flag以及进程优先级
        if (s.app != null) {
            if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                s.app.treatLikeActivity = true;
            }
            if (s.whitelistManager) {
                s.app.whitelistManager = true;
            }
            // This could have made the service more important.
            mAm.updateLruProcessLocked(s.app,
                    (callerApp.hasActivitiesOrRecentTasks() && s.app.hasClientActivities())
                            || (callerApp.getCurProcState() <= ActivityManager.PROCESS_STATE_TOP
                                    && (flags & Context.BIND_TREAT_LIKE_ACTIVITY) != 0),
                    b.client);
            mAm.updateOomAdjLocked(s.app, OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
        }

        if (s.app != null && b.intent.received) {
            // Service is already running, so we can immediately
            // publish the connection.
            //如果服务之前就已经在运行，即Service.onBind方法已经被执行，返回的IBinder对象也已经被保存
            //调用LoadedApk$ServiceDispatcher$InnerConnection.connected方法
            //回调ServiceConnection.onServiceConnected方法
            c.conn.connected(s.name, b.intent.binder, false);

            // If this is the first app connected back to this binding,
            // and the service had previously asked to be told when
            // rebound, then do so.
            //当服务解绑，调用到Service.onUnbind方法时返回true，此时doRebind变量就会被赋值为true
            //此时，当再次建立连接时，服务会回调Service.onRebind方法
            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                requestServiceBindingLocked(s, b.intent, callerFg, true);
            }
        } else if (!b.intent.requested) {
            //如果服务是因这次绑定而创建的
            //请求执行Service.onBind方法，获取返回的IBinder对象
            //发布Service，回调ServiceConnection.onServiceConnected方法
            requestServiceBindingLocked(s, b.intent, callerFg, false);
        }

        maybeLogBindCrossProfileService(userId, callingPackage, callerApp.info.uid);

        //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
        //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
        getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    //返回值大于0则视为成功
    return 1;
}
```

这个方法做的事就比较多了，我们挑重点来说：

- 获取或创建各种连接记录（`ActivityServiceConnectionsHolder`、`AppBindRecord`、`ConnectionRecord`等）

- 校验各种`flags`

- 查找相应的`Service`

- 检查是否需要用户手动确认权限并弹出权限确认弹窗

- 向各个连接记录类中添加`ConnectionRecord`连接信息

- 如果`flags`设置了`BIND_AUTO_CREATE`便会调用`bringUpServiceLocked`方法尝试拉起服务，如果服务未创建，则会创建服务并调用其`onCreate`方法，如果服务已创建，则什么都不会做：这里和`startService`路径一样都调用到了`bringUpServiceLocked`方法，但最终调用的结果却不太一样，这是因为`startService`路径中，`ServiceRecord.startRequested`为`true`并且向`ServiceRecord.pendingStarts`中添加了启动项，而`bindService`路径不会向`ServiceRecord.pendingStarts`中添加启动项，并且由于`ServiceRecord.startRequested`为`false`，因此也不会去添加假的启动项，所以和`startService`不同，最终不会回调`Service.onStartCommand`方法

- 如果服务之前就已经在运行，则表示`Service.onBind`方法已经被执行，返回的`IBinder`对象也已经被保存，此时直接调用`LoadedApk$ServiceDispatcher$InnerConnection.connected`方法，在这个方法中会回调`ServiceConnection.onServiceConnected`方法

- 如果服务是因这次绑定而创建的，则调用`requestServiceBindingLocked`方法请求执行`Service.onBind`方法，获取返回的`IBinder`对象，然后发布`Service`，回调`ServiceConnection.onServiceConnected`方法

- 最后调用`ActiveServices$ServiceMap.ensureNotStartingBackgroundLocked`方法继续调度后台`Service`的启动

`bringUpServiceLocked`方法我们之前已经分析过了，我们接下来看服务创建后所要调用的`requestServiceBindingLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
        boolean execInFg, boolean rebind) throws TransactionTooLargeException {
    if (r.app == null || r.app.thread == null) {
        // If service is not currently running, can't yet bind.
        return false;
    }
    if ((!i.requested || rebind) && i.apps.size() > 0) {
        try {
            //记录Service执行操作并设置超时回调
            //前台服务超时时间为20s，后台服务超时时间为200s
            bumpServiceExecutingLocked(r, execInFg, "bind");
            //设置进程状态
            r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //回到App进程，调度执行Service的bind操作
            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                    r.app.getReportedProcState());
            //请求绑定完成
            if (!rebind) {
                i.requested = true;
            }
            i.hasBound = true;
            i.doRebind = false;
        } catch (TransactionTooLargeException e) {
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            throw e;
        } catch (RemoteException e) {
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            return false;
        }
    }
    return true;
}
```

和之前一样，这里也是调用`App`进程中的`ActivityThread$ApplicationThread.scheduleBindService`方法进行绑定操作

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public final void scheduleBindService(IBinder token, Intent intent,
        boolean rebind, int processState) {
    //更新进程信息
    updateProcessState(processState, false);
    BindServiceData s = new BindServiceData();
    s.token = token;
    s.intent = intent;
    s.rebind = rebind;

    sendMessage(H.BIND_SERVICE, s);
}
```


将绑定`Service`的必要信息包装成`BindServiceData`对象后，通过`Handler`依次发送`Message`处理服务启动，这里最终调用的是`ActivityThread.handleBindService`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleBindService(BindServiceData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            try {
                if (!data.rebind) {
                    //正常情况下回调Service.onBind方法，获得控制Service的IBinder对象
                    IBinder binder = s.onBind(data.intent);
                    //发布Service
                    ActivityManager.getService().publishService(
                            data.token, data.intent, binder);
                } else {
                    //当服务解绑，调用到Service.onUnbind方法时返回true，此时doRebind变量就会被赋值为true
                    //此时，当再次建立连接时，服务会回调Service.onRebind方法
                    s.onRebind(data.intent);
                    //Service相关任务执行完成
                    //这一步中会把之前的启动超时定时器取消
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to bind to service " + s
                        + " with " + data.intent + ": " + e.toString(), e);
            }
        }
    }
}
```

在这个方法里面调用了`Service.onBind`方法，获得到了控制`Service`的`IBinder`对象，然后再调用`AMS.publishService`发布服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void publishService(IBinder token, Intent intent, IBinder service) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        if (!(token instanceof ServiceRecord)) {
            throw new IllegalArgumentException("Invalid service token");
        }
        //转交给ActiveServices处理
        mServices.publishServiceLocked((ServiceRecord)token, intent, service);
    }
}

//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (r != null) {
            Intent.FilterComparison filter
                    = new Intent.FilterComparison(intent);
            //获取Intent绑定记录
            IntentBindRecord b = r.bindings.get(filter);
            if (b != null && !b.received) {
                //保存控制Service的IBinder对象
                b.binder = service;
                //请求绑定完成
                b.requested = true;
                //IBinder对象获取完成
                b.received = true;
                //遍历所有与此服务绑定的客户端连接
                ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                for (int conni = connections.size() - 1; conni >= 0; conni--) {
                    ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                    for (int i=0; i<clist.size(); i++) {
                        ConnectionRecord c = clist.get(i);
                        if (!filter.equals(c.binding.intent.intent)) {
                            continue;
                        }
                        //调用LoadedApk$ServiceDispatcher$IServiceConnection.connected方法
                        //回调ServiceConnection.onServiceConnected方法
                        c.conn.connected(r.name, service, false);
                    }
                }
            }
            //Service相关任务执行完成
            //这一步中会把之前的启动超时定时器取消
            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

在这个方法中，首先将通过`Service.onBind`获得到的控制`Service`的`IBinder`对象保存在`IntentBindRecord`中，这样之后再有其他`client`绑定服务，就只需要用它作为参数回调`ServiceConnection.onServiceConnected`方法就可以了

接下来遍历所有与此服务绑定的客户端连接，对符合条件的连接执行`LoadedApk$ServiceDispatcher$IServiceConnection.connected`方法

```java
//frameworks/base/core/java/android/app/LoadedApk.java
public void connected(ComponentName name, IBinder service, boolean dead)
        throws RemoteException {
    //mDispatcher是对ServiceDispatcher的弱引用
    LoadedApk.ServiceDispatcher sd = mDispatcher.get();
    if (sd != null) {
        //调用ServiceDispatcher.connected方法
        sd.connected(name, service, dead);
    }
}

public void connected(ComponentName name, IBinder service, boolean dead) {
    //RunConnection里也是调用了doConnected方法
    if (mActivityExecutor != null) {
        mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
    } else if (mActivityThread != null) {
        mActivityThread.post(new RunConnection(name, service, 0, dead));
    } else {
        doConnected(name, service, dead);
    }
}

public void doConnected(ComponentName name, IBinder service, boolean dead) {
    ServiceDispatcher.ConnectionInfo old;
    ServiceDispatcher.ConnectionInfo info;

    synchronized (this) {
        if (mForgotten) {
            // We unbound before receiving the connection; ignore
            // any connection received.
            return;
        }
        old = mActiveConnections.get(name);
        //如果旧的连接信息中的IBinder对象和本次调用传入的IBinder对象是同一个对象
        if (old != null && old.binder == service) {
            // Huh, already have this one.  Oh well!
            return;
        }

        if (service != null) {
            // A new service is being connected... set it all up.
            //建立一个新的连接信息
            info = new ConnectionInfo();
            info.binder = service;
            info.deathMonitor = new DeathMonitor(name, service);
            try {
                //注册Binder死亡通知
                service.linkToDeath(info.deathMonitor, 0);
                //保存本次连接信息
                mActiveConnections.put(name, info);
            } catch (RemoteException e) {
                // This service was dead before we got it...  just
                // don't do anything with it.
                //服务已死亡，移除连接信息
                mActiveConnections.remove(name);
                return;
            }
        } else {
            // The named service is being disconnected... clean up.
            mActiveConnections.remove(name);
        }

        //移除Binder死亡通知
        if (old != null) {
            old.binder.unlinkToDeath(old.deathMonitor, 0);
        }
    }

    // If there was an old service, it is now disconnected.
    //回调ServiceConnection.onServiceDisconnected
    //通知client之前的连接已被断开
    if (old != null) {
        mConnection.onServiceDisconnected(name);
    }
    //如果Service死亡需要回调ServiceConnection.onBindingDied通知client服务死亡
    if (dead) {
        mConnection.onBindingDied(name);
    }
    // If there is a new viable service, it is now connected.
    if (service != null) {
        //回调ServiceConnection.onServiceConnected方法
        //告知client已建立连接
        mConnection.onServiceConnected(name, service);
    } else {
        // The binding machinery worked, but the remote returned null from onBind().
        //当Service.onBind方法返回null时，回调ServiceConnection.onNullBinding方法
        mConnection.onNullBinding(name);
    }
}
```

可以看到，在这个方法里最终执行了`ServiceConnection.onServiceConnected`回调，通知客户端已与`Service`建立连接

至此，整个`bindService`的流程就结束了

# 总结

`Service`的整个启动流程到这里基本上都分析完了，至于`Service`的停止，重建等流程，我将会在后面的文章中再慢慢分析