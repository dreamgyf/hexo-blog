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

由于`Service`的停止与重建在实际情况下会收到各种条件影响产生不同的情况，所以这里的流程图我也只画最简路径，以供大家参考

![service主动停止流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E7%9A%84%E5%81%9C%E6%AD%A2%E4%B8%8E%E9%87%8D%E5%BB%BA_service%E4%B8%BB%E5%8A%A8%E5%81%9C%E6%AD%A2%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

![service主动解绑流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E7%9A%84%E5%81%9C%E6%AD%A2%E4%B8%8E%E9%87%8D%E5%BB%BA_service%E4%B8%BB%E5%8A%A8%E8%A7%A3%E7%BB%91%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

![Task移除流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E7%9A%84%E5%81%9C%E6%AD%A2%E4%B8%8E%E9%87%8D%E5%BB%BA_Task%E7%A7%BB%E9%99%A4%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

![service重建流程图](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Service%E7%9A%84%E5%81%9C%E6%AD%A2%E4%B8%8E%E9%87%8D%E5%BB%BA_service%E9%87%8D%E5%BB%BA%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

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

    //对于主动停止的Service，不需要重启
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

可以看到，这里回调了`Service.onUnbind`方法，它的返回值表示当`Service`后面再与其他客户端建立连接时，是否需要回调`Service.onRebind`方法，但是这有一个前提，那就是`Service`中途没有被停止，具体原因以及`rebind`流程我们稍后再分析

到了这一步，解绑就完成了，接下来和之前在`stopSelf`章节里分析的后续流程就一样了，最终调用`ActivityThread.handleStopService`方法停止服务，还记得我们之前分析在这个方法中触发完`Service.onDestroy`回调，之后还需要调用`ContextImpl.scheduleFinalCleanup`方法吗？现在我们就来看看这个方法又做了什么事情

```java
//frameworks/base/core/java/android/app/ContextImpl.java
final void scheduleFinalCleanup(String who, String what) {
    mMainThread.scheduleContextCleanup(this, who, what);
}
```

这里的`mMainThread`就是应用的`ActivityThread`

```java
//frameworks/base/core/java/android/app/ActivityThread.java
final void scheduleContextCleanup(ContextImpl context, String who,
        String what) {
    ContextCleanupInfo cci = new ContextCleanupInfo();
    cci.context = context;
    cci.who = who;
    cci.what = what;
    sendMessage(H.CLEAN_UP_CONTEXT, cci);
}
```

还是老样子通过`Handler`发消息，最终调用的是`ContextImpl.performFinalCleanup`方法

```java
//frameworks/base/core/java/android/app/ContextImpl.java
final void performFinalCleanup(String who, String what) {
    mPackageInfo.removeContextRegistrations(getOuterContext(), who, what);
}
```

然后调用`LoadedApk.removeContextRegistrations`方法执行清理操作

```java
//frameworks/base/core/java/android/app/LoadedApk.java
public void removeContextRegistrations(Context context,
        String who, String what) {
    ... //清理广播接收器

    synchronized (mServices) {
        ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> smap =
                mServices.remove(context);
        if (smap != null) {
            for (int i = 0; i < smap.size(); i++) {
                LoadedApk.ServiceDispatcher sd = smap.valueAt(i);
                ... //报告ServiceConnection泄露
                try {
                    ActivityManager.getService().unbindService(
                            sd.getIServiceConnection());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
                sd.doForget();
            }
        }
        mUnboundServices.remove(context);
    }
}
```

可能看到这里，有很多小伙伴会有疑问，为什么之前已经清理过了，这里还要再进行清理、这里为什么会有`ServiceConnection`泄露、这里为什么还要再次`unbindService`？那我们需要注意了，这里被调用的对象到底是谁？其实是`Service`的`Context`，假设我们在`Serivce`里调用`bindService`又绑定了一个其他`Service`，那这个`Service`被销毁后，它和另一个`Service`的连接怎么办？是不是就产生了泄露？为了防止这种情况，所以我们需要在`Service`销毁时调用一下这个方法解除它与其他`Service`的绑定

而且这个方法不仅会在这里被调用到哦，在我们之前分析过的 [Android源码分析 - Activity销毁流程](https://juejin.cn/post/7216142711614799930) 中，也存在它的身影，当`Activity`被销毁，走到`handleDestroyActivity`方法时，会调用到我们`ContextImpl.scheduleFinalCleanup`方法，进行广播接收器的清理以及服务的解绑

至此，`Service`的主动停止流程我们都分析完了，还有一些细枝末节的事情可以说一说

# rebind流程

之前我们说过，`rebind`的前提是`Service`中途没有被停止，为什么呢？带着疑问，我们来看之前没有分析的，当`Service.onUnbind`方法返回值为`true`时，会调用的`AMS.unbindFinished`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void unbindFinished(IBinder token, Intent intent, boolean doRebind) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        mServices.unbindFinishedLocked((ServiceRecord)token, intent, doRebind);
    }
}
```

还是转交给`ActiveServices`去实现

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
void unbindFinishedLocked(ServiceRecord r, Intent intent, boolean doRebind) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (r != null) {
            Intent.FilterComparison filter
                    = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter);

            boolean inDestroying = mDestroyingServices.contains(r);
            if (b != null) {
                //服务unbind的前提就是IntentBindRecord.apps.size == 0
                if (b.apps.size() > 0 && !inDestroying) {
                    ...
                } else {
                    // Note to tell the service the next time there is
                    // a new client.
                    //将doRebind标记置为true，下一次再次建立连接时
                    //服务会回调Service.onRebind方法
                    b.doRebind = true;
                }
            }

            serviceDoneExecutingLocked(r, inDestroying, false);
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

根据我们之前的分析，我们知道，回调`Service.onUnbind`的前提就是这个`Service`没有任何连接与其绑定了，即`IntentBindRecord.apps.size == 0`，在这个case下，这个方法会将`IntentBindRecord`的`doRebind`变量置为`true`

此时让我们再回顾一下上一篇文章 [Android源码分析 - Service启动流程](https://juejin.cn/post/7276363520554795064#heading-10) 中分析的`bindService`流程，在`ActiveServices.bindServiceLocked`方法中有这么一段代码

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection, int flags,
        String instanceName, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    ...
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
    ...
}
```

如果`Service`被停止了，那么它相应的`ServiceRecord`会从缓存`mServicesByInstanceName`和`mServicesByIntent`中移除，那么等到重新启动`Service`时会新建出一个`ServiceRecord`，此时里面的变量全部被初始化，`b.intent.received == false`，`b.intent.requested == false`，`b.intent.doRebind == false`，在这种情况下，调用`requestServiceBindingLocked`方法的最后一个入参`rebind`为`false`，就会直接回调`Service.onBind`方法，而不会回调`Service.onRebind`方法

如果`Service`没有被停止，且之前有被绑定过，那么`b.intent.received == true`，代表`IBinder`对象已获取到，此时如果之前的`Service.onUnbind`回调返回值为`true`，那么这里的`b.intent.doRebind`也为`true`，再加上如果这是`Service`断开所有连接后建立的第一次连接，即`b.intent.apps.size() == 1`，那么此时调用的`requestServiceBindingLocked`方法的最后一个入参`rebind`为`true`，就会直接回调`Service.onRebind`方法，而不会回调`Service.onBind`方法

# 混合启动的Service该如何停止

单一启动方式的`Service`的停止很简单，那么混合启动的`Service`该如何停止呢？

何为混合启动？指的是通过`startService`方式启动`Service`并且以`BIND_AUTO_CREATE`标志调用`bindService`方法绑定`Service`，两者不分先后

在上面的分析中，我们注意到，不管用哪种方式停止服务，最后都会走到`bringDownServiceIfNeededLocked`方法中，在这个方法里又会调用`isServiceNeededLocked`判断是否`Service`是否被需要，那被需要的条件是什么呢？`Service.startRequested`为`true`，并且没有标志为`BIND_AUTO_CREATE`的连接绑定，那么，只要不符合这两个条件，服务自然就可以被停止了

没有标志为`BIND_AUTO_CREATE`的连接绑定这个简单，只需要把标记为`BIND_AUTO_CREATE`的连接全部解绑了就好，那么怎么让`Service.startRequested`为`false`呢？我们回顾一下之前对`stopSelf`和`stopService`的分析，在他们调用`bringDownServiceIfNeededLocked`方法之前，都会先将`Service.startRequested`设置为`false`，所以答案就很明显了：只要`unbindService`掉所有`BIND_AUTO_CREATE`的标志的连接，然后`stopService`就能停止掉混合启动的服务，当然你也可以先`stopService`，再`unbindService`掉所有`BIND_AUTO_CREATE`的标志的连接

# 小测验

经过以上一系列的分析，我给大家出几个小问题：

1. 先通过`startService`启动服务，然后再用`BIND_AUTO_CREATE`标志`bindService`（连接1），此时，`Service`的生命周期是怎样的？`ServiceConnection`会回调哪些方法？

2. 在上一题的基础上，再使用一个非`BIND_AUTO_CREATE`标志`bindService`（连接2），此时，`Service`的生命周期是怎样的？`ServiceConnection`会回调哪些方法？

3. 此时，使用`unbindService`解绑连接1，会发生什么？`ServiceConnection`会回调哪些方法？

4. 此时，使用`unbindService`解绑连接2，会发生什么？`ServiceConnection`会回调哪些方法？

5. 接着，再使用连接1`bindService`，会发生什么？`ServiceConnection`会回调哪些方法？

6. 然后，调用`stopService`停止服务，服务真的会被停止吗？`Service`的生命周期是怎样的？连接1的`ServiceConnection`会回调哪些方法？

7. 紧接着，再使用连接2`bindService`，会发生什么？连接2的`ServiceConnection`会回调哪些方法？

8. 最后，调用`unbindService`解绑连接1，会发生什么？`Service`的生命周期是怎样的？`ServiceConnection`会回调哪些方法？

请大家思考片刻，可以去回顾之前的`Service`启动分析以及停止流程分析，如果以上问题能全部回答正确，证明你对`Service`已经有了一个很深刻的理解，接下来揭晓答案：

1. 生命周期：`onCreate` -> `onStartCommand` -> `onBind`，如果`onBind`的返回值不为`null`，`ServiceConnection`会回调`onServiceConnected`方法，如果`onBind`的返回值为`null`，`ServiceConnection`会回调`onNullBinding`方法

2. 生命周期不会发生变化，如果`onBind`的返回值不为`null`，`ServiceConnection`会回调`onServiceConnected`方法，如果`onBind`的返回值为`null`，`ServiceConnection`会回调`onNullBinding`方法

3. 生命周期不会发生变化，`ServiceConnection`不会回调任何方法

4. 生命周期：`onUnbind`，`ServiceConnection`不会回调任何方法

5. 如果之前`onUnbind`的返回值为`true`，则生命周期为：`onRebind`，否则生命周期不会发生变化，如果之前`onBind`的返回值不为`null`，`ServiceConnection`会回调`onServiceConnected`方法，如果之前`onBind`的返回值为`null`，`ServiceConnection`会回调`onNullBinding`方法

6. 不会真的被停止，生命周期不会发生变化，`ServiceConnection`不会回调任何方法

7. 生命周期不会发生变化，如果之前`onBind`的返回值不为`null`，`ServiceConnection`会回调`onServiceConnected`方法，如果之前`onBind`的返回值为`null`，`ServiceConnection`会回调`onNullBinding`方法

8. 服务会被停止，生命周期：`onUnbind` -> `onDestroy`，连接1的`ServiceConnection`不会回调任何方法，如果之前`onBind`的返回值不为`null`，连接2的`ServiceConnection`会回调`onServiceDisconnected`、`onBindingDied`以及`onNullBinding`方法，如果之前`onBind`的返回值为`null`，连接2的`ServiceConnection`会回调`onBindingDied`以及`onNullBinding`方法

# 被动停止

`Service`除了主动停止，还会因为各种情况导致被动停止

## 用户手动从最近任务移除Task

注：由于各家系统对进程或任务调度策略不同，所以这里的`Task`移除逻辑和`Service`停止逻辑可能会有些许不同，我们还是以原生`Android`为准分析

不管用户是从最近任务划走了一个`Task`，还是点击了全部清除，最终都会走到`ActivityStackSupervisor.removeTaskById`方法

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
boolean removeTaskById(int taskId, boolean killProcess, boolean removeFromRecents,
        String reason) {
    //通过id查询Task
    final Task task =
            mRootWindowContainer.anyTaskForId(taskId, MATCH_TASK_IN_STACKS_OR_RECENT_TASKS);
    if (task != null) {
        removeTask(task, killProcess, removeFromRecents, reason);
        return true;
    }
    return false;
}
```

这里的入参，`killProcess`为`true`，如果是移除单一`Task`，那么`removeFromRecents`为`true`，如果是清除全部`Task`，那么`removeFromRecents`为`false`

先通过`id`，使用`RootWindowContainer`查询相应的`Task`，然后再调用`removeTask`方法继续移除`Task`

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
void removeTask(Task task, boolean killProcess, boolean removeFromRecents, String reason) {
    //如果Task正在清理中则直接返回
    if (task.mInRemoveTask) {
        // Prevent recursion.
        return;
    }
    //标记Task正在清理中
    task.mInRemoveTask = true;
    try {
        //销毁此Task下所有Activity
        task.performClearTask(reason);
        //清理Task
        cleanUpRemovedTaskLocked(task, killProcess, removeFromRecents);
        //关闭屏幕固定
        mService.getLockTaskController().clearLockedTask(task);
        //通知任务栈发生变化
        mService.getTaskChangeNotificationController().notifyTaskStackChanged();
        //启动任务持久化程序，将任何挂起的任务写入磁盘
        if (task.isPersistable) {
            mService.notifyTaskPersisterLocked(null, true);
        }
    } finally {
        //取消标记
        task.mInRemoveTask = false;
    }
}
```

这个方法最重要的有两个部分，一个是销毁此`Task`下所有`Activity`，对于我们本次而言不需要关注，另一个是调用`cleanUpRemovedTaskLocked`继续清理`Task`

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
void cleanUpRemovedTaskLocked(Task task, boolean killProcess, boolean removeFromRecents) {
    //从最近任务列表中移除
    //对于清除全部Task而言，之前在RecentTasks中已经进行过移除操作了
    //所以传入的removeFromRecents为false
    if (removeFromRecents) {
        mRecentTasks.remove(task);
    }
    ComponentName component = task.getBaseIntent().getComponent();
    if (component == null) {
        return;
    }

    // Find any running services associated with this app and stop if needed.
    //清理和此Task有关联的服务
    final Message msg = PooledLambda.obtainMessage(ActivityManagerInternal::cleanUpServices,
            mService.mAmInternal, task.mUserId, component, new Intent(task.getBaseIntent()));
    mService.mH.sendMessage(msg);

    //如果不需要杀死进程，到这里就为止了
    if (!killProcess) {
        return;
    }

    // Determine if the process(es) for this task should be killed.
    final String pkg = component.getPackageName();
    ArrayList<Object> procsToKill = new ArrayList<>();
    ArrayMap<String, SparseArray<WindowProcessController>> pmap = 
            mService.mProcessNames.getMap();
    //遍历App进程，确定要杀死的进程
    for (int i = 0; i < pmap.size(); i++) {
        SparseArray<WindowProcessController> uids = pmap.valueAt(i);
        for (int j = 0; j < uids.size(); j++) {
            WindowProcessController proc = uids.valueAt(j);
            //不要杀死其他用户下的进程
            if (proc.mUserId != task.mUserId) {
                // Don't kill process for a different user.
                continue;
            }
            //不要杀死首页进程
            //HomeProcess指的是含有分类为android.intent.category.HOME的进程
            //也就是能成为首页Launcher的进程
            if (proc == mService.mHomeProcess) {
                // Don't kill the home process along with tasks from the same package.
                continue;
            }
            //不要杀死和这个Task无关的进程
            if (!proc.mPkgList.contains(pkg)) {
                // Don't kill process that is not associated with this task.
                continue;
            }

            //如果这个进程有Activity在不同的Task里，并且这个Task也在最近任务里
            //或者有Activity还没有被停止，则不要杀死进程
            if (!proc.shouldKillProcessForRemovedTask(task)) {
                // Don't kill process(es) that has an activity in a different task that is also
                // in recents, or has an activity not stopped.
                return;
            }

            //有前台服务的话不要杀死进程
            if (proc.hasForegroundServices()) {
                // Don't kill process(es) with foreground service.
                return;
            }

            // Add process to kill list.
            procsToKill.add(proc);
        }
    }

    // Kill the running processes. Post on handle since we don't want to hold the service lock
    // while calling into AM.
    //杀死进程
    final Message m = PooledLambda.obtainMessage(
            ActivityManagerInternal::killProcessesForRemovedTask, mService.mAmInternal,
            procsToKill);
    mService.mH.sendMessage(m);
}
```

这个方法主要做两件事，一是清理和此`Task`有关联的服务，二是杀死应该杀死的进程

清理服务这一步用到了池化技术，这里大家不用管，就当做调用了`mService.mAmInternal.cleanUpServices`即可，这里的`mService.mAmInternal`是`AMS`里的一个内部类`LocalService`

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void cleanUpServices(int userId, ComponentName component, Intent baseIntent) {
    synchronized(ActivityManagerService.this) {
        mServices.cleanUpServices(userId, component, baseIntent);
    }
}
```

同样，关于`Service`的工作都转交给`ActiveServices`

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
void cleanUpServices(int userId, ComponentName component, Intent baseIntent) {
    ArrayList<ServiceRecord> services = new ArrayList<>();
    //获得此用户下所有的活动Service
    ArrayMap<ComponentName, ServiceRecord> alls = getServicesLocked(userId);
    //筛选出此Task下的所有活动Service
    for (int i = alls.size() - 1; i >= 0; i--) {
        ServiceRecord sr = alls.valueAt(i);
        if (sr.packageName.equals(component.getPackageName())) {
            services.add(sr);
        }
    }

    // Take care of any running services associated with the app.
    for (int i = services.size() - 1; i >= 0; i--) {
        ServiceRecord sr = services.get(i);
        if (sr.startRequested) {
            if ((sr.serviceInfo.flags&ServiceInfo.FLAG_STOP_WITH_TASK) != 0) {
                //如果在manifest里设置了stopWithTask，那么会直接停止Service
                stopServiceLocked(sr);
            } else {
                //如果没有设置stopWithTask的话，则会回调Service.onTaskRemoved方法
                sr.pendingStarts.add(new ServiceRecord.StartItem(sr, true,
                        sr.getLastStartId(), baseIntent, null, 0));
                if (sr.app != null && sr.app.thread != null) {
                    // We always run in the foreground, since this is called as
                    // part of the "remove task" UI operation.
                    try {
                        sendServiceArgsLocked(sr, true, false);
                    } catch (TransactionTooLargeException e) {
                        // Ignore, keep going.
                    }
                }
            }
        }
    }
}
```

首先要获取到所有需要清理的`Service`记录，然后再对它们进行处理

对于在`manifest`文件里设置了`stopWithTask`标识的`Service`，直接调用`stopServiceLocked`方法停止服务，而对于没有这个标识的`Service`，则是增加一个启动项，接着调用`sendServiceArgsLocked`处理这个启动项

我们观察这个启动项的构建，第二个参数为`true`，我们可以去这个类的构造方法那里看到，第二个参数的名字为`_taskRemoved`，意思很明显了，然后根据我们在上一篇文章 [Android源码分析 - Service启动流程](https://juejin.cn/post/7276363520554795064#heading-9) 中分析的`sendServiceArgsLocked`方法可以知道，它最终会走到`ActivityThread.handleServiceArgs`方法中

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleServiceArgs(ServiceArgsData data) {
    ...
    int res;
    if (!data.taskRemoved) {
        //正常情况调用
        res = s.onStartCommand(data.args, data.flags, data.startId);
    } else {
        //用户关闭Task栈时调用
        s.onTaskRemoved(data.args);
        res = Service.START_TASK_REMOVED_COMPLETE;
    }
    ...
}
```

可以发现，当`taskRemoved`变量为`true`时，会回调`Service.onTaskRemoved`方法

我们接着回到`cleanUpRemovedTaskLocked`方法中，当它清理完服务后，便会尝试杀死进程，这里面其他的判断条件我们都不用管，我们只需要关心其中的一段，有前台服务的话不要杀死进程

以上是我根据`AOSP`源码分析得出的结果，在模拟器上也验证通过，但在我的小米MIX4上表现却完全不是这样，大家开发时还是要以实际为准

## 内存不足

当内存不足时，系统会通过`OOM Killer`或`Low Memory Killer`等手段杀死各种进程，如果被杀死的进程里有`Service`正在运行，那自然也会被停止

# 重建

当`Service`所在进程被杀死后，根据`Service.onStartCommand`的返回值，系统会决定是否重建，怎么重建。我们先把其可能的返回值以及产生的结果先列出来：

- `START_STICKY_COMPATIBILITY`：`targetSdkVersion` < 5 (`Android 2.0`) 的App默认会返回这个，`Service`被杀后会被重建，但`onStartCommand`方法不会被执行
- `START_STICKY`：`targetSdkVersion` >= 5 (`Android 2.0`) 的App默认会返回这个，`Service`被杀后会被重建，`onStartCommand`方法也会被执行，但此时`onStartCommand`方法的第一个参数`Intent`为`null`
- `START_NOT_STICKY`：`Service`被杀后不会被重建
- `START_REDELIVER_INTENT`：`Service`被杀后会被重建，`onStartCommand`方法也会被执行，此时`onStartCommand`方法的第一个参数`Intent`为`Service`被杀死前最后一次调用`onStartCommand`方法时传递的`Intent`

对于其返回值我们需要先了解一下是怎么处理的，这需要回顾一下上一篇文章分析的`handleServiceArgs`方法了

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            ...
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

可以看到，这里从`Service.onStartCommand`得到返回值后以其作为参数调用了`AMS.serviceDoneExecuting`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
    synchronized(this) {
        ...
        mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
    }
}
```

转交给了`ActiveServices.serviceDoneExecutingLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
    boolean inDestroying = mDestroyingServices.contains(r);
    if (r != null) {
        if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
            // This is a call from a service start...  take care of
            // book-keeping.
            r.callStart = true;
            switch (res) {
                case Service.START_STICKY_COMPATIBILITY:
                case Service.START_STICKY: {
                    // We are done with the associated start arguments.
                    r.findDeliveredStart(startId, false, true);
                    // Don't stop if killed.
                    r.stopIfKilled = false;
                    break;
                }
                case Service.START_NOT_STICKY: {
                    // We are done with the associated start arguments.
                    r.findDeliveredStart(startId, false, true);
                    if (r.getLastStartId() == startId) {
                        // There is no more work, and this service
                        // doesn't want to hang around if killed.
                        r.stopIfKilled = true;
                    }
                    break;
                }
                case Service.START_REDELIVER_INTENT: {
                    // We'll keep this item until they explicitly
                    // call stop for it, but keep track of the fact
                    // that it was delivered.
                    ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
                    if (si != null) {
                        si.deliveryCount = 0;
                        si.doneExecutingCount++;
                        // Don't stop if killed.
                        r.stopIfKilled = true;
                    }
                    break;
                }
                case Service.START_TASK_REMOVED_COMPLETE: {
                    // Special processing for onTaskRemoved().  Don't
                    // impact normal onStartCommand() processing.
                    r.findDeliveredStart(startId, true, true);
                    break;
                }
                default:
                    throw new IllegalArgumentException(
                            "Unknown service start result: " + res);
            }
            if (res == Service.START_STICKY_COMPATIBILITY) {
                r.callStart = false;
            }
        } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
            ...
        }
        ...
    } else { ... }
}
```

我们就只看对`Service.onStartCommand`的返回值进行处理的部分

首先，不管返回值是什么，都会调用`ServiceRecord.findDeliveredStart`方法，只不过入参不同

```java
//frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java
public StartItem findDeliveredStart(int id, boolean taskRemoved, boolean remove) {
    final int N = deliveredStarts.size();
    for (int i=0; i<N; i++) {
        StartItem si = deliveredStarts.get(i);
        if (si.id == id && si.taskRemoved == taskRemoved) {
            if (remove) deliveredStarts.remove(i);
            return si;
        }
    }

    return null;
}
```

前两个参数，不管返回值是什么，传进来的都是一样的，而第三个参数`remove`就不同了，当返回值为`START_REDELIVER_INTENT`的时候，它为`false`，其他情况都为`true`，意味着要删除这个已分发的启动项，`START_REDELIVER_INTENT`由于需要保留最后一次调用`onStartCommand`时的`Intent`，所以它不应该被删除

接着我们回到`serviceDoneExecutingLocked`方法，可以发现，`START_STICKY`和`START_STICKY_COMPATIBILITY`情况下的`ServiceRecord.stopIfKilled`被置为了`false`，其他则被置为了`true`，这和我们之前对结果的描述不同啊？不是说好了返回`START_REDELIVER_INTENT`也会重启吗？这是因为`START_REDELIVER_INTENT`比较特殊，它的重启不需要看`stopIfKilled`这个标志位，这个等到我们后面分析到怎么判断是否应该停止服务时就知道了

以上的内容我们先记下来，会在后面的重启流程中发挥作用

然后我们来看进程死亡后会发生什么，我们曾在之前的文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128#heading-7) 中提到过，当App启动时，`AMS`会为其注册一个App进程死亡回调`AppDeathRecipient`，当App进程死亡后便会回调其`binderDied`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
private final class AppDeathRecipient implements IBinder.DeathRecipient {
    final ProcessRecord mApp;
    final int mPid;
    final IApplicationThread mAppThread;

    AppDeathRecipient(ProcessRecord app, int pid,
            IApplicationThread thread) {
        mApp = app;
        mPid = pid;
        mAppThread = thread;
    }

    @Override
    public void binderDied() {
        synchronized(ActivityManagerService.this) {
            appDiedLocked(mApp, mPid, mAppThread, true, null);
        }
    }
}
```

接着便会调用`appDiedLocked`方法处理进程死亡

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final void appDiedLocked(ProcessRecord app, int pid, IApplicationThread thread,
        boolean fromBinderDied, String reason) {
    // First check if this ProcessRecord is actually active for the pid.
    //检查pid所属ProcessRecord是否与传入ProcessRecord相符
    synchronized (mPidsSelfLocked) {
        ProcessRecord curProc = mPidsSelfLocked.get(pid);
        if (curProc != app) {
            return;
        }
    }

    ... //记录电池统计信息

    //如果App进程尚未死亡的话，杀死进程
    if (!app.killed) {
        if (!fromBinderDied) {
            killProcessQuiet(pid);
            mProcessList.noteAppKill(app, ApplicationExitInfo.REASON_OTHER,
                    ApplicationExitInfo.SUBREASON_UNKNOWN, reason);
        }
        ProcessList.killProcessGroup(app.uid, pid);
        app.killed = true;
    }

    // Clean up already done if the process has been re-started.
    if (app.pid == pid && app.thread != null &&
            app.thread.asBinder() == thread.asBinder()) {
        //一般情况下非自动化测试，先置为true
        boolean doLowMem = app.getActiveInstrumentation() == null;
        boolean doOomAdj = doLowMem;
        if (!app.killedByAm) { //不通过AMS杀死的进程，一般就是被 Low Memory Killer (LMK) 杀死的
            ... //报告信息
            //被LMK杀死，说明系统内存不足
            mAllowLowerMemLevel = true;
        } else { //通过AMS杀死的进程
            // Note that we always want to do oom adj to update our state with the
            // new number of procs.
            mAllowLowerMemLevel = false;
            //正常情况下非内存不足
            doLowMem = false;
        }
        ... //事件记录
        //继续处理App进程死亡
        handleAppDiedLocked(app, false, true);

        //调整进程优先级
        if (doOomAdj) {
            updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_PROCESS_END);
        }
        //当因为内存不足而杀死App进程时
        //调用App层各处的的onLowMemory方法，释放内存
        if (doLowMem) {
            doLowMemReportIfNeededLocked(app);
        }
    } else if (app.pid != pid) { //新进程已启动
        // A new process has already been started.
        ... //报告记录信息
    }
    ...
}
```

这里的其他代码我们都不用关心，直接看`handleAppDiedLocked`方法继续处理App进程死亡

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final void handleAppDiedLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart) {
    ...
    //清理进程的主要方法
    boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1,
            false /*replacingPid*/);
    ...
}

final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
    ...
    mServices.killServicesLocked(app, allowRestart);
    ...
}
```

可以看到，这里调用了`ActiveServices.killServicesLocked`方法停止还在运行中的服务

```java
final void killServicesLocked(ProcessRecord app, boolean allowRestart) {
    // Clean up any connections this application has to other services.
    //清理所有连接
    for (int i = app.connections.size() - 1; i >= 0; i--) {
        ConnectionRecord r = app.connections.valueAt(i);
        removeConnectionLocked(r, app, null);
    }
    updateServiceConnectionActivitiesLocked(app);
    app.connections.clear();

    app.whitelistManager = false;

    // Clear app state from services.
    //遍历所有正在运行中的服务
    for (int i = app.numberOfRunningServices() - 1; i >= 0; i--) {
        ServiceRecord sr = app.getRunningServiceAt(i);
        synchronized (sr.stats.getBatteryStats()) {
            sr.stats.stopLaunchedLocked();
        }
        if (sr.app != app && sr.app != null && !sr.app.isPersistent()) {
            //记录服务已停止
            sr.app.stopService(sr);
            //更新绑定的客户端uids
            sr.app.updateBoundClientUids();
        }
        sr.setProcess(null);
        sr.isolatedProc = null;
        sr.executeNesting = 0;
        sr.forceClearTracker();
        if (mDestroyingServices.remove(sr)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "killServices remove destroying " + sr);
        }

        final int numClients = sr.bindings.size();
        for (int bindingi=numClients-1; bindingi>=0; bindingi--) {
            IntentBindRecord b = sr.bindings.valueAt(bindingi);
            //释放对服务Binder的引用
            b.binder = null;
            //重置
            b.requested = b.received = b.hasBound = false;
            // If this binding is coming from a cached process and is asking to keep
            // the service created, then we'll kill the cached process as well -- we
            // don't want to be thrashing around restarting processes that are only
            // there to be cached.
            ... //遍历客户端进程（实际上没做任何事）
        }
    }

    ServiceMap smap = getServiceMapLocked(app.userId);

    // Now do remaining service cleanup.
    for (int i = app.numberOfRunningServices() - 1; i >= 0; i--) {
        ServiceRecord sr = app.getRunningServiceAt(i);

        // Unless the process is persistent, this process record is going away,
        // so make sure the service is cleaned out of it.
        //非持久化进程
        if (!app.isPersistent()) {
            //记录服务已停止
            app.stopService(sr);
            //更新绑定的客户端uids
            app.updateBoundClientUids();
        }

        // Sanity check: if the service listed for the app is not one
        // we actually are maintaining, just let it drop.
        //一致性检查
        final ServiceRecord curRec = smap.mServicesByInstanceName.get(sr.instanceName);
        if (curRec != sr) {
            if (curRec != null) {
                Slog.wtf(TAG, "Service " + sr + " in process " + app
                        + " not same as in map: " + curRec);
            }
            continue;
        }

        // Any services running in the application may need to be placed
        // back in the pending list.
        //允许重启，但Service崩溃的次数超出重试上限（默认为16），并且它不是系统应用
        if (allowRestart && sr.crashCount >= mAm.mConstants.BOUND_SERVICE_MAX_CRASH_RETRY
                && (sr.serviceInfo.applicationInfo.flags
                    &ApplicationInfo.FLAG_PERSISTENT) == 0) {
            ... //记录
            //停止服务
            bringDownServiceLocked(sr);
        } else if (!allowRestart
                || !mAm.mUserController.isUserRunning(sr.userId, 0)) {
            //不允许重启或者服务进程所在用户不在运行，停止服务
            bringDownServiceLocked(sr);
        } else {
            //尝试调度重启服务
            final boolean scheduled = scheduleServiceRestartLocked(sr, true /* allowCancel */);

            // Should the service remain running?  Note that in the
            // extreme case of so many attempts to deliver a command
            // that it failed we also will stop it here.
            if (!scheduled) { //未调度重启
                //停止服务
                bringDownServiceLocked(sr);
            } else if (sr.canStopIfKilled(false /* isStartCanceled */)) {
                // Update to stopped state because the explicit start is gone. The service is
                // scheduled to restart for other reason (e.g. connections) so we don't bring
                // down it.
                //将服务的启动状态更新为停止
                sr.startRequested = false;
                ... //记录
            }
        }
    }

    //不允许重启的话
    if (!allowRestart) {
        //停止所有服务
        app.stopAllServices();
        //清理绑定的客户端uids
        app.clearBoundClientUids();

        // Make sure there are no more restarting services for this process.
        //确保这个进程不再会重启服务，清理所有待重启待启动的服务
        for (int i=mRestartingServices.size()-1; i>=0; i--) {
            ServiceRecord r = mRestartingServices.get(i);
            if (r.processName.equals(app.processName) &&
                    r.serviceInfo.applicationInfo.uid == app.info.uid) {
                mRestartingServices.remove(i);
                clearRestartingIfNeededLocked(r);
            }
        }
        for (int i=mPendingServices.size()-1; i>=0; i--) {
            ServiceRecord r = mPendingServices.get(i);
            if (r.processName.equals(app.processName) &&
                    r.serviceInfo.applicationInfo.uid == app.info.uid) {
                mPendingServices.remove(i);
            }
        }
    }

    // Make sure we have no more records on the stopping list.
    //清理所有停止中的服务
    int i = mDestroyingServices.size();
    while (i > 0) {
        i--;
        ServiceRecord sr = mDestroyingServices.get(i);
        if (sr.app == app) {
            sr.forceClearTracker();
            mDestroyingServices.remove(i);
        }
    }

    //清理所有执行中的服务
    //这里的执行中指的是有事务正在运行，比如说正在停止过程中，不是指运行中
    app.executingServices.clear();
}
```

这里做了各种清理工作，客户端进程的清理呀，服务端进程的清理，然后就是`Service`重启的核心，`scheduleServiceRestartLocked`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final boolean scheduleServiceRestartLocked(ServiceRecord r, boolean allowCancel) {
    //系统正在关机，直接返回
    if (mAm.mAtmInternal.isShuttingDown()) {
        return false;
    }

    //一致性检查
    ServiceMap smap = getServiceMapLocked(r.userId);
    if (smap.mServicesByInstanceName.get(r.instanceName) != r) {
        ServiceRecord cur = smap.mServicesByInstanceName.get(r.instanceName);
        Slog.wtf(TAG, "Attempting to schedule restart of " + r
                + " when found in map: " + cur);
        return false;
    }

    final long now = SystemClock.uptimeMillis();

    final String reason;
    if ((r.serviceInfo.applicationInfo.flags
            &ApplicationInfo.FLAG_PERSISTENT) == 0) { //对于非系统应用
        //服务至少要过多长时间才能重启，默认1000ms
        long minDuration = mAm.mConstants.SERVICE_RESTART_DURATION;
        //服务被杀死重启后需要运行多长时间，默认60s
        long resetTime = mAm.mConstants.SERVICE_RESET_RUN_DURATION;
        boolean canceled = false;

        // Any delivered but not yet finished starts should be put back
        // on the pending list.
        //对应着返回值为START_REDELIVER_INTENT的情况
        final int N = r.deliveredStarts.size();
        if (N > 0) {
            for (int i=N-1; i>=0; i--) {
                ServiceRecord.StartItem si = r.deliveredStarts.get(i);
                si.removeUriPermissionsLocked();
                if (si.intent == null) {
                    // We'll generate this again if needed.
                } else if (!allowCancel || (si.deliveryCount < ServiceRecord.MAX_DELIVERY_COUNT
                        && si.doneExecutingCount < ServiceRecord.MAX_DONE_EXECUTING_COUNT)) {
                    //如果该启动项的失败次数小于最大容忍次数
                    //MAX_DELIVERY_COUNT默认为3
                    //MAX_DONE_EXECUTING_COUNT默认为6
                    r.pendingStarts.add(0, si);
                    //这种情况下延时是现在距离启动时间的两倍
                    long dur = SystemClock.uptimeMillis() - si.deliveredTime;
                    dur *= 2;
                    if (minDuration < dur) minDuration = dur;
                    if (resetTime < dur) resetTime = dur;
                } else {
                    //如果该启动项的失败次数大于等于最大容忍次数
                    canceled = true;
                }
            }
            r.deliveredStarts.clear();
        }

        if (allowCancel) {
            //判断是否应该停止（只考虑非绑定启动的情况）
            final boolean shouldStop = r.canStopIfKilled(canceled);
            if (shouldStop && !r.hasAutoCreateConnections()) {
                // Nothing to restart.
                //如果应该停止，直接返回
                return false;
            }
            reason = (r.startRequested && !shouldStop) ? "start-requested" : "connection";
        } else {
            reason = "always";
        }

        r.totalRestartCount++;
        if (r.restartDelay == 0) { //第一次重启的情况
            r.restartCount++;
            r.restartDelay = minDuration;
        } else if (r.crashCount > 1) { //Service所在进程在Service运行过程中发生崩溃导致重启的话
            //重启延时为 30min * (崩溃次数 - 1)
            r.restartDelay = mAm.mConstants.BOUND_SERVICE_CRASH_RESTART_DURATION
                    * (r.crashCount - 1);
        } else { //非第一次重启的情况
            // If it has been a "reasonably long time" since the service
            // was started, then reset our restart duration back to
            // the beginning, so we don't infinitely increase the duration
            // on a service that just occasionally gets killed (which is
            // a normal case, due to process being killed to reclaim memory).
            if (now > (r.restartTime+resetTime)) {
                //如果服务重启后运行达到了一定时间，则重启延时为
                r.restartCount = 1;
                r.restartDelay = minDuration;
            } else {
                //如果服务重启后运行没有达到一定时间（短时间内又要重启）
                //则增长重启延时，默认因子为4
                r.restartDelay *= mAm.mConstants.SERVICE_RESTART_DURATION_FACTOR;
                if (r.restartDelay < minDuration) {
                    r.restartDelay = minDuration;
                }
            }
        }

        //确定重启时间
        r.nextRestartTime = now + r.restartDelay;

        // Make sure that we don't end up restarting a bunch of services
        // all at the same time.
        //确保不会在同一时间启动大量服务
        boolean repeat;
        do {
            repeat = false;
            final long restartTimeBetween = mAm.mConstants.SERVICE_MIN_RESTART_TIME_BETWEEN;
            for (int i=mRestartingServices.size()-1; i>=0; i--) {
                ServiceRecord r2 = mRestartingServices.get(i);
                if (r2 != r && r.nextRestartTime >= (r2.nextRestartTime-restartTimeBetween)
                        && r.nextRestartTime < (r2.nextRestartTime+restartTimeBetween)) {
                    r.nextRestartTime = r2.nextRestartTime + restartTimeBetween;
                    r.restartDelay = r.nextRestartTime - now;
                    repeat = true;
                    break;
                }
            }
        } while (repeat);

    } else { //对于系统进程，立马重启
        // Persistent processes are immediately restarted, so there is no
        // reason to hold of on restarting their services.
        r.totalRestartCount++;
        r.restartCount = 0;
        r.restartDelay = 0;
        r.nextRestartTime = now;
        reason = "persistent";
    }

    //添加到重启列表中
    if (!mRestartingServices.contains(r)) {
        r.createdFromFg = false;
        mRestartingServices.add(r);
        r.makeRestarting(mAm.mProcessStats.getMemFactorLocked(), now);
    }

    //取消前台服务通知
    cancelForegroundNotificationLocked(r);

    //通过Handler调度重启
    mAm.mHandler.removeCallbacks(r.restarter);
    mAm.mHandler.postAtTime(r.restarter, r.nextRestartTime);
    r.nextRestartTime = SystemClock.uptimeMillis() + r.restartDelay;
    ... //事件记录
    return true;
}
```

这个方法主要做了几件事情，一是判断服务需不需要重启，二是计算服务的下次重启时间，最后通过`Handler`执行延时重启

还记得我们前面说的当返回值为`START_REDELIVER_INTENT`时，不会从`ServiceRecord.deliveredStarts`中删除启动项吗？这里就体现出了这一点，遍历整个`deliveredStarts`列表，从中找出符合重启条件的启动项，将其加入到`pendingStarts`列表中，需要注意的是，在这种情况下，重启延时为现在距离启动时间的两倍，所以一般情况下`START_REDELIVER_INTENT`比`START_STICKY`重启的要更慢

接下来便要判断服务需不需要重启，这里调用了`ServiceRecord.canStopIfKilled`方法

```java
//frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java
boolean canStopIfKilled(boolean isStartCanceled) {
    return startRequested && (stopIfKilled || isStartCanceled) && pendingStarts.isEmpty();
}
```

之前说过，如果返回值为`START_STICKY`或`START_STICKY_COMPATIBILITY`，那这里的`stopIfKilled`就为`false`，所以整体会返回`false`，而对于返回值`START_REDELIVER_INTENT`而言，之前已经进行过操作，将启动项添加到`pendingStarts`列表中了，所以只要这里为`false`，整体就为`false`，`stopIfKilled`的值就不重要了

最后就是通过`Handler`执行延时重启了，这里`Handler`传入的`Runnable`是`ServiceRecord.restarter`，它是在服务启动，调用`retrieveServiceLocked`方法时被创建的

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private class ServiceRestarter implements Runnable {
    private ServiceRecord mService;

    void setService(ServiceRecord service) {
        mService = service;
    }

    public void run() {
        synchronized(mAm) {
            performServiceRestartLocked(mService);
        }
    }
}
```

可以看到，在一定的延时后，会调用到`performServiceRestartLocked`方法重启服务

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
final void performServiceRestartLocked(ServiceRecord r) {
    if (!mRestartingServices.contains(r)) {
        return;
    }
    if (!isServiceNeededLocked(r, false, false)) {
        // Paranoia: is this service actually needed?  In theory a service that is not
        // needed should never remain on the restart list.  In practice...  well, there
        // have been bugs where this happens, and bad things happen because the process
        // ends up just being cached, so quickly killed, then restarted again and again.
        // Let's not let that happen.
        Slog.wtf(TAG, "Restarting service that is not needed: " + r);
        return;
    }
    try {
        //参考上一篇文章，Service启动流程
        bringUpServiceLocked(r, r.intent.getIntent().getFlags(), r.createdFromFg, true, false);
    } catch (TransactionTooLargeException e) {
        // Ignore, it's been logged and nothing upstack cares.
    }
}
```

最终调用了`bringUpServiceLocked`方法启动服务，这一部分可以看上一篇文章 [Android源码分析 - Service启动流程](https://juejin.cn/post/7276363520554795064) 中的分析，还记得上一篇文章中分析的`realStartServiceLocked`方法中，有一个逻辑是：如果`Service`已经启动，并且没有启动项，则构建一个假的启动参数供`onStartCommand`使用 吗？之前看到这个逻辑的时候我还有些疑惑为什么需要这样，现在就豁然开朗了，原来这是为`Service`重启做的逻辑，而对于返回值`START_REDELIVER_INTENT`而言，`pendingStarts`列表本身就不为空，直接正常执行启动任务就可以了

# 总结

至此，整个`Service`篇章就到此结束了，通过这次的写作，我自身也是受益匪浅，了解到了很多我以前不知道的知识，也纠正了一些我以前错误的认知，如果也能帮到正在看文章的你们，那就再好不过了