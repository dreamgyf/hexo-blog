---
title: Android源码分析 - Service的停止与重建
date: 2023-09-14 11:32:56
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

在上一篇文章 [Android源码分析 - Service启动流程](https://juejin.cn/post/7276363520554795064) 中，我们分析了一个`Service`是怎么启动的，这次我们再来看看一个`Service`是如何被停止的，什么情况下`Service`会被重建以及它的重建过程

# 流程图

# 主动停止

首先，我们来看主动停止的情况，主动停止也分三种：

- `Service`自己调用`stopSelf`或`stopSelfResult`方法停止

- 使用`startService`启动的`Service`，使用`stopService`方法停止

- 使用`bindService`启动的`Service`，使用`unbindService`方法解除绑定，当没有任何`Client`和`Service`绑定时，`Service`就会自行停止

## Service.stopSelf 或 Service.stopSelfResult

`stopSelf`和`stopSelfResult`方法唯一的区别是一个没返回值，一个会返回是否成功

```java
//frameworks/base/core/java/android/app/Service.java
public final void stopSelf() {
    stopSelf(-1);
}

public final void stopSelf(int startId) {
    if (mActivityManager == null) {
        return;
    }
    try {
        mActivityManager.stopServiceToken(
                new ComponentName(this, mClassName), mToken, startId);
    } catch (RemoteException ex) {
    }
}

public final boolean stopSelfResult(int startId) {
    if (mActivityManager == null) {
        return false;
    }
    try {
        return mActivityManager.stopServiceToken(
                new ComponentName(this, mClassName), mToken, startId);
    } catch (RemoteException ex) {
    }
    return false;
}
```

这个方法也是直接调用了`AMS.stopServiceToken`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public boolean stopServiceToken(ComponentName className, IBinder token,
        int startId) {
    synchronized(this) {
        return mServices.stopServiceTokenLocked(className, token, startId);
    }
}
```

然后`AMS`转手调用了`ActiveServices.stopServiceTokenLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
boolean stopServiceTokenLocked(ComponentName className, IBinder token,
        int startId) {
    //通过className查找相应的ServiceRecord
    //在Service启动过程中调用的retrieveServiceLocked方法会查找Service，创建ServiceRecord
    //并将其添加到Map中，findServiceLocked方法只需要从这个Map中去获取即可
    ServiceRecord r = findServiceLocked(className, token, UserHandle.getCallingUserId());
    if (r != null) {
        if (startId >= 0) {
            // Asked to only stop if done with all work.  Note that
            // to avoid leaks, we will take this as dropping all
            // start items up to and including this one.
            //查找startId所对应的已分发的启动项
            ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
            //从已分发启动请求列表中移除
            if (si != null) {
                while (r.deliveredStarts.size() > 0) {
                    ServiceRecord.StartItem cur = r.deliveredStarts.remove(0);
                    cur.removeUriPermissionsLocked();
                    if (cur == si) {
                        break;
                    }
                }
            }

            //如果传入的启动ID不是Service最后一次启动的ID，则不能停止服务
            //ps：每次启动Service，startId都会递增，初始值为1
            if (r.getLastStartId() != startId) {
                return false;
            }
        }

        ... //记录
        //重置启动状态（重要，后文中会分析）
        r.startRequested = false;
        r.callStart = false;
        final long origId = Binder.clearCallingIdentity();
        //接着停止Service
        bringDownServiceIfNeededLocked(r, false, false);
        Binder.restoreCallingIdentity(origId);
        return true;
    }
    return false;
}
```

### startId机制

这里要说一下`Service`的`startId`机制，每次启动`Service`时，`ActiveServices.startServiceLocked`方法会向`ServiceRecord.pendingStarts`列表中添加一个启动项`ServiceRecord.StartItem`，构建这个启动项需要提供一个`startId`，而这个`startId`是由`ServiceRecord.makeNextStartId`生成的

```java
//frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java
public int makeNextStartId() {
    lastStartId++;
    if (lastStartId < 1) {
        lastStartId = 1;
    }
    return lastStartId;
}
```

由于`lastStartId`的初始值为 0 ，所以第一次调用这个方法，得到的`startId`就是 1 ，即`startId`是从 1 开始递增的，由于当`Service`被停止后，`ServiceRecord`会从之前的缓存Map中移除，所以下一次再启动`Service`时会重新创建`ServiceRecord`，`startId`会被重置

当我们调用`stopSelf`停止服务时，如果传入了大于等于 0 的`startId`，此时便会判断这个`startId`是不是最后一次启动所对应的`startId`，如果不是的话，则不能停止这个`Service`

这个`startId`设计的意义是什么呢？我们从`IntentService`的设计中可以管中窥豹：

`IntentService`是一个运行在另一个线程的，每次处理完任务都会自动停止的`Service`，但如果你调用多次`startService`会发现，`IntentService.onDestroy`方法只会调用一次，这是为什么呢？因为`IntentService`每次停止，调用`stopSelf`方法都是带上这次启动的`startId`的，这样如果一次性有多个启动请求，前面的任务执行完，停止时发现，此次启动请求的`startId`不是最后一个`startId`，这样就不会停止掉自身，直到最后一个任务处理完成，避免了`Service`的多次停止启动消耗系统资源

---

接着，在这个方法的最后，调用`bringDownServiceIfNeededLocked`方法继续停止服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
        boolean hasConn) {
    //检查此服务是否还被需要
    //在用stopSelf停止服务的这种情况下
    //检查的就是是否有auto-create的连接（flag为BIND_AUTO_CREATE）
    //如有则不能停止服务
    if (isServiceNeededLocked(r, knowConn, hasConn)) {
        return;
    }

    // Are we in the process of launching?
    //不要停止正在启动中的Service
    if (mPendingServices.contains(r)) {
        return;
    }

    //继续停止服务
    bringDownServiceLocked(r);
}
```

这个方法主要就做了一些能否停止服务的检查，主要的停止操作都在下一个`bringDownServiceLocked`方法中

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void bringDownServiceLocked(ServiceRecord r) {
    ... //处理Client与Serivce的连接，进行断开连接以及解除绑定操作

    // Check to see if the service had been started as foreground, but being
    // brought down before actually showing a notification.  That is not allowed.
    //如果此Service是以前台服务的形式启动，并且当前还尚未成为前台服务
    if (r.fgRequired) {
        r.fgRequired = false;
        r.fgWaiting = false;
        ... //记录
        //将前台服务的超时回调取消
        mAm.mHandler.removeMessages(
                ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG, r);
        //这种情况直接令App崩溃，杀死应用
        if (r.app != null) {
            Message msg = mAm.mHandler.obtainMessage(
                    ActivityManagerService.SERVICE_FOREGROUND_CRASH_MSG);
            msg.obj = r.app;
            msg.getData().putCharSequence(
                ActivityManagerService.SERVICE_RECORD_KEY, r.toString());
            mAm.mHandler.sendMessage(msg);
        }
    }

    //记录销毁时间
    r.destroyTime = SystemClock.uptimeMillis();

    //从缓存中移除ServiceRecord
    final ServiceMap smap = getServiceMapLocked(r.userId);
    ServiceRecord found = smap.mServicesByInstanceName.remove(r.instanceName);

    // Note when this method is called by bringUpServiceLocked(), the service is not found
    // in mServicesByInstanceName and found will be null.
    if (found != null && found != r) {
        // This is not actually the service we think is running...  this should not happen,
        // but if it does, fail hard.
        //如果找到的服务不是我们目前停止的服务，应该是一个不可能的情况
        //碰到这种情况，将ServiceRecord重新放回去并抛出异常
        smap.mServicesByInstanceName.put(r.instanceName, found);
        throw new IllegalStateException("Bringing down " + r + " but actually running "
                + found);
    }
    //清除ServiceRecord
    smap.mServicesByIntent.remove(r.intent);
    r.totalRestartCount = 0;
    //取消之前的Service重启任务（如果有）
    unscheduleServiceRestartLocked(r, 0, true);

    // Also make sure it is not on the pending list.
    //从待启动Service列表中移除
    for (int i=mPendingServices.size()-1; i>=0; i--) {
        if (mPendingServices.get(i) == r) {
            mPendingServices.remove(i);
        }
    }

    //关闭前台服务通知
    cancelForegroundNotificationLocked(r);
    //对于已经成为前台服务的Service
    if (r.isForeground) {
        //修改应用的活动前台计数，如果计数小于等于0，将其从mActiveForegroundApps列表中移除
        decActiveForegroundAppLocked(smap, r);
        ... //更新统计信息
    }

    //各种清理操作
    r.isForeground = false;
    r.foregroundId = 0;
    r.foregroundNoti = null;
    r.mAllowWhileInUsePermissionInFgs = false;

    // Clear start entries.
    r.clearDeliveredStartsLocked();
    r.pendingStarts.clear();
    smap.mDelayedStartList.remove(r);

    if (r.app != null) {
        synchronized (r.stats.getBatteryStats()) {
            r.stats.stopLaunchedLocked();
        }
        //从ProcessRecord中移除Service记录
        r.app.stopService(r);
        //更新服务进程绑定的应用uids
        r.app.updateBoundClientUids();
        //允许绑定Service的应用程序管理白名单
        if (r.whitelistManager) {
            updateWhitelistManagerLocked(r.app);
        }
        if (r.app.thread != null) {
            //更新进程前台服务信息
            updateServiceForegroundLocked(r.app, false);
            try {
                //记录Service执行操作并设置超时回调
                //前台服务超时时间为20s，后台服务超时时间为200s
                bumpServiceExecutingLocked(r, false, "destroy");
                //添加到销毁中Service列表中
                mDestroyingServices.add(r);
                //标记正在销毁中
                r.destroying = true;
                //更新进程优先级
                mAm.updateOomAdjLocked(r.app, true,
                        OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                //回到App进程，调度执行Service的stop操作
                r.app.thread.scheduleStopService(r);
            } catch (Exception e) {
                serviceProcessGoneLocked(r);
            }
        }
    }

    //清除连接
    if (r.bindings.size() > 0) {
        r.bindings.clear();
    }

    //对于主动停止的Service，不需要重启 TODO
    if (r.restarter instanceof ServiceRestarter) {
        ((ServiceRestarter)r.restarter).setService(null);
    }

    ... //记录

    //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
    //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
    smap.ensureNotStartingBackgroundLocked(r);
}
```

这个方法主要做了以下几件事：

1. 处理`Client`与`Serivce`的连接，进行断开连接以及解除绑定操作（具体等到后面分析`unbindService`时再说）
2. 对于以前台服务形式启动，并且当前还尚未成为前台服务的`Service`，直接杀死App
3. 各种重置，清理操作
4. 关闭前台服务通知
5. 将`Service`添加到销毁中服务列表，并调度执行停止操作，最终回调`Service.onDestroy`
6. 将`ServiceRestarter`内的`ServiceRecord`变量设为`null`，避免其后续重启

我们重点看第5步，在这个方法中调用了`ActivityThread$ApplicationThread.scheduleStopService`去调度执行停止服务操作

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public final void scheduleStopService(IBinder token) {
    sendMessage(H.STOP_SERVICE, token);
}
```

同样的，也是通过`Handler`发送`Message`处理服务停止，这里最终调用的是`ActivityThread.handleStopService`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleStopService(IBinder token) {
    //从Service列表中取出并移除此服务
    Service s = mServices.remove(token);
    if (s != null) {
        try {
            //调用Service.onDestroy
            s.onDestroy();
            //解绑以及清理（解除对ServiceRecord的Binder远程对象的引用）
            s.detachAndCleanUp();
            //执行清理操作，具体来说就是断开其他客户端与Service的连接以及解除绑定
            Context context = s.getBaseContext();
            if (context instanceof ContextImpl) {
                final String who = s.getClassName();
                ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
            }

            //确保其他异步任务执行完成
            QueuedWork.waitToFinish();

            try {
                //Service相关任务执行完成
                //这一步中会把之前的超时定时器取消
                ActivityManager.getService().serviceDoneExecuting(
                        token, SERVICE_DONE_EXECUTING_STOP, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to stop service " + s
                        + ": " + e.toString(), e);
            }
        }
    }
}
```

可以看到，这里首先从`mServices`列表中取出并移除此服务，然后触发`Service.onDestroy`回调，之后还需要调用`ContextImpl.scheduleFinalCleanup`方法执行一些清理工作，这一部分的分析我们留到后面`unbindService`章节里再讲，这样，整个`Service`的停止流程就到此结束了

## stopService

接下来我们来看一下调用方调用`stopService`停止服务的情况

```java
//frameworks/base/core/java/android/app/ContextImpl.java
public boolean stopService(Intent service) {
    warnIfCallingFromSystemProcess();
    return stopServiceCommon(service, mUser);
}

private boolean stopServiceCommon(Intent service, UserHandle user) {
    try {
        //验证Intent有效性
        validateServiceIntent(service);
        //跨进程处理
        service.prepareToLeaveProcess(this);
        //调用AMS.stopService
        int res = ActivityManager.getService().stopService(
            mMainThread.getApplicationThread(), service,
            service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to stop service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

这里基本上是直接调用`AMS.stopService`进入系统进程处理服务停止

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public int stopService(IApplicationThread caller, Intent service,
        String resolvedType, int userId) {
    enforceNotIsolatedCaller("stopService");
    // Refuse possible leaked file descriptors
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        //转交给ActiveServices处理
        return mServices.stopServiceLocked(caller, service, resolvedType, userId);
    }
}

//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
int stopServiceLocked(IApplicationThread caller, Intent service,
        String resolvedType, int userId) {
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
    if (caller != null && callerApp == null) {
        throw new SecurityException(
                "Unable to find app for caller " + caller
                + " (pid=" + Binder.getCallingPid()
                + ") when stopping service " + service);
    }

    // If this service is active, make sure it is stopped.
    //查找相应的Service，其中入参createIfNeeded为false，所以如果从缓存中找不到ServiceRecord的话则会直接返回null
    ServiceLookupResult r = retrieveServiceLocked(service, null, resolvedType, null,
            Binder.getCallingPid(), Binder.getCallingUid(), userId, false, false, false, false);
    if (r != null) {
        if (r.record != null) {
            final long origId = Binder.clearCallingIdentity();
            try {
                //接着处理停止服务
                stopServiceLocked(r.record);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return 1;
        }
        return -1;
    }

    return 0;
}

private void stopServiceLocked(ServiceRecord service) {
    if (service.delayed) {
        // If service isn't actually running, but is being held in the
        // delayed list, then we need to keep it started but note that it
        // should be stopped once no longer delayed.
        service.delayedStop = true;
        return;
    }
    ... //统计信息记录

    //重置启动状态（重要，后文中会分析）
    service.startRequested = false;
    service.callStart = false;

    //和上文一样，停止Service
    bringDownServiceIfNeededLocked(service, false, false);
}
```

可以看到，这里和上文中分析的以`stopSelf`方式停止服务一样，先重置启动状态，然后调用`bringDownServiceIfNeededLocked`停止服务

## unbindService

接下来的是通过`bindService`方法，并且`flag`为`BIND_AUTO_CREATE`启动的`Service`，我们需要通过`unbindService`方法解除绑定，当最终没有任何`flag`为`BIND_AUTO_CREATE`的客户端与`Service`绑定，这个`Service`就会被停止

```java
//frameworks/base/core/java/android/app/ContextImpl.java
public void unbindService(ServiceConnection conn) {
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        //将ServiceDispatcher的状态设置为forgotten，之后便不再会回调ServiceConnection任何方法
        IServiceConnection sd = mPackageInfo.forgetServiceDispatcher(
                getOuterContext(), conn);
        try {
            //调用AMS.unbindService方法
            ActivityManager.getService().unbindService(sd);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } else {
        throw new RuntimeException("Not supported in system context");
    }
}
```

这个方法中首先调用了`LoadedApk.forgetServiceDispatcher`方法

```java
//frameworks/base/core/java/android/app/LoadedApk.java
public final IServiceConnection forgetServiceDispatcher(Context context,
        ServiceConnection c) {
    synchronized (mServices) {
        ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map
                = mServices.get(context);
        LoadedApk.ServiceDispatcher sd = null;
        if (map != null) {
            sd = map.get(c);
            if (sd != null) {
                //移除ServiceDispatcher
                map.remove(c);
                //清理连接并标记遗忘
                sd.doForget();
                if (map.size() == 0) {
                    mServices.remove(context);
                }
                ... //debug
                return sd.getIServiceConnection();
            }
        }
        ... //debug
        ... //异常
    }
}
```

将`ServiceDispatcher`从缓存中移除并调用`LoadedApk$ServiceDispatcher.doForget`方法

```java
//frameworks/base/core/java/android/app/LoadedApk.java
void doForget() {
    synchronized(this) {
        for (int i=0; i<mActiveConnections.size(); i++) {
            ServiceDispatcher.ConnectionInfo ci = mActiveConnections.valueAt(i);
            ci.binder.unlinkToDeath(ci.deathMonitor, 0);
        }
        mActiveConnections.clear();
        mForgotten = true;
    }
}
```

这里将所有连接的`binder`死亡回调移除，然后清除所有连接，再将`mForgotten`标记设为`true`

接着我们会走到`AMS.unbindService`方法中

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public boolean unbindService(IServiceConnection connection) {
    synchronized (this) {
        return mServices.unbindServiceLocked(connection);
    }
}
```

同样的，将工作转交给`ActiveServices`

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
boolean unbindServiceLocked(IServiceConnection connection) {
    IBinder binder = connection.asBinder();
    ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
    //找不到连接记录，直接返回
    if (clist == null) {
        return false;
    }

    final long origId = Binder.clearCallingIdentity();
    try {
        //遍历连接
        while (clist.size() > 0) {
            ConnectionRecord r = clist.get(0);
            //移除连接
            removeConnectionLocked(r, null, null);
            //removeConnectionLocked方法会将此ConnectionRecord从连接列表中移除
            //如果此ConnectionRecord仍然存在的话，是一个严重的错误，这里再移除一次
            if (clist.size() > 0 && clist.get(0) == r) {
                // In case it didn't get removed above, do it now.
                Slog.wtf(TAG, "Connection " + r + " not removed for binder " + binder);
                clist.remove(0);
            }

            if (r.binding.service.app != null) {
                if (r.binding.service.app.whitelistManager) {
                    updateWhitelistManagerLocked(r.binding.service.app);
                }
                // This could have made the service less important.
                if ((r.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    r.binding.service.app.treatLikeActivity = true;
                    mAm.updateLruProcessLocked(r.binding.service.app,
                            r.binding.service.app.hasClientActivities()
                            || r.binding.service.app.treatLikeActivity, null);
                }
            }
        }

        //更新顶层应用进程优先级
        mAm.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);

    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    return true;
}
```

没看到什么特别重要的逻辑，看来重点应该在`removeConnectionLocked`这个方法中了

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
void removeConnectionLocked(ConnectionRecord c, ProcessRecord skipApp,
        ActivityServiceConnectionsHolder skipAct) {
    IBinder binder = c.conn.asBinder();
    AppBindRecord b = c.binding;
    ServiceRecord s = b.service;
    //这里的clist是ServiceRecord中的列表，和上一个方法中的clist不是一个对象
    ArrayList<ConnectionRecord> clist = s.getConnections().get(binder);
    //移除各种连接
    if (clist != null) {
        clist.remove(c);
        if (clist.size() == 0) {
            s.removeConnection(binder);
        }
    }
    b.connections.remove(c);
    c.stopAssociation();
    if (c.activity != null && c.activity != skipAct) {
        c.activity.removeConnection(c);
    }
    if (b.client != skipApp) {
        b.client.connections.remove(c);
        ... //各种flag的处理
        //更新是否有与Service建立连接的Activity
        if (s.app != null) {
            updateServiceClientActivitiesLocked(s.app, c, true);
        }
    }
    //将连接从mServiceConnections列表中移除
    //这个clist才是和上一个方法是同一个对象
    clist = mServiceConnections.get(binder);
    if (clist != null) {
        clist.remove(c);
        if (clist.size() == 0) {
            mServiceConnections.remove(binder);
        }
    }

    mAm.stopAssociationLocked(b.client.uid, b.client.processName, s.appInfo.uid,
            s.appInfo.longVersionCode, s.instanceName, s.processName);

    //如果调用方App没有其他连接和Service绑定
    //则将整个AppBindRecord移除
    if (b.connections.size() == 0) {
        b.intent.apps.remove(b.client);
    }

    if (!c.serviceDead) {
        //如果服务端进程存活并且没有其他连接绑定了，同时服务还处在绑定关系中（尚未回调过Service.onUnbind）
        if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0
                && b.intent.hasBound) {
            try {
                bumpServiceExecutingLocked(s, false, "unbind");
                if (b.client != s.app && (c.flags&Context.BIND_WAIVE_PRIORITY) == 0
                        && s.app.setProcState <= ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) {
                    // If this service's process is not already in the cached list,
                    // then update it in the LRU list here because this may be causing
                    // it to go down there and we want it to start out near the top.
                    mAm.updateLruProcessLocked(s.app, false, null);
                }
                mAm.updateOomAdjLocked(s.app, true,
                        OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                //标记为未绑定
                b.intent.hasBound = false;
                // Assume the client doesn't want to know about a rebind;
                // we will deal with that later if it asks for one.
                b.intent.doRebind = false;
                //回到App进程，调度执行Service的unbind操作
                s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
            } catch (Exception e) {
                serviceProcessGoneLocked(s);
            }
        }

        // If unbound while waiting to start and there is no connection left in this service,
        // remove the pending service
        if (s.getConnections().isEmpty()) {
            mPendingServices.remove(s);
        }

        if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
            //是否有其他含有BIND_AUTO_CREATE标记的连接
            boolean hasAutoCreate = s.hasAutoCreateConnections();
            ... //记录
            bringDownServiceIfNeededLocked(s, true, hasAutoCreate);
        }
    }
}
```

这个方法主要做了以下几件事：

1. 执行各种移除操作

2. 如果目标服务在此次解绑后不再有任何其他连接与其绑定，则调度执行`Service`的`unbind`操作

3. 如果此次断开的连接的`flag`中包含`BIND_AUTO_CREATE`，则调用`bringDownServiceIfNeededLocked`尝试停止服务

2、3两点都很重要，我们首先看第2点，什么情况下会在这里调度执行`Service`的`unbind`操作，前面描述的其实不是很准确，准确的来说应该是目标服务在同一个`IntentBindRecord`下，此次解绑后不再有任何其他连接与其绑定。那么什么叫同一个`IntentBindRecord`呢？这和我们启动服务传入的`Intent`有关，`IntentBindRecord`的第一次创建是在我们调用`bindService`后，走到`ActiveServices.bindServiceLocked`方法中，其中有一段代码调用了`ServiceRecord.retrieveAppBindingLocked`方法产生的

```java
//frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java
public AppBindRecord retrieveAppBindingLocked(Intent intent,
        ProcessRecord app) {
    Intent.FilterComparison filter = new Intent.FilterComparison(intent);
    IntentBindRecord i = bindings.get(filter);
    if (i == null) {
        i = new IntentBindRecord(this, filter);
        bindings.put(filter, i);
    }
    AppBindRecord a = i.apps.get(app);
    if (a != null) {
        return a;
    }
    a = new AppBindRecord(this, i, app);
    i.apps.put(app, a);
    return a;
}
```

在这个方法中，它将我们传入的`Intent`包装成了`Intent.FilterComparison`对象，然后尝试用它作为`key`从`ArrayMap``bindings`中取获取`IntentBindRecord`，如果获取不到则会创建一个新的，那么是否是同一个`IntentBindRecord`的判断标准就是包装后的`Intent.FilterComparison`对象的`HashCode`是否相等，我们来看一下它的`HashCode`是怎样计算的：

```java
//frameworks/base/core/java/android/content/Intent.java
public static final class FilterComparison {
    private final Intent mIntent;
    private final int mHashCode;

    public FilterComparison(Intent intent) {
        mIntent = intent;
        mHashCode = intent.filterHashCode();
    }
    ...
    @Override
    public int hashCode() {
        return mHashCode;
    }
}

public int filterHashCode() {
    int code = 0;
    if (mAction != null) {
        code += mAction.hashCode();
    }
    if (mData != null) {
        code += mData.hashCode();
    }
    if (mType != null) {
        code += mType.hashCode();
    }
    if (mIdentifier != null) {
        code += mIdentifier.hashCode();
    }
    if (mPackage != null) {
        code += mPackage.hashCode();
    }
    if (mComponent != null) {
        code += mComponent.hashCode();
    }
    if (mCategories != null) {
        code += mCategories.hashCode();
    }
    return code;
}
```

可以看到，只有以上参数全部相等，才会被视为同一个`Intent`，而我们通常使用`Intent`只会设置它的`mComponent`，所以在一般情况下`Service`的`onBind`和`onUnbind`也只会触发一次（在`Service`没有被销毁的情况下）

接着我们来看第3点，如果此次断开的连接的`flag`中包含`BIND_AUTO_CREATE`，首先会去查询是否有其他含有`BIND_AUTO_CREATE`标记的连接，然后以此作为参数调用`bringDownServiceIfNeededLocked`尝试停止服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
        boolean hasConn) {
    //检查此服务是否还被需要
    if (isServiceNeededLocked(r, knowConn, hasConn)) {
        return;
    }

    // Are we in the process of launching?
    //不要停止正在启动中的Service
    if (mPendingServices.contains(r)) {
        return;
    }

    //继续停止服务
    bringDownServiceLocked(r);
}

private final boolean isServiceNeededLocked(ServiceRecord r, boolean knowConn,
        boolean hasConn) {
    // Are we still explicitly being asked to run?
    //Service之前是否通过startService启动过并且未stop
    if (r.startRequested) {
        return true;
    }

    // Is someone still bound to us keeping us running?
    //这里我们传入的是true
    //因为我们之前已经做了检查，知道了是否还有其他auto-create的连接
    if (!knowConn) {
        hasConn = r.hasAutoCreateConnections();
    }
    //如果还有其他auto-create的连接
    //则此服务还被需要
    if (hasConn) {
        return true;
    }

    return false;
}
```

可以看到，经过上述检查，如果发现此`Service`确实可以被停止了，则会调用`bringDownServiceLocked`方法停止服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void bringDownServiceLocked(ServiceRecord r) {
    //断开所有连接
    ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
    for (int conni = connections.size() - 1; conni >= 0; conni--) {
        ArrayList<ConnectionRecord> c = connections.valueAt(conni);
        for (int i=0; i<c.size(); i++) {
            ConnectionRecord cr = c.get(i);
            // There is still a connection to the service that is
            // being brought down.  Mark it as dead.
            //将服务标记为死亡
            cr.serviceDead = true;
            cr.stopAssociation();
            //回调ServiceConnection各种方法
            //通知client服务断开连接以及死亡
            cr.conn.connected(r.name, null, true);
        }
    }

    // Tell the service that it has been unbound.
    //通知Service解除绑定
    if (r.app != null && r.app.thread != null) {
        boolean needOomAdj = false;
        //遍历所有连接，解除绑定
        for (int i = r.bindings.size() - 1; i >= 0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            //如果还处在绑定关系中（尚未回调过Service.onUnbind）
            if (ibr.hasBound) {
                try {
                    //记录Service执行操作并设置超时回调
                    //前台服务超时时间为20s，后台服务超时时间为200s
                    bumpServiceExecutingLocked(r, false, "bring down unbind");
                    needOomAdj = true;
                    //标记为未绑定
                    ibr.hasBound = false;
                    ibr.requested = false;
                    //回到App进程，调度执行Service的unbind操作
                    r.app.thread.scheduleUnbindService(r,
                            ibr.intent.getIntent());
                } catch (Exception e) {
                    needOomAdj = false;
                    serviceProcessGoneLocked(r);
                    break;
                }
            }
        }
        //更新服务端进程优先级
        if (needOomAdj) {
            mAm.updateOomAdjLocked(r.app, true,
                    OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
        }
    }
    ... //和上文相同
}
```

这个方法我们在前面分析`stopSelf`的时候说过了，这次我们只看和绑定服务有关的部分

首先断开所有连接，回调`ServiceConnection`各种方法，通知客户端服务断开连接以及死亡，这里需要注意的是，我们本次执行`unbindService`操作的连接已经在上一步中从`ServiceRecord.connections`列表中移除，所以并不会回调它的`ServiceConnection`的任何方法，这也是很多人对`unbindService`方法的误解（包括我自己），`bindService`方法在成功绑定服务后会回调`ServiceConnection.onServiceConnected`方法，但`unbindService`方法在成功解绑服务后并不会回调`ServiceConnection.onServiceDisconnected`以及任何其它方法，这些方法只会在`Service`被以其他方式停止（比如后面会分析的混合启动的服务如何停止）或者`Service`意外停止（比如服务端应用崩溃或被杀死）的情况才会被调用

所以这里处理的是断开其他的连接，我们假设一个场景，使用同一个`Intent`和两个不同的`ServiceConnection`，一个使用`BIND_AUTO_CREATE`标记，一个使用其他标记，先绑定`BIND_AUTO_CREATE`标记的`Service`，然后再绑定其他标记的`Service`，接着我们对`BIND_AUTO_CREATE`标记的`Serivce`调用`unbindService`解绑，此时就会走到这个方法中，`ServiceRecord.connections`列表中会存在那个使用其他标记的连接，然后其内部成员变量`conn`的`connected`方法，这个`conn`是一个`IServiceConnection`类型，实际上的实现类为`LoadedApk$ServiceDispatcher$InnerConnection`，最终会调用到`LoadedApk$ServiceDispatcher.doConnected`方法

```java
public void doConnected(ComponentName name, IBinder service, boolean dead) {
    ServiceDispatcher.ConnectionInfo old;
    ServiceDispatcher.ConnectionInfo info;

    synchronized (this) {
        //被标记为遗忘则不处理任何事情
        //调用unbindService就会将这个标志设为true
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
        //当Service.onBind方法返回null，或者Service停止时
        //回调ServiceConnection.onNullBinding方法
        mConnection.onNullBinding(name);
    }
}
```

这个方法其实我们在上一篇文章中已经说过了，不过在上一篇文章中我们关注的是`Service`绑定的部分，而这次我们关注的是解绑的部分

首先映入眼帘的就是对`mForgotten`变量的判断，它在客户端调用`unbindService`就会被设为`true`，然后便会直接返回，不再处理后续事项。当然，实际上执行完`unbindService`方法后，客户端与`Service`的连接会被移除，理论上应该也不会再走到这个方法里才对（这里我也感觉有点疑惑）

根据这段代码，我们能看出来`Service`停止后，对客户端的回调是什么：

- 当`Service.onBind`方法的返回不为`null`时，此时会依次回调`ServiceConnection.onServiceDisconnected`、`ServiceConnection.onBindingDied`和`ServiceConnection.onNullBinding`方法

- 当`Service.onBind`方法的返回为`null`时，此时会依次回调`ServiceConnection.onBindingDied`和`ServiceConnection.onNullBinding`方法

大家也可以自己写写`Demo`来检验一下我说的是否正确

这一步处理完后，接下来要做的便是处理`Service`那边的解绑，遍历`IntentBindRecord`列表，调用`ActivityThread$ApplicationThread.scheduleUnbindService`去调度执行服务解绑操作，这里通过`Handler`最终调用的是`ActivityThread.handleUnbindService`方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleUnbindService(BindServiceData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            //回调Service.onUnbind方法，如果返回值为true
            //当再次建立连接时，服务会回调Service.onRebind方法
            boolean doRebind = s.onUnbind(data.intent);
            try {
                if (doRebind) {
                    ActivityManager.getService().unbindFinished(
                            data.token, data.intent, doRebind);
                } else {
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to unbind to service " + s
                        + " with " + data.intent + ": " + e.toString(), e);
            }
        }
    }
}
```

# 混合启动的Service该如何停止