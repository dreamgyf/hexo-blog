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
                        //将ContentProvider记录保存到进程发布ContentProvider中
                        if (!proc.pubProviders.containsKey(cpi.name)) {
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

第三部分，如果目标`ContentProvider`未在运行，先通过`PMS`获取`ContentProvider`信息，接着尝试通过Class（`android:name`属性）获取`ContentProviderRecord`，如果获取不到，说明这个`ContentProvider`是第一次运行（开机后），这种情况下需要新建`ContentProviderRecorder`，如果获取到了，但是其所在进程被标记为正在死亡，此时同样需要新建`ContentProviderRecorder`，不要复用旧的`ContentProviderRecord`，避免出现预期之外的问题

接下来同样检查目标`ContentProvider`是否可以在调用者进程中直接运行，如果可以直接返回一个新的`ContentProviderHolder`让调用者进程自己启动获取`ContentProvider`

接着检查正在启动中的`ContentProvider`列表，如果不在列表中，我们需要手动启动它，此时又有两种情况：

1. `ContentProvider`所在进程已启动：调用目标进程中的`ApplicationThread.scheduleInstallProvider`方法安装启动`ContentProvider`
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