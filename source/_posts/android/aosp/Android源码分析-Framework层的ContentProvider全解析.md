---
title: Android源码分析 - Framework层的ContentProvider全解析
date: 2023-06-20 17:29:17
tags: 
- Android源码
- ContentProvider
categories: 
- [Android, 源码分析]
- [Android, ContentProvider]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

在四大组件中，可能我们平时用到最少的便是`ContentProvider`了，`ContentProvider`是用来帮助应用管理其自身和其他应用所存储数据的访问，并提供与其他应用共享数据的方法，使用`ContentProvider`可以安全的在应用之间共享和修改数据，比如说访问图库，通讯录等

在之前的文章中，我们提到了`ContentProvider`的启动时机，不妨顺水推舟，干脆把这一块分析个明白，本篇文章并不会教大家怎样使用`ContentProvider`，只将精力集中在`ContentProvider`在系统层面的启动与交互上

# 基础知识

## ContentResolver

想要通过`ContentProvider`访问应用数据，我们通常需要借助`ContentResolver`的API，我们可以通过`Context.getContentResolver`方法获取其实例对象

`ContentResolver`是一个抽象类，它的抽象方法由`ContextImpl.ApplicationContentResolver`继承实现，我们实际上获取到的也是这个实例对象

## Uri格式

`ContentProvider`的使用需要先获得提供者的Uri，它的格式如下：

1. Scheme：固定为`content://`
2. Authority：为提供者在`AndroidManifest`里设置的`android:authorities`属性
3. 资源相对路径
4. 资源ID

其中，资源相对路径和资源ID不是必须的，要看资源存储的数量及形式

举个栗子，外部存储中某张图片的Uri为：`content://media/external/images/media/${id}`，其中`media`为Authority，`/external/images/media`为外部存储图片的相对路径，`id`为这张图片资源在数据库中存储的`id`

# 获取ContentProvider

`ContentProvider`作为共享数据的桥梁，最主要的几个功能无非是增、删、改、查，我们就以查作为入口来分析`ContentProvider`对象是怎么获取的

```java
//ContentResolver.query
public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable Bundle queryArgs,
        @Nullable CancellationSignal cancellationSignal) {
    ...

    //尝试获取unstableProvider
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    if (unstableProvider == null) {
        return null;
    }
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
        ...
        try {
            //调用远程对象query
            qCursor = unstableProvider.query(mPackageName, mAttributionTag, uri, projection,
                    queryArgs, remoteCancellationSignal);
        } catch (DeadObjectException e) {
            // The remote process has died...  but we only hold an unstable
            // reference though, so we might recover!!!  Let's try!!!!
            // This is exciting!!1!!1!!!!1
            unstableProviderDied(unstableProvider);
            //尝试获取stableProvider
            stableProvider = acquireProvider(uri);
            if (stableProvider == null) {
                return null;
            }
            //调用远程对象query
            qCursor = stableProvider.query(mPackageName, mAttributionTag, uri, projection,
                    queryArgs, remoteCancellationSignal);
        }
        if (qCursor == null) {
            return null;
        }

        // Force query execution.  Might fail and throw a runtime exception here.
        qCursor.getCount();
        ...

        // Wrap the cursor object into CursorWrapperInner object.
        //将qCursor和provider包装成CursorWrapperInner对象返回
        final IContentProvider provider = (stableProvider != null) ? stableProvider
                : acquireProvider(uri);
        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
        stableProvider = null;
        qCursor = null;
        return wrapper;
    } catch (RemoteException e) {
        // Arbitrary and not worth documenting, as Activity
        // Manager will kill this process shortly anyway.
        return null;
    } finally {
        //释放资源
        if (qCursor != null) {
            qCursor.close();
        }
        if (cancellationSignal != null) {
            cancellationSignal.setRemote(null);
        }
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}
```

我们可以将这个方法大致分成以下几个步骤：

1. 获取`unstableProvider`远程对象
2. 调用`unstableProvider`对象的`query`方法，获取`qCursor`
3. 如果`query`过程中远程对象死亡，尝试获取`stableProvider`并调用`query`方法获取`qCursor`
4. 获取`stableProvider`（如果之前没获取的话）
5. 将`qCursor`和`stableProvider`包装成`CursorWrapperInner`对象返回
6. 释放资源

既然`ContentProvider`可以在应用之前共享数据，那它必然是支持跨进程的，没错，用的还是我们熟悉的`Binder`通信，`IContentProvider`对象即是给调用方进程使用的远程`Binder`对象，回顾这个方法我们发现，`IContentProvider`远程对象是通过`acquireUnstableProvider`或`acquireProvider`获取的，我们接下来看看这两个方法做了什么

这里有一个关于`unstable`和`stable`的概念，对于通过这两种方式获取的`ContentProvider`分别会有`unstableCount`和`stableCount`两种引用计数，如果远程`ContentProvider`所在进程死亡，且其`stableCount > 0`的话，则会将其通过`stable`方式关联的调用方进程一同杀死，具体的流程我们会在后面分析

```java
public final IContentProvider acquireUnstableProvider(Uri uri) {
    if (!SCHEME_CONTENT.equals(uri.getScheme())) {
        return null;
    }
    String auth = uri.getAuthority();
    if (auth != null) {
        return acquireUnstableProvider(mContext, uri.getAuthority());
    }
    return null;
}

public final IContentProvider acquireProvider(Uri uri) {
    if (!SCHEME_CONTENT.equals(uri.getScheme())) {
        return null;
    }
    final String auth = uri.getAuthority();
    if (auth != null) {
        return acquireProvider(mContext, auth);
    }
    return null;
}

public final IContentProvider acquireUnstableProvider(String name) {
    if (name == null) {
        return null;
    }
    return acquireUnstableProvider(mContext, name);
}

public final IContentProvider acquireProvider(String name) {
    if (name == null) {
        return null;
    }
    return acquireProvider(mContext, name);
}

// ContextImpl.ApplicationContentResolver 内实现
protected IContentProvider acquireUnstableProvider(Context c, String auth) {
    return mMainThread.acquireProvider(c,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), false);
}

// ContextImpl.ApplicationContentResolver 内实现
protected IContentProvider acquireProvider(Context context, String auth) {
    return mMainThread.acquireProvider(context,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), true);
}
```

## ActivityThread.acquireProvider

Android系统是通过`Authority`来区分不同的`ContentProvider`的，经过一些简单的判断处理后，最终调用了`ActivityThread.acquireProvider`方法去获取`ContentProvider`，而`acquireUnstableProvider`和`acquireProvider`的区别只是最后一个布尔值入参不同罢了

```java
public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
    //尝试从本地缓存中获取ContentProvider对象
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }

    // There is a possible race here.  Another thread may try to acquire
    // the same provider at the same time.  When this happens, we want to ensure
    // that the first one wins.
    // Note that we cannot hold the lock while acquiring and installing the
    // provider since it might take a long time to run and it could also potentially
    // be re-entrant in the case where the provider is in the same process.
    ContentProviderHolder holder = null;
    try {
        synchronized (getGetProviderLock(auth, userId)) {
            //使用AMS获取ContentProvider对象
            holder = ActivityManager.getService().getContentProvider(
                    getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
        }
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    if (holder == null) {
        ...
        return null;
    }

    // Install provider will increment the reference count for us, and break
    // any ties in the race.
    //安装ContentProvider
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```

这个方法大概做了以下几件事：

1. 首先从缓存中尝试获取`IContentProvider`对象
2. 使用`AMS`获取`ContentProviderHolder`对象
3. 安装`ContentProvider`
4. 返回`IContentProvider`对象

### ActivityThread.acquireExistingProvider

我们首先看通过`acquireExistingProvider`方法尝试从缓存中获取`IContentProvider`对象

```java
public final IContentProvider acquireExistingProvider(
        Context c, String auth, int userId, boolean stable) {
    synchronized (mProviderMap) {
        //从缓存Map中查找
        final ProviderKey key = new ProviderKey(auth, userId);
        final ProviderClientRecord pr = mProviderMap.get(key);
        if (pr == null) {
            return null;
        }

        IContentProvider provider = pr.mProvider;
        IBinder jBinder = provider.asBinder();
        //判断远端进程是否已被杀死
        if (!jBinder.isBinderAlive()) {
            // The hosting process of the provider has died; we can't
            // use this one.
            //清理ContentProvider
            handleUnstableProviderDiedLocked(jBinder, true);
            return null;
        }

        // Only increment the ref count if we have one.  If we don't then the
        // provider is not reference counted and never needs to be released.
        ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
        if (prc != null) {
            //更新引用计数
            incProviderRefLocked(prc, stable);
        }
        return provider;
    }
}
```

首先通过`Authority`和`userId`来从Map中查找是否已存在对应的`ProviderClientRecord`对象，然后从中取出`IContentProvider`对象，再检查其中的远程`Binder`对象是否已被杀死，最后一切无误，增加`ContentProvider`的引用计数

### AMS.getContentProvider

如果这一步没有获取到，程序会继续从`AMS`获取`ContentProvider`

```java
public final ContentProviderHolder getContentProvider(
        IApplicationThread caller, String callingPackage, String name, int userId,
        boolean stable) {
    if (caller == null) {
        String msg = "null IApplicationThread when getting content provider "
                + name;
        Slog.w(TAG, msg);
        throw new SecurityException(msg);
    }
    // The incoming user check is now handled in checkContentProviderPermissionLocked() to deal
    // with cross-user grant.
    final int callingUid = Binder.getCallingUid();
    if (callingPackage != null && mAppOpsService.checkPackage(callingUid, callingPackage)
            != AppOpsManager.MODE_ALLOWED) {
        throw new SecurityException("Given calling package " + callingPackage
                + " does not match caller's uid " + callingUid);
    }
    return getContentProviderImpl(caller, name, null, callingUid, callingPackage,
            null, stable, userId);
}
```

经过一些检查后调用`getContentProviderImpl`方法，这个方法有点长，我们分段来看

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;
    boolean providerRunning = false;

    synchronized(this) {
        //获取调用方所在进程记录
        ProcessRecord r = null;
        if (caller != null) {
            r = getRecordForAppLocked(caller);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when getting content provider " + name);
            }
        }

        boolean checkCrossUser = true;

        // First check if this content provider has been published...
        //检查需要的ContentProvider是否已被发布
        cpr = mProviderMap.getProviderByName(name, userId);
        // If that didn't work, check if it exists for user 0 and then
        // verify that it's a singleton provider before using it.
        //如果没找到，尝试从系统用户中查找已发布的ContentProvider
        //并确保它是可用的单例组件，条件如下：
        //是用户级应用程序且组件设置了单例flag且拥有INTERACT_ACROSS_USERS权限 或 App运行在system进程中 或 组件设置了单例flag且是同一个App
        if (cpr == null && userId != UserHandle.USER_SYSTEM) {
            cpr = mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
            if (cpr != null) {
                cpi = cpr.info;
                if (isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        && isValidSingletonCall(r == null ? callingUid : r.uid,
                                cpi.applicationInfo.uid)) {
                    userId = UserHandle.USER_SYSTEM;
                    checkCrossUser = false;
                } else {
                    cpr = null;
                    cpi = null;
                }
            }
        }

        //判断ContentProvider所在进程是否已死亡
        ProcessRecord dyingProc = null;
        if (cpr != null && cpr.proc != null) {
            providerRunning = !cpr.proc.killed;

            // Note if killedByAm is also set, this means the provider process has just been
            // killed by AM (in ProcessRecord.kill()), but appDiedLocked() hasn't been called
            // yet. So we need to call appDiedLocked() here and let it clean up.
            // (See the commit message on I2c4ba1e87c2d47f2013befff10c49b3dc337a9a7 to see
            // how to test this case.)
            if (cpr.proc.killed && cpr.proc.killedByAm) {
                Slog.wtf(TAG, cpr.proc.toString() + " was killed by AM but isn't really dead");
                // Now we are going to wait for the death before starting the new process.
                dyingProc = cpr.proc;
            }
        }
        ...
    }
    ...
}
```

首先，第一部分，检查目标`ContentProvider`是否已被发布并记录在了`mProviderMap`中，注意这里的`mProviderMap`是`AMS`中的一个成员变量，一系列Map的一个集合，和`ActivityThread`中的`mProviderMap`不是一个东西。如果在当前用户中找不到，且当前用户不是系统用户（UserHandle.USER_SYSTEM == 0），则尝试从系统用户中查找合法可用的单例`ContentProvider`，符合以下任一一个条件的`ContentProvider`即可被视作单例`ContentProvider`：

- App是用户级应用程序（uid >= 10000）且`ContentProvider`组件设置了单例flag（`android:singleUser`）且App拥有`INTERACT_ACROSS_USERS`权限
- App运行在`system`进程中
- `ContentProvider`组件设置了单例flag（`android:singleUser`）且是同一个App

至于为什么跨用户访问需要单例这个条件，这个和多用户相关，我也不是很清楚，以后如果分析到了多用户这块再回来补充。目前国内厂商的应用分身、手机分身功能大部分用的就是多用户技术

接着通过目标`ContentProviderRecord`是否存在和其所在进程是否还存活判断目标`ContentProvider`是否在运行中

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;
    boolean providerRunning = false;

    synchronized(this) {
        ...
        //ContentProvider正在运行中
        if (providerRunning) {
            cpi = cpr.info;

            //如果此ContentProvider可以在调用者进程中直接运行（同一个App的同进程 或 同一个App且Provider组件支持多进程）
            //直接返回一个新的ContentProviderHolder让调用者进程自己启动ContentProvider
            if (r != null && cpr.canRunHere(r)) {
                ... //权限检查

                // This provider has been published or is in the process
                // of being published...  but it is also allowed to run
                // in the caller's process, so don't make a connection
                // and just let the caller instantiate its own instance.
                ContentProviderHolder holder = cpr.newHolder(null);
                // don't give caller the provider object, it needs
                // to make its own.
                holder.provider = null;
                return holder;
            }

            // Don't expose providers between normal apps and instant apps
            try {
                if (AppGlobals.getPackageManager()
                        .resolveContentProvider(name, 0 /*flags*/, userId) == null) {
                    return null;
                }
            } catch (RemoteException e) {
            }

            ... //权限检查

            final long origId = Binder.clearCallingIdentity();

            // In this case the provider instance already exists, so we can
            // return it right away.
            //获取连接并更新引用计数
            conn = incProviderCountLocked(r, cpr, token, callingUid, callingPackage, callingTag,
                    stable);
            if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
                if (cpr.proc != null
                        && r != null && r.setAdj <= ProcessList.PERCEPTIBLE_LOW_APP_ADJ) {
                    // If this is a perceptible app accessing the provider,
                    // make sure to count it as being accessed and thus
                    // back up on the LRU list.  This is good because
                    // content providers are often expensive to start.
                    //更新进程优先级
                    mProcessList.updateLruProcessLocked(cpr.proc, false, null);
                }
            }

            final int verifiedAdj = cpr.proc.verifiedAdj;
            //更新进程adj
            boolean success = updateOomAdjLocked(cpr.proc, true,
                    OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
            // XXX things have changed so updateOomAdjLocked doesn't actually tell us
            // if the process has been successfully adjusted.  So to reduce races with
            // it, we will check whether the process still exists.  Note that this doesn't
            // completely get rid of races with LMK killing the process, but should make
            // them much smaller.
            if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
                success = false;
            }
            maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
            // NOTE: there is still a race here where a signal could be
            // pending on the process even though we managed to update its
            // adj level.  Not sure what to do about this, but at least
            // the race is now smaller.
            if (!success) {
                // Uh oh...  it looks like the provider's process
                // has been killed on us.  We need to wait for a new
                // process to be started, and make sure its death
                // doesn't kill our process.
                Slog.wtf(TAG, "Existing provider " + cpr.name.flattenToShortString()
                        + " is crashing; detaching " + r);
                //ContentProvider所在进程被杀了，更新引用计数
                boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
                //仍有别的地方对这个ContentProvider有引用，直接返回null（需要等待进程清理干净才能重启）
                if (!lastRef) {
                    // This wasn't the last ref our process had on
                    // the provider...  we will be killed during cleaning up, bail.
                    return null;
                }
                // We'll just start a new process to host the content provider
                //将运行状态标为false，使得重新启动ContentProvider所在进程
                providerRunning = false;
                conn = null;
                dyingProc = cpr.proc;
            } else {
                cpr.proc.verifiedAdj = cpr.proc.setAdj;
            }

            Binder.restoreCallingIdentity(origId);
        }
        ...
    }
    ...
}
```

第二部分，如果目标`ContentProvider`正在运行中，首先检查目标`ContentProvider`是否可以在调用者进程中直接运行，需要满足以下任一一个条件：

- 调用者和目标`ContentProvider`是同一个App中的同进程
- 调用者和目标`ContentProvider`属同一个App且`ContentProvider`组件支持多进程（`android:multiprocess`）

在这种情况下，直接返回一个新的`ContentProviderHolder`让调用者进程自己处理获得`ContentProvider`即可，具体逻辑在`ActivityThread.installProvider`方法中，后面会分析

如果不满足这种情况，即调用方进程和目标`ContentProvider`不在一个进程中，需要跨进程调用，获取`ContentProviderConnection`连接并更新引用计数

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;
    boolean providerRunning = false;

    synchronized(this) {
        ...
        //ContentProvider未在运行
        if (!providerRunning) {
            //通过PMS获取ContentProvider信息
            try {
                cpi = AppGlobals.getPackageManager().
                    resolveContentProvider(name,
                        STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
            } catch (RemoteException ex) {
            }
            if (cpi == null) {
                return null;
            }
            // If the provider is a singleton AND
            // (it's a call within the same user || the provider is a
            // privileged app)
            // Then allow connecting to the singleton provider
            boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                    cpi.name, cpi.flags)
                    && isValidSingletonCall(r == null ? callingUid : r.uid,
                            cpi.applicationInfo.uid);
            if (singleton) {
                userId = UserHandle.USER_SYSTEM;
            }
            cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);

            ... //各项检查

            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
            //通过Class（android:name属性）获取ContentProviderRecord
            cpr = mProviderMap.getProviderByClass(comp, userId);
            //此ContentProvider是第一次运行
            boolean firstClass = cpr == null;
            if (firstClass) {
                final long ident = Binder.clearCallingIdentity();

                ... //权限处理

                try {
                    //获取应用信息
                    ApplicationInfo ai =
                        AppGlobals.getPackageManager().
                            getApplicationInfo(
                                    cpi.applicationInfo.packageName,
                                    STOCK_PM_FLAGS, userId);
                    if (ai == null) {
                        Slog.w(TAG, "No package info for content provider "
                                + cpi.name);
                        return null;
                    }
                    ai = getAppInfoForUser(ai, userId);
                    //新建ContentProvider记录
                    cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
                } catch (RemoteException ex) {
                    // pm is in same process, this will never happen.
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            } else if (dyingProc == cpr.proc && dyingProc != null) {
                // The old stable connection's client should be killed during proc cleaning up,
                // so do not re-use the old ContentProviderRecord, otherwise the new clients
                // could get killed unexpectedly.
                //旧的ContentProvider进程在死亡过程中，不要复用旧的ContentProviderRecord，避免出现预期之外的问题
                cpr = new ContentProviderRecord(cpr);
                // This is sort of "firstClass"
                firstClass = true;
            }

            //如果此ContentProvider可以在调用者进程中直接运行（同一个App的同进程 或 同一个App且Provider组件支持多进程）
            //直接返回一个新的ContentProviderHolder让调用者进程自己启动ContentProvider
            if (r != null && cpr.canRunHere(r)) {
                // If this is a multiprocess provider, then just return its
                // info and allow the caller to instantiate it.  Only do
                // this if the provider is the same user as the caller's
                // process, or can run as root (so can be in any process).
                return cpr.newHolder(null);
            }

            // This is single process, and our app is now connecting to it.
            // See if we are already in the process of launching this
            // provider.
            //查找正在启动中的ContentProvider
            final int N = mLaunchingProviders.size();
            int i;
            for (i = 0; i < N; i++) {
                if (mLaunchingProviders.get(i) == cpr) {
                    break;
                }
            }

            // If the provider is not already being launched, then get it
            // started.
            //目标ContentProvider不在启动中
            if (i >= N) {
                final long origId = Binder.clearCallingIdentity();

                try {
                    // Content provider is now in use, its package can't be stopped.
                    //将App状态置为unstopped，设置休眠状态为false
                    try {
                        AppGlobals.getPackageManager().setPackageStoppedState(
                                cpr.appInfo.packageName, false, userId);
                    } catch (RemoteException e) {
                    } catch (IllegalArgumentException e) {
                        Slog.w(TAG, "Failed trying to unstop package "
                                + cpr.appInfo.packageName + ": " + e);
                    }

                    // Use existing process if already started
                    //获取目标ContentProvider所在进程记录
                    ProcessRecord proc = getProcessRecordLocked(
                            cpi.processName, cpr.appInfo.uid, false);
                    if (proc != null && proc.thread != null && !proc.killed) { //进程存活
                        if (!proc.pubProviders.containsKey(cpi.name)) {
                            //将ContentProviderRecord保存到进程已发布ContentProvider列表中
                            proc.pubProviders.put(cpi.name, cpr);
                            try {
                                //调度ActivityThread直接安装ContentProvider
                                proc.thread.scheduleInstallProvider(cpi);
                            } catch (RemoteException e) {
                            }
                        }
                    } else { //进程死亡
                        //启动App（App启动过程中会自动启动ContentProvider）
                        proc = startProcessLocked(cpi.processName,
                                cpr.appInfo, false, 0,
                                new HostingRecord("content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name)),
                                ZYGOTE_POLICY_FLAG_EMPTY, false, false, false);
                        if (proc == null) {
                            ...
                            return null;
                        }
                    }
                    cpr.launchingApp = proc;
                    //将目标ContentProvider添加到启动中列表
                    mLaunchingProviders.add(cpr);
                } finally {
                    Binder.restoreCallingIdentity(origId);
                }
            }

            // Make sure the provider is published (the same provider class
            // may be published under multiple names).
            if (firstClass) {
                mProviderMap.putProviderByClass(comp, cpr);
            }

            mProviderMap.putProviderByName(name, cpr);
            //获取连接并更新引用计数
            conn = incProviderCountLocked(r, cpr, token, callingUid, callingPackage, callingTag,
                    stable);
            if (conn != null) {
                conn.waiting = true;
            }
        }

        grantImplicitAccess(userId, null /*intent*/, callingUid,
                UserHandle.getAppId(cpi.applicationInfo.uid));
    }
    ...
}
```

第三部分，如果目标`ContentProvider`未在运行，先通过`PMS`获取`ContentProvider`信息，接着尝试通过Class（`android:name`属性）获取`ContentProviderRecord`，如果获取不到，说明这个`ContentProvider`是第一次运行（开机后），这种情况下需要新建`ContentProviderRecord`，如果获取到了，但是其所在进程被标记为正在死亡，此时同样需要新建`ContentProviderRecord`，不要复用旧的`ContentProviderRecord`，避免出现预期之外的问题

接下来同样检查目标`ContentProvider`是否可以在调用者进程中直接运行，如果可以直接返回一个新的`ContentProviderHolder`让调用者进程自己启动获取`ContentProvider`

接着检查正在启动中的`ContentProvider`列表，如果不在列表中，我们可能需要手动启动它，此时又有两种情况：

1. `ContentProvider`所在进程已启动：如果进程已发布`ContentProvider`列表中不包含这个`ContentProviderRecord`，则将其添加到列表中，然后调用目标进程中的`ApplicationThread.scheduleInstallProvider`方法安装启动`ContentProvider`
2. `ContentProvider`所在进程未启动：启动目标进程，目标进程启动过程中会自动安装启动`ContentProvider`（`ActivityThread.handleBindApplication`方法中）

最后更新`mProviderMap`，获取`ContentProviderConnection`连接并更新引用计数

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;
    boolean providerRunning = false;

    // Wait for the provider to be published...
    final long timeout =
            SystemClock.uptimeMillis() + ContentResolver.CONTENT_PROVIDER_READY_TIMEOUT_MILLIS;
    boolean timedOut = false;
    synchronized (cpr) {
        while (cpr.provider == null) {
            //ContentProvider启动过程中进程死亡，返回null
            if (cpr.launchingApp == null) {
                ...
                return null;
            }
            try {
                //计算最大等待时间
                final long wait = Math.max(0L, timeout - SystemClock.uptimeMillis());
                if (conn != null) {
                    conn.waiting = true;
                }
                //释放锁，等待ContentProvider启动完成
                cpr.wait(wait);
                //等待时间已过，ContentProvider还是没能启动完成并发布，超时
                if (cpr.provider == null) {
                    timedOut = true;
                    break;
                }
            } catch (InterruptedException ex) {
            } finally {
                if (conn != null) {
                    conn.waiting = false;
                }
            }
        }
    }
    if (timedOut) {
        ... //超时处理
        return null;
    }

    //返回新的ContentProviderHolder
    return cpr.newHolder(conn);
}
```

第四部分，如果`ContentProvider`已存在，直接新建一个`ContentProviderHolder`返回，如果`ContentProvider`之前不存在，现在正在启动中，则以当前时间加上`CONTENT_PROVIDER_READY_TIMEOUT_MILLIS`推算出一个超时时间，给目标`ContentProviderRecord`上锁后，调用`wait`方法等待，直到`ContentProvider`成功发布后`notify`解除`wait`状态（在`AMS.publishContentProviders`方法中，之后会分析到），或一直等待直到超时。`wait`状态解除后，判断内部`ContentProvider`是否已被赋值，如果没有，则可以断定超时，此时返回`null`，如有，则返回一个新的`ContentProviderHolder`

### ActivityThread.installProvider

由于这个方法同时包含了启动安装本地`ContentProvider`和获取安装远程`ContentProvider`的逻辑，所以放到后面`启动ContentProvider`章节里一起分析

# 启动ContentProvider

从前面的章节`获取ContentProvider`中，我们已经归纳出`ContentProvider`的启动分为两种情况，接着我们就来分析在这两种情况下，`ContentProvider`的启动路径

## 进程已启动

在进程已启动的情况下，如果进程已发布`ContentProvider`列表中不包含这个`ContentProviderRecord`，则将其添加到列表中，然后调用目标进程中的`ApplicationThread.scheduleInstallProvider`方法安装启动`ContentProvider`

`ApplicationThread.scheduleInstallProvider`会通过`Hander`发送一条`what`值为`H.INSTALL_PROVIDER`的消息，我们根据这个`what`值搜索，发现会走到`ActivityThread.handleInstallProvider`方法中，在这个方法内又会调用`installContentProviders`方法安装启动`ContentProvider`

## 进程未启动

在进程未启动的情况下，直接启动目标进程，在之前的文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128#heading-7) 里，我们分析了App的启动流程，其中有两个地方对启动`ContentProvider`至关重要

### AMS.attachApplicationLocked

在这个方法中会调用`generateApplicationProvidersLocked`方法

```java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    ...
    //normalMode一般情况下均为true
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
    ...
}

private final List<ProviderInfo> generateApplicationProvidersLocked(ProcessRecord app) {
    List<ProviderInfo> providers = null;
    try {
        //通过PMS获取App中同一个进程内的所有的ContentProvider组件信息
        providers = AppGlobals.getPackageManager()
                .queryContentProviders(app.processName, app.uid,
                        STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS
                                | MATCH_DEBUG_TRIAGED_MISSING, /*metadastaKey=*/ null)
                .getList();
    } catch (RemoteException ex) {
    }
    int userId = app.userId;
    if (providers != null) {
        int N = providers.size();
        //有必要的情况下进行Map扩容
        app.pubProviders.ensureCapacity(N + app.pubProviders.size());
        for (int i=0; i<N; i++) {
            // TODO: keep logic in sync with installEncryptionUnawareProviders
            ProviderInfo cpi =
                (ProviderInfo)providers.get(i);
            //对于单例ContentProvider，需要在默认用户中启动，如果不是默认用户的话则直接将其丢弃掉，不启动
            boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                    cpi.name, cpi.flags);
            if (singleton && UserHandle.getUserId(app.uid) != UserHandle.USER_SYSTEM) {
                // This is a singleton provider, but a user besides the
                // default user is asking to initialize a process it runs
                // in...  well, no, it doesn't actually run in this process,
                // it runs in the process of the default user.  Get rid of it.
                providers.remove(i);
                N--;
                i--;
                continue;
            }

            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
            ContentProviderRecord cpr = mProviderMap.getProviderByClass(comp, userId);
            if (cpr == null) {
                //新建ContentProviderRecord并将其加入到缓存中
                cpr = new ContentProviderRecord(this, cpi, app.info, comp, singleton);
                mProviderMap.putProviderByClass(comp, cpr);
            }
            //将ContentProviderRecord保存到进程已发布ContentProvider列表中
            app.pubProviders.put(cpi.name, cpr);
            if (!cpi.multiprocess || !"android".equals(cpi.packageName)) {
                // Don't add this if it is a platform component that is marked
                // to run in multiple processes, because this is actually
                // part of the framework so doesn't make sense to track as a
                // separate apk in the process.
                //将App添加至进程中运行的包列表中
                app.addPackage(cpi.applicationInfo.packageName,
                        cpi.applicationInfo.longVersionCode, mProcessStats);
            }
            notifyPackageUse(cpi.applicationInfo.packageName,
                                PackageManager.NOTIFY_PACKAGE_USE_CONTENT_PROVIDER);
        }
    }
    return providers;
}
```

这个方法主要是获取需要启动的`ContentProvider`的`ContentProviderRecord`，如果是第一次启动这个`ContentProvider`则需要新建一个`ContentProviderRecord`并将其存入缓存，然后将其保存到进程已发布`ContentProvider`列表中，以供后面使用。同时这个方法返回了需要启动的`ProviderInfo`列表，`AMS.attachApplicationLocked`方法可以根据这个列表判断是否有需要启动的`ContentProvider`并设置`ContentProvider`启动超时检测

### ActivityThread.handleBindApplication

```java
private void handleBindApplication(AppBindData data) {
    ...
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        //创建Application
        app = data.info.makeApplication(data.restrictedBackupMode, null);
    
        ...

        mInitialApplication = app;

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        //在非受限模式下启动ContentProvider
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                installContentProviders(app, data.providers);
            }
        }

        ...

        //执行Application的onCreate方法
        try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    } finally {
        // If the app targets < O-MR1, or doesn't change the thread policy
        // during startup, clobber the policy to maintain behavior of b/36951662
        if (data.appInfo.targetSdkVersion < Build.VERSION_CODES.O_MR1
                || StrictMode.getThreadPolicy().equals(writesAllowedPolicy)) {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
    ...
}
```

可以看到，在这个方法中直接调用了`installContentProviders`方法安装启动`ContentProvider`

另外提一点，为什么我要把`Application`的创建和`onCreate`也放进来呢？现在市面上有很多库，包括很多教程告诉我们，可以通过注册`ContentProvider`的方式初始化SDK，获取全局`Context`，比如说著名的内存泄漏检测工具`LeakCanary`的新版本，想要使用它，直接添加它的依赖就行了，不需要对代码做哪怕一点的改动，究其原理，就是因为`ContentProvider`的启动时机是在`Application`创建后，`Application.onCreate`调用前，并且`ContentProvider`内的`Context`成员变量大概率就是`Application`，大家以后在开发过程中也可以妙用这一点

## ActivityThread.installContentProviders

好了，现在这两种情况最终都走到了`ActivityThread.installContentProviders`方法中，那我们接下来就好好分析`ContentProvider`实际的启动安装流程

```java
private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();

    for (ProviderInfo cpi : providers) {
        //逐个启动
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        if (cph != null) {
            cph.noReleaseNeeded = true;
            results.add(cph);
        }
    }

    try {
        //发布ContentProvider
        ActivityManager.getService().publishContentProviders(
            getApplicationThread(), results);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```

这个方法很简单，便利所有待启动的`ContentProvider`信息列表，逐个启动安装`ContentProvider`，最后一起发布

### ActivityThread.installProvider

我们先看`installProvider`方法，我们在上一章中分析到，获取`ContentProvider`的时候也会调用这个方法，这次我们就结合起来一起分析

通过上文的代码，我们发现，有两处地方会调用`installProvider`方法，方法的入参有三种形式，分别为：

- `holder`不为`null`，`info`不为`null`，`holder.provider`为`null`：在`ActivityThread.acquireProvider`方法中被调用，路径为 没有获取到已存在的`ContentProvider` -> `AMS.getContentProvider` -> `AMS.getContentProviderImpl` -> 发现目标`ContentProvider`可以在调用者进程中直接运行 -> 直接返回一个新的`ContentProviderHolder`（包含`ProviderInfo`） -> `ActivityThread.installProvider`，在这种情况下`installProvider`方法会在本地启动安装`ContentProvider`

- `holder`为`null`，`info`不为`null`：在`ActivityThread.installContentProviders`方法中被调用，两条路径，一是App进程启动后自动执行，二是在`AMS.getContentProvider`方法中发现目标进程已启动但是`ContentProvider`未启动，调用`ActivityThread.scheduleInstallProvider`方法执行，在这种情况下`installProvider`方法会在本地启动安装`ContentProvider`

- `holder`不为`null`，`holder.provider`不为`null`：在`ActivityThread.acquireProvider`方法中被调用，路径为 没有获取到已存在的`ContentProvider` -> `AMS.getContentProvider` -> `AMS.getContentProviderImpl` -> 获取到目标进程的远程`ContentProvider`引用 -> 包装成`ContentProviderHolder`返回 -> `ActivityThread.installProvider`，在这种情况下`installProvider`方法直接可以获取到远程`ContentProvider`引用，然后进行处理

我们将这三种情况分成两种case分别分析

#### 本地启动ContentProvider

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    if (holder == null || holder.provider == null) { //启动本地ContentProvider
        Context c = null;
        ApplicationInfo ai = info.applicationInfo;
        //首先获取Context，一般情况下就是Application
        if (context.getPackageName().equals(ai.packageName)) {
            c = context;
        } else if (mInitialApplication != null &&
                mInitialApplication.getPackageName().equals(ai.packageName)) {
            c = mInitialApplication;
        } else {
            try {
                c = context.createPackageContext(ai.packageName,
                        Context.CONTEXT_INCLUDE_CODE);
            } catch (PackageManager.NameNotFoundException e) {
                // Ignore
            }
        }
        if (c == null) {
            return null;
        }

        //Split Apks动态加载相关
        if (info.splitName != null) {
            try {
                c = c.createContextForSplit(info.splitName);
            } catch (NameNotFoundException e) {
                throw new RuntimeException(e);
            }
        }

        try {
            final java.lang.ClassLoader cl = c.getClassLoader();
            //获取应用信息
            LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
            if (packageInfo == null) {
                // System startup case.
                packageInfo = getSystemContext().mPackageInfo;
            }
            //通过AppComponentFactory实例化ContentProvider
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
            //Transport类，继承自ContentProviderNative（Binder服务端）
            provider = localProvider.getIContentProvider();
            if (provider == null) {
                return null;
            }
            // XXX Need to create the correct context for this provider.
            //初始化ContentProvider，调用其onCreate方法
            localProvider.attachInfo(c, info);
        } catch (java.lang.Exception e) {
            if (!mInstrumentation.onException(null, e)) {
                throw new RuntimeException(
                        "Unable to get provider " + info.name
                        + ": " + e.toString(), e);
            }
            return null;
        }
    } else { //获取外部ContentProvider
        ...
    }

    ContentProviderHolder retHolder;

    synchronized (mProviderMap) {
        //对于本地ContentProvider来说，这里的实际类型是Transport，继承自ContentProviderNative（Binder服务端）
        IBinder jBinder = provider.asBinder();
        if (localProvider != null) { //本地启动ContentProvider的情况
            ComponentName cname = new ComponentName(info.packageName, info.name);
            ProviderClientRecord pr = mLocalProvidersByName.get(cname);
            if (pr != null) {
                //如果已经存在相应的ContentProvider记录，使用其内部已存在的ContentProvider
                provider = pr.mProvider;
            } else {
                //否则使用新创建的ContentProvider
                holder = new ContentProviderHolder(info);
                holder.provider = provider;
                //对于本地ContentProvider来说，不存在释放引用这种情况
                holder.noReleaseNeeded = true;
                //创建ProviderClientRecord并将其保存到mProviderMap本地缓存中
                pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                //保存ProviderClientRecord到本地缓存中
                mLocalProviders.put(jBinder, pr);
                mLocalProvidersByName.put(cname, pr);
            }
            retHolder = pr.mHolder;
        } else { //获取远程ContentProvider的情况
            ...
        }
    }
    return retHolder;
}
```

我们在这里找到了`ContentProvider`创建并启动的入口，首先通过传入的`Context`（实际上就是`Application`）判断并确定创建并给`ContentProvider`使用的的`Context`是什么（一般情况下也是`Application`），然后获取到应用信息`LoadedApk`，再通过它得到`AppComponentFactory`（前面的文章中介绍过，如果没有在`AndroidManifest`中设置`android:appComponentFactory`属性，使用的便是默认的`AppComponentFactory`），接着通过`AppComponentFactory.instantiateProvider`方法实例化`ContentProvider`对象

```java
public @NonNull ContentProvider instantiateProvider(@NonNull ClassLoader cl,
        @NonNull String className)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (ContentProvider) cl.loadClass(className).newInstance();
}
```

默认的话就是通过`ClassName`反射调用默认构造函数实例化`ContentProvider`对象，最后再调用`ContentProvider.attachInfo`方法初始化

```java
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}

private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    ...
    if (mContext == null) {
        mContext = context;
        ...
        ContentProvider.this.onCreate();
    }
}
```

详细内容我们就不分析了，只需要知道这里给`mContext`赋了值，然后调用了`ContentProvider.onCreate`方法就可以了

到了这一步，`ContentProvider`就算是启动完成了，接下来需要执行一些安装步骤，其实也就是对缓存等进行一些处理。在`ContentProvider`实例化后，会调用其`getIContentProvider`方法给`provider`变量赋值，这里获得的对象其实是一个`Transport`对象，继承自`ContentProviderNative`，是一个`Binder`服务端对象，在`ContentProvider`初始化后，会对`Transport`对象调用`asBinder`方法获得`Binder`对象，这里获得的其实还是自己本身，接着从缓存中尝试获取`ProviderClientRecord`对象，如果获取到了，说明已经存在了相应的`ContentProvider`，使用`ProviderClientRecord`内部的`ContentProvider`，刚刚新创建的那个就可以丢弃了，如果没获取到，就去新建`ContentProviderHolder`以及`ProviderClientRecord`，然后将他们添加到各种缓存中，至此，`ContentProvider`的安装过程也到此结束

#### 获取处理远程ContentProvider

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    if (holder == null || holder.provider == null) { //启动本地ContentProvider
        ...
    } else { //获取外部ContentProvider
        //实际类型为ContentProviderProxy
        provider = holder.provider;
    }

    ContentProviderHolder retHolder;

    synchronized (mProviderMap) {
        //对于外部ContentProvider来说，这里的实际类型是BinderProxy
        IBinder jBinder = provider.asBinder();
        if (localProvider != null) { //本地启动ContentProvider的情况
            ...
        } else { //获取远程ContentProvider的情况
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) { //如果ContentProvider引用已存在
                // We need to transfer our new reference to the existing
                // ref count, releasing the old one...  but only if
                // release is needed (that is, it is not running in the
                // system process).
                //对于远程ContentProvider来说，如果目标App为system应用（UID为ROOT_UID或SYSTEM_UID）
                //并且目标App不为设置（包名不为com.android.settings），则noReleaseNeeded为true
                if (!noReleaseNeeded) {
                    //增加已存在的ContentProvider引用的引用计数
                    incProviderRefLocked(prc, stable);
                    try {
                        //释放传入的引用，移除ContentProviderConnection相关信息，更新引用计数
                        ActivityManager.getService().removeContentProvider(
                                holder.connection, stable);
                    } catch (RemoteException e) {
                        //do nothing content provider object is dead any way
                    }
                }
            } else {
                //创建ProviderClientRecord并将其保存到mProviderMap本地缓存中
                ProviderClientRecord client = installProviderAuthoritiesLocked(
                        provider, localProvider, holder);
                if (noReleaseNeeded) { //同上，目标App为system应用，不需要释放引用
                    //新建一个ProviderRefCount，但引用计数初始化为一个较大的数值
                    //这样后续无论调用方进程的ContentProvider引用计数如何变动都不会影响到AMS
                    prc = new ProviderRefCount(holder, client, 1000, 1000);
                } else { //需要释放引用的情况下
                    //正常的新建初始化一个ProviderRefCount
                    prc = stable
                            ? new ProviderRefCount(holder, client, 1, 0)
                            : new ProviderRefCount(holder, client, 0, 1);
                }
                //保存至缓存
                mProviderRefCountMap.put(jBinder, prc);
            }
            retHolder = prc.holder;
        }
    }
    return retHolder;
}
```

对于`holder.provider`不为`null`的情况，直接获取远程`ContentProvider`引用，然后进行处理就可以了。这里获取到的`IContentProvider`的实际类型是`ContentProviderProxy`，然后对其调用`asBinder`方法，获取到的是`BinderProxy`对象，接着从缓存中尝试获取`ProviderRefCount`对象，如果缓存中已经有相应的引用对象了，则在需要释放引用（`!noReleaseNeeded`）的情况下使用原有的引用，释放参数传入进来的`ContentProvider`引用

这里`noReleaseNeeded`是在`ContentProviderRecord`构造时赋值的，为`true`的条件是目标App为system应用（`UID`为`ROOT_UID`或`SYSTEM_UID`）并且目标App不为设置（包名不为`com.android.settings`）

如果缓存中没有查找到相应的`ProviderRefCount`对象，新建`ProviderClientRecord`和`ProviderRefCount`对象，并将他们保存到缓存中，至于为什么在`noReleaseNeeded`的情况下，新建的`ProviderRefCount`的引用计数初始值为1000，我猜测是因为`noReleaseNeeded`代表了不需要释放引用，所以这里干脆设置一个比较大的值，这样无论调用方进程的`ContentProvider`引用计数怎样变动，都不会再调用到`AMS`的方法中去处理引用的变化，在非常早期的`Android`版本中（`Android 4.0.1`），这个值曾被设置为`10000`

至此，远程`ContentProvider`的安装也结束了

#### ActivityThread.installProviderAuthoritiesLocked

接下来我们再简单的看一下两种case都会走到的`installProviderAuthoritiesLocked`方法吧

```java
private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
        ContentProvider localProvider, ContentProviderHolder holder) {
    final String auths[] = holder.info.authority.split(";");
    final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid);
    ...
    final ProviderClientRecord pcr = new ProviderClientRecord(
            auths, provider, localProvider, holder);
    for (String auth : auths) {
        final ProviderKey key = new ProviderKey(auth, userId);
        final ProviderClientRecord existing = mProviderMap.get(key);
        if (existing != null) {
            Slog.w(TAG, "Content provider " + pcr.mHolder.info.name
                    + " already published as " + auth);
        } else {
            mProviderMap.put(key, pcr);
        }
    }
    return pcr;
}
```

这个方法很简单，新建了一个`ProviderClientRecord`并将其添加到`mProviderMap`缓存中，这里的操作对应着最前面的`acquireExistingProvider`方法，有了这个缓存，以后就可以直接拿，而不用再复杂的经过一系列的`AMS`跨进程操作了

### AMS.publishContentProviders

`ContentProvider`全部启动安装完后，便要调用`AMS.publishContentProviders`将他们发布出去，供外部使用了

```java
public final void publishContentProviders(IApplicationThread caller,
        List<ContentProviderHolder> providers) {
    if (providers == null) {
        return;
    }

    synchronized (this) {
        final ProcessRecord r = getRecordForAppLocked(caller);
        if (r == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when publishing content providers");
        }

        final long origId = Binder.clearCallingIdentity();

        final int N = providers.size();
        for (int i = 0; i < N; i++) {
            ContentProviderHolder src = providers.get(i);
            if (src == null || src.info == null || src.provider == null) {
                continue;
            }
            //App进程启动时或AMS.getContentProvider中已经将相应ContentProviderRecord添加到了pubProviders中
            ContentProviderRecord dst = r.pubProviders.get(src.info.name);
            if (dst != null) {
                //保存至缓存中
                ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                mProviderMap.putProviderByClass(comp, dst);
                String names[] = dst.info.authority.split(";");
                for (int j = 0; j < names.length; j++) {
                    mProviderMap.putProviderByName(names[j], dst);
                }

                //ContentProvider已经启动完毕，将其从正在启动的ContentProvider列表中移除
                int launchingCount = mLaunchingProviders.size();
                int j;
                boolean wasInLaunchingProviders = false;
                for (j = 0; j < launchingCount; j++) {
                    if (mLaunchingProviders.get(j) == dst) {
                        mLaunchingProviders.remove(j);
                        wasInLaunchingProviders = true;
                        j--;
                        launchingCount--;
                    }
                }
                //移除ContentProvider启动超时监听
                if (wasInLaunchingProviders) {
                    mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                }
                // Make sure the package is associated with the process.
                // XXX We shouldn't need to do this, since we have added the package
                // when we generated the providers in generateApplicationProvidersLocked().
                // But for some reason in some cases we get here with the package no longer
                // added...  for now just patch it in to make things happy.
                r.addPackage(dst.info.applicationInfo.packageName,
                        dst.info.applicationInfo.longVersionCode, mProcessStats);
                synchronized (dst) {
                    dst.provider = src.provider;
                    dst.setProcess(r);
                    //让出锁，通知其他wait的地方
                    //对应着AMS.getContentProvider的第四部分：等待ContentProvider启动完成
                    dst.notifyAll();
                }
                dst.mRestartCount = 0;
                updateOomAdjLocked(r, true, OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
                maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                        src.info.authority);
            }
        }

        Binder.restoreCallingIdentity(origId);
    }
}
```

遍历整个待发布的`ContentProvider`列表，从`ProcessRecord.pubProviders`中查找相对应的`ContentProviderRecord`，我们在之前的章节中已经分析过了，App进程启动时或`AMS.getContentProvider`中已经将相应`ContentProviderRecord`添加到了`pubProviders`中，然后就是将其保存到各个缓存中，由于`ContentProvider`已经启动完毕，所以需要将其从正在启动的`ContentProvider`列表中移除，在`ContentProvider`正常启动的情况下，我们需要将`ContentProvider`的启动超时监听移除，最后，获取`ContentProviderRecord`同步锁，将准备好的`ContentProvider`赋值到`ContentProviderRecord`中，接着调用`notifyAll`方法通知其他调用过`wait`的地方，将锁让出，这里对应的就是`AMS.getContentProvider`的第四部分：等待`ContentProvider`启动完成

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
    ...
    //这里的cpr和在publishContentProviders获得的dst是一个对象
    synchronized (cpr) {
        while (cpr.provider == null) {
            ...
            //释放锁，等待ContentProvider启动完成
            cpr.wait(wait);
            ...
        }
    }
    ...
}
```

这样，`ContentProvider`一发布，这里就会收到通知，解除`wait`状态，获得到`ContentProvider`，返回出去，是不是感觉一切都串起来了？

# ContentProvider引用计数

`ContentProvider`的获取与启动分析完了，接下来我们聊聊它的引用计数，为下一小节分析`ContentProvider`死亡杀死调用方进程的过程做准备

`ActivityThread`层的引用计数是和`AMS`层的引用计数分开的，`ActivityThread`记录的是目标`ContentProvider`在本进程中有多少处正在使用，而`AMS`记录的是目标`ContentProvider`正在被多少个进程使用

## ActivityThread层的引用计数

### 增加引用计数

我们先从`ActivityThread`层增加引用计数开始说起，在`ActivityThread`获取`ContentProvider`时，便会调用`incProviderRefLocked`方法来增加引用计数，具体的时机为`acquireExistingProvider`或`installProvider`时，代码我就不重复放了，大家看前面几个小节就行（后同）

```java
private final void incProviderRefLocked(ProviderRefCount prc, boolean stable) {
    if (stable) {
        //增加ActivityThread的stable引用计数
        prc.stableCount += 1;
        //本进程对目标ContentProvider产生了stable引用关系
        if (prc.stableCount == 1) {
            // We are acquiring a new stable reference on the provider.
            int unstableDelta;
            //正在移除ContentProvider引用中（释放ContentProvider后发现stable和unstable引用计数均为0）
            if (prc.removePending) {
                // We have a pending remove operation, which is holding the
                // last unstable reference.  At this point we are converting
                // that unstable reference to our new stable reference.
                //当ActivityThread释放一个stable的ContentProvider时，如果释放完后，
                //发现stable和unstable引用计数均为0，则会暂时保留一个unstable引用
                //所以这里需要为 -1 ，将这个unstable引用移除
                unstableDelta = -1;
                // Cancel the removal of the provider.
                prc.removePending = false;
                // There is a race! It fails to remove the message, which
                // will be handled in completeRemoveProvider().
                //取消移除ContentProvider引用
                mH.removeMessages(H.REMOVE_PROVIDER, prc);
            } else {
                //对于正常情况，只需要增加stable引用计数，不需要动unstable引用计数
                unstableDelta = 0;
            }
            try {
                //AMS层修改引用计数
                ActivityManager.getService().refContentProvider(
                        prc.holder.connection, 1, unstableDelta);
            } catch (RemoteException e) {
                //do nothing content provider object is dead any way
            }
        }
    } else {
        //增加ActivityThread的unstable引用计数
        prc.unstableCount += 1;
        //本进程对目标ContentProvider产生了unstable引用关系
        if (prc.unstableCount == 1) {
            // We are acquiring a new unstable reference on the provider.
            //正在移除ContentProvider引用中（释放ContentProvider后发现stable和unstable引用计数均为0）
            if (prc.removePending) {
                // Oh look, we actually have a remove pending for the
                // provider, which is still holding the last unstable
                // reference.  We just need to cancel that to take new
                // ownership of the reference.
                //取消移除ContentProvider引用
                prc.removePending = false;
                mH.removeMessages(H.REMOVE_PROVIDER, prc);
            } else {
                // First unstable ref, increment our count in the
                // activity manager.
                try {
                    //增加AMS层的unstable引用计数
                    ActivityManager.getService().refContentProvider(
                            prc.holder.connection, 0, 1);
                } catch (RemoteException e) {
                    //do nothing content provider object is dead any way
                }
            }
        }
    }
}
```

这里的逻辑需要配合着`ContentProvider`释放引用那里一起看才好理解，我先提前解释一下

首先，`removePending`这个变量表示此`ContentProvider`正在移除中，当`ActivityThread`减少引用计数，检查到`stable`和`unstable`引用计数均为`0`后被赋值为`true`，并且会向`Handler`发送一条`what`值为`REMOVE_PROVIDER`的延时消息，在一定时间后便会触发`ContentProvider`移除操作，清理本地缓存，再将`removePending`重新置为`false`，所以当这里`removePending`为`true`则说明此`ContentProvider`还没完全被移除，我们把这个消息取消掉继续使用这个`ContentProvider`

对于`stable`引用的情况下，当`ActivityThread`减少引用计数，检查到`stable`和`unstable`引用计数均为`0`后，会暂时保留一个`unstable`引用，等到后面真正触发到了移除`ContentProvider`的时候再将这个`unstable`引用移除，所以在增加引用计数的时候需要考虑到这一点，在这种情况下要将`AMS`层的`unstable`引用计数减一

对于其他的情况就是正常的增加`ActivityThread`层引用计数，然后调用`AMS.refContentProvider`方法操作`AMS`层的引用计数

### 减少引用计数

`ContentProvider`使用完后会调用`ActivityThread.releaseProvider`方法，以`query`方法为例，最后会调用`releaseUnstableProvider`和`releaseProvider`方法，最终都会走到这里来

```java
public final boolean releaseProvider(IContentProvider provider, boolean stable) {
    if (provider == null) {
        return false;
    }

    IBinder jBinder = provider.asBinder();
    synchronized (mProviderMap) {
        ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
        if (prc == null) {
            // The provider has no ref count, no release is needed.
            return false;
        }

        boolean lastRef = false;
        if (stable) {
            //引用计数已经为0，无法再减了
            if (prc.stableCount == 0) {
                return false;
            }
            //减少ActivityThread的stable引用计数
            prc.stableCount -= 1;
            if (prc.stableCount == 0) {
                // What we do at this point depends on whether there are
                // any unstable refs left: if there are, we just tell the
                // activity manager to decrement its stable count; if there
                // aren't, we need to enqueue this provider to be removed,
                // and convert to holding a single unstable ref while
                // doing so.
                lastRef = prc.unstableCount == 0;
                try {
                    //如果是最后的引用，则暂时保留一个unstable引用
                    ActivityManager.getService().refContentProvider(
                            prc.holder.connection, -1, lastRef ? 1 : 0);
                } catch (RemoteException e) {
                    //do nothing content provider object is dead any way
                }
            }
        } else {
            //引用计数已经为0，无法再减了
            if (prc.unstableCount == 0) {
                return false;
            }
            //减少ActivityThread的unstable引用计数
            prc.unstableCount -= 1;
            if (prc.unstableCount == 0) {
                // If this is the last reference, we need to enqueue
                // this provider to be removed instead of telling the
                // activity manager to remove it at this point.
                lastRef = prc.stableCount == 0;
                //如果是最后的引用，则不进入到这里，暂时保留一个unstable引用
                if (!lastRef) {
                    try {
                        //减少AMS引用计数
                        ActivityManager.getService().refContentProvider(
                                prc.holder.connection, 0, -1);
                    } catch (RemoteException e) {
                        //do nothing content provider object is dead any way
                    }
                }
            }
        }

        if (lastRef) {
            if (!prc.removePending) {
                // Schedule the actual remove asynchronously, since we don't know the context
                // this will be called in.
                //表面此ContentProvider正在移除中
                prc.removePending = true;
                //发送延时消息，等待一定时间后移除ContentProvider
                Message msg = mH.obtainMessage(H.REMOVE_PROVIDER, prc);
                mH.sendMessageDelayed(msg, CONTENT_PROVIDER_RETAIN_TIME);
            } else {
                Slog.w(TAG, "Duplicate remove pending of provider " + prc.holder.info.name);
            }
        }
        return true;
    }
}
```

可以看到，在减完引用计数后，如果发现是最后一个引用，即`stable`和`unstable`引用计数均为`0`，此时无论是`stable`还是`unstable`都会让`AMS`暂时保留一个`unstable`引用，然后发送一条`what`值为`REMOVE_PROVIDER`的延时消息，等待一定时间后移除`ContentProvider`，当时间到了触发这条消息时，会调用到`ActivityThread.completeRemoveProvider`方法

```java
final void completeRemoveProvider(ProviderRefCount prc) {
    synchronized (mProviderMap) {
        if (!prc.removePending) {
            // There was a race!  Some other client managed to acquire
            // the provider before the removal was completed.
            // Abort the removal.  We will do it later.
            return;
        }

        // More complicated race!! Some client managed to acquire the
        // provider and release it before the removal was completed.
        // Continue the removal, and abort the next remove message.
        prc.removePending = false;

        //移除缓存
        final IBinder jBinder = prc.holder.provider.asBinder();
        ProviderRefCount existingPrc = mProviderRefCountMap.get(jBinder);
        if (existingPrc == prc) {
            mProviderRefCountMap.remove(jBinder);
        }

        //移除缓存
        for (int i=mProviderMap.size()-1; i>=0; i--) {
            ProviderClientRecord pr = mProviderMap.valueAt(i);
            IBinder myBinder = pr.mProvider.asBinder();
            if (myBinder == jBinder) {
                mProviderMap.removeAt(i);
            }
        }
    }

    try {
        //处理AMS层引用计数
        ActivityManager.getService().removeContentProvider(
                prc.holder.connection, false);
    } catch (RemoteException e) {
        //do nothing content provider object is dead any way
    }
}
```

这个方法将进程内所持有的`ContentProvider`相关缓存清除，然后调用`AMS.removeContentProvider`方法通知`AMS`移除`ContentProvider`，处理相应的引用计数。这里我们发现，调用`AMS.removeContentProvider`方法传入的最后一个参数`stable`为`false`，因为我们之前在`stable`和`unstable`引用计数均为`0`的情况下，保留了一个`unstable`引用，所以这时移除的`ContentProvider`引用也是`unstable`引用

## AMS层的引用计数

接着我们来看`AMS`层的引用计数

### AMS.refContentProvider

我们就先从我们刚刚分析的`ActivityThread`层的引用计数修改后续：`refContentProvider` 看起

```java
public boolean refContentProvider(IBinder connection, int stable, int unstable) {
    ContentProviderConnection conn;
    ...
    conn = (ContentProviderConnection)connection;
    ...

    synchronized (this) {
        if (stable > 0) {
            conn.numStableIncs += stable;
        }
        stable = conn.stableCount + stable;
        if (stable < 0) {
            throw new IllegalStateException("stableCount < 0: " + stable);
        }

        if (unstable > 0) {
            conn.numUnstableIncs += unstable;
        }
        unstable = conn.unstableCount + unstable;
        if (unstable < 0) {
            throw new IllegalStateException("unstableCount < 0: " + unstable);
        }

        if ((stable+unstable) <= 0) {
            throw new IllegalStateException("ref counts can't go to zero here: stable="
                    + stable + " unstable=" + unstable);
        }
        conn.stableCount = stable;
        conn.unstableCount = unstable;
        return !conn.dead;
    }
}
```

这个方法很简单，应该不需要再多做分析了吧？就是简单的修改`ContentProviderConnection`的引用计数值

### AMS.incProviderCountLocked

接下来我们看`AMS`层引用计数的增加，`AMS.incProviderCountLocked`这个方法的触发时机是在`AMS.getContentProviderImpl`方法中

```java
ContentProviderConnection incProviderCountLocked(ProcessRecord r,
        final ContentProviderRecord cpr, IBinder externalProcessToken, int callingUid,
        String callingPackage, String callingTag, boolean stable) {
    if (r != null) {
        for (int i=0; i<r.conProviders.size(); i++) {
            ContentProviderConnection conn = r.conProviders.get(i);
            //如果连接已存在，在其基础上增加引用计数
            if (conn.provider == cpr) {
                if (stable) {
                    conn.stableCount++;
                    conn.numStableIncs++;
                } else {
                    conn.unstableCount++;
                    conn.numUnstableIncs++;
                }
                return conn;
            }
        }
        //新建ContentProviderConnection连接
        ContentProviderConnection conn = new ContentProviderConnection(cpr, r, callingPackage);
        //建立关联
        conn.startAssociationIfNeeded();
        if (stable) {
            conn.stableCount = 1;
            conn.numStableIncs = 1;
        } else {
            conn.unstableCount = 1;
            conn.numUnstableIncs = 1;
        }
        //添加连接
        cpr.connections.add(conn);
        r.conProviders.add(conn);
        //建立关联
        startAssociationLocked(r.uid, r.processName, r.getCurProcState(),
                cpr.uid, cpr.appInfo.longVersionCode, cpr.name, cpr.info.processName);
        return conn;
    }
    cpr.addExternalProcessHandleLocked(externalProcessToken, callingUid, callingTag);
    return null;
}
```

如果调用方进程已存在对应`ContentProviderConnection`连接，则在其基础上增加引用计数，否则新建连接，然后初始化引用计数值

### AMS.decProviderCountLocked

然后是减少引用计数，之前在`ActivityThread`减引用到0后，会延时调用`ActivityThread.completeRemoveProvider`方法，在这个方法中会调用到`AMS.removeContentProvider`方法

```java
public void removeContentProvider(IBinder connection, boolean stable) {
    long ident = Binder.clearCallingIdentity();
    try {
        synchronized (this) {
            ContentProviderConnection conn = (ContentProviderConnection)connection;
            ...
            //减少引用计数
            if (decProviderCountLocked(conn, null, null, stable)) {
                //更新进程优先级
                updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_REMOVE_PROVIDER);
            }
        }
    } finally {
        Binder.restoreCallingIdentity(ident);
    }
}
```

在这个方法中便会调用`AMS.decProviderCountLocked`减少引用计数，然后更新进程优先级

```java
boolean decProviderCountLocked(ContentProviderConnection conn,
        ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
    if (conn != null) {
        cpr = conn.provider;
        //减少引用计数值
        if (stable) {
            conn.stableCount--;
        } else {
            conn.unstableCount--;
        }
        if (conn.stableCount == 0 && conn.unstableCount == 0) {
            //停止关联
            conn.stopAssociation();
            //移除连接
            cpr.connections.remove(conn);
            conn.client.conProviders.remove(conn);
            if (conn.client.setProcState < PROCESS_STATE_LAST_ACTIVITY) {
                // The client is more important than last activity -- note the time this
                // is happening, so we keep the old provider process around a bit as last
                // activity to avoid thrashing it.
                if (cpr.proc != null) {
                    cpr.proc.lastProviderTime = SystemClock.uptimeMillis();
                }
            }
            //停止关联
            stopAssociationLocked(conn.client.uid, conn.client.processName, cpr.uid,
                    cpr.appInfo.longVersionCode, cpr.name, cpr.info.processName);
            return true;
        }
        return false;
    }
    cpr.removeExternalProcessHandleLocked(externalProcessToken);
    return false;
}
```

减少引用计数值，如果`stable`和`unstable`引用计数均为`0`，则将这个连接移除

# ContentProvider死亡杀死调用方进程的过程

我们前面提到过，`ContentProvider`所在进程死亡会将与其所有有`stable`关联的调用方进程杀死，这是怎么做到的呢？在之前的文章中，我们介绍过进程启动时，在调用`AMS.attachApplicationLocked`时会注册一个App进程死亡回调，我们就从进程死亡，触发死亡回调开始分析

```java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    ...
    //注册App进程死亡回调
    AppDeathRecipient adr = new AppDeathRecipient(
            app, pid, thread);
    thread.asBinder().linkToDeath(adr, 0);
    app.deathRecipient = adr;
    ...
}
```

注册了死亡回调后，如果对应`binder`进程死亡，便会回调`IBinder.DeathRecipient.binderDied`方法，我们来看一下`AppDeathRecipient`对这个方法的实现

```java
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

直接转手调用`AMS.appDiedLocked`方法，然后经过`handleAppDiedLocked`调用到`cleanUpApplicationRecordLocked`方法中

```java
final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
    ...

    boolean restart = false;

    // Remove published content providers.
    //清除已发布的ContentProvider
    for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
        ContentProviderRecord cpr = app.pubProviders.valueAt(i);
        if (cpr.proc != app) {
            // If the hosting process record isn't really us, bail out
            continue;
        }
        final boolean alwaysRemove = app.bad || !allowRestart;
        final boolean inLaunching = removeDyingProviderLocked(app, cpr, alwaysRemove);
        if (!alwaysRemove && inLaunching && cpr.hasConnectionOrHandle()) {
            // We left the provider in the launching list, need to
            // restart it.
            restart = true;
        }

        cpr.provider = null;
        cpr.setProcess(null);
    }
    app.pubProviders.clear();

    // Take care of any launching providers waiting for this process.
    //清除正在启动中的ContentProvider
    if (cleanupAppInLaunchingProvidersLocked(app, false)) {
        mProcessList.noteProcessDiedLocked(app);
        restart = true;
    }

    // Unregister from connected content providers.
    //清除已连接的ContentProvider
    if (!app.conProviders.isEmpty()) {
        for (int i = app.conProviders.size() - 1; i >= 0; i--) {
            ContentProviderConnection conn = app.conProviders.get(i);
            conn.provider.connections.remove(conn);
            stopAssociationLocked(app.uid, app.processName, conn.provider.uid,
                    conn.provider.appInfo.longVersionCode, conn.provider.name,
                    conn.provider.info.processName);
        }
        app.conProviders.clear();
    }
    ...
}
```

可以看到，这个方法中遍历了`ProcessRecord.pubProviders`，逐个对发布的`ContentProvider`调用`removeDyingProviderLocked`方法执行移除操作

```java
private final boolean removeDyingProviderLocked(ProcessRecord proc,
        ContentProviderRecord cpr, boolean always) {
    boolean inLaunching = mLaunchingProviders.contains(cpr);
    if (inLaunching && !always && ++cpr.mRestartCount > ContentProviderRecord.MAX_RETRY_COUNT) {
        // It's being launched but we've reached maximum attempts, force the removal
        always = true;
    }

    if (!inLaunching || always) {
        synchronized (cpr) {
            cpr.launchingApp = null;
            cpr.notifyAll();
        }
        final int userId = UserHandle.getUserId(cpr.uid);
        // Don't remove from provider map if it doesn't match
        // could be a new content provider is starting
        //移除缓存
        if (mProviderMap.getProviderByClass(cpr.name, userId) == cpr) {
            mProviderMap.removeProviderByClass(cpr.name, userId);
        }
        String names[] = cpr.info.authority.split(";");
        for (int j = 0; j < names.length; j++) {
            // Don't remove from provider map if it doesn't match
            // could be a new content provider is starting
            //移除缓存
            if (mProviderMap.getProviderByName(names[j], userId) == cpr) {
                mProviderMap.removeProviderByName(names[j], userId);
            }
        }
    }

    for (int i = cpr.connections.size() - 1; i >= 0; i--) {
        ContentProviderConnection conn = cpr.connections.get(i);
        if (conn.waiting) {
            // If this connection is waiting for the provider, then we don't
            // need to mess with its process unless we are always removing
            // or for some reason the provider is not currently launching.
            if (inLaunching && !always) {
                continue;
            }
        }
        ProcessRecord capp = conn.client;
        conn.dead = true;
        if (conn.stableCount > 0) {
            if (!capp.isPersistent() && capp.thread != null
                    && capp.pid != 0
                    && capp.pid != MY_PID) {
                //当调用方与被杀死的目标ContentProvider进程间有stable连接
                //并且调用方App进程非persistent进程并且非system_server进程中的情况下
                //杀死调用方进程
                capp.kill("depends on provider "
                        + cpr.name.flattenToShortString()
                        + " in dying proc " + (proc != null ? proc.processName : "??")
                        + " (adj " + (proc != null ? proc.setAdj : "??") + ")",
                        ApplicationExitInfo.REASON_DEPENDENCY_DIED,
                        ApplicationExitInfo.SUBREASON_UNKNOWN,
                        true);
            }
        } else if (capp.thread != null && conn.provider.provider != null) {
            try {
                //通知调用方移除ContentProvider
                capp.thread.unstableProviderDied(conn.provider.provider.asBinder());
            } catch (RemoteException e) {
            }
            // In the protocol here, we don't expect the client to correctly
            // clean up this connection, we'll just remove it.
            //移除连接
            cpr.connections.remove(i);
            if (conn.client.conProviders.remove(conn)) {
                stopAssociationLocked(capp.uid, capp.processName, cpr.uid,
                        cpr.appInfo.longVersionCode, cpr.name, cpr.info.processName);
            }
        }
    }

    if (inLaunching && always) {
        mLaunchingProviders.remove(cpr);
        cpr.mRestartCount = 0;
        inLaunching = false;
    }
    return inLaunching;
}
```

可以看到，在这个方法中遍历了`ContentProvider`下的所有连接，当发现有其他进程与自己建立了`stable`连接（`conn.stableCount > 0`），且调用方进程不是`persistent`进程（常驻进程，只有拥有系统签名的App设置这个属性才生效），也不是运行在`system_server`进程，调用`ProcessRecord.kill`方法直接杀死进程，对于没有建立`stable`连接的调用方进程，调用`IApplicationThread.unstableProviderDied`方法通知调用方进程移除相应的`ContentProvider`

所以，使用`ContentProvider`是有一定风险的，大家要注意规避

# 总结

到这里，整个`Framework`层关于`ContentProvider`的内容应该都分析完了，希望大家看完后能获得一些收获，接下来的文章应该会去分析`Service`相关源码，敬请期待~