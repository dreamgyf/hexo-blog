---
title: Android源码分析 - ContentProvider的启动与交互
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

## installContentProviders

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

我们先看`installProvider`方法，我们在上一章中分析到，获取`ContentProvider`的时候也会调用这个方法，这次我们就结合起来一起分析

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
        provider = holder.provider;
    }

    ContentProviderHolder retHolder;

    synchronized (mProviderMap) {
        //对于本地ContentProvider来说，这里的实际类型是Transport，继承自ContentProviderNative（Binder服务端）
        //对于外部ContentProvider来说，这里的实际类型是BinderProxy
        IBinder jBinder = provider.asBinder();
        if (localProvider != null) {
            ComponentName cname = new ComponentName(info.packageName, info.name);
            ProviderClientRecord pr = mLocalProvidersByName.get(cname);
            if (pr != null) {
                //如果已经存在相应的ContentProvider记录，使用其内部已存在的ContentProvider
                provider = pr.mProvider;
            } else {
                //否则使用新创建的ContentProvider
                holder = new ContentProviderHolder(info);
                holder.provider = provider;
                holder.noReleaseNeeded = true;
                pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                mLocalProviders.put(jBinder, pr);
                mLocalProvidersByName.put(cname, pr);
            }
            retHolder = pr.mHolder;
        } else {
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) {
                if (DEBUG_PROVIDER) {
                    Slog.v(TAG, "installProvider: lost the race, updating ref count");
                }
                // We need to transfer our new reference to the existing
                // ref count, releasing the old one...  but only if
                // release is needed (that is, it is not running in the
                // system process).
                if (!noReleaseNeeded) {
                    incProviderRefLocked(prc, stable);
                    try {
                        ActivityManager.getService().removeContentProvider(
                                holder.connection, stable);
                    } catch (RemoteException e) {
                        //do nothing content provider object is dead any way
                    }
                }
            } else {
                ProviderClientRecord client = installProviderAuthoritiesLocked(
                        provider, localProvider, holder);
                if (noReleaseNeeded) {
                    prc = new ProviderRefCount(holder, client, 1000, 1000);
                } else {
                    prc = stable
                            ? new ProviderRefCount(holder, client, 1, 0)
                            : new ProviderRefCount(holder, client, 0, 1);
                }
                mProviderRefCountMap.put(jBinder, prc);
            }
            retHolder = prc.holder;
        }
    }
    return retHolder;
}
```