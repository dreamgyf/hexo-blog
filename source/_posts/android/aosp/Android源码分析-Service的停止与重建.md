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
        //重置启动状态
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

接着，在这个方法的最后，调用`bringDownServiceIfNeededLocked`方法继续停止服务

```java
private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
        boolean hasConn) {
    //检查是否有auto-create的连接（flag为BIND_AUTO_CREATE）
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
            //回调ServiceConnection.onServiceConnected和ServiceConnection.onBindingDied方法
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
            if (ibr.hasBound) {
                try {
                    //记录Service执行操作并设置超时回调
                    //前台服务超时时间为20s，后台服务超时时间为200s
                    bumpServiceExecutingLocked(r, false, "bring down unbind");
                    needOomAdj = true;
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
        //更新进程优先级
        if (needOomAdj) {
            mAm.updateOomAdjLocked(r.app, true,
                    OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
        }
    }

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

    int memFactor = mAm.mProcessStats.getMemFactorLocked();
    long now = SystemClock.uptimeMillis();
    if (r.tracker != null) {
        r.tracker.setStarted(false, memFactor, now);
        r.tracker.setBound(false, memFactor, now);
        if (r.executeNesting == 0) {
            r.tracker.clearCurrentOwner(r, false);
            r.tracker = null;
        }
    }

    smap.ensureNotStartingBackgroundLocked(r);
}
```