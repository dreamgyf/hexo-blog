---
title: Android源码分析 - Framework层的Binder（服务端篇）
date: 2022-07-05 18:48:28
tags: 
- Android源码
- Binder
categories: 
- [Android, 源码分析]
- [Android, Binder]
---

# 开篇

**本篇以aosp分支`android-11.0.0_r25`，kernel分支`android-msm-wahoo-4.4-android11`作为基础解析**

我们在上一片文章[Android源码分析 - Framework层的Binder（客户端篇）](https://juejin.cn/post/7113760814409973790)中，分析了客户端是怎么向服务端通过`binder`驱动发起请求，然后再接收服务端的返回的。本篇文章，我们将会以服务端的视角，分析服务端是怎么通过`binder`驱动接收客户端的请求，处理，然后再返回给客户端的。

# ServiceManager

上篇文章我们是以`ServiceManager`作为服务端分析的，本篇文章我们还是围绕着它来做分析，它也是一个比较特殊的服务端，我们正好可以顺便分析一下它是怎么成为`binder`驱动的`context_manager`的

## 进程启动

`ServiceManager`是在独立的进程中运行的，它是由`init`进程从`rc`文件中解析并启动的，路径为`frameworks/native/cmds/servicemanager/servicemanager.rc`

```
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    onrestart restart thermalservice
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

这个服务的入口函数位于`frameworks/native/cmds/servicemanager/main.cpp`的`main`函数中

```cpp
int main(int argc, char** argv) {
    //根据上面的rc文件，argc == 1, argv[0] == "/system/bin/servicemanager"
    if (argc > 2) {
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";
    }

    //此时，要使用的binder驱动为/dev/binder
    const char* driver = argc == 2 ? argv[1] : "/dev/binder";

    //初始化binder驱动
    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ps->setThreadPoolMaxThreadCount(0);
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);

    //实例化ServiceManager
    sp<ServiceManager> manager = new ServiceManager(std::make_unique<Access>());
    //将自身作为服务添加
    if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
        LOG(ERROR) << "Could not self register servicemanager";
    }

    //设置服务端Bbinder对象
    IPCThreadState::self()->setTheContextObject(manager);
    //设置成为binder驱动的context manager
    ps->becomeContextManager(nullptr, nullptr);

    //通过Looper epoll机制处理binder事务
    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);

    BinderCallback::setupTo(looper);
    ClientCallbackCallback::setupTo(looper, manager);

    while(true) {
        looper->pollAll(-1);
    }

    //正常走不到这里
    return EXIT_FAILURE;
}
```

### 初始化Binder

首先读取参数，按照之前的rc文件来看，这里的`driver`为`/dev/binder`，然后根据此`driver`初始化此进程的`ProcessState`单例，根据我们上一章的分析我们知道此时会执行`binder_open`和`binder_mmap`，接着对这个单例做一些配置

```cpp
sp<ProcessState> ProcessState::initWithDriver(const char* driver)
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        // Allow for initWithDriver to be called repeatedly with the same
        // driver.
        //如果已经被初始化过了，并且传入的driver参数和已初始化的驱动名一样，直接返回之前初始化的单例
        if (!strcmp(gProcess->getDriverName().c_str(), driver)) {
            return gProcess;
        }
        //否则异常退出
        LOG_ALWAYS_FATAL("ProcessState was already initialized.");
    }

    //判断指定的driver是否存在并可读
    if (access(driver, R_OK) == -1) {
        ALOGE("Binder driver %s is unavailable. Using /dev/binder instead.", driver);
        //回滚默认binder驱动
        driver = "/dev/binder";
    }

    gProcess = new ProcessState(driver);
    return gProcess;
}
```

#### access

文档：<https://man7.org/linux/man-pages/man2/access.2.html>

原型：`int access(const char *pathname, int mode);`

这个函数是用来检查调用进程是否可以对指定文件执行某种操作的，成功返回0，失败返回-1并设置error

`mode`参数可以为以下几个值：

- `F_OK`：文件存在

- `R_OK`：文件可读

- `W_OK`：文件可写

- `X_OK`：文件可执行

### 注册成为Binder驱动的context manager

接着调用了`ProcessState`的`becomeContextManager`函数注册成为`Binder`驱动的`context manager`

```cpp
bool ProcessState::becomeContextManager()
{
    AutoMutex _l(mLock);

    flat_binder_object obj {
        .flags = FLAT_BINDER_FLAG_TXN_SECURITY_CTX,
    };

    int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);

    // fallback to original method
    if (result != 0) {
        android_errorWriteLog(0x534e4554, "121035042");

        int unused = 0;
        result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR, &unused);
    }

    if (result == -1) {
        ALOGE("Binder ioctl to become context manager failed: %s\n", strerror(errno));
    }

    return result == 0;
}
```

这里通过`binder_ioctl`，以`BINDER_SET_CONTEXT_MGR_EXT`为命令码请求`binder`驱动

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ...
    switch (cmd) {
    ...
    case BINDER_SET_CONTEXT_MGR_EXT: {
        struct flat_binder_object fbo;

        if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
            ret = -EINVAL;
            goto err;
        }
        ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
        if (ret)
            goto err;
        break;
    }
    ...
```

```cpp
static int binder_ioctl_set_ctx_mgr(struct file *filp,
                    struct flat_binder_object *fbo)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    struct binder_context *context = proc->context;
    struct binder_node *new_node;
    kuid_t curr_euid = current_euid();

    mutex_lock(&context->context_mgr_node_lock);
    //binder的context manager只能设置一次
    if (context->binder_context_mgr_node) {
        pr_err("BINDER_SET_CONTEXT_MGR already set\n");
        ret = -EBUSY;
        goto out;
    }
    //判断调用进程是否有权限设置context manager
    ret = security_binder_set_context_mgr(proc->tsk);
    if (ret < 0)
        goto out;
    //context->binder_context_mgr_uid != -1
    if (uid_valid(context->binder_context_mgr_uid)) {
        if (!uid_eq(context->binder_context_mgr_uid, curr_euid)) {
            pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
                   from_kuid(&init_user_ns, curr_euid),
                   from_kuid(&init_user_ns,
                     context->binder_context_mgr_uid));
            ret = -EPERM;
            goto out;
        }
    } else {
        //设置Binder驱动context manager所在进程的用户ID
        context->binder_context_mgr_uid = curr_euid;
    }
    //新建binder节点
    new_node = binder_new_node(proc, fbo);
    if (!new_node) {
        ret = -ENOMEM;
        goto out;
    }
    binder_node_lock(new_node);
    new_node->local_weak_refs++;
    new_node->local_strong_refs++;
    new_node->has_strong_ref = 1;
    new_node->has_weak_ref = 1;
    //设置binder驱动context manager节点
    context->binder_context_mgr_node = new_node;
    binder_node_unlock(new_node);
    binder_put_node(new_node);
out:
    mutex_unlock(&context->context_mgr_node_lock);
    return ret;
}
```

这里的过程也很简单，首先检查之前是否设置过`context manager`，然后做权限校验，通过后通过`binder_new_node`创建出一个新的`binder`节点，并将它作为`context manager`节点

### Looper循环处理Binder事务

这里的`Looper`和我们平常应用开发所说的`Looper`是一个东西，本篇就不做过多详解了，只需要知道，可以通过`Looper::addFd`函数监听文件描述符，通过`Looper::pollAll`或`Looper::pollOnce`函数接收消息，消息抵达后会回调`LooperCallback::handleEvent`函数

了解了这些后我们来看一下`BinderCallback`这个类

```cpp
class BinderCallback : public LooperCallback {
public:
    static sp<BinderCallback> setupTo(const sp<Looper>& looper) {
        sp<BinderCallback> cb = new BinderCallback;

        int binder_fd = -1;
        //向binder驱动发送BC_ENTER_LOOPER事务请求，并获得binder设备的文件描述符
        IPCThreadState::self()->setupPolling(&binder_fd);
        LOG_ALWAYS_FATAL_IF(binder_fd < 0, "Failed to setupPolling: %d", binder_fd);

        // Flush after setupPolling(), to make sure the binder driver
        // knows about this thread handling commands.
        IPCThreadState::self()->flushCommands();

        //监听binder文件描述符
        int ret = looper->addFd(binder_fd,
                                Looper::POLL_CALLBACK,
                                Looper::EVENT_INPUT,
                                cb,
                                nullptr /*data*/);
        LOG_ALWAYS_FATAL_IF(ret != 1, "Failed to add binder FD to Looper");

        return cb;
    }

    int handleEvent(int /* fd */, int /* events */, void* /* data */) override {
        //从binder驱动接收到消息并处理
        IPCThreadState::self()->handlePolledCommands();
        return 1;  // Continue receiving callbacks.
    }
};
```

在`servicemanager`进程启动的过程中调用了`BinderCallback::setupTo`函数，这个函数首先想`binder`驱动发起了一个`BC_ENTER_LOOPER`事务请求，获得`binder`设备的文件描述符，然后调用`Looper::addFd`函数监听`binder`设备文件描述符，这样当`binder`驱动发来消息后，就可以通过`Looper::handleEvent`函数接收并处理了

```cpp
status_t IPCThreadState::setupPolling(int* fd)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    //设置binder请求码
    mOut.writeInt32(BC_ENTER_LOOPER);
    //检查写缓存是否有可写数据，有的话发送给binder驱动
    flushCommands();
    //赋值binder驱动的文件描述符
    *fd = mProcess->mDriverFD;
    pthread_mutex_lock(&mProcess->mThreadCountLock);
    mProcess->mCurrentThreads++;
    pthread_mutex_unlock(&mProcess->mThreadCountLock);
    return 0;
}
```

## Binder事务处理

`BinderCallback`类重写了`handleEvent`函数，里面调用了`IPCThreadState::handlePolledCommands`函数来接收处理`binder`事务

```cpp
status_t IPCThreadState::handlePolledCommands()
{
    status_t result;

    //当读缓存中数据未消费完时，持续循环
    do {
        result = getAndExecuteCommand();
    } while (mIn.dataPosition() < mIn.dataSize());

    //当我们清空执行完所有的命令后，最后处理BR_DECREFS和BR_RELEASE
    processPendingDerefs();
    flushCommands();
    return result;
}
```

### 读取并处理响应

这个函数的重点在`getAndExecuteCommand`，首先无论如何从binder驱动那里读取并处理一次响应，如果处理完后发现读缓存中还有数据尚未消费完，继续循环这个处理过程（理论来说此时不会再从`binder`驱动那里读写数据，只会处理剩余读缓存）

```cpp
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    //从binder驱动中读写数据（理论来说此时写缓存dataSize为0，也就是只读数据）
    result = talkWithDriver(/* true */);
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        //读取BR响应码
        cmd = mIn.readInt32();
        ...
        result = executeCommand(cmd);
        ...
    }

    return result;
}
```

### 处理响应

这里有很多线程等其他操作，我们不需要关心，我在这里把他们简化掉了，剩余的代码很清晰，首先从binder驱动中读取数据，然后从数据中读取出`BR`响应码，接着调用`executeCommand`函数继续往下处理

```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    ...
    case BR_TRANSACTION_SEC_CTX:
    case BR_TRANSACTION:
        {
            binder_transaction_data_secctx tr_secctx;
            binder_transaction_data& tr = tr_secctx.transaction_data;

            if (cmd == (int) BR_TRANSACTION_SEC_CTX) {
                result = mIn.read(&tr_secctx, sizeof(tr_secctx));
            } else {
                result = mIn.read(&tr, sizeof(tr));
                tr_secctx.secctx = 0;
            }

            ALOG_ASSERT(result == NO_ERROR,
                "Not enough command data for brTRANSACTION");
            if (result != NO_ERROR) break;

            //读取数据到缓冲区
            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            ...

            Parcel reply;
            status_t error;
            //对于ServiceManager的binder节点来说，是没有ptr的
            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                //对于其他binder服务端来说，tr.cookie为本地BBinder对象指针
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                //对于ServiceManager来说，使用the_context_object这个BBinder对象
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }

            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);
                //非TF_ONE_WAY模式下需要Reply
                sendReply(reply, 0);
            } else {
                ... //TF_ONE_WAY模式下不需要Reply，这里只打了些日志
            }
            ...
        }
        break;
        ...
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }

    return result;
}
```

我们重点分析这个函数在`BR_TRANSACTION`下的case，其余的都删减掉

首先，这个函数从读缓存中读取了`binder_transaction_data`，我们知道这个结构体记录了实际数据的地址、大小等信息，然后实例化了一个`Parcel`对象作为缓冲区，从`binder_transaction_data`中将实际数据读取出来

接着找到本地`BBinder`对象，对于`ServiceManager`来说就是之前在`main`函数中`setTheContextObject`的`ServiceManager`对象，而对于其他`binder`服务端来说，则是通过`tr.cookie`获取，然后调用`BBinder`的`transact`函数

```cpp
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    //确保从头开始读取数据
    data.setDataPosition(0);

    if (reply != nullptr && (flags & FLAG_CLEAR_BUF)) {
        //标记这个Parcel在释放时需要将内存中数据用0覆盖（涉及安全）
        reply->markSensitive();
    }

    status_t err = NO_ERROR;
    //这里的code是由binder客户端请求传递过来的
    //是客户端与服务端的一个约定
    //它标识了客户端像服务端发起的是哪种请求
    switch (code) {
        ...
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }

    // In case this is being transacted on in the same process.
    if (reply != nullptr) {
        //设置数据指针偏移为0，这样后续读取数据便会从头开始
        reply->setDataPosition(0);
        if (reply->dataSize() > LOG_REPLIES_OVER_SIZE) {
            ALOGW("Large reply transaction of %zu bytes, interface descriptor %s, code %d",
                  reply->dataSize(), String8(getInterfaceDescriptor()).c_str(), code);
        }
    }

    return err;
}
```

### onTransact

这个函数主要调用了`onTransact`函数，它是一个虚函数，可以被子类重写。我们观察`ServiceManager`这个类，它继承了`BnServiceManager`，在`BnServiceManager`中重写了这个`onTransact`函数，它们的继承关系如下：

`ServiceManager` -> `BnServiceManager` -> `BnInterface<IServiceManager>` -> `IServiceManager` & `BBinder`

这里的`BnServiceManager`是通过`AIDL`工具生成出来的（`AIDL`既可以生成`Java`代码，也可以生成`C++`代码），我们找到一份生成后的代码

```cpp
::android::status_t BnServiceManager::onTransact(uint32_t _aidl_code, const ::android::Parcel& _aidl_data, ::android::Parcel* _aidl_reply, uint32_t _aidl_flags) {
    ::android::status_t _aidl_ret_status = ::android::OK;
    switch (_aidl_code) {
        case BnServiceManager::TRANSACTION_getService: {
            //参数name
            ::std::string in_name;
            ::android::sp<::android::IBinder> _aidl_return;
            //类型检查
            if (!(_aidl_data.checkInterface(this))) {
                _aidl_ret_status = ::android::BAD_TYPE;
                break;
            }
            //读取参数name
            _aidl_ret_status = _aidl_data.readUtf8FromUtf16(&in_name);
            if (((_aidl_ret_status) != (::android::OK))) {
                break;
            }
            //确认数据已读完
            if (auto st = _aidl_data.enforceNoDataAvail();
            !st.isOk()) {
                _aidl_ret_status = st.writeToParcel(_aidl_reply);
                break;
            }
            //执行真正的getService函数
            ::android::binder::Status _aidl_status(getService(in_name, &_aidl_return));
            //将状态值写入reply
            _aidl_ret_status = _aidl_status.writeToParcel(_aidl_reply);
            if (((_aidl_ret_status) != (::android::OK))) {
                break;
            }
            if (!_aidl_status.isOk()) {
                break;
            }
            //将返回值写入reply
            _aidl_ret_status = _aidl_reply->writeStrongBinder(_aidl_return);
            if (((_aidl_ret_status) != (::android::OK))) {
                break;
            }
        }
        break;
        ...
    }
    if (_aidl_ret_status == ::android::UNEXPECTED_NULL) {
        _aidl_ret_status = ::android::binder::Status::fromExceptionCode(::android::binder::Status::EX_NULL_POINTER).writeOverParcel(_aidl_reply);
    }
    return _aidl_ret_status;
}
```

生成出的代码格式比较丑，不易阅读，我把它格式化了一下，提取出我们需要的部分。这个函数主要流程就是先从`data`中读取所需要的参数，然后根据参数执行相对应的函数，然后将状态值写入`reply`，最后再将返回值写入`reply`。这里我们将上一章节`AIDL`生成出的`java`文件那部分拿过来做对比，我们可以发现，这里`Parcel`的写入和那里`Parcel`的读取顺序是严格一一对应的

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
        //先读取状态值
        _reply.readException();
        //再读取返回值
        _result = _reply.readStrongBinder();
    }
    finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

### 实际功能实现

然后我们来看真正功能实现的地方：`getService`函数，根据之前所说的继承关系，`ServiceManager`继承自`IServiceManager`，实现了其中的纯虚函数，其中就包括了`getService`

```cpp
Status ServiceManager::getService(const std::string& name, sp<IBinder>* outBinder) {
    *outBinder = tryGetService(name, true);
    // returns ok regardless of result for legacy reasons
    return Status::ok();
}
```

```cpp
sp<IBinder> ServiceManager::tryGetService(const std::string& name, bool startIfNotFound) {
    auto ctx = mAccess->getCallingContext();

    //返回值
    sp<IBinder> out;
    Service* service = nullptr;
    //从map中寻找相应的服务
    if (auto it = mNameToService.find(name); it != mNameToService.end()) {
        service = &(it->second);

        if (!service->allowIsolated) {
            uid_t appid = multiuser_get_app_id(ctx.uid);
            bool isIsolated = appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END;

            if (isIsolated) {
                return nullptr;
            }
        }
        //返回值指向对应service的binder对象
        out = service->binder;
    }

    if (!mAccess->canFind(ctx, name)) {
        return nullptr;
    }

    if (!out && startIfNotFound) {
        tryStartService(name);
    }

    if (out) {
        // Setting this guarantee each time we hand out a binder ensures that the client-checking
        // loop knows about the event even if the client immediately drops the service
        service->guaranteeClient = true;
    }

    return out;
}
```

这里面的实现我们就没必要细看了，只需要注意它返回了相应`service`的`binder`对象，根据上面的代码来看，会将其写入到`reply`中

### Reply

实际的功能处理完成后，我们回到`IPCThreadState::executeCommand`中来。对于非`TF_ONE_WAY`模式，我们要将`reply`发送给请求方客户端

```cpp
status_t IPCThreadState::sendReply(const Parcel& reply, uint32_t flags)
{
    status_t err;
    status_t statusBuffer;
    //将binder reply请求打包好写入写缓冲区
    err = writeTransactionData(BC_REPLY, flags, -1, 0, reply, &statusBuffer);
    if (err < NO_ERROR) return err;

    return waitForResponse(nullptr, nullptr);
}
```

`writeTransactionData`在上一章中已经分析过了，这里就不多做描述了，`waitForResponse`我们上一章也分析过了，根据我们在上一章所描述的非`TF_ONE_WAY`的通信过程，在向`binder`驱动发送`BC_REPLY`请求后我们会收到`BR_TRANSACTION_COMPLETE`响应，根据我们传入`waitForResponse`的两个参数值，会直接跳出函数中的循环，结束此次`binder`通信

![non_oneway](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Framework%E5%B1%82%E7%9A%84Binder%EF%BC%88%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%AF%87%EF%BC%89_non_oneway.png)

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

        switch (cmd) {
        ...
        case BR_TRANSACTION_COMPLETE:
            //参数为两个nullptr，直接跳转到finish结束
            if (!reply && !acquireResult) goto finish;
            break;
        ...
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

至此，`binder`服务端的一次消息处理到这就结束了，`Looper`会持续监听着`binder`驱动`fd`，等待下一条`binder`消息的到来

# 结束

经过这么多篇文章的分析，整个`Binder`架构的大致通信原理、过程，我们应该都了解的差不多了，至于一些边边角角的细节，以后有机会的话我会慢慢再补充