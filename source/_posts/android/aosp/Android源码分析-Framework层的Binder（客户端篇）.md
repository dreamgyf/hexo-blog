---
title: Android源码分析 - Framework层的Binder（客户端篇）
date: 2022-06-27 11:45:00
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以`aosp`分支`android-11.0.0_r25`作为基础解析**

我们在之前的文章中，从驱动层面分析了`Binder`是怎样工作的，但`Binder`驱动只涉及传输部分，待传输对象是怎么产生的呢，这就是`framework`层的工作了。我们要彻底了解`Binder`的工作原理，不仅要去看驱动层，还得去看`framework`层以及应用层（`AIDL`）

# ServiceManager

## getIServiceManager

我们还是以第一次见到`Binder`的地方`ServiceManager`开始分析，我们选取`getService`方法来分析（这个方法既有入参也有返回），抛除掉它缓存和`log`的部分，最核心的代码就一句`getIServiceManager().getService(name)` 

```java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative
            .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    return sServiceManager;
}
```

### BinderInternal.getContextObject

我们从`BinderInternal.getContextObject()`开始看起，这个函数是一个`native`函数，他被实现在`frameworks/base/core/jni/android_util_Binder.cpp`中

```cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```

#### ProcessState

我们在这里可以发现一个比较关键的类`ProcessState`，它是一个负责打开`binder`驱动并进行`mmap`映射的单例对象，这从它的`self`函数就可以看出来，每个进程只存在一个`ProcessState`实例

位置：`frameworks/native/libs/binder/ProcessState.cpp`

```cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}
```

我们来看看它的构造函数

```cpp
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))    //打开binder驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }
}
```

这里的`:`后是`c++`构造函数初始化赋值的一种语法，可以看到其中调用了`open_driver`函数打开`binder`驱动

```cpp
static int open_driver(const char *driver)
{
    //打开binder驱动
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    int vers = 0;
    //验证binder版本
    status_t result = ioctl(fd, BINDER_VERSION, &vers);
    if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
        ...
    }
    //设置binder最大线程数
    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
    result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
    return fd;
}
```

这里做了三件事，打开`binder`驱动、验证`binder`版本、设置`binder`最大线程数，接着构造函数调用`mmap`建立`binder`映射，这里面的实现我们已经在[Android源码分析 - Binder驱动（上）](https://juejin.cn/post/7062654742329032740)、[（中）](https://juejin.cn/post/7069675794028560391)、[（下）](https://juejin.cn/post/7073783503791325214)中分析过了，感兴趣的同学可以回过头去看一看

`ProcessState::self`函数执行完后，当前进程的`binder`初始化工作已经执行完毕，接下来我们回过头来看它的`getContextObject`函数

```cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);

    if (context == nullptr) {
       ALOGW("Not able to get context object on %s.", mDriverName.c_str());
    }

    // The root object is special since we get it directly from the driver, it is never
    // written by Parcell::writeStrongBinder.
    internal::Stability::tryMarkCompilationUnit(context.get());

    return context;
}
```

我们在`binder`驱动篇就提到了，`handle`句柄`0`代表的就是`ServiceManager`，所以这里调用`getStrongProxyForHandle`函数的参数为`0`

```cpp
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    //查找或建立handle对应的handle_entry
    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                //当handle为ServiceManager的特殊情况
                //需要确保在创建Binder引用之前，context manager已经被binder注册
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
                if (status == DEAD_OBJECT)
                   return nullptr;
            }
            //创建BpBinder并保存下来以便后面再次查找
            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

```cpp
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    //新建一个handle_entry并插入到vector中
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = nullptr;
        e.refs = nullptr;
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return nullptr;
    }
    return &mHandleToObject.editItemAt(handle);
}
```

整条链路下来还是比较清晰的，最终获得了一个`BpBinder`对象，这是`native`中的类型，需要将它转换成`java`中的类型，这里调用了`javaObjectForIBinder`函数，位于`frameworks/base/core/jni/android_util_Binder.cpp`中

#### javaObjectForIBinder

```cpp
// If the argument is a JavaBBinder, return the Java object that was used to create it.
// Otherwise return a BinderProxy for the IBinder. If a previous call was passed the
// same IBinder, and the original BinderProxy is still alive, return the same BinderProxy.
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    if (val == NULL) return NULL;

    //JavaBBinder返回true，其他类均返回flase
    if (val->checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }

    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
    nativeData->mOrgue = new DeathRecipientList;
    nativeData->mObject = val;

    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    if (env->ExceptionCheck()) {
        // In the exception case, getInstance still took ownership of nativeData.
        return NULL;
    }
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    //如果object是刚刚新建出来的BinderProxy
    if (actualNativeData == nativeData) {
        //处理proxy计数
        ...
    } else {
        delete nativeData;
    }

    return object;
}
```

我们先看一看这个`gBinderProxyOffsets`是什么

```cpp
static struct binderproxy_offsets_t
{
    // Class state.
    jclass mClass;
    jmethodID mGetInstance;
    jmethodID mSendDeathNotice;

    // Object state.
    //指向BinderProxyNativeData的指针
    jfieldID mNativeData;  // Field holds native pointer to BinderProxyNativeData.
} gBinderProxyOffsets;

const char* const kBinderProxyPathName = "android/os/BinderProxy";

static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    ...
    jclass clazz = FindClassOrDie(env, kBinderProxyPathName);
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mGetInstance = GetStaticMethodIDOrDie(env, clazz, "getInstance",
            "(JJ)Landroid/os/BinderProxy;");
    gBinderProxyOffsets.mSendDeathNotice =
            GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice",
                                   "(Landroid/os/IBinder$DeathRecipient;Landroid/os/IBinder;)V");
    gBinderProxyOffsets.mNativeData = GetFieldIDOrDie(env, clazz, "mNativeData", "J");
    ...
}
```

可以看到，`gBinderProxyOffsets`实际上是一个用来记录一些`java`中对应类、方法以及字段的结构体，用于从`native`层调用`java`层代码

接下来我们看`javaObjectForIBinder`函数的具体内容

```cpp
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    if (val == NULL) return NULL;

    //JavaBBinder返回true，其他类均返回flase
    if (val->checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }
    ...
}
```

首先有一个`IBinder`类型检查的判断，我看了一圈发现目前只有当`IBinder`的实际类型为`JavaBBinder`的时候会返回`true`，其他子类均返回`false`。`JavaBBinder`类继承自`BBinder`，里面保存了对`java`层`Binder`对象的引用，所以在这种情况下，直接返回里面的`object`就好了。

从这里可以看出，`native`层的`javaBBinder`与`java`层的`Binder`是对应关系

我们这里传进来的是`BpBinder`，会接着往下走

```cpp
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    ...
    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
    nativeData->mOrgue = new DeathRecipientList;
    nativeData->mObject = val;

    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    ...
}
```

接着实例化一个`BinderProxyNativeData`，将`Binder`死亡回调`DeathRecipientList`和`Binder`对象（这里为`BpBinder`）赋值给它，然后调用`java`层方法。`gBinderProxyOffsets`之前说过了，类为`android.os.BinderProxy`，方法为`getInstance`，所以这里调用的即为`android.os.BinderProxy.getInstance(nativeData, iBinder)`，`BinderProxy`的路径为`frameworks/base/core/java/android/os/BinderProxy.java`

```java
private static BinderProxy getInstance(long nativeData, long iBinder) {
    BinderProxy result;
    synchronized (sProxyMap) {
        try {
            result = sProxyMap.get(iBinder);
            if (result != null) {
                return result;
            }
            result = new BinderProxy(nativeData);
        } catch (Throwable e) {
            // We're throwing an exception (probably OOME); don't drop nativeData.
            NativeAllocationRegistry.applyFreeFunction(NoImagePreloadHolder.sNativeFinalizer,
                    nativeData);
            throw e;
        }
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(result, nativeData);
        // The registry now owns nativeData, even if registration threw an exception.
        sProxyMap.set(iBinder, result);
    }
    return result;
}
```

这里的逻辑比较简单，以`iBinder`为 **key** 尝试从`sProxyMap`取出`BinderProxy`，如果取到值了就直接将它返回出去，如果没取到，用之前传进来的`BinderProxyNativeData`指针为参数实例化一个`BinderProxy`，并将其设置到`sProxyMap`中

从这里可以看出每一个服务的`BinderProxy`都是以单例形式存在的，并且`native`层的`BinderProxyNativeData`与`java`层的`BinderProxy`是对应关系

```cpp
BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
}

jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    ...
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    //如果object是刚刚新建出来的BinderProxy
    if (actualNativeData == nativeData) {
        //处理proxy计数
        ...
    } else {
        delete nativeData;
    }

    return object;
}
```

接下来判断我们通过`BinderProxy.getInstance`方法获得的`BinderProxy`是不是刚刚创建出来的，如果是新建的则需要处理一下proxy计数，这里是通过对比`BinderProxy`中的`mNativeData`和我们新建出来的`nativeData`地址判断的

### ServiceManagerNative.asInterface

我们将目光放回`getIServiceManager`方法，现在我们知道`BinderInternal.getContextObject()`方法返回了`ServiceManager`对应的`BinderProxy`，接着会调用`Binder.allowBlocking`方法，这个方法只是改变了`BinderProxy`中的一个参数，使其允许阻塞调用，这样的话`getIServiceManager`就可以被简化成如下代码

```java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative
            .asInterface(/* BinderProxy */);
    return sServiceManager;
}
```

我们看到`asInterface`方法实际上是直接实例化了一个`ServiceManagerProxy`对象

```java
public static IServiceManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }

    // ServiceManager is never local
    return new ServiceManagerProxy(obj);
}
```

## ServiceManagerProxy

从名字就能听出来，`ServiceManagerProxy`其实是一个代理类，它其实是`IServiceManager.Stub.Proxy`的代理，实际上是没有什么必要的，可以发现作者也在注释中标注了`This class should be deleted and replaced with IServiceManager.Stub whenever mRemote is no longer used`，我们看一下它的构造方法

```cpp
public ServiceManagerProxy(IBinder remote) {
    mRemote = remote;
    mServiceManager = IServiceManager.Stub.asInterface(remote);
}
```

`ServiceManagerProxy`实现了`IServiceManager`接口，但这个方法的实现都是直接调用`mServiceManager`，以`addService`举例

```java
public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
        throws RemoteException {
    mServiceManager.addService(name, service, allowIsolated, dumpPriority);
}
```

这与直接使用`IServiceManager.Stub.asInterface(remote)`得到`IServiceManager`并没有什么区别

## IServiceManager

我们将重点转到`IServiceManager`上，我们在源码中搜索不到`IServiceManager.java`文件，因为实际上这个文件是通过`aidl`生成的

关于`aidl`我们到后面再详细分析，现在我们只需要知道它其实是辅助我们进行`binder`通信的一种工具，`aidl`文件会在编译过程中生成出与之对应的`java`文件

`IServiceManager`的`aidl`文件路径为`frameworks/native/libs/binder/aidl/android/os/IServiceManager.aidl`

我们来看一下它生成出的`IServiceManager.Stub.asInterface`方法

```java
public static android.os.IServiceManager asInterface(android.os.IBinder obj)
{
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof android.os.IServiceManager))) {
        return ((android.os.IServiceManager) iin);
    }
    return new android.os.IServiceManager.Stub.Proxy(obj);
}
```

这里我们传入的`IBinder`是`BinderProxy`，它的`queryLocalInterface`永远返回`null`，所以这里返回的是`IServiceManager.Stub.Proxy`对象，我们接着看之前调用的`getService`方法

```java
@Override public android.os.IBinder getService(java.lang.String name) throws android.os.RemoteException
{
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    android.os.IBinder _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeString(name);
        boolean _status = mRemote.transact(Stub.TRANSACTION_getService, _data, _reply, 0);
        _reply.readException();
        _result = _reply.readStrongBinder();
    }
    finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

### Parcel

`Parcel`是一个存放读取数据的容器，它的基本功能和使用相信进阶`Android`开发应该都懂，我们在这里只介绍一些关键性函数的含义，其他就不多赘述了，有机会的话以后单独开一章分析它

|函数|作用|
|---|---|
|obtain|获取一个新的Parcel对象|
|ipcData、data|数据区首地址|
|ipcDataSize、dataSize|数据大小|
|ipcObjects|偏移数组首地址|
|ipcObjectsCount|IPC对象数量|
|dataPosition|数据指针当前的位置|
|dataCapacity|数据区的总容量（始终 >= dataSize）|


这里获取了两个`Parcel`，一个`_data`用来传递参数数据，一个`_reply`用来接收回应。接着，`_data`首先调用`writeInterfaceToken`方法，这里的`token`是客户端与服务端的一个协定，服务端会校验我们写入的这个`token`，然后按照顺序将参数依次写入到`_data`中（序列化），然后通过`binder`调用远程服务真正的方法，然后检查异常。

对于无返回值的方法来说，到这一步已经结束了，但我们这个方法是有返回值的，所以我们需要一个`_result`，从`_reply`中读取出数据（反序列化），赋给`_result`，然后返回出去

## BinderProxy.transact

我们重点看`transact`这一部分，通过我们之前的分析，我们知道`mRemote`是一个`BinderProxy`类型的对象，我们来看他的`transact`方法

```java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    //检查Parcel大小
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");

    ...

    //trace
    ...

    //Binder事务处理回调
    ...

    //AppOpsManager信息记录
    ...

    try {
        final boolean result = transactNative(code, data, reply, flags);
        
        if (reply != null && !warnOnBlocking) {
            reply.addFlags(Parcel.FLAG_IS_REPLY_FROM_BLOCKING_ALLOWED_OBJECT);
        }

        return result;
    } finally {
        ...
    }
}
```

我这里简化了一下代码，可以看到，首先就是对`Parcel`大小的检查

```java
static void checkParcel(IBinder obj, int code, Parcel parcel, String msg) {
    if (CHECK_PARCEL_SIZE && parcel.dataSize() >= 800*1024) {
        // Trying to send > 800k, this is way too much.
        StringBuilder sb = new StringBuilder();
        sb.append(msg);
        sb.append(": on ");
        sb.append(obj);
        sb.append(" calling ");
        sb.append(code);
        sb.append(" size ");
        sb.append(parcel.dataSize());
        sb.append(" (data: ");
        parcel.setDataPosition(0);
        sb.append(parcel.readInt());
        sb.append(", ");
        sb.append(parcel.readInt());
        sb.append(", ");
        sb.append(parcel.readInt());
        sb.append(")");
        Slog.wtfStack(TAG, sb.toString());
    }
}
```

`Android`默认设置了`Parcel`数据传输不能超过**800k**，当然，各个厂商是可以随意改动这里的代码的，如果超过了的话，便会调用`Slog.wtfStack`打印日志，需要注意的是，在当前进程不是系统进程并且系统也不是工程版本的情况下，这个方法是会结束进程的，所以在应用开发的时候，我们需要注意跨进程数据传输的大小，避免因此引发crash

省去中间的一些`log`、回调，接下来便是调用`transactNative`方法，这是一个`native`方法，实现在`frameworks/base/core/jni/android_util_Binder.cpp`中

```cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }

    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }

    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    //log
    ...
    
    status_t err = target->transact(code, *data, reply, flags);
    
    //log
    ...

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
    return JNI_FALSE;
}
```

这里首先是获得`native`层对应的`Parcel`并执行判断，`Parcel`实际上功能是在`native`中实现的，`java`中的`Parcel`类使用`mNativePtr`成员变量保存了其对应`native`中的`Parcel`的指针

然后调用`getBPNativeData`函数获得`BinderProxy`在`native`中对应的`BinderProxyNativeData`，再通过里面的`mObject`域成员变量得到其对应的`BpBinder`，这个函数在之前分析`javaObjectForIBinder`的时候已经出现过了

## BpBinder.transact

之后便是调用`BpBinder`的`transact`函数了

```cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    //判断binder服务是否存活
    if (mAlive) {
        ...
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}
```

这里有一个`Alive`判断，可以避免对一个已经死亡的`binder`服务再发起事务，浪费资源，除此之外便是调用`IPCThreadState`的`transact`函数了

## IPCThreadState

路径：`frameworks/native/libs/binder/IPCThreadState.cpp`

还记得我们之前提到的`ProcessState`吗？`IPCThreadState`和它很像，`ProcessState`负责打开`binder`驱动并进行`mmap`映射，而`IPCThreadState`则是负责与`binder`驱动进行具体的交互

`IPCThreadState`也有一个`self`函数，与`ProcessState`的`self`不同的是，`ProcessState`是进程单例，而`IPCThreadState`是线程单例，我们来看看它是怎么实现的

```cpp
IPCThreadState* IPCThreadState::self()
{
    //不是初次调用的情况
    if (gHaveTLS.load(std::memory_order_acquire)) {
restart:
        //初次调用，生成线程私有变量key后
        const pthread_key_t k = gTLS;
        //先从线程本地储存空间中尝试获取值
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        //没有的话就实例化一个
        return new IPCThreadState;
    }

    //IPCThreadState shutdown后不能再获取
    if (gShutdown.load(std::memory_order_relaxed)) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
        return nullptr;
    }

    //首次获取时gHaveTLS为false，会先走这里
    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS.load(std::memory_order_relaxed)) {
        //创建一个key，作为存放线程本地变量的key
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
                    strerror(key_create_value));
            return nullptr;
        }
        //创建完毕，gHaveTLS置为true
        gHaveTLS.store(true, std::memory_order_release);
    }
    pthread_mutex_unlock(&gTLSMutex);
    //回到gHaveTLS为true的case
    goto restart;
}
```

`gHaveTLS`是一个原子类型的`bool`值，它在存取过程中需要指定内存序`std::memory_order_xxx`，在这里我们直接忽略掉，把它当成一个纯粹的`bool`值就好了

在这里，`TLS`的全称为`Thread Local Storage`，表示线程本地储存空间，和`java`中的`ThreadLocal`其实是一个作用

当一个线程初次获取`IPCThreadState`的时候，会先走到`gHaveTLS`为`false`的case，此时程序会创建一个`key`，作为存放线程本地变量的`key`，创建成功后将`gHaveTLS`置为`true`，然后`goto`到`gHaveTLS`为`true`的case，此时线程本地储存空间中暂时还是没有数据的，所以会`new`一个`IPCThreadState`出来，在`IPCThreadState`的构造函数中，会将自己保存到线程本地储存空间中，这样，当线程第二次再获取`IPCThreadState`的时候，便会直接走到`pthread_getspecific`这里获取并返回

```cpp
IPCThreadState::IPCThreadState()
      : mProcess(ProcessState::self()),
        mServingStackPointer(nullptr),
        mServingStackPointerGuard(nullptr),
        mWorkSource(kUnsetWorkSource),
        mPropagateWorkSource(false),
        mIsLooper(false),
        mIsFlushing(false),
        mStrictModePolicy(0),
        mLastTransactionBinderFlags(0),
        mCallRestriction(mProcess->mCallRestriction) {
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```

我们通过构造函数可以发现，它调用了`pthread_setspecific`函数将自身保存在了线程本地储存空间中

`IPCThreadState`中，成员变量`mIn`用于接收来自`binder`设备的数据，`mOut`用于储存发往`binder`设备的数据，他们的默认容量都为`256字节`

### transact

我们接着看它的`transact`函数

```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    LOG_ALWAYS_FATAL_IF(data.isForRpc(), "Parcel constructed for RPC, but being used with binder.");

    status_t err;

    flags |= TF_ACCEPT_FDS;

    //log
    ...
    
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {    //binder事务不为TF_ONE_WAY
        //当线程限制binder事务不为TF_ONE_WAY时
        if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) {
            if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) {
                //这个限制只是log记录
                ALOGE("Process making non-oneway call (code: %u) but is restricted.", code);
                CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(),
                    ANDROID_LOG_ERROR);
            } else /* FATAL_IF_NOT_ONEWAY */ {
                //这个限制会终止线程
                LOG_ALWAYS_FATAL("Process may not make non-oneway calls (code: %u).", code);
            }
        }

        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }

        //log
        ...
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}
```

这个函数的重点在于`writeTransactionData`和`waitForResponse`，我们依次分析

#### writeTransactionData

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    //目标binder句柄值，ServiceManager为0
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        //数据大小
        tr.data_size = data.ipcDataSize();
        //数据区起始地址
        tr.data.ptr.buffer = data.ipcData();
        //传递的偏移数组大小
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        //偏移数组的起始地址
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    //这里为BC_TRANSACTION
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

在分析这个函数之前，我们需要先回忆一下在前面`binder`驱动章节我们所学习的`binder`结构和通信过程：[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391#heading-13)

`binder_tansaction`首先会读取一个请求码`cmd`，当`binder`请求码为`BC_TRANSACTION`/`BC_REPLY`的时候，`binder`驱动所接收的参数为`binder_transaction_data`结构体，所以在这个函数中，我们将`binder`请求码（这里为`BC_TRANSACTION`）和`binder_transaction_data`结构体依次写入到`mOut`中，为之后`binder_tansaction`做准备

#### waitForResponse

数据准备好后，接着便来到了`waitForResponse`函数

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_ONEWAY_SPAM_SUSPECT:
            ...
        case BR_TRANSACTION_COMPLETE:
            //当TF_ONE_WAY模式下收到BR_TRANSACTION_COMPLETE直接返回，本次binder通信结束
            if (!reply && !acquireResult) goto finish;
            break;
        case BR_DEAD_REPLY:
            ...
        case BR_FAILED_REPLY:
            ...
        case BR_FROZEN_REPLY:
            ...
        case BR_ACQUIRE_RESULT:
            ...
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                //失败直接返回
                if (err != NO_ERROR) goto finish;

                if (reply) {    //客户端需要接收replay
                    if ((tr.flags & TF_STATUS_CODE) == 0) {    //正常reply内容
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer /*释放缓冲区*/);
                    } else {    //内容只是一个32位的状态码
                        //接收状态码
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        //释放缓冲区
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {    //客户端不需要接收replay
                    //释放缓冲区
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;
        default:
            //这里是binder服务端部分的处理，现在不需要关注
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
        logExtendedError();
    }

    return err;
}
```

这里有一个循环，正如函数名所描述，会一直等待到一整条`binder`事务链结束返回后才会退出这个循环，在这个循环的开头，便是`talkWithDriver`方法

##### talkWithDriver

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    //检查打开的binder设备的fd
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    //dataPosition >= dataSize说明上一次读取到的数据已经消费完
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    //需要写的数据大小，这里的doReceive默认为true，如果上一次的数据还没读完，则不会写入任何内容
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        //将read_size设置为读缓存可用容量
        bwr.read_size = mIn.dataCapacity();
        //设置读缓存起始地址
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // Return immediately if there is nothing to do.
    //没有要读写的数据就直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        //调用binder驱动的ioctl
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
            
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        //写数据被消费了
        if (bwr.write_consumed > 0) {
            //写数据没有被消费完
            if (bwr.write_consumed < mOut.dataSize())
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed,
                                 mOut.dataSize());
            else {
                //写数据消费完了，将数据大小设置为0，这样下次就不会再写数据了
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        //读到了数据
        if (bwr.read_consumed > 0) {
            //设置数据大小及数据指针偏移，这样后面就可以从中读取出来数据了
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }

    return err;
}
```

这里的`binder_write_read`也是一个我们熟悉的结构，我们在之前的文章[Android源码分析 - Binder驱动（中）](https://juejin.cn/post/7069675794028560391#heading-10)中了解过，关于`binder`通信的代码，我们需要结合着`binder`驱动一起看才能理解

在`binder`驱动层中，`binder_ioctl_write_read`函数会从用户空间读取一个`binder_write_read`结构，这个结构体主要描述了数据传输的大小和位置以及消费情况（已读/写数据大小），这么看来，`talkWithDriver`函数的结构就很清晰了：

1. 创建出`binder_write_read`结构，根据之前的读取情况，决定是否读写数据，设置写数据内容和大小，设置读数据的空间和容量

2. 调用`binder`驱动的`ioctl`

3. 重置写缓存，根据`ioctl`的结果设置读缓存

这之后，`waitForResponse`函数就可以从读缓存`mIn`中读到数据了，我们回到这个函数中，发现它首先从读缓存中读取了一个`binder`响应码，然后根据这个响应码再处理接下来的工作

##### 处理Reply

在此之前，我们先回顾一下一次`binder_tansaction`的整个过程，根据事务类型，分为两种情况：

- `TF_ONE_WAY`

![binder_oneway](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Framework%E5%B1%82%E7%9A%84Binder%EF%BC%88%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%AF%87%EF%BC%89_oneway.png)

- `非 TF_ONE_WAY`

![binder_non_oneway](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Framework%E5%B1%82%E7%9A%84Binder%EF%BC%88%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%AF%87%EF%BC%89_non_oneway.png)

我们先对照着看`TF_ONE_WAY`的情况

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
        switch (cmd) {
        ...
        case BR_TRANSACTION_COMPLETE:
            //当TF_ONE_WAY模式下收到BR_TRANSACTION_COMPLETE直接返回，本次binder通信结束
            if (!reply && !acquireResult) goto finish;
            break;
        ...
        }
    }
}
```

对于`TF_ONE_WAY`模式来说，客户端在收到`BR_TRANSACTION_COMPLETE`响应码后则返回，不会再等待`BR_REPLY`

而对于非`TF_ONE_WAY`模式来说，客户端不仅会收到`BR_TRANSACTION_COMPLETE`响应码，之后还会阻塞等待`binder`驱动给它发来`BR_REPLY`响应码，这之后一次`binder_transaction`才算完成

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
        switch (cmd) {
        ...
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                //失败直接返回
                if (err != NO_ERROR) goto finish;

                if (reply) {    //客户端需要接收replay
                    if ((tr.flags & TF_STATUS_CODE) == 0) {    //正常reply内容
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer /*释放缓冲区*/);
                    } else {    //内容只是一个32位的状态码
                        //接收状态码
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        //释放缓冲区
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {    //客户端不需要接收replay
                    //释放缓冲区
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;
        ...
        }
    }
}
```

一般来说，非`TF_ONE_WAY`模式肯定是需要一个`reply`来接收的，即`reply != null`，此时我们来看看接收正常`reply`的过程（接收32位状态码没什么好说的，直接从读缓冲区中强制类型转换出一个32位的code就完事了）

这里我们就需要看一下`Parcel`的`ipcSetDataReference`函数了

```cpp
void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize, const binder_size_t* objects,
                                 size_t objectsCount, release_func relFunc) {
    // this code uses 'mOwner == nullptr' to understand whether it owns memory
    LOG_ALWAYS_FATAL_IF(relFunc == nullptr, "must provide cleanup function");
    //先清理重置一下数据和状态
    freeData();

    auto* kernelFields = maybeKernelFields();
    LOG_ALWAYS_FATAL_IF(kernelFields == nullptr); // guaranteed by freeData.

    mData = const_cast<uint8_t*>(data);
    mDataSize = mDataCapacity = dataSize;
    kernelFields->mObjects = const_cast<binder_size_t*>(objects);
    kernelFields->mObjectsSize = kernelFields->mObjectsCapacity = objectsCount;
    mOwner = relFunc;

    //检查数据
    binder_size_t minOffset = 0;
    for (size_t i = 0; i < kernelFields->mObjectsSize; i++) {
        binder_size_t offset = kernelFields->mObjects[i];
        if (offset < minOffset) {
            ALOGE("%s: bad object offset %" PRIu64 " < %" PRIu64 "\n",
                  __func__, (uint64_t)offset, (uint64_t)minOffset);
            kernelFields->mObjectsSize = 0;
            break;
        }
        const flat_binder_object* flat
            = reinterpret_cast<const flat_binder_object*>(mData + offset);
        uint32_t type = flat->hdr.type;
        //binder类型出现异常
        if (!(type == BINDER_TYPE_BINDER || type == BINDER_TYPE_HANDLE ||
              type == BINDER_TYPE_FD)) {
            ...
            kernelFields->mObjectsSize = 0;
            break;
        }
        minOffset = offset + sizeof(flat_binder_object);
    }
    scanForFds();
}
```

其实这个函数也不复杂，我们知道`binder_mmap`做到了一次拷贝，将数据拷贝到了内核物理内存中，然后将其与用户空间虚拟内存做了映射，所以这个函数此时只需要将数据的地址，大小等等无脑赋值进去，客户端后续便可以用`Parcel`提供的函数方便的从中读取数据了

##### freeBuffer

最后我们再来看一下`freeBuffer`这个释放缓冲区的方法，

```cpp
void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
                                size_t /*dataSize*/,
                                const binder_size_t* /*objects*/,
                                size_t /*objectsSize*/)
{
    ...
    if (parcel != nullptr) parcel->closeFileDescriptors();
    IPCThreadState* state = self();
    state->mOut.writeInt32(BC_FREE_BUFFER);
    state->mOut.writePointer((uintptr_t)data);
    state->flushIfNeeded();
}
```

可以看到，这里向`binder`驱动发送了一个`BC_FREE_BUFFER`请求，然后`binder`驱动会负责回收这块缓冲区内存

我们在`Parcel::ipcSetDataReference`函数中可以发现，它将`freeBuffer`函数指针赋值给了`mOwner`，等到什么时候不需要这个`Parcel`了，便会调用这个函数进行缓冲区内存回收

# 结束

到这里，我们客户端与`binder`驱动沟通交互的分析就结束了，相比`binder`驱动而言，`framework`层的`binder`就好理解多了，下一章我们会从服务端的角度来看，它是怎么从`binder`驱动接收并处理客户端的请求的
