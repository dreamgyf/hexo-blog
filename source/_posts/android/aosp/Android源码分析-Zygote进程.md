---
title: Android源码分析 - Zygote进程
date: 2022-01-10 17:30:00
tags: Android源码
categories: 
- [Android, 源码分析]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

上一篇文章[Android源码分析 - init进程](https://juejin.cn/post/7049277873877680142)，我们分析了Android第一个用户进程init进程的启动过程和之后的守护服务

init进程启动了很多服务，例如Zygote，ServiceManager，MediaServer，SurfaceFlinger等，我们平常写Android应用都是使用Java语言，这次我们就先从Java世界的半边天：Zygote进程 开始分析

# 介绍

Zygote意为受精卵，它有两大作用，一是启动SystemServer，二是孵化启动App

# 启动服务

我们已经知道了init进程会从`init.rc`文件中解析并启动服务，那zygote是在哪定义的呢，`init.rc`的头几行就有一个import：`import /system/etc/init/hw/init.${ro.zygote}.rc`

我们在init.rc同目录下就能找到几个对应的文件：`init.zygote32_64.rc` `init.zygote32.rc` `init.zygote64_32.rc` `init.zygote64.rc`，具体import哪个文件与具体设备硬件有关，现在64位手机这么普及了，我们就以`init.zygote64.rc`为目标分析

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

下面的子项我们暂时不用关心，先记住`app_process64`的启动参数`-Xzygote /system/bin --zygote --start-system-server`即可

Zygote启动的源文件为`frameworks/base/cmds/app_process/app_main.cpp`

```c++
int main(int argc, char* const argv[])
{   
    ...
    //创建了一个AppRuntime，继承自AndroidRuntime，重写了一些回调方法
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    //在启动服务时，传进来的参数是包含文件路径的
    //我们不需要这个参数，就减一下个数，移一下指针
    argc--;
    argv++;

    ...
    
    //处理参数，这里只添加了一个-Xzygote参数
    int i;
    for (i = 0; i < argc; i++) {
        ...
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        ...
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    //跳过参数/system/bin，这个参数目前没有被使用
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            //有--zygote参数
            zygote = true;
            //ZYGOTE_NICE_NAME在64位下为zygote64，32位下为zygote
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            //有-start-system-server参数
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        //这个分支不会进入Zygote模式
        ...
    } else {
        // We're in zygote mode.
        //新建Dalvik缓存目录
        maybeCreateDalvikCache();

        //添加启动参数
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        //Zygote模式下没有其他参数了
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        //设置程序名以及进程名
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        //执行AndroidRuntime::start方法
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

整体结构还是比较简单的，就是处理一下参数，进入zygote对应的分支，执行AndroidRuntime::start方法，第一个参数传的是ZygoteInit在Java中的类名，第二个参数传了一些选项（start-system-server和abi-list），第三个参数传了true，代表启动虚拟机的时候需要额外添加一些JVM参数

# AndroidRuntime::start

```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    static const String8 startSystemServer("start-system-server");
    // Whether this is the primary zygote, meaning the zygote which will fork system server.
    //64_32位兼容设备上会启动两个Zygote，一个叫zygote，一个叫zygote_secondary
    bool primary_zygote = false;

    //有start-system-server选项则代表是主Zygote
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
            primary_zygote = true;
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    //检查和配置一些环境变量
    ...

    /* start the virtual machine */
    //加载libart.so
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    
    JNIEnv* env;
    //启动JVM
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    //回调AppRuntime中重写的方法
    onVmCreated(env);

    /*
     * Register android functions.
     */
    //注册Android JNI函数
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    //创建一个Java层的String数组用来装参数
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    //第一个参数是类名com.android.internal.os.ZygoteInit
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    //剩下来参数分别是start-system-server和abi-list
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    //将Java类名中的"."替换成"/"，这是JNI中的类名规则
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        //获取ZygoteInit中的main方法，参数为String类型，返回值为void
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            //执行ZygoteInit的main方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

            //后面的代码除非JVM挂了，否则不会执行
            ...
        }
    }
    ...
}
```

首先判断选项中是否携带参数start-system-server，如有，则将它视为主Zygote，接着就开始启动JVM了

# 启动JVM

## JniInvocation

使用JniInvocation初始化Android ART虚拟机环境，它的路径是`libnativehelper/include_platform/nativehelper/JniInvocation.h`，我们来看一下它是怎么做的

我们首先看一下它的构造函数

```c++
/* libnativehelper/include_platform/nativehelper/JniInvocation.h */
class JniInvocation final {
  public:
    JniInvocation() {
        impl_ = JniInvocationCreate();
    }

    ~JniInvocation() {
        JniInvocationDestroy(impl_);
    }
  ...
}
```

调用`JniInvocationCreate`方法创建了一个JniInvocationImpl实例对象

```c++
JniInvocationImpl* JniInvocationCreate() {
    return new JniInvocationImpl();
}
```

接着调用`JniInvocation::Init`方法

```c++
bool Init(const char* library) {
    return JniInvocationInit(impl_, library) != 0;
}

int JniInvocationInit(JniInvocationImpl* instance, const char* library) {
    return instance->Init(library) ? 1 : 0;
}
```

可以看到，`JniInvocation`实际上是个代理类，内部实现是交给`JniInvocationImpl`的，路径为`libnativehelper/JniInvocation.cpp`

```c++
bool JniInvocationImpl::Init(const char* library) {
  ...
  //非debug一律为libart.so
  library = GetLibrary(library, buffer);
  //加载libart.so库
  handle_ = OpenLibrary(library);
  if (handle_ == NULL) {
    //如果是加载libart.so库失败，直接返回false
    if (strcmp(library, kLibraryFallback) == 0) {
      // Nothing else to try.
      ALOGE("Failed to dlopen %s: %s", library, GetError().c_str());
      return false;
    }
    ...
    //如果是加载其他库失败，尝试回退加载libart.so库
    library = kLibraryFallback;
    handle_ = OpenLibrary(library);
    if (handle_ == NULL) {
      ALOGE("Failed to dlopen %s: %s", library, GetError().c_str());
      return false;
    }
  }
  //从libart.so库获得三个JVM相关的函数地址
  if (!FindSymbol(reinterpret_cast<FUNC_POINTER*>(&JNI_GetDefaultJavaVMInitArgs_),
                  "JNI_GetDefaultJavaVMInitArgs")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<FUNC_POINTER*>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<FUNC_POINTER*>(&JNI_GetCreatedJavaVMs_),
                  "JNI_GetCreatedJavaVMs")) {
    return false;
  }
  return true;
}
```

### 加载libart.so库

我们先看`GetLibrary`方法

```c++
static const char* kLibraryFallback = "libart.so";

const char* JniInvocationImpl::GetLibrary(const char* library,
                                          char* buffer,
                                          bool (*is_debuggable)(),
                                          int (*get_library_system_property)(char* buffer)) {
#ifdef __ANDROID__
  const char* default_library;

  if (!is_debuggable()) {
    library = kLibraryFallback;
    default_library = kLibraryFallback;
  } else {
    ...
  }
#else
  ...
  const char* default_library = kLibraryFallback;
#endif
  if (library == NULL) {
    library = default_library;
  }

  return library;
}
```

可以看到，在debug模式或`library`参数为`NULL`的情况下都是直接返回的`libart.so`

而`OpenLibrary`方法是使用了`dlopen`函数，加载`libart.so`库

```c++
void* OpenLibrary(const char* filename) {
  ...
  const int kDlopenFlags = RTLD_NOW | RTLD_NODELETE;
  return dlopen(filename, kDlopenFlags);
  ...
}
```

#### dlopen

原型：`void *dlopen(const char *filename, int flags); `

文档：https://man7.org/linux/man-pages/man3/dlopen.3.html

这是一个Linux函数，用来加载一个动态链接库，当加载成功时，会返回一个句柄

上面的这两个参数，`RTLD_NOW`代表立即计算库的依赖性，`RTLD_NODELETE`代表不要再`dlclose`期间卸载库，这样当再次加载库的时候不会重新初始化对象的静态全局变量，使用这个flag是为了确保`libart.so`在关闭时不会被取消映射。因为即使在 `JNI_DeleteJavaVM` 调用之后，某些线程仍可能尚未完成退出，如果卸载该库，则可能导致段错误

### 从libart.so库中寻找函数地址

接着调用`FindSymbol`函数查找函数地址

```c++
#define FUNC_POINTER void*

bool JniInvocationImpl::FindSymbol(FUNC_POINTER* pointer, const char* symbol) {
  //获得函数地址
  *pointer = GetSymbol(handle_, symbol);
  //获取失败，卸载libart.so库
  if (*pointer == NULL) {
    ALOGE("Failed to find symbol %s: %s\n", symbol, GetError().c_str());
    CloseLibrary(handle_);
    handle_ = NULL;
    return false;
  }
  return true;
}

FUNC_POINTER GetSymbol(void* handle, const char* symbol) {
  ...
  return dlsym(handle, symbol);
  ...
}
```

#### dlsym

原型：`void *dlsym(void *restrict handle , const char *restrict symbol); `

文档：https://man7.org/linux/man-pages/man3/dlsym.3.html

也是一个Linux函数，用来从已加载的动态链接库中获取一个函数的地址

传入的第一个参数为之前加载库时返回的句柄，第二个参数为函数名

### 总结

回顾一下全局，`JniInvocationImpl::Init`的作用是，加载`libart.so`库，并从中获取三个函数指针：

- JNI_GetDefaultJavaVMInitArgs：获取虚拟机的默认初始化参数
- JNI_CreateJavaVM：创建虚拟机实例
- JNI_GetCreatedJavaVMs：获取创建的虚拟机实例

这几个函数被定义在`jni.h`中，后面我们创建JVM的时候会用到这些函数

## AndroidRuntime::startVm

```c++
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote, bool primary_zygote)
{
    JavaVMInitArgs initArgs;
    ...
    //配置了一大堆JVM选项
    
    initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;

    //创建并初始化JVM
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

    return 0;
}
```

`JNI_CreateJavaVM`方法就是我们之前从`libart.so`里获得的方法，被编译进`libart.so`前，源码的路径为`art/runtime/jni/java_vm_ext.cc`，接下来属于ART虚拟机的工作，我们就不再往下深究了

后面有一个`onVmCreated`回调，但在zygote模式下没做任何事

# 注册JNI函数

接着调用`startReg`函数注册Android JNI函数

```c++
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ...
    //设置Native创建线程的函数，通过javaCreateThreadEtc这个函数创建的线程
    //会把创建的线程attach到JVM中，使其既能执行c/c++代码，也能执行Java代码
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    //创建局部引用栈帧
    env->PushLocalFrame(200);

    //注册jni函数
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    //将当前栈帧出栈，释放其中所有局部引用
    env->PopLocalFrame(NULL);

    return 0;
}
```

首先hook了Native创建线程的函数，之后创建线程便会调用我们设置的`javaCreateThreadEtc`函数，
会把创建的线程attach到JVM中，使其既能执行c/c++代码，也能执行Java代码。这个等之后看到Android线程创建的时候再分析

`PushLocalFrame`和`PopLocalFrame`是一对函数，它们是用来管理JNI的局部引用的

首先，`PushLocalFrame`会创建出一个局部引用栈帧，之后JNI创建出来的局部引用都会放在这个栈帧里，等使用结束后调用PopLocalFrame函数，会将当前栈帧出栈，并且释放其中所有的局部引用

接下来我们看`register_jni_procs`函数

```c++
struct RegJNIRec {
    int (*mProc)(JNIEnv*);
};
    
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
            ...
            return -1;
        }
    }
    return 0;
}
```

很简单，就是循环执行`gRegJNI`数组中所有的函数

```c++
#define REG_JNI(name)      { name }

static const RegJNIRec gRegJNI[] = {
        REG_JNI(register_com_android_internal_os_RuntimeInit),
        REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit),
        REG_JNI(register_android_os_SystemClock),
        REG_JNI(register_android_util_EventLog),
        REG_JNI(register_android_util_Log),
        ...
};
```

`gRegJNI`数组中存放着很多Java类注册JNI函数的函数，后面大家如果阅读源码看到了Android中的native方法可以来这边找它所对应的c++实现

这边注册的类非常多，我们就取第一个`register_com_android_internal_os_RuntimeInit`为例分析一下

```c++
typedef struct {
    const char* name;
    const char* signature;
    void*       fnPtr;
} JNINativeMethod;

int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    const JNINativeMethod methods[] = {
            {"nativeFinishInit", "()V",
             (void*)com_android_internal_os_RuntimeInit_nativeFinishInit},
            {"nativeSetExitWithoutCleanup", "(Z)V",
             (void*)com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup},
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
        methods, NELEM(methods));
}
```

创建了一个JNINativeMethod结构体，第一个成员是Java中的方法名，第二个成员是Java中对应方法的签名，第三个成员是Java方法对应native函数的函数指针，然后调用`jniRegisterNativeMethods`函数

```c++
int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);

    ALOGV("Registering %s's %d native methods...", className, numMethods);

    scoped_local_ref<jclass> c(env, findClass(env, className));
    ALOG_ALWAYS_FATAL_IF(c.get() == NULL,
                         "Native registration unable to find class '%s'; aborting...",
                         className);

    int result = e->RegisterNatives(c.get(), gMethods, numMethods);
    ALOG_ALWAYS_FATAL_IF(result < 0, "RegisterNatives failed for '%s'; aborting...",
                         className);

    return 0;
}
```

这个函数先通过Java类名获得一个jclass对象，接着调用`JNIEnv::RegisterNatives`函数，这个函数定义在`jni.h`中，实现在`libart.so`库中，我们在平时开发jni的时候，动态注册native方法的时候就会使用到它，这里就不再往下分析了

# 进入JAVA世界

JVM启动好了，JNI函数也注册完毕了，接下来就该进入到JAVA世界了

```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    //创建一个Java层的String数组用来装参数
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    //第一个参数是类名com.android.internal.os.ZygoteInit
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    //剩下来参数分别是start-system-server和abi-list
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    //将Java类名中的"."替换成"/"，这是JNI中的类名规则
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        //获取ZygoteInit中的main方法，参数为String类型，返回值为void
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            //执行ZygoteInit的main方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

            //后面的代码除非JVM挂了，否则不会执行
            ...
        }
    }
    ...
}
```

这里看不懂的自己先补一下JNI知识，总之就是调用了`com.android.internal.os.ZygoteInit`类的静态方法`main`，以`com.android.internal.os.ZygoteInit`，`start-system-server`和`abi-list`作为参数

# ZygoteInit

`ZygoteInit`类的源码路径为`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

我们这就开始分析它的main方法

```java
public static void main(String argv[]) {
    ZygoteServer zygoteServer = null;
    
    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    //标记着zygote开始启动，不允许创建线程（Zygote必须保证单线程）
    ZygoteHooks.startZygoteNoThreadCreation();
    
    // Zygote goes into its own process group.
    //设置进程组id
    try {
        Os.setpgid(0, 0);
    } catch (ErrnoException ex) {
        throw new RuntimeException("Failed to setpgid(0,0)", ex);
    }

    Runnable caller;
    try {
        ...
        //配置参数
        boolean startSystemServer = false;
        String zygoteSocketName = "zygote";
        String abiList = null;
        boolean enableLazyPreload = false;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if ("--enable-lazy-preload".equals(argv[i])) {
                enableLazyPreload = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }

        //public static final String PRIMARY_SOCKET_NAME = "zygote";
        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
        ...
        if (!enableLazyPreload) {
            ...
            //预加载
            preload(bootTimingsTraceLog);
            ...
        }
        ...

        //调用Java层的垃圾回收
        gcAndFinalize();
        ...
        //回调AppRRuntime中的onZygoteInit函数
        Zygote.initNativeState(isPrimaryZygote);

        //解除创建线程限制（马上就要执行fork了，子进程要有能力创建线程）
        ZygoteHooks.stopZygoteNoThreadCreation();

        //创建socket
        zygoteServer = new ZygoteServer(isPrimaryZygote);

        //启动SystemServer
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }

        Log.i(TAG, "Accepting command socket connections");

        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        //执行死循环监听socket，负责接收事件，启动App
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }

    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    //接收到AMS的启动App请求后，fork出子进程，处理App启动
    if (caller != null) {
        caller.run();
    }
}
```

先调用`ZygoteHooks.startZygoteNoThreadCreation()`禁止创建线程，Zygote必须保证单线程，这和`fork`机制有关，`fork`函数只会将当前线程复制到子进程，同时，`fork`会将锁也复制到子进程中，如果在`fork`之前，有一个线程持有了锁，但是`fork`的时候没把这个线程复制到子进程中，这把锁就被永久持有了，会造成死锁

## android.system.Os

我们看一下`Os`是什么，根据import我们知道它的全限定类名为`android.system.Os`，它的源码路径为`libcore/luni/src/main/java/android/system/Os.java`

```java
...
import libcore.io.Libcore;

public final class Os {
    private Os() {}
    ...
    public static void setpgid(int pid, int pgid) throws ErrnoException { Libcore.os.setpgid(pid, pgid); }
    ...
}
```

它里面全是这种形式的静态代理方法，实际调用`Libcore.os`执行，我们就以`setpgid`方法去追踪一下

`Libcore`位于`libcore/luni/src/main/java/libcore/io/Libcore.java`，`os`是其中的一个静态变量

```java
public final class Libcore {
    private Libcore() { }

    /**
     * Direct access to syscalls. Code should strongly prefer using {@link #os}
     * unless it has a strong reason to bypass the helpful checks/guards that it
     * provides.
     */
    public static final Os rawOs = new Linux();

    /**
     * Access to syscalls with helpful checks/guards.
     * For read access only; the only supported way to update this field is via
     * {@link #compareAndSetOs}.
     */
    @UnsupportedAppUsage
    public static volatile Os os = new BlockGuardOs(rawOs);

    public static Os getOs() {
        return os;
    }
    ...
}
```

`os`的类型为`BlockGuardOs`，以`Linux`类型的常量`rawOs`作为构造方法参数实例化，它继承自`ForwardingOs`

```java
public class ForwardingOs implements Os {
    @UnsupportedAppUsage
    private final Os os;

    @UnsupportedAppUsage
    @libcore.api.CorePlatformApi
    protected ForwardingOs(Os os) {
        this.os = Objects.requireNonNull(os);
    }
    ...
    public void setpgid(int pid, int pgid) throws ErrnoException { os.setpgid(pid, pgid); }
    ...
}
```

可以看到，这其实又是一个代理类，实际上是直接调用了`Linux`类的方法，至于`BlockGuardOs`，它在部分方法上做了一些回调监听，除此之外也是直接调用了`Linux`类的方法

```java
public final class Linux implements Os {
    Linux() { }

    ...
    public native int getpgid(int pid);
    ...
}
```

里面基本上都是JNI调用native方法，对应的c++源码路径为`libcore/luni/src/main/native/libcore_io_Linux.cpp`，下面是注册JNI函数的函数

```c++
#define NATIVE_METHOD(className, functionName, signature)                \
  MAKE_JNI_NATIVE_METHOD(#functionName, signature, className ## _ ## functionName)
  
#define MAKE_JNI_NATIVE_METHOD(name, signature, function)                      \
  _NATIVEHELPER_JNI_MAKE_METHOD(kNormalNative, name, signature, function)
  
#define _NATIVEHELPER_JNI_MAKE_METHOD(kind, name, sig, fn) \
  MAKE_CHECKED_JNI_NATIVE_METHOD(kind, name, sig, fn)
  
#define MAKE_CHECKED_JNI_NATIVE_METHOD(native_kind, name_, signature_, fn) \
  ([]() {                                                                \
    using namespace nativehelper::detail;  /* NOLINT(google-build-using-namespace) */ \
    static_assert(                                                       \
        MatchJniDescriptorWithFunctionType<native_kind,                  \
                                           decltype(fn),                 \
                                           fn,                           \
                                           sizeof(signature_)>(signature_),\
        "JNI signature doesn't match C++ function type.");               \
    /* Suppress implicit cast warnings by explicitly casting. */         \
    return JNINativeMethod {                                             \
        const_cast<decltype(JNINativeMethod::name)>(name_),              \
        const_cast<decltype(JNINativeMethod::signature)>(signature_),    \
        reinterpret_cast<void*>(&(fn))};                                 \
  })()

static JNINativeMethod gMethods[] = {
    ...
    NATIVE_METHOD(Linux, setpgid, "(II)V"),
    ...
};
void register_libcore_io_Linux(JNIEnv* env) {
    ...
    jniRegisterNativeMethods(env, "libcore/io/Linux", gMethods, NELEM(gMethods));
}
```

可以看到，Java层方法对应native方法的格式为`Linux_方法名`，我们通过这种规则找到`setpgid`方法对应的函数

```c++
static void Linux_setpgid(JNIEnv* env, jobject, jint pid, int pgid) {
    throwIfMinusOne(env, "setpgid", TEMP_FAILURE_RETRY(setpgid(pid, pgid)));
}
```

可以看到是直接调用了Linux系统层函数

### 总结

综上所述，`android.system.Os`类存在的意义是可以使Java层能够方便的调用Linux系统方法

## 预加载

接下来就是一些参数配置工作，然后调用`preload`预加载

```java
static void preload(TimingsTraceLog bootTimingsTraceLog) {
    ...
    //预加载Java类
    preloadClasses();
    ...
    //加载三个jar文件
    /* /system/framework/android.hidl.base-V1.0-java.jar */
    /* /system/framework/android.hidl.manager-V1.0-java.jar */
    /* /system/framework/android.test.base.jar */
    cacheNonBootClasspathClassLoaders();
    ...
    //预加载系统资源
    preloadResources();
    ...
    //预加载硬件抽象层？
    nativePreloadAppProcessHALs();
    ...
    //预加载opengl
    maybePreloadGraphicsDriver();
    ...
    //预加载动态库
    preloadSharedLibraries();
    //TextView预加载Font
    preloadTextResources();
    //预加载webviewchromium
    WebViewFactory.prepareWebViewInZygote();
    ...
}
```

### preloadClasses

我们看看Java类的预加载

```java
private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";

private static void preloadClasses() {
    final VMRuntime runtime = VMRuntime.getRuntime();

    InputStream is;
    try {
        is = new FileInputStream(PRELOADED_CLASSES);
    } catch (FileNotFoundException e) {
        Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");
        return;
    }
    ...
    try {
        BufferedReader br =
                new BufferedReader(new InputStreamReader(is), Zygote.SOCKET_BUFFER_SIZE);

        int count = 0;
        String line;
        while ((line = br.readLine()) != null) {
            // Skip comments and blank lines.
            line = line.trim();
            if (line.startsWith("#") || line.equals("")) {
                continue;
            }

            ...
            //Java类加载器加载类
            Class.forName(line, true, null);
            count++;
            ...
        }

        Log.i(TAG, "...preloaded " + count + " classes in "
                + (SystemClock.uptimeMillis() - startTime) + "ms.");
    } catch (IOException e) {
        Log.e(TAG, "Error reading " + PRELOADED_CLASSES + ".", e);
    } finally {
        ...
    }
}
```

主要的代码就是从`/system/etc/preloaded-classes`这个文件中读取出需要预加载的类，再通过`Class.forName`使用类加载器加载一遍，编译前的路径为`frameworks/base/config/preloaded-classes`

### 为什么需要预加载

Zygote进程的一大作用就是孵化App，那是怎么孵化的呢？这过程中肯定要使用到`fork`，我们知道，`fork`后，父子进程是可以共享资源的，既然我们每启动一个App，都需要使用虚拟机、加载一些View等必要的类等等，那为何不在父进程中加载好这些，fork后子进程不就可以直接使用它们了吗？这就是Zygote进程预加载的原因

## 启动binder线程池

预加载结束后，会先清理一下Java层的垃圾，然后调用`Zygote.initNativeState(isPrimaryZygote)`方法，这个方法调用了native方法`nativeInitNativeState`，这个方法是在`AndroidRuntime`中注册的，同时也实现在`AndroidRuntime`中，

```c++
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

我们之前分析过，执行的是`AndroidRuntime`的子类`AppRuntime`的`onZygoteInit`函数

```c++
virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}
```

通过这个函数启动Binder线程池，至于Binder的细节，我们留到以后再分析

## 启动SystemServer

`Zygote`进程孵化的第一个进程便是`SystemServer`，具体怎么孵化的，孵化后SystemServer又做了什么，留在下一节我们再分析

# ZygoteServer

## 构造方法

我们知道，我们App都是从`Zygote`孵化而来的，App启动是从`ActivityManagerService`的`startActivity`方法开始的，那么AMS是怎么和Zygote通信的呢，答案是通过socket

我们先从ZygoteServer的构造方法开始看起

```java
ZygoteServer(boolean isPrimaryZygote) {
    ...
    if (isPrimaryZygote) {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
        ...
    } else {
        ...
    }
    ...
}
```

其中有一些东西是和USAP机制有关的，但在AOSP中默认是关闭的，关于USAP机制我们以后再分析，现在只需要关注`mZygoteSocket`就可以了，它是通过调用`Zygote.createManagedSocketFromInitSocket`赋值的

```java
static LocalServerSocket createManagedSocketFromInitSocket(String socketName) {
    int fileDesc;
    //ANDROID_SOCKET_zygote
    final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;

    try {
        //获得文件描述符
        String env = System.getenv(fullSocketName);
        fileDesc = Integer.parseInt(env);
    } catch (RuntimeException ex) {
        throw new RuntimeException("Socket unset or invalid: " + fullSocketName, ex);
    }

    try {
        FileDescriptor fd = new FileDescriptor();
        fd.setInt$(fileDesc);
        return new LocalServerSocket(fd);
    } catch (IOException ex) {
        throw new RuntimeException(
            "Error building socket from file descriptor: " + fileDesc, ex);
    }
}
```

很简单，就是从系统属性中获取一个fd，然后实例化`LocalServerSocket`，路径为`frameworks/base/core/java/android/net/LocalServerSocket.java`

```java
public LocalServerSocket(FileDescriptor fd) throws IOException
{
    impl = new LocalSocketImpl(fd);
    impl.listen(LISTEN_BACKLOG);
    localAddress = impl.getSockAddress();
}
```

在内部创建了一个`LocalSocketImpl`，然后调用了`listen`方法声明开始监听这个fd，内部调用了Linux的`listen`函数

## runSelectLoop

然后我们来看在`ZygoteInit`中调用的`runSelectLoop`方法

```java
Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
    ArrayList<ZygoteConnection> peers = new ArrayList<>();

    //将server socket fd加到列表头（后面需要判断是否为server socket）
    socketFDs.add(mZygoteSocket.getFileDescriptor());
    peers.add(null);
    ...
    while (true) {
        ...
        StructPollfd[] pollFDs;
        ...
        pollFDs = new StructPollfd[socketFDs.size()];
        
        int pollIndex = 0;
        for (FileDescriptor socketFD : socketFDs) {
            pollFDs[pollIndex] = new StructPollfd();
            pollFDs[pollIndex].fd = socketFD;
            pollFDs[pollIndex].events = (short) POLLIN;
            ++pollIndex;
        }
        ... //上面一大段都与USAP机制有关，这里先不关注

        int pollReturnValue;
        try {
            //等待文件描述符上的事件
            pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }

        if (pollReturnValue == 0) { //没有接收到事件（超时），从循环开头重新开始等待事件
            ...
        } else {
            ...
            while (--pollIndex >= 0) {
                //没有要读取的数据，跳过
                if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                    continue;
                }

                if (pollIndex == 0) {
                    //pollIndex == 0说明这个fd是ZygoteServer socket的fd

                    //接受并建立一个socket连接
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    //将client socket fd加入列表
                    socketFDs.add(newPeer.getFileDescriptor());

                } else if (pollIndex < usapPoolEventFDIndex) {
                    //不使用USAP机制的话，pollIndex < usapPoolEventFDIndex条件一定成立
                    //进入这边说明是client socket

                    try {
                        //内部执行fork，返回一个待执行Runnable用于处理子进程后续任务
                        ZygoteConnection connection = peers.get(pollIndex);
                        final Runnable command = connection.processOneCommand(this);

                        //fork后，在子进程中会将这个变量设为true
                        if (mIsForkChild) { //子进程中
                            if (command == null) {
                                throw new IllegalStateException("command == null");
                            }
                            //return出去，由ZygoteInit执行这个Runnable
                            return command;
                        } else { //父进程中
                            if (command != null) {
                                throw new IllegalStateException("command != null");
                            }

                            //读取完了，关闭这个socket，清理列表
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(pollIndex);
                                socketFDs.remove(pollIndex);
                            }
                        }
                    } catch (Exception e) {
                        ...
                    } finally {
                        mIsForkChild = false;
                    }
                } else {
                    ... //不开启USAP机制不会走到这个分支
                }
            }
            ...
        }
        ...
    }
}
```

创建两个列表，`socketFDs`和`peers`的下标是一一对应的，首先将server socket fd添加到列表头，方便后续判断事件是来自client或是server socker，`peers`列表也要添加一个`null`作为`socketFDs`的对应

接着就开始执行死循环，`Zygote`进程永远不会退出这个循环，只有`fork`出子进程后，子进程会主动return

### poll

为了理解后面的内容，我们先要学习一下poll函数

poll是Linux中的字符设备驱动中的一个函数，它的作用是等待文件描述符上的某个事件

原型：`int poll(struct pollfd * fds , nfds_t nfds , int timeout );`

文档：https://man7.org/linux/man-pages/man2/poll.2.html

第一个参数是一个`pollfd`结构体指针

```c++
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

- fd不用多说，就是文件描述符
- event代表关注哪些事件，`POLLIN`代表可读，`POLLOUT`代表可写等等
- revents是由内核通知的，函数返回的时候，会设置对应的fd实际发生的事件，比如fd有可读的事件，设置`POLLIN`

第二个参数`nfds`表示fd的个数，即`pollfd`数组的size

第三个参数表示超时时间

返回值：

- 大于0：表示有fd事件产生，值为有产生事件的fd的个数
- 等于0：表示超时
- 小于0：表示有错误产生

#### StructPollfd & pollfd

弄懂poll是干嘛的后，我们再来接着看`runSelectLoop`方法

死循环中首先创建了一个`StructPollfd`数组，它根据`socketFDs`依次创建出一个个`StructPollfd`对象，并将他们的事件都设为`POLLIN`可读

`StructPollfd`和c中的结构体`pollfd`是对应的，目的是为了方便Java层调用Linux的`poll`函数

`StructPollfd`的路径为`libcore/luni/src/main/java/android/system/StructPollfd.java`

```java
public final class StructPollfd {
    public FileDescriptor fd;
    public short events;
    public short revents;
    public Object userData;
    ...
}
```

#### 调用

然后调用`Os.poll`方法

关于`Os`我们上面刚分析过，知道他调用了JNI函数，native函数命名格式为`Linux_函数名`，我们去`libcore/luni/src/main/native/libcore_io_Linux.cpp`中找一下

```java
static jint Linux_poll(JNIEnv* env, jobject, jobjectArray javaStructs, jint timeoutMs) {
    ... //把Java对象StructPollfd数组转换成c中的struct pollfd数组
    int rc;
    while (true) {
        ...
        rc = poll(fds.get(), count, timeoutMs);
        if (rc >= 0 || errno != EINTR) {
            break;
        }
        ...
    }
    if (rc == -1) {
        throwErrnoException(env, "poll");
        return -1;
    }
    ... //设置Java对象StructPollfd的revents值
    return rc;
}
```

简单看一下，就是把传进去的`StructPollfd`数组转换成了`struct pollfd`数组，然后调用Linux `poll`函数，再把`revents`写进`StructPollfd`对象中，最后返回

再看回`runSelectLoop`方法，如果poll执行返回值为-1，会直接引发一个Java异常，其他情况先判断一下`poll`的返回值，如果为0，则没有事件产生，否则会从后向前依次判断`pollFDs`的`revents`，如果为`POLLIN`可读，则处理，不可读则跳过

### 建立连接

我们先看第一次`poll`到事件的情况，这时候，`pollFDs`中只有一个zygote socket fd，收到可读事件，说明有客户端socket向zygote socket请求发起连接，这时候我们调用`acceptCommandPeer`方法建立新连接

```java
private ZygoteConnection acceptCommandPeer(String abiList) {
    try {
        return createNewConnection(mZygoteSocket.accept(), abiList);
    } catch (IOException ex) {
        throw new RuntimeException(
                "IOException during accept()", ex);
    }
}
```

调用了`mZygoteSocket.accept`方法

```java
public LocalSocket accept() throws IOException
{
    LocalSocketImpl acceptedImpl = new LocalSocketImpl();

    impl.accept(acceptedImpl);

    return LocalSocket.createLocalSocketForAccept(acceptedImpl);
}
```

新建了一个`LocalSocketImpl`(client socket) 实例，然后调用`LocalSocketImpl`(zygote socket) 的`accept`方法

```java
protected void accept(LocalSocketImpl s) throws IOException {
    if (fd == null) {
        throw new IOException("socket not created");
    }

    try {
        s.fd = Os.accept(fd, null /* address */);
        s.mFdCreatedInternally = true;
    } catch (ErrnoException e) {
        throw e.rethrowAsIOException();
    }
}
```

调用了Linux的accept函数，接受建立连接，并返回了一个新的client socket fd，将`LocalSocketImpl`中的fd变量设置为这个fd，接着调用`LocalSocket.createLocalSocketForAccept`将`LocalSocketImpl`包装成`LocalSocket`

```java
static LocalSocket createLocalSocketForAccept(LocalSocketImpl impl) {
    return createConnectedLocalSocket(impl, SOCKET_UNKNOWN);
}

private static LocalSocket createConnectedLocalSocket(LocalSocketImpl impl, int sockType) {
    LocalSocket socket = new LocalSocket(impl, sockType);
    socket.isConnected = true;
    socket.isBound = true;
    socket.implCreated = true;
    return socket;
}
```

然后使用这个`LocalSocket`创建了一个`ZygoteConnection`包装socket连接

```java
protected ZygoteConnection createNewConnection(LocalSocket socket, String abiList)
        throws IOException {
    return new ZygoteConnection(socket, abiList);
}
```

我们看一下构造方法做了什么

```java
ZygoteConnection(LocalSocket socket, String abiList) throws IOException {
    mSocket = socket;
    this.abiList = abiList;

    mSocketOutStream = new DataOutputStream(socket.getOutputStream());
    mSocketReader =
            new BufferedReader(
                    new InputStreamReader(socket.getInputStream()), Zygote.SOCKET_BUFFER_SIZE);
    ...
    isEof = false;
}
```

打开了client socket的输入输出流，准备读写数据了

然后将这个连接和fd分别添加进`peers`和`socketFDs`

### 执行client socket命令

在第二次循环中`pollFDs`数组中便包括了新建立连接的client socket了，这时调用`Os.poll`，可以获得到这个client socket的可读事件，此时调用`connection.processOneCommand`方法

```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    String[] args;

    try {
        //读取从client socket传来的参数
        args = Zygote.readArgumentList(mSocketReader);
    } catch (IOException ex) {
        throw new IllegalStateException("IOException on command socket", ex);
    }
    ...
    int pid;
    FileDescriptor childPipeFd = null;
    FileDescriptor serverPipeFd = null;

    //解析参数
    ZygoteArguments parsedArgs = new ZygoteArguments(args);

    ... //一系列参数校验工作

    //创建子进程
    pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
            parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mIsTopApp,
            parsedArgs.mPkgDataInfoList, parsedArgs.mWhitelistedDataInfoList,
            parsedArgs.mBindMountAppDataDirs, parsedArgs.mBindMountAppStorageDirs);

    try {
        if (pid == 0) { //子进程中
            //设置mIsForkChild = true
            zygoteServer.setForkChild();
            //子进程中关闭fork复制来的zygote socket
            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;

            return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
        } else { //父进程中
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            handleParentProc(pid, serverPipeFd);
            return null;
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```

# 启动APP进程

首先读取从client socket传来的参数，然后校验这些参数，完毕后调用`Zygote.forkAndSpecialize`方法fork出子进程

```java
static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
        int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir,
        boolean isTopApp, String[] pkgDataInfoList, String[] whitelistedDataInfoList,
        boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs) {
    //停止其他线程
    ZygoteHooks.preFork();

    //fork进程
    int pid = nativeForkAndSpecialize(
            uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
            fdsToIgnore, startChildZygote, instructionSet, appDataDir, isTopApp,
            pkgDataInfoList, whitelistedDataInfoList, bindMountAppDataDirs,
            bindMountAppStorageDirs);
    ...
    //恢复其他线程
    ZygoteHooks.postForkCommon();
    return pid;
}
```

Zygote进程启动了4个线程：

- HeapTaskDaemon
- ReferenceQueueDaemon
- FinalizerDaemon
- FinalizerWatchdogDaemon

之前上面也分析过了多线程对fork会产生影响，所以这里先把其他线程停了，等fork完了再重新启动

然后执行native函数`nativeForkAndSpecialize`，路径为`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`

```c++
static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits,
        jint mount_external, jstring se_info, jstring nice_name,
        jintArray managed_fds_to_close, jintArray managed_fds_to_ignore, jboolean is_child_zygote,
        jstring instruction_set, jstring app_data_dir, jboolean is_top_app,
        jobjectArray pkg_data_info_list, jobjectArray whitelisted_data_info_list,
        jboolean mount_data_dirs, jboolean mount_storage_dirs) {
    ...
    pid_t pid = ForkCommon(env, false, fds_to_close, fds_to_ignore, true);

    if (pid == 0) {
      SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits,
                       capabilities, capabilities,
                       mount_external, se_info, nice_name, false,
                       is_child_zygote == JNI_TRUE, instruction_set, app_data_dir,
                       is_top_app == JNI_TRUE, pkg_data_info_list,
                       whitelisted_data_info_list,
                       mount_data_dirs == JNI_TRUE,
                       mount_storage_dirs == JNI_TRUE);
    }
    return pid;
}
```

先调用`ForkCommon`，再在子进程调用`SpecializeCommon`

```c++
static pid_t ForkCommon(JNIEnv* env, bool is_system_server,
                        const std::vector<int>& fds_to_close,
                        const std::vector<int>& fds_to_ignore,
                        bool is_priority_fork) {
  //设置子进程信号处理函数
  SetSignalHandlers();
  ...
  //fork前先阻塞SIGCHLD信号
  BlockSignal(SIGCHLD, fail_fn);
  ...
  //执行fork
  pid_t pid = fork();
  ...
  //恢复SIGCHLD信号
  UnblockSignal(SIGCHLD, fail_fn);

  return pid;
}
```

```c++
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
  const char* process_name = is_system_server ? "system_server" : "zygote";
  ...
  //创建进程组
  if (!is_system_server && getuid() == 0) {
    const int rc = createProcessGroup(uid, getpid());
    if (rc == -EROFS) {
      ALOGW("createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?");
    } else if (rc != 0) {
      ALOGE("createProcessGroup(%d, %d) failed: %s", uid, /* pid= */ 0, strerror(-rc));
    }
  }
  //设置GroupId
  SetGids(env, gids, is_child_zygote, fail_fn);
  //设置资源Limit
  SetRLimits(env, rlimits, fail_fn);
  ...
  //设置调度策略
  SetSchedulerPolicy(fail_fn, is_top_app);
  ...
  //设置线程名
  if (nice_name.has_value()) {
    SetThreadName(nice_name.value());
  } else if (is_system_server) {
    SetThreadName("system_server");
  }

  //子进程中不再处理SIGCHLD信号
  UnsetChldSignalHandler();
  ...

  if (is_child_zygote) {
      initUnsolSocketToSystemServer();
  }

  //调用Zygote.callPostForkChildHooks方法
  env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, runtime_flags,
                            is_system_server, is_child_zygote, managed_instruction_set);

  //设置默认进程优先级
  setpriority(PRIO_PROCESS, 0, PROCESS_PRIORITY_DEFAULT);

  if (env->ExceptionCheck()) {
    fail_fn("Error calling post fork hooks.");
  }
}
```

子进程创建完成后，`ZygoteConnection`在子进程中会返回`handleChildProc`，在父进程中会返回`null`

在`ZygoteServer`中做了判断，如果为子进程且command不为null，返回common到`ZygoteInit`，如果是父进程，继续socket poll循环

在`ZygoteInit.runSelectLoop`后，如果返回值`caller`（对应`ZygoteServer`中的`command`）不为`null`，则执行这个`Runnable`

我们看一下`handleChildProc`做了什么

```java
private Runnable handleChildProc(ZygoteArguments parsedArgs,
        FileDescriptor pipeFd, boolean isZygote) {
    //在子进程中关闭发起启动App请求的client socket
    closeSocket();
    //设置进程名
    Zygote.setAppProcessName(parsedArgs, TAG);

    ...
    if (parsedArgs.mInvokeWith != null) {
        //和进程内存泄露或溢出有关？
        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.mRemainingArgs);

        // Should not get here.
        throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
    } else {
        if (!isZygote) {
            //根据参数，执行这个方法
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mDisabledCompatChanges,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        } else {
            return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        }
    }
}
```

执行`ZygoteInit.zygoteInit`

```java
public static final Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    ...
    //重定向Log
    RuntimeInit.redirectLogStreams();
    //通用初始化
    RuntimeInit.commonInit();
    //之前有提到，开启binder线程池
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

然后执行`RuntimeInit.applicationInit`

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

这里的startClass为`android.app.ActivityThread`

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

这里使用了Java中的反射，找到了`ActivityThread`中对应的`main`方法，并用其创建了一个`Runnable`对象`MethodAndArgsCaller`

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
            //执行ActivityThread.main方法
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

之前说了，在`ZygoteInit.runSelectLoop`后，如果返回值`caller`不为`null`，则执行这个`Runnable`，即执行`MethodAndArgsCaller`的`run`方法，反射调用`ActivityThread.main`方法

# 结束

至此，Zygote进程部分的分析就到此为止了，后面fork出App进程那段讲的很粗糙，后面写到App启动那块的时候，我会重新梳理一遍这里的逻辑，补充上去