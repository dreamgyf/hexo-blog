---
title: Android源码分析 - SystemServer（上）
date: 2022-01-17 20:42:00
tags: 
- Android源码
- SystemServer
categories: 
- [Android, 源码分析]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

上一篇文章[Android源码分析 - Zygote进程](https://juejin.cn/post/7051507161955827720 "Android源码分析 - Zygote进程")，我们分析了Android `Zygote`进程的启动和之后是如何接收消息创建App进程的

在上一章中，我们说了，`Zygote`的一大作用就是启动`SystemServer`，那么`SystemServer`是怎么启动的呢？启动后又做了些什么呢？我们分上下两篇来分析，本篇介绍`SystemServer`是如何启动的

# 介绍

`SystemServer`主要是用来创建系统服务的，譬如我们熟知的`ActivityManagerService`，`PackageManagerService`都是由它创建的

# 启动SystemServer

我们从上一篇文章的`ZygoteInit`开始，`ZygoteInit`类的源码路径为`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

```java
public static void main(String argv[]) {
    ...
    boolean startSystemServer = false;
    ...
    for (int i = 1; i < argv.length; i++) {
        //参数中有start-system-server
        if ("start-system-server".equals(argv[i])) {
            startSystemServer = true;
        }
        ...
    }
    ...
    //启动SystemServer
    if (startSystemServer) {
        Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

        //子进程中才会满足r != null
        if (r != null) {
            //此时执行这个Runnable
            r.run();
            return;
        }
    }
}
```

之前在c++代码中JNI调用Java函数的时候，带了参数`start-system-server`，在这里就会通过这个参数判断是否启动`SystemServer`，接下来调用`forkSystemServer`方法

```java
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    //设置Linux capabilities
    long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_IPC_LOCK,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_PTRACE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG,
            OsConstants.CAP_WAKE_ALARM,
            OsConstants.CAP_BLOCK_SUSPEND
    );
    //移除一些当前线程都不可用的特权
    StructCapUserHeader header = new StructCapUserHeader(
            OsConstants._LINUX_CAPABILITY_VERSION_3, 0);
    StructCapUserData[] data;
    try {
        data = Os.capget(header);
    } catch (ErrnoException ex) {
        throw new RuntimeException("Failed to capget()", ex);
    }
    //data[0].effective为当前线程所可用的特权，data[1].effective貌似为0
    capabilities &= ((long) data[0].effective) | (((long) data[1].effective) << 32);

    //设置fork参数
    String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                    + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",
    };
    ZygoteArguments parsedArgs = null;

    int pid;

    try {
        //解析设置的参数
        parsedArgs = new ZygoteArguments(args);
        ... //进一步设置参数
        pid = Zygote.forkSystemServer(
                parsedArgs.mUid, parsedArgs.mGid,
                parsedArgs.mGids,
                parsedArgs.mRuntimeFlags,
                null,
                parsedArgs.mPermittedCapabilities,
                parsedArgs.mEffectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    //SystemServer子进程
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        
        //关闭zygote server socket
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

## Capabilities

这里需要先了解一下Linux Capabilities机制：[Linux Capabilities机制](https://juejin.cn/post/7052933216948191269)

这里先定义了`SystemServer`进程的`Permitted`和`Effective`能力集合

```java
private static long posixCapabilitiesAsBits(int... capabilities) {
    long result = 0;
    for (int capability : capabilities) {
        //非法capability，直接抛出异常
        if ((capability < 0) || (capability > OsConstants.CAP_LAST_CAP)) {
            throw new IllegalArgumentException(String.valueOf(capability));
        }
        //为或操作，构建capabilities集合
        result |= (1L << capability);
    }
    return result;
}
```

检查一下有无非法`capability`，然后做位或运算，构建出一个`capabilities`集合

然后通过`Os.capget`方法获取当前线程的`capabilities`集合，上一篇文章中我们已经分析过了Os的作用，最终通过`Linux_capget`JNI函数调用Linux`capget`函数，通过返回回来的值，剔除一些当前线程不支持的特权

## Fork

接着设置一些fork参数，通过`ZygoteArguments`去解析它

然后调用`Zygote.forkSystemServer`方法，这个和上一章里说的fork App的过程差不多

```java
static int forkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    //停止其他线程
    ZygoteHooks.preFork();

    int pid = nativeForkSystemServer(
            uid, gid, gids, runtimeFlags, rlimits,
            permittedCapabilities, effectiveCapabilities);

    //设置默认线程优先级
    Thread.currentThread().setPriority(Thread.NORM_PRIORITY);
    //恢复其他线程
    ZygoteHooks.postForkCommon();
    return pid;
}
```

先把子线程都停止掉，fork完后再恢复，调用native函数`nativeForkSystemServer`，路径为`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`

```c++
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits, jlong permitted_capabilities,
        jlong effective_capabilities) {
  ...
  pid_t pid = ForkCommon(env, true,
                         fds_to_close,
                         fds_to_ignore,
                         true);
  if (pid == 0) {
      // System server prcoess does not need data isolation so no need to
      // know pkg_data_info_list.
      SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits,
                       permitted_capabilities, effective_capabilities,
                       MOUNT_EXTERNAL_DEFAULT, nullptr, nullptr, true,
                       false, nullptr, nullptr, /* is_top_app= */ false,
                       /* pkg_data_info_list */ nullptr,
                       /* whitelisted_data_info_list */ nullptr, false, false);
  } else if (pid > 0) {
      ...
      gSystemServerPid = pid;
      //检查SystemServer进程状态
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          //如果SystemServer进程死亡，重启整个Zygote
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
      }

      //如果是低内存设备，限制SystemServer进程使用内存大小
      if (UsePerAppMemcg()) {
          if (!SetTaskProfiles(pid, std::vector<std::string>{"SystemMemoryProcess"})) {
              ALOGE("couldn't add process %d into system memcg group", pid);
          }
      }
  }
  return pid;
}
```

### ForkCommon

我们先看`ForkCommon`函数

```java
static pid_t ForkCommon(JNIEnv* env, bool is_system_server,
                        const std::vector<int>& fds_to_close,
                        const std::vector<int>& fds_to_ignore,
                        bool is_priority_fork) {
  //设置子进程信号处理器
  SetSignalHandlers();

  //C++中的一种可调用对象，ZygoteFailure函数接收4个参数，前三个参数都已提供，最后一个参数占位等待调用方填入
  auto fail_fn = std::bind(ZygoteFailure, env, is_system_server ? "system_server" : "zygote",
                           nullptr, _1);

  //在fork期间阻塞住SIGCHLD信号，避免在SIGCHLD信号处理函数中打印log，导致后面关闭的日志fd重新被打开
  BlockSignal(SIGCHLD, fail_fn);

  //关闭所有日志相关fd
  __android_log_close();
  AStatsSocket_close();

  //SystemServer是Zygote进程起来后第一个fork的出来进程，创建打开的文件描述符表
  if (gOpenFdTable == nullptr) {
    gOpenFdTable = FileDescriptorTable::Create(fds_to_ignore, fail_fn);
  } else {
    gOpenFdTable->Restat(fds_to_ignore, fail_fn);
  }

  android_fdsan_error_level fdsan_error_level = android_fdsan_get_error_level();

  //立即清除任何未使用的内存
  mallopt(M_PURGE, 0);

  pid_t pid = fork();

  if (pid == 0) {
    //fork SystemServer时，此参数为true
    if (is_priority_fork) {
      //设置最高进程优先级
      setpriority(PRIO_PROCESS, 0, PROCESS_PRIORITY_MAX);
    } else {
      setpriority(PRIO_PROCESS, 0, PROCESS_PRIORITY_MIN);
    }

    // The child process.
    PreApplicationInit();

    //清除所有需要立即关闭的fd
    DetachDescriptors(env, fds_to_close, fail_fn);

    //USAP机制我们现在不关注
    ClearUsapTable();

    //重新打开剩余打开的文件描述符，避免文件描述符通过fork在SystemServer和Zygote之间共享
    gOpenFdTable->ReopenOrDetach(fail_fn);

    //Sanitizer机制，用来检测程序异常
    android_fdsan_set_error_level(fdsan_error_level);

    // Reset the fd to the unsolicited zygote socket
    gSystemServerSocketFd = -1;
  } else {
    ALOGD("Forked child process %d", pid);
  }

  //取消之前阻塞的SIGCHLD信号
  UnblockSignal(SIGCHLD, fail_fn);

  return pid;
}
```

#### 处理子进程信号

先设置子进程信号处理器

```java
static void SetSignalHandlers() {
    struct sigaction sig_chld = {.sa_flags = SA_SIGINFO, .sa_sigaction = SigChldHandler};

    if (sigaction(SIGCHLD, &sig_chld, nullptr) < 0) {
        ALOGW("Error setting SIGCHLD handler: %s", strerror(errno));
    }

  struct sigaction sig_hup = {};
  sig_hup.sa_handler = SIG_IGN;
  if (sigaction(SIGHUP, &sig_hup, nullptr) < 0) {
    ALOGW("Error setting SIGHUP handler: %s", strerror(errno));
  }
}
```

关于信号的处理，我们在[Android源码分析 - init进程](https://juejin.cn/post/7049277873877680142#heading-31)中已经了解过一次，`SA_SIGINFO`这个flag代表调用信号处理函数`sa_sigaction`的时候，会将信号的信息通过参数`siginfo_t`传入

`SIGHUP`表示终端断开信号，`SIG_IGN`表示忽略信号，即忽略终端断开信号

我们看一下`Zygote`是怎么处理其子进程信号的

```java
static void SigChldHandler(int /*signal_number*/, siginfo_t* info, void* /*ucontext*/) {
    pid_t pid;
    int status;
    ...
    int saved_errno = errno;

    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        //通知SystemServer，Zygote收到了一个SIGCHLD信号
        sendSigChildStatus(pid, info->si_uid, status);
        ... //打印子进程状态日志
        //如果崩溃的进程是SystemServer，整个Zygote都会退出，再通过init进程重启
        if (pid == gSystemServerPid) {
            async_safe_format_log(ANDROID_LOG_ERROR, LOG_TAG,
                                  "Exit zygote because system server (pid %d) has terminated", pid);
            kill(getpid(), SIGKILL);
        }
    }
    ...
    errno = saved_errno;
}
```

如果检测到有子进程退出，通知`SystemServer`，如果这个进程是`SystemServer`进程，杀掉`Zygote`进程重启

#### ZygoteFailure

这里先需要理解一下[C++11 中的std::function和std::bind](https://www.jianshu.com/p/f191e88dcc80)

简单来说，`std::bind`返回了一个`std::function`对象，它是一个可调用对象，实际调用的就是传入的第一个参数：`ZygoteFailure`函数，这个函数接受4个参数，前三个参数都在`std::bind`时提供好了，第四个参数以`_1`占位符替代（`std::placeholders::_1`）

实际上调用`fail_fn(msg)`就相当于调用函数`ZygoteFailure(env, "system_server", nullptr, msg)`

```java
static void ZygoteFailure(JNIEnv* env,
                          const char* process_name,
                          jstring managed_process_name,
                          const std::string& msg) {
  std::unique_ptr<ScopedUtfChars> scoped_managed_process_name_ptr = nullptr;
  if (managed_process_name != nullptr) {
    scoped_managed_process_name_ptr.reset(new ScopedUtfChars(env, managed_process_name));
    if (scoped_managed_process_name_ptr->c_str() != nullptr) {
      process_name = scoped_managed_process_name_ptr->c_str();
    }
  }

  const std::string& error_msg =
      (process_name == nullptr) ? msg : StringPrintf("(%s) %s", process_name, msg.c_str());
  //抛出异常
  env->FatalError(error_msg.c_str());
  __builtin_unreachable();
}
```

当发生错误后，最终向Java层抛出了一个异常

#### BlockSignal & UnblockSignal

在`fork`期间需要阻塞住`SIGCHLD`信号，避免在`SIGCHLD`信号处理函数中打印log，导致后面关闭的日志fd重新被打开

```java
static void BlockSignal(int signum, fail_fn_t fail_fn) {
  sigset_t sigs;
  sigemptyset(&sigs);
  sigaddset(&sigs, signum);

  if (sigprocmask(SIG_BLOCK, &sigs, nullptr) == -1) {
    fail_fn(CREATE_ERROR("Failed to block signal %s: %s", strsignal(signum), strerror(errno)));
  }
}
```

等`fork`结束，取消阻塞`SIGCHLD`信号

```java
static void UnblockSignal(int signum, fail_fn_t fail_fn) {
  sigset_t sigs;
  sigemptyset(&sigs);
  sigaddset(&sigs, signum);

  if (sigprocmask(SIG_UNBLOCK, &sigs, nullptr) == -1) {
    fail_fn(CREATE_ERROR("Failed to un-block signal %s: %s", strsignal(signum), strerror(errno)));
  }
}
```

信号集函数我们之前已经在[Android源码分析 - init进程](https://juejin.cn/post/7049277873877680142#heading-32)中介绍过了，很简单，就是将`SIGCHLD`信号添加到屏蔽集中，`fork`完后再将这个信号从屏蔽集中移除

### SpecializeCommon

至此，fork操作结束，我们看一下在SystemServer进程中执行的`SpecializeCommon`函数

```java
static void SpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray gids,
                             jint runtime_flags, jobjectArray rlimits,
                             jlong permitted_capabilities, jlong effective_capabilities,
                             jint mount_external, jstring managed_se_info,
                             jstring managed_nice_name, bool is_system_server,
                             bool is_child_zygote, jstring managed_instruction_set,
                             jstring managed_app_data_dir, bool is_top_app,
                             jobjectArray pkg_data_info_list,
                             jobjectArray whitelisted_data_info_list,
                             bool mount_data_dirs, bool mount_storage_dirs) {
  //process_name = "system_server"
  const char* process_name = is_system_server ? "system_server" : "zygote";
  auto fail_fn = std::bind(ZygoteFailure, env, process_name, managed_nice_name, _1);
  auto extract_fn = std::bind(ExtractJString, env, process_name, managed_nice_name, _1);

  //均为nullptr
  auto se_info = extract_fn(managed_se_info);
  auto nice_name = extract_fn(managed_nice_name);
  auto instruction_set = extract_fn(managed_instruction_set);
  auto app_data_dir = extract_fn(managed_app_data_dir);

  //当UID发生改变时（root->非root）保留capabilities
  if (uid != 0) {
    EnableKeepCapabilities(fail_fn);
  }
  //设置Inheritable集合
  SetInheritable(permitted_capabilities, fail_fn);
  //从Bounding集合中移除调用线程相关能力
  DropCapabilitiesBoundingSet(fail_fn);
  ...
  //创建私有挂载命名空间，挂载虚拟存储
  MountEmulatedStorage(uid, mount_external, need_pre_initialize_native_bridge, fail_fn);

  ...
  //设置GroupId
  SetGids(env, gids, is_child_zygote, fail_fn);
  //设置资源Limit
  SetRLimits(env, rlimits, fail_fn);
  ...
  //设置gid及访问权限
  if (setresgid(gid, gid, gid) == -1) {
    fail_fn(CREATE_ERROR("setresgid(%d) failed: %s", gid, strerror(errno)));
  }
  //capabilities集合中仍然存在CAP_SYS_ADMIN，需要过滤系统调用
  SetUpSeccompFilter(uid, is_child_zygote);
  //设置调度策略
  SetSchedulerPolicy(fail_fn, is_top_app);
  //设置uid及访问权限
  if (setresuid(uid, uid, uid) == -1) {
    fail_fn(CREATE_ERROR("setresuid(%d) failed: %s", uid, strerror(errno)));
  }
  ...
  //设置Capabilities
  SetCapabilities(permitted_capabilities, effective_capabilities, permitted_capabilities, fail_fn);
  //关闭所有日志相关fd
  __android_log_close();
  AStatsSocket_close();
  ...
  //设置线程名
  if (nice_name.has_value()) {
    SetThreadName(nice_name.value());
  } else if (is_system_server) { //nice_name为nullptr, 进入此分支
    SetThreadName("system_server");
  }

  //取消掉之前设置的SIGCHID信号处理函数
  UnsetChldSignalHandler();

  if (is_system_server) {
    //调用ZygoteHooks.postForkSystemServer(runtime_flags);
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkSystemServerHooks, runtime_flags);
    if (env->ExceptionCheck()) {
      fail_fn("Error calling post fork system server hooks.");
    }
    ...
  }
  ...
  //调用ZygoteHooks.postForkChild(runtime_flags, true, false, null);
  env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, runtime_flags,
                            is_system_server, is_child_zygote, managed_instruction_set);

  //设置默认进程优先级
  setpriority(PRIO_PROCESS, 0, PROCESS_PRIORITY_DEFAULT);

  if (env->ExceptionCheck()) {
    fail_fn("Error calling post fork hooks.");
  }
}
```

这里做了很多工作，有`Capabilities`相关，`selinux`相关，权限相关等等，有点太多了，我标了注释，就不再一一分析了

接下来回到`nativeForkSystemServer`中，在`Zygote`进程中继续执行

```c++
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits, jlong permitted_capabilities,
        jlong effective_capabilities) {
  ...
  if (pid == 0) {
      ...
  } else if (pid > 0) {
      ...
      gSystemServerPid = pid;
      //检查SystemServer进程状态
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          //如果SystemServer进程死亡，重启整个Zygote
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
      }

      //如果是低内存设备，限制SystemServer进程使用内存大小
      if (UsePerAppMemcg()) {
          if (!SetTaskProfiles(pid, std::vector<std::string>{"SystemMemoryProcess"})) {
              ALOGE("couldn't add process %d into system memcg group", pid);
          }
      }
  }
  return pid;
}
```

通过Linux函数`waitpid`检查`SystemServer`进程状态，这个函数和之前在[Android源码分析 - init进程](https://juejin.cn/post/7049277873877680142#heading-38)中提过的`waitid`函数类似，`WNOHANG`表示非阻塞等待

如果`SystemServer`进程死亡，重启整个`Zygote`

```java
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    ...
    //SystemServer子进程
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        
        //关闭zygote server socket
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

### cgroups

如果是小内存设备，使用Linux的`cgroups`机制，限制`SystemServer`进程使用内存大小

```c++
bool UsePerAppMemcg() {
    bool low_ram_device = GetBoolProperty("ro.config.low_ram", false);
    return GetBoolProperty("ro.config.per_app_memcg", low_ram_device);
}
```

关于Linux的`cgroups`机制，可以查看这篇文档：[cgroups(7) — Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html)

关于Android的`Cgroups`机制，可以看这篇官方文档：[Cgroup 抽象层](https://source.android.google.cn/devices/tech/perf/cgroups?hl=zh-cn)

## 运行

### 初始化

```java
private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
    //将umask设置为0077，这样新的文件和目录将默认为仅属于所有者的权限
    Os.umask(S_IRWXG | S_IRWXO);
    //设置进程名
    if (parsedArgs.mNiceName != null) {
        Process.setArgV0(parsedArgs.mNiceName);
    }

    //对classpath中的apk，分别进行dex优化操作，由installd真正执行
    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        performSystemServerDexOpt(systemServerClasspath);
        ...
    }

    if (parsedArgs.mInvokeWith != null) {
        ...
    } else {
        //SystemServer进入这个分支
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = createPathClassLoader(systemServerClasspath, parsedArgs.mTargetSdkVersion);

            Thread.currentThread().setContextClassLoader(cl);
        }

        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs, cl);
    }
}
```

处理一些初始化操作，然后调用`ZygoteInit.zygoteInit`方法

```java
public static final Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    ...
    //通用初始化
    RuntimeInit.commonInit();
    //开启binder线程池
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}
```

`RuntimeInit`的路径为`frameworks/base/core/java/com/android/internal/os/RuntimeInit.java`，先执行通用初始化

```java
protected static final void commonInit() {
    //设置默认线程异常处理器
    LoggingHandler loggingHandler = new LoggingHandler();
    RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));

    //设置时区
    RuntimeHooks.setTimeZoneIdSupplier(() -> SystemProperties.get("persist.sys.timezone"));

    //重置Log配置
    LogManager.getLogManager().reset();
    new AndroidConfig();

    //设置网络UA信息
    String userAgent = getDefaultUserAgent();
    System.setProperty("http.agent", userAgent);

    //初始化网络流量统计
    NetworkManagementSocketTagger.install();
    ...
    initialized = true;
}
```

接着执行`RuntimeInit.applicationInit`

```java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    //如果应用程序调用System.exit()，则立即终止该进程，不运行任何hook函数
    nativeSetExitWithoutCleanup(true);
    //设置虚拟机参数
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
    //解析参数
    final Arguments args = new Arguments(argv);
    ...
    //查找startClass中的main方法
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

### 参数解析

我们看一下它是怎么解析参数的

```java
Arguments(String args[]) throws IllegalArgumentException {
    parseArgs(args);
}

private void parseArgs(String args[])
        throws IllegalArgumentException {
    int curArg = 0;
    for (; curArg < args.length; curArg++) {
        String arg = args[curArg];

        if (arg.equals("--")) {
            curArg++;
            break;
        } else if (!arg.startsWith("--")) {
            break;
        }
    }

    if (curArg == args.length) {
        throw new IllegalArgumentException("Missing classname argument to RuntimeInit!");
    }

    startClass = args[curArg++];
    startArgs = new String[args.length - curArg];
    System.arraycopy(args, curArg, startArgs, 0, startArgs.length);
}
```

循环读参数直到有一项参数为"--"或者不以"--"开头，然后以下一个参数作为`startClass`，用再下一个参数到args数组结尾生成一个新的数组作为`startArgs`，我们观察一下`forkSystemServer`方法中设置的args

```java
String args[] = {
                "--setuid=1000",
                "--setgid=1000",
                "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                        + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
                "--capabilities=" + capabilities + "," + capabilities,
                "--nice-name=system_server",
                "--runtime-args",
                "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
                "com.android.server.SystemServer",
        };
```

可以看出，`startClass`应该为`com.android.server.SystemServer`，`startArgs`数组为空

### 反射执行

接着调用`findStaticMain`方法

```java
protected static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }

    /*
        * This throw gets caught in ZygoteInit.main(), which responds
        * by invoking the exception's run() method. This arrangement
        * clears up all the stack frames that were required in setting
        * up the process.
        */
    return new MethodAndArgsCaller(m, argv);
}
```

这里使用了Java中的反射，找到了`SystemServer`中对应的`main`方法，并用其创建了一个`Runnable`对象`MethodAndArgsCaller`

```java
static class MethodAndArgsCaller implements Runnable {
    private final Method mMethod;
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            //执行SystemServer.main方法
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```

我们最后再回到`ZygoteInit`的`main`方法中

```java
public static void main(String argv[]) {
    ...
    //启动SystemServer
    if (startSystemServer) {
        Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

        //子进程中才会满足r != null
        if (r != null) {
            //此时执行这个Runnable
            r.run();
            return;
        }
    }
}
```

执行这个在子进程中返回出去的`Runnable`：`MethodAndArgsCaller`，反射调用`SystemServer.main`方法

# 结束

至此，`SystemServer`的启动我们就分析完了，下一篇我们将分析`SystemServer`启动后做了什么