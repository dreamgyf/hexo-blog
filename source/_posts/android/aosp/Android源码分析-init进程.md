---
title: Android源码分析 - init进程
date: 2022-01-04 17:19:00
tags: 
- Android源码
- init进程
categories: 
- [Android, 源码分析]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

PC启动会通过BIOS引导，从0x7c00处找到以0xaa55为结尾的引导程序启动。而Android通常使用在移动设备上，没有PC的BIOS，取而代之的是BootLoader。

# BootLoader

在CPU上电复位完成后，会从一个固定的地址加载一段程序，即BootLoader，不同的CPU可能这个地址不同。BootLoader是一段引导程序，其中最为常见的为U-boot，它一般会先检测用户是否按下某些特别按键，这些特别按键是uboot在编译时预先被约定好的，用于进入调试模式。如果用户没有按这些特别的按键，则uboot会从NAND Flash中装载Linux内核，装载的地址是在编译uboot时预先约定好的。

# 进程

## idle

Linux内核启动后，便会创建第一个进程idle。idle进程是Linux中的第一个进程，pid为0，是唯一一个没有通过fork产生的进程，它的优先级非常低，用于CPU没有任务的时候进行空转。

## init

init进程由idle进程创建，是Linux系统的第一个用户进程，pid为1，是系统所有用户进程的直接或间接父进程，本篇重点讲的就是它。

## kthreadd

kthreadd进程同样由idle进程创建，pid为2，它始终运行在内核空间，负责所有内核线程的调度与管理。

# init进程

Android的init进程代码在`system/core/init/main.cpp`中，以main方法作为入口，分为几个阶段：

```c++
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif

    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }

    return FirstStageMain(argc, argv);
}
```

这里的ueventd和subcontext，都是在各自守护进程中执行的，不在init进程中执行，这里就不多介绍了

## FirstStageMain

默认不加任何参数启动init的话，便会开始init第一阶段，进入到FirstStageMain函数中，代码在`system/core/init/first_stage_init.cpp`中

### umask

文档：<https://man7.org/linux/man-pages/man2/umask.2.html>

原型：`mode_t umask(mode_t mask);`

这个方法是用来设置创建目录或文件时所应该赋予权限的掩码

Linux中，文件默认最大权限是666，目录最大权限是777，当创建目录时，假设掩码为022，那赋予它的权限为（777 & ~022）= 755

在执行init第一阶段时，先执行umask(0)，使创建的目录或文件的默认权限为最高

### 创建目录、设备节点，挂载

```c++
int FirstStageMain(int argc, char** argv) {
    ...
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);

    // Clear the umask.
    umask(0);

    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
    CHECKCALL(mkdir("/dev/pts", 0755));
    CHECKCALL(mkdir("/dev/socket", 0755));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));
#define MAKE_STR(x) __STRING(x)
    CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
#undef MAKE_STR
    CHECKCALL(chmod("/proc/cmdline", 0440));
    std::string cmdline;
    android::base::ReadFileToString("/proc/cmdline", &cmdline);
    gid_t groups[] = {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL));

    CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));

    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11)));
    }

    CHECKCALL(mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)));
    CHECKCALL(mknod("/dev/ptmx", S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3)));
    CHECKCALL(mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=1000"));
    CHECKCALL(mkdir("/mnt/vendor", 0755));
    CHECKCALL(mkdir("/mnt/product", 0755));
    CHECKCALL(mount("tmpfs", "/debug_ramdisk", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
#undef CHECKCALL
    ...
}
```

### 初始化日志

#### SetStdioToDevNull

由于Linux内核打开了/dev/console作为标准输入输出流（stdin/stdout/stderr）的文件描述符，而init进程在用户空间，无权访问/dev/console，后续如果执行printf的话可能会导致错误，所以先调用`SetStdioToDevNull`函数来将标准输入输出流（stdin/stdout/stderr）用/dev/null文件描述符替换

/dev/null被称为空设备，是一个特殊的设备文件，它会丢弃一切写入其中的数据，读取它会立即得到一个EOF

#### InitKernelLogging

接着调用`InitKernelLogging`函数，初始化了一个简单的kernel日志系统

### 创建设备，挂载分区

```c++
int FirstStageMain(int argc, char** argv) {
    ...
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline) : 0;

    if (!LoadKernelModules(IsRecoveryMode() && !ForceNormalBoot(cmdline), want_console)) {
        if (want_console != FirstStageConsoleParam::DISABLED) {
            LOG(ERROR) << "Failed to load kernel modules, starting console";
        } else {
            LOG(FATAL) << "Failed to load kernel modules";
        }
    }

    if (want_console == FirstStageConsoleParam::CONSOLE_ON_FAILURE) {
        StartConsole();
    }

    if (ForceNormalBoot(cmdline)) {
        mkdir("/first_stage_ramdisk", 0755);
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            LOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }
    ...
    if (!DoFirstStageMount()) {
        LOG(FATAL) << "Failed to mount required partitions early ...";
    }
    ...
}
```

### 结束

至此，第一阶段的init结束，通过execv函数带参执行init文件，进入`SetupSelinux`

```c++
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";
```

#### exec系列函数

用exec系列函数可以把当前进程替换为一个新进程，且新进程与原进程有相同的PID。

这里在末尾直接打log是因为，exec系列函数如果执行正常是不会返回的，所以只要执行到下面就代表exec执行出错了

## SetupSelinux

启动Selinux安全机制，初始化selinux，加载SELinux规则，配置SELinux相关log输出，并启动第二阶段：`SecondStageMain`

```c++
int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    MountMissingSystemPartitions();

    // Set up SELinux, loading the SELinux policy.
    SelinuxSetupKernelLogging();
    SelinuxInitialize();

    // We're in the kernel domain and want to transition to the init domain.  File systems that
    // store SELabels in their xattrs, such as ext4 do not need an explicit restorecon here,
    // but other file systems do.  In particular, this is needed for ramdisks such as the
    // recovery image for A/B devices.
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);

    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never return from this function.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```

## SecondStageMain

使用`second_stage`参数启动init的话，便会开始init第二阶段，进入到`SecondStageMain`函数中，代码在`system/core/init/init.cpp`中

```c++
int SecondStageMain(int argc, char** argv) {
    ...
    //和第一阶段一样，初始化日志
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";
    ...
    // Set init and its forked children's oom_adj.
    //设置init进程和以后fork出来的进程的OOM等级，这里的值为-1000，保证进程不会因为OOM被杀死
    if (auto result =
                WriteFile("/proc/1/oom_score_adj", StringPrintf("%d", DEFAULT_OOM_SCORE_ADJUST));
        !result.ok()) {
        LOG(ERROR) << "Unable to write " << DEFAULT_OOM_SCORE_ADJUST
                   << " to /proc/1/oom_score_adj: " << result.error();
    }
    ...
    // Indicate that booting is in progress to background fw loaders, etc.
    //设置一个标记，代表正在启动过程中
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
    ...
    //初始化系统属性
    PropertyInit();
    ...
    // Mount extra filesystems required during second stage init
    //挂载额外的文件系统
    MountExtraFilesystems();

    // Now set up SELinux for second stage.
    //设置SELinux
    SelinuxSetupKernelLogging();
    SelabelInitialize();
    SelinuxRestoreContext();

    //使用epoll，注册信号处理函数，守护进程服务
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }

    InstallSignalFdHandler(&epoll);
    InstallInitNotifier(&epoll);
    //启动系统属性服务
    StartPropertyService(&property_fd);
    ...
    //设置commands指令所对应的函数map
    const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    Action::set_function_map(&function_map);
    ...
    //解析init.rc脚本
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);
    ...
    //构建了一些Action，Trigger等事件对象加入事件队列中
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    Keychords keychords;
    am.QueueBuiltinAction(
            [&epoll, &keychords](const BuiltinArguments& args) -> Result<void> {
                for (const auto& svc : ServiceList::GetInstance()) {
                    keychords.Register(svc->keycodes());
                }
                keychords.Start(&epoll, HandleKeychord);
                return {};
            },
            "KeychordInit");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    //死循环，等待事件处理
    while (true) {
        // By default, sleep until something happens.
        auto epoll_timeout = std::optional<std::chrono::milliseconds>{};
        ...
        //执行从init.rc脚本解析出来的每条指令
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        ...
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout = 0ms;
        }

        auto pending_functions = epoll.Wait(epoll_timeout);
        if (!pending_functions.ok()) {
            LOG(ERROR) << pending_functions.error();
        } else if (!pending_functions->empty()) {
            // We always reap children before responding to the other pending functions. This is to
            // prevent a race where other daemons see that a service has exited and ask init to
            // start it again via ctl.start before init has reaped it.
            //处理子进程退出后的相关事项
            ReapAnyOutstandingChildren();
            for (const auto& function : *pending_functions) {
                (*function)();
            }
        }
        ...
    }

    return 0;
}
```

### Linux OOM Killer机制

Linux下有一种 OOM KILLER 的机制，它会在系统内存耗尽的情况下，启用自己算法有选择性的杀掉一些进程，这个算法和三个值有关：

- /proc/PID/oom_score ,OOM 最终得分，值越大越有可能被杀掉
- /proc/PID/oom_score_adj ，取值范围为-1000到1000，计算oom_score时会加上该参数
- /proc/PID/oom_adj ，取值是-17到+15，该参数主要是为兼容旧版内核

在init过程中，代码设置了init进程和以后fork出来的进程的OOM等级，这里的值为-1000，设置为这个值就可以保证进程永远不会因为OOM被杀死

### 解析init.rc脚本

#### Android Init Language

rc文件，是用`Android Init Language`编写的特殊文件。用这种语法编写的文件，统一用".rc"后缀

它的语法说明可以在aosp源码`system/core/init/README.md`中找到，这里就简单说明一下语法规则

##### Actions

`Actions`是一系列命令的开始，一个`Action`会有一个触发器，用于确定`Action`何时执行。当一个与`Action`的触发器匹配的事件发生时，该动作被添加到待执行队列的尾部

格式如下：

```
on <trigger> [&& <trigger>]* 
    <command> 
    <command> 
    <command>
```

##### Triggers（触发器）

触发器作用于`Actions`，可用于匹配某些类型的事件，并用于导致操作发生

##### Commands

`Commands`就是一个个命令的集合了

`Action`, `Triggers`, `Commands`共同组成了一个单元，举个例子：

```
on zygote-start && property:ro.crypto.state=unencrypted 
    # A/B update verifier that marks a successful boot. 
    exec_start update_verifier_nonencrypted 
    start statsd 
    start netd 
    start zygote 
    start zygote_secondary
```

##### Services

`Services`是对一些程序的定义，格式如下：

```
service <name> <pathname> [ <argument> ]*
    <option>
    <option>
    ...
```

其中：

- name：定义的服务名
- pathname：这个程序的路径
- argument：程序运行的参数
- option：服务选项，后文将介绍

##### Options

`Options`是对`Services`的修饰，它们影响着服务运行的方式和时间

`Services`, `Options`组成了一个单元，举个例子：

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
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
    task_profiles ProcessCapacityHigh MaxPerformance
```

##### Imports

导入其他的rc文件或目录解析，如果path是一个目录，目录中的每个文件都被解析为一个rc文件。它不是递归的，嵌套的目录将不会被解析。

格式如下：

```
import <path>
```

`Imports`的内容会放到最后解析

上文所述的`Commands`，`Options`等具体命令，可以网上搜索一下，或者自己看`system/core/init/README.md`

`Commands`的定义可以在`system/core/init/builtins.cpp`中找到

`Options`的定义可以在`system/core/init/service_parser.cpp`中找到

#### 解析

```c++
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```

这个函数会从这些地方寻找rc文件解析，`/system/etc/init/hw/init.rc`是主rc文件，剩下的目录，如果system分区尚未挂载的话，就把它们加入到`late_import_paths`中，等到后面`mount_all`时再加载

主rc文件在编译前的位置为`system/core/rootdir/init.rc`

简单分析一下：

首先，以`ActionManager`和`ServiceList`作为参数创建了一个Parser解析器，解析后的结果会存放在`ActionManager`和`ServiceList`中，这里的两个传进来的参数都是单例模式

```c++
Parser CreateParser(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser;

    parser.AddSectionParser("service", std::make_unique<ServiceParser>(
                                               &service_list, GetSubcontext(), std::nullopt));
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&action_manager, GetSubcontext()));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));

    return parser;
}
```

先创建了一个`Parser`对象，然后往里面添加了`ServiceParser`、`ActionParser`以及`ImportParser`，这三个类都是继承自`ServiceParser`，这里的`std::make_unique`是new了一个对象，并用其原始指针构造出了一个智能指针

接着走到`Parser::ParseConfig`方法中：

```c++
bool Parser::ParseConfig(const std::string& path) {
    if (is_dir(path.c_str())) {
        return ParseConfigDir(path);
    }
    return ParseConfigFile(path);
}
```

判断是否是目录，如果是目录，就把目录中的所有文件加入容器中排序后依次解析

```c++
bool Parser::ParseConfigDir(const std::string& path) {
    LOG(INFO) << "Parsing directory " << path << "...";
    std::unique_ptr<DIR, decltype(&closedir)> config_dir(opendir(path.c_str()), closedir);
    if (!config_dir) {
        PLOG(INFO) << "Could not import directory '" << path << "'";
        return false;
    }
    dirent* current_file;
    std::vector<std::string> files;
    while ((current_file = readdir(config_dir.get()))) {
        // Ignore directories and only process regular files.
        if (current_file->d_type == DT_REG) {
            std::string current_path =
                android::base::StringPrintf("%s/%s", path.c_str(), current_file->d_name);
            files.emplace_back(current_path);
        }
    }
    // Sort first so we load files in a consistent order (bug 31996208)
    std::sort(files.begin(), files.end());
    for (const auto& file : files) {
        if (!ParseConfigFile(file)) {
            LOG(ERROR) << "could not import file '" << file << "'";
        }
    }
    return true;
}
```

可以看到，最终都调用了`Parser::ParseConfigFile`方法

```c++
bool Parser::ParseConfigFile(const std::string& path) {
    LOG(INFO) << "Parsing file " << path << "...";
    android::base::Timer t;
    auto config_contents = ReadFile(path);
    if (!config_contents.ok()) {
        LOG(INFO) << "Unable to read config file '" << path << "': " << config_contents.error();
        return false;
    }

    ParseData(path, &config_contents.value());

    LOG(VERBOSE) << "(Parsing " << path << " took " << t << ".)";
    return true;
}
```

从文件中读取出字符串，并继续调用`Parser::ParseData`方法

```c++
void Parser::ParseData(const std::string& filename, std::string* data) {
    data->push_back('\n');  // TODO: fix tokenizer
    data->push_back('\0');

    parse_state state;
    state.line = 0;
    state.ptr = data->data();
    state.nexttoken = 0;

    SectionParser* section_parser = nullptr;
    int section_start_line = -1;
    std::vector<std::string> args;

    // If we encounter a bad section start, there is no valid parser object to parse the subsequent
    // sections, so we must suppress errors until the next valid section is found.
    bool bad_section_found = false;

    auto end_section = [&] {
        bad_section_found = false;
        if (section_parser == nullptr) return;

        if (auto result = section_parser->EndSection(); !result.ok()) {
            parse_error_count_++;
            LOG(ERROR) << filename << ": " << section_start_line << ": " << result.error();
        }

        section_parser = nullptr;
        section_start_line = -1;
    };

    for (;;) {
        switch (next_token(&state)) {
            case T_EOF:
                end_section();

                for (const auto& [section_name, section_parser] : section_parsers_) {
                    section_parser->EndFile();
                }

                return;
            case T_NEWLINE: {
                state.line++;
                if (args.empty()) break;
                // If we have a line matching a prefix we recognize, call its callback and unset any
                // current section parsers.  This is meant for /sys/ and /dev/ line entries for
                // uevent.
                auto line_callback = std::find_if(
                    line_callbacks_.begin(), line_callbacks_.end(),
                    [&args](const auto& c) { return android::base::StartsWith(args[0], c.first); });
                if (line_callback != line_callbacks_.end()) {
                    end_section();

                    if (auto result = line_callback->second(std::move(args)); !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                    }
                } else if (section_parsers_.count(args[0])) {
                    end_section();
                    section_parser = section_parsers_[args[0]].get();
                    section_start_line = state.line;
                    if (auto result =
                                section_parser->ParseSection(std::move(args), filename, state.line);
                        !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                        section_parser = nullptr;
                        bad_section_found = true;
                    }
                } else if (section_parser) {
                    if (auto result = section_parser->ParseLineSection(std::move(args), state.line);
                        !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                    }
                } else if (!bad_section_found) {
                    parse_error_count_++;
                    LOG(ERROR) << filename << ": " << state.line
                               << ": Invalid section keyword found";
                }
                args.clear();
                break;
            }
            case T_TEXT:
                args.emplace_back(state.text);
                break;
        }
    }
}

```

这里新建了一个`parse_state`结构体，用来以行为单位，分割整个文件字符串，根据分割出来的结果返回相应的TYPE，`Parser::ParseData`方法再通过TYPE来做逐行解析

这个结构体以及TPYE和分割分割方法的定义在`system/core/init/tokenizer.h`中，在`system/core/init/tokenizer.cpp`中实现

```c++
int next_token(struct parse_state *state)
{
    char *x = state->ptr;
    char *s;

    if (state->nexttoken) {
        int t = state->nexttoken;
        state->nexttoken = 0;
        return t;
    }

    for (;;) {
        switch (*x) {
        case 0:
            state->ptr = x;
            return T_EOF;
        case '\n':
            x++;
            state->ptr = x;
            return T_NEWLINE;
        case ' ':
        case '\t':
        case '\r':
            x++;
            continue;
        case '#':
            while (*x && (*x != '\n')) x++;
            if (*x == '\n') {
                state->ptr = x+1;
                return T_NEWLINE;
            } else {
                state->ptr = x;
                return T_EOF;
            }
        default:
            goto text;
        }
    }

textdone:
    state->ptr = x;
    *s = 0;
    return T_TEXT;
text:
    state->text = s = x;
textresume:
    for (;;) {
        switch (*x) {
        case 0:
            goto textdone;
        case ' ':
        case '\t':
        case '\r':
            x++;
            goto textdone;
        case '\n':
            state->nexttoken = T_NEWLINE;
            x++;
            goto textdone;
        case '"':
            x++;
            for (;;) {
                switch (*x) {
                case 0:
                        /* unterminated quoted thing */
                    state->ptr = x;
                    return T_EOF;
                case '"':
                    x++;
                    goto textresume;
                default:
                    *s++ = *x++;
                }
            }
            break;
        case '\\':
            x++;
            switch (*x) {
            case 0:
                goto textdone;
            case 'n':
                *s++ = '\n';
                x++;
                break;
            case 'r':
                *s++ = '\r';
                x++;
                break;
            case 't':
                *s++ = '\t';
                x++;
                break;
            case '\\':
                *s++ = '\\';
                x++;
                break;
            case '\r':
                    /* \ <cr> <lf> -> line continuation */
                if (x[1] != '\n') {
                    x++;
                    continue;
                }
                x++;
                FALLTHROUGH_INTENDED;
            case '\n':
                    /* \ <lf> -> line continuation */
                state->line++;
                x++;
                    /* eat any extra whitespace */
                while((*x == ' ') || (*x == '\t')) x++;
                continue;
            default:
                    /* unknown escape -- just copy */
                *s++ = *x++;
            }
            continue;
        default:
            *s++ = *x++;
        }
    }
    return T_EOF;
}
```

简单来说就是先看看有没有遇到结束符（换行 \n 或者EOF \0）或者注释（#）如果遇到了就返回`T_NEWLINE`或者`T_EOF`代表这上一行结束了或者整个文件读取完了，没有遇到的话说明读取的是可解析的正文，跳到text段，将文本内容写到`state.text`中，直到碰到换行符或空格等分割标志（以空格或换行等作为分隔符，一小段一小段的进行分割），将读取到的最后一个正文的位置+1处的字符置为\0，`state.text`里的内容便称为了完整一段的内容，接着返回`T_TEXT`表示已读入一段文本

接着回到`Parser::ParseData`方法中，如果读到的TYPE是`T_TEXT`，就将这一段内容先添加到容器中，当读到`T_NEWLINE`时，解析之前读入的一整行内容，先用args[0]（一行的开头）去寻找我们之前添加的`SectionParser`，如果能找到，说明这一行是service、on或者import，将`section_parser`赋值为相应`SectionParser`子类的指针，调用其`ParseSection`方法解析，如果读入的一行里，不是以service、on或者import开头，并且之前定义的`section_parser`不为空指针，说明是service或者on参数的子参数，调用`ParseLineSection`方法解析子参数，并加入到父参数中。

最后，每次读取完都会执行`args.clear()`清楚这一行的数据，当读取到新的service、on或者import时，需要先执行`EndSection`方法，将之前解析好的结构添加到列表中

### 执行任务

回到`SecondStageMain`中，可以看到，最后有一个死循环，用来等待事件处理

```c++
int SecondStageMain(int argc, char** argv) {
    ...
    while (true) {
        // By default, sleep until something happens.
        auto epoll_timeout = std::optional<std::chrono::milliseconds>{};
        ...
        //执行从init.rc脚本解析出来的每条指令
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        ...
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout = 0ms;
        }

        auto pending_functions = epoll.Wait(epoll_timeout);
        if (!pending_functions.ok()) {
            LOG(ERROR) << pending_functions.error();
        } else if (!pending_functions->empty()) {
            // We always reap children before responding to the other pending functions. This is to
            // prevent a race where other daemons see that a service has exited and ask init to
            // start it again via ctl.start before init has reaped it.
            //处理子进程退出后的相关事项
            ReapAnyOutstandingChildren();
            for (const auto& function : *pending_functions) {
                (*function)();
            }
        }
        ...
    }
    
    return 0;
}
```

其中`am.ExecuteOneCommand()`方法便是执行从rc文件中解析出来的指令

```c++
void ActionManager::ExecuteOneCommand() {
    {
        auto lock = std::lock_guard{event_queue_lock_};
        // Loop through the event queue until we have an action to execute
        //当前正在执行的action队列为空，但等待执行的事件队列不为空
        while (current_executing_actions_.empty() && !event_queue_.empty()) {
            for (const auto& action : actions_) {
                //从等待执行的事件队列头取出一个元素event， 
                //然后调用action的CheckEvent检查此event是否匹配当前action 
                //如果匹配，将这个action加入到正在执行的actions队列的队尾
                if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                               event_queue_.front())) {
                    current_executing_actions_.emplace(action.get());
                }
            }
            event_queue_.pop();
        }
    }

    if (current_executing_actions_.empty()) {
        return;
    }
    
    //从队列头取一个action（front不会使元素出队）
    auto action = current_executing_actions_.front();

    //如果是第一次执行这个action
    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        LOG(INFO) << "processing action (" << trigger_name << ") from (" << action->filename()
                  << ":" << action->line() << ")";
    }

    //这个current_command_是个成员变量，标志着执行到了哪一行
    action->ExecuteOneCommand(current_command_);

    // If this was the last command in the current action, then remove
    // the action from the executing list.
    // If this action was oneshot, then also remove it from actions_.
    ++current_command_;
    //current_command_等于action的commands数量，说明这个action以及全部执行完了
    if (current_command_ == action->NumCommands()) {
        //此action出队
        current_executing_actions_.pop();
        //重置计数器
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action](std::unique_ptr<Action>& a) { return a.get() == action; };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser),
                           actions_.end());
        }
    }
}
```

里面会执行`Action::ExecuteOneCommand`方法

```c++
void Action::ExecuteOneCommand(std::size_t command) const {
    // We need a copy here since some Command execution may result in
    // changing commands_ vector by importing .rc files through parser
    Command cmd = commands_[command];
    ExecuteCommand(cmd);
}
```

接着调用到了`Action::ExecuteCommand`方法

```c++
void Action::ExecuteCommand(const Command& command) const {
    android::base::Timer t;
    //这一行是具体的执行
    auto result = command.InvokeFunc(subcontext_);
    auto duration = t.duration();

    // Any action longer than 50ms will be warned to user as slow operation
    //失败、超时或者debug版本都需要打印结果
    if (!result.has_value() || duration > 50ms ||
        android::base::GetMinimumLogSeverity() <= android::base::DEBUG) {
        std::string trigger_name = BuildTriggersString();
        std::string cmd_str = command.BuildCommandString();

        LOG(INFO) << "Command '" << cmd_str << "' action=" << trigger_name << " (" << filename_
                  << ":" << command.line() << ") took " << duration.count() << "ms and "
                  << (result.ok() ? "succeeded" : "failed: " + result.error().message());
    }
}
```

接着会调用`Command::InvokeFunc`方法

```c++
Result<void> Command::InvokeFunc(Subcontext* subcontext) const {
    //从 /vendor 或 /oem 解析出来的rc文件都会走这里 
    //涉及到selinux权限问题，Google为了保证安全 
    //队对厂商定制的rc文件中的命令执行，以及由此启动的服务的权限都会有一定限制
    if (subcontext) {
        if (execute_in_subcontext_) {
            return subcontext->Execute(args_);
        }

        auto expanded_args = subcontext->ExpandArgs(args_);
        if (!expanded_args.ok()) {
            return expanded_args.error();
        }
        return RunBuiltinFunction(func_, *expanded_args, subcontext->context());
    }
    
    //系统原生的rc文件命令都会走这里
    return RunBuiltinFunction(func_, args_, kInitContext);
}
```

系统原生的rc文件命令都会走到`RunBuiltinFunction`方法中

```c++
Result<void> RunBuiltinFunction(const BuiltinFunction& function,
                                const std::vector<std::string>& args, const std::string& context) {
    auto builtin_arguments = BuiltinArguments(context);

    builtin_arguments.args.resize(args.size());
    builtin_arguments.args[0] = args[0];
    for (std::size_t i = 1; i < args.size(); ++i) {
        auto expanded_arg = ExpandProps(args[i]);
        if (!expanded_arg.ok()) {
            return expanded_arg.error();
        }
        builtin_arguments.args[i] = std::move(*expanded_arg);
    }

    return function(builtin_arguments);
}
```

这里的`function`是一个以`BuiltinArguments`为参数的`std::function`函数包装器模板，可以包装函数、函数指针、类成员函数指针或任意类型的函数对象，在Command对象new出来的时候构造函数就指定了这个func_，我们可以看一下`Action::AddCommand`方法：

```c++
Result<void> Action::AddCommand(std::vector<std::string>&& args, int line) {
    if (!function_map_) {
        return Error() << "no function map available";
    }

    //从function_map_中进行键值对查找
    auto map_result = function_map_->Find(args);
    if (!map_result.ok()) {
        return Error() << map_result.error();
    }

    commands_.emplace_back(map_result->function, map_result->run_in_subcontext, std::move(args),
                           line);
    return {};
}
```

可以看到，是通过rc文件中的字符串去一个`function_map_`常量中查找得到的，而这个`function_map_`是在哪赋值的呢，答案是在`SecondStageMain`函数中

```c++
int SecondStageMain(int argc, char** argv) {
    ...
    //设置commands指令所对应的函数map
    const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    Action::set_function_map(&function_map);
    ...
}
```

这个在前文代码中有提及，map的定义在`system/core/init/builtins.cpp`中

```c++
const BuiltinFunctionMap& GetBuiltinFunctionMap() {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    // clang-format off
    static const BuiltinFunctionMap builtin_functions = {
        {"bootchart",               {1,     1,    {false,  do_bootchart}}},
        {"chmod",                   {2,     2,    {true,   do_chmod}}},
        {"chown",                   {2,     3,    {true,   do_chown}}},
        {"class_reset",             {1,     1,    {false,  do_class_reset}}},
        {"class_reset_post_data",   {1,     1,    {false,  do_class_reset_post_data}}},
        {"class_restart",           {1,     1,    {false,  do_class_restart}}},
        {"class_start",             {1,     1,    {false,  do_class_start}}},
        {"class_start_post_data",   {1,     1,    {false,  do_class_start_post_data}}},
        {"class_stop",              {1,     1,    {false,  do_class_stop}}},
        {"copy",                    {2,     2,    {true,   do_copy}}},
        {"domainname",              {1,     1,    {true,   do_domainname}}},
        {"enable",                  {1,     1,    {false,  do_enable}}},
        {"exec",                    {1,     kMax, {false,  do_exec}}},
        {"exec_background",         {1,     kMax, {false,  do_exec_background}}},
        {"exec_start",              {1,     1,    {false,  do_exec_start}}},
        {"export",                  {2,     2,    {false,  do_export}}},
        {"hostname",                {1,     1,    {true,   do_hostname}}},
        {"ifup",                    {1,     1,    {true,   do_ifup}}},
        {"init_user0",              {0,     0,    {false,  do_init_user0}}},
        {"insmod",                  {1,     kMax, {true,   do_insmod}}},
        {"installkey",              {1,     1,    {false,  do_installkey}}},
        {"interface_restart",       {1,     1,    {false,  do_interface_restart}}},
        {"interface_start",         {1,     1,    {false,  do_interface_start}}},
        {"interface_stop",          {1,     1,    {false,  do_interface_stop}}},
        {"load_persist_props",      {0,     0,    {false,  do_load_persist_props}}},
        {"load_system_props",       {0,     0,    {false,  do_load_system_props}}},
        {"loglevel",                {1,     1,    {false,  do_loglevel}}},
        {"mark_post_data",          {0,     0,    {false,  do_mark_post_data}}},
        {"mkdir",                   {1,     6,    {true,   do_mkdir}}},
        // TODO: Do mount operations in vendor_init.
        // mount_all is currently too complex to run in vendor_init as it queues action triggers,
        // imports rc scripts, etc.  It should be simplified and run in vendor_init context.
        // mount and umount are run in the same context as mount_all for symmetry.
        {"mount_all",               {0,     kMax, {false,  do_mount_all}}},
        {"mount",                   {3,     kMax, {false,  do_mount}}},
        {"perform_apex_config",     {0,     0,    {false,  do_perform_apex_config}}},
        {"umount",                  {1,     1,    {false,  do_umount}}},
        {"umount_all",              {0,     1,    {false,  do_umount_all}}},
        {"update_linker_config",    {0,     0,    {false,  do_update_linker_config}}},
        {"readahead",               {1,     2,    {true,   do_readahead}}},
        {"remount_userdata",        {0,     0,    {false,  do_remount_userdata}}},
        {"restart",                 {1,     1,    {false,  do_restart}}},
        {"restorecon",              {1,     kMax, {true,   do_restorecon}}},
        {"restorecon_recursive",    {1,     kMax, {true,   do_restorecon_recursive}}},
        {"rm",                      {1,     1,    {true,   do_rm}}},
        {"rmdir",                   {1,     1,    {true,   do_rmdir}}},
        {"setprop",                 {2,     2,    {true,   do_setprop}}},
        {"setrlimit",               {3,     3,    {false,  do_setrlimit}}},
        {"start",                   {1,     1,    {false,  do_start}}},
        {"stop",                    {1,     1,    {false,  do_stop}}},
        {"swapon_all",              {0,     1,    {false,  do_swapon_all}}},
        {"enter_default_mount_ns",  {0,     0,    {false,  do_enter_default_mount_ns}}},
        {"symlink",                 {2,     2,    {true,   do_symlink}}},
        {"sysclktz",                {1,     1,    {false,  do_sysclktz}}},
        {"trigger",                 {1,     1,    {false,  do_trigger}}},
        {"verity_update_state",     {0,     0,    {false,  do_verity_update_state}}},
        {"wait",                    {1,     2,    {true,   do_wait}}},
        {"wait_for_prop",           {2,     2,    {false,  do_wait_for_prop}}},
        {"write",                   {2,     2,    {true,   do_write}}},
    };
    // clang-format on
    return builtin_functions;
}
```

### 启动服务

以下面一段rc脚本为例，我们看一下一个服务是怎么启动的

```
on zygote-start 
    start zygote
```

首先这是一个action，当init进程在死循环中执行到`ActionManager::ExecuteOneCommand`方法时，检查到这个action刚好符合`event_queue_`队首的`EventTrigger`，便会执行这个`action`下面的`commands`。`commands`怎么执行在上面已经分析过了，我们去`system/core/init/builtins.cpp`里的map中找key-value对应关系，发现start对应着`do_start`函数：

```c++
static Result<void> do_start(const BuiltinArguments& args) {
    Service* svc = ServiceList::GetInstance().FindService(args[1]);
    if (!svc) return Error() << "service " << args[1] << " not found";
    if (auto result = svc->Start(); !result.ok()) {
        return ErrorIgnoreEnoent() << "Could not start service: " << result.error();
    }
    return {};
}
```

`ServiceList`通过`args[1]`即定义的服务名去寻找之前解析好的service，并执行`Service::Start`方法：

```c++
Result<void> Service::Start() {
    ... 
    //上面基本上是一些检查和准备工作，这里先忽略

    pid_t pid = -1;
    //通过namespaces_.flags判断使用哪种方式创建进程
    if (namespaces_.flags) {
        pid = clone(nullptr, nullptr, namespaces_.flags | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }

    if (pid == 0) {
        //设置权限掩码
        umask(077);
        ...
        //内部调用execv函数启动文件
        if (!ExpandArgsAndExecv(args_, sigstop_)) {
            PLOG(ERROR) << "cannot execv('" << args_[0]
                        << "'). See the 'Debugging init' section of init's README.md for tips";
        }

        _exit(127);
    }

    if (pid < 0) {
        pid_ = 0;
        return ErrnoError() << "Failed to fork";
    }

    ...
    return {};
}
```

```c++
static bool ExpandArgsAndExecv(const std::vector<std::string>& args, bool sigstop) {
    std::vector<std::string> expanded_args;
    std::vector<char*> c_strings;

    expanded_args.resize(args.size());
    //将要执行的文件路径先加入容器
    c_strings.push_back(const_cast<char*>(args[0].data()));
    for (std::size_t i = 1; i < args.size(); ++i) {
        auto expanded_arg = ExpandProps(args[i]);
        if (!expanded_arg.ok()) {
            LOG(FATAL) << args[0] << ": cannot expand arguments': " << expanded_arg.error();
        }
        expanded_args[i] = *expanded_arg;
        c_strings.push_back(expanded_args[i].data());
    }
    c_strings.push_back(nullptr);

    if (sigstop) {
        kill(getpid(), SIGSTOP);
    }

    //调用execv函数，带参执行文件
    return execv(c_strings[0], c_strings.data()) == 0;
}
```

这里先`fork`（或`clone`）出了一个子进程，再在这个子进程中调用`execv`函数执行文件

到此为止，一个服务便被启动起来了

### 守护服务

当服务启动起来后，`init`进程也要负责服务的守护，为什么呢？

假设`zygote`进程挂了，那`zygote`进程下的所有子进程都可能会被杀，整个`Android`系统会出现大问题，那怎么办呢？得把`zygote`进程重启起来呀。`init`进程守护服务做的就是这些事，当接收到子进程退出信号，就会触发对应的函数进行处理，去根据这个进程所对应的服务，处理后事（重启等）

代码在这个位置：

```c++
int SecondStageMain(int argc, char** argv) {
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }

    InstallSignalFdHandler(&epoll);
    InstallInitNotifier(&epoll);
}
```

先创建出来一个epoll句柄，再用它去`InstallSignalFdHandler`装载信号handler：

```c++
static void InstallSignalFdHandler(Epoll* epoll) {
    // Applying SA_NOCLDSTOP to a defaulted SIGCHLD handler prevents the signalfd from receiving
    // SIGCHLD when a child process stops or continues (b/77867680#comment9).
    //设置SIGCHLD信号的处理方式
    const struct sigaction act { .sa_handler = SIG_DFL, .sa_flags = SA_NOCLDSTOP };
    sigaction(SIGCHLD, &act, nullptr);

    //在init进程中屏蔽SIGCHLD、SIGTERM信号
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGCHLD);

    if (!IsRebootCapable()) {
        // If init does not have the CAP_SYS_BOOT capability, it is running in a container.
        // In that case, receiving SIGTERM will cause the system to shut down.
        sigaddset(&mask, SIGTERM);
    }

    if (sigprocmask(SIG_BLOCK, &mask, nullptr) == -1) {
        PLOG(FATAL) << "failed to block signals";
    }

    // Register a handler to unblock signals in the child processes.
    //在子进程中取消SIGCHLD、SIGTERM信号屏蔽
    const int result = pthread_atfork(nullptr, nullptr, &UnblockSignals);
    if (result != 0) {
        LOG(FATAL) << "Failed to register a fork handler: " << strerror(result);
    }

    //创建用于接受信号的文件描述符
    signal_fd = signalfd(-1, &mask, SFD_CLOEXEC);
    if (signal_fd == -1) {
        PLOG(FATAL) << "failed to create signalfd";
    }

    //注册信号处理器
    if (auto result = epoll->RegisterHandler(signal_fd, HandleSignalFd); !result.ok()) {
        LOG(FATAL) << result.error();
    }
}
```

#### sigaction函数

先介绍一下`sigaction`函数，它是用来检查和设置一个信号的处理方式的

文档：<https://man7.org/linux/man-pages/man2/sigaction.2.html>

第一个参数`signum`，定义在`signal.h`中，用来指定信号的编号（需要设置哪个信号）

第二个参数`act`是一个结构体：

```c++
struct sigaction {
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;
    int        sa_flags;
    void     (*sa_restorer)(void);
};
```

其中，`sa_handler`表示信号的处理方式，`sa_flags`用来设置信号处理的其他相关操作

第三个参数`oldact`，如果不为`null`，会将此信号原来的处理方式保存进去

对应一下`InstallSignalFdHandler`里的调用，`.sa_handler = SIG_DFL`表示使用默认的信号处理，`.sa_flags = SA_NOCLDSTOP`当参数`signum`为`SIGCHLD`的时候生效，表示当子进程暂停时不会通知父进程

#### 信号集函数

接下来`InstallSignalFdHandler`函数调用了一些信号集函数

##### sigemptyset

原型：`int sigemptyset(sigset_t *set);`

文档：<https://man7.org/linux/man-pages/man3/sigemptyset.3p.html>

该函数的作用是将信号集初始化为空

##### sigaddset

原型：`int sigaddset(sigset_t *set, int signo);`

文档：<https://man7.org/linux/man-pages/man3/sigaddset.3p.html>

该函数的作用是把信号signo添加到信号集set中

##### sigpromask

原型：`int sigpromask(int how, const sigset_t *set, sigset_t *oldset);`

文档：<https://man7.org/linux/man-pages/man2/sigprocmask.2.html>

该函数可以根据参数指定的方法修改进程的信号屏蔽字

第一个参数`how`有3种取值：

- `SIG_BLOCK`：将set中的信号添加到信号屏蔽字中（不改变原有已存在信号屏蔽字，相当于用set中的信号与原有信号取并集设置）
- `SIG_UNBLOCK`：将set中的信号移除信号屏蔽字（相当于用set中的信号的补集与原有信号取交集设置）
- `SIG_SETMASK`：使用set中的信号直接代替原有信号屏蔽字中的信号

第二个参数`set`是一个信号集，怎么使用和参数how相关

第三个参数`oldset`，如果不为null，会将原有信号屏蔽字的信号集保存进去

为什么init进程要屏蔽这些信号呢？因为它后面会特殊处理这些信号

#### pthread_atfork

这也是一个Linux函数，用来注册fork的handlers

原型：`int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));`

调用这个函数后，当进程再调用fork时，内部创建子进程钱会先在父进程中调用`prepare`函数，创建子进程成功后，会在父进程中调用`parent`函数，子进程中调用`child`函数

对应到`InstallSignalFdHandler`里来，即当init进程fork出子进程后调用`UnblockSignals`函数

```c++
static void UnblockSignals() {
    const struct sigaction act { .sa_handler = SIG_DFL };
    sigaction(SIGCHLD, &act, nullptr);

    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGCHLD);
    sigaddset(&mask, SIGTERM);

    if (sigprocmask(SIG_UNBLOCK, &mask, nullptr) == -1) {
        PLOG(FATAL) << "failed to unblock signals for PID " << getpid();
    }
}

```

也就是，先在init进程中屏蔽了`SIGCHLD`、`SIGTERM`信号，再在子进程中解除了这两个信号的屏蔽

#### signalfd函数

同样也是Linux函数，用来创建用于接受信号的文件描述符

原型：`int signalfd(int fd, const sigset_t *mask, int flags);`

参数fd如果为-1，则该函数会创建一个新的文件描述符与mask信号集相关联，如果不为-1，则该函数会用mask替换之前与这个fd相关联的信号集

flags：

- SFD_NONBLOCK：给新打开的文件描述符设置`O_NONBLOCK`标志，非阻塞I/O模式
- SFD_CLOEXEC：给新打开的文件描述符设置`O_CLOEXEC`标志，当exec函数执行成功后，会自动关闭这个文件描述符

对应到`InstallSignalFdHandler`中，它创建了一个用于接受`SIGCHLD`、`SIGTERM`信号的文件描述符。回忆一下之前对启动服务的分析，是先调用fork创建进程，在exec执行文件，将flags设置为`SFD_CLOEXEC`，这样就可以保证在子进程中关闭由fork得到的接收信号的文件描述符

注册信号处理器

最后调用`Epoll::RegisterHandler`方法注册处理器，内部调用了`epoll_ctl`函数，感兴趣可以自己看一下，文档：<https://man7.org/linux/man-pages/man2/epoll_ctl.2.html>

这样，当init进程接收到`SIGCHLD`、`SIGTERM`信号时便会调用`HandleSignalFd`方法：

```c++
static void HandleSignalFd() {
    signalfd_siginfo siginfo;
    //从信号集文件描述符中读取信息
    ssize_t bytes_read = TEMP_FAILURE_RETRY(read(signal_fd, &siginfo, sizeof(siginfo)));
    if (bytes_read != sizeof(siginfo)) {
        PLOG(ERROR) << "Failed to read siginfo from signal_fd";
        return;
    }

    switch (siginfo.ssi_signo) {
        case SIGCHLD:
            ReapAnyOutstandingChildren();
            break;
        case SIGTERM:
            HandleSigtermSignal(siginfo);
            break;
        default:
            PLOG(ERROR) << "signal_fd: received unexpected signal " << siginfo.ssi_signo;
            break;
    }
}
```

我们这里主要看`SIGCHLD`，当子进程退出，init进程便会捕获到`SIGCHLD`，执行`ReapAnyOutstandingChildren`方法：

```c++
void ReapAnyOutstandingChildren() {
    while (ReapOneProcess() != 0) {
    }
}
```

```c++
static pid_t ReapOneProcess() {
    siginfo_t siginfo = {};
    // This returns a zombie pid or informs us that there are no zombies left to be reaped.
    // It does NOT reap the pid; that is done below.
    //获取一个已经退出的子进程，但暂时先不销毁
    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT)) != 0) {
        PLOG(ERROR) << "waitid failed";
        return 0;
    }

    auto pid = siginfo.si_pid;
    if (pid == 0) return 0;

    // At this point we know we have a zombie pid, so we use this scopeguard to reap the pid
    // whenever the function returns from this point forward.
    // We do NOT want to reap the zombie earlier as in Service::Reap(), we kill(-pid, ...) and we
    // want the pid to remain valid throughout that (and potentially future) usages.
    //最后，销毁这个子进程
    auto reaper = make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });

    std::string name;
    std::string wait_string;
    Service* service = nullptr;

    if (SubcontextChildReap(pid)) {
        name = "Subcontext";
    } else {
        //通过pid获得service
        service = ServiceList::GetInstance().FindService(pid, &Service::pid);
        ...
    }
    ...
    if (!service) return pid;

    //处理service后事
    service->Reap(siginfo);

    if (service->flags() & SVC_TEMPORARY) {
        ServiceList::GetInstance().RemoveService(*service);
    }

    return pid;
}
```

##### waitid函数

Linux函数，用于等待一个子进程状态的改变

原型：`int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);`

文档：<https://man7.org/linux/man-pages/man3/waitid.3p.html>

第一个参数`idtype`：

- P_PID：等待的子进程的pid必须和参数id匹配
- P_GID：等待的子进程的组id必须和参数id匹配
- P_ADD：等待所有子进程，此时，参数id被忽略

这个函数会将执行的结果保存在第三个参数infop中

`options`：

- WCONTINUED：等待那些由SIGCONT重新启动的子进程
- WEXITED：等待那些已经退出的子进程
- WSTOPPED：等待那些被信号暂停的子进程
- WNOHANG：非阻塞等待
- WNOWAIT：保持返回的子进程处于可等待状态（后续可以再对这个子进程进行wait）

回到`ReapOneProcess`函数中来，它先调用`waitid`函数，获得一个状态发生改变的子进程（options设置了`WEXITED`，即已退出的子进程），使用了`WNOWAIT`参数，也就是暂时先不销毁子进程，使用非阻塞的方式获取

ScopeGuard

`ScopeGuard`的意思是，出作用域后，自动执行某段代码

函数中那段`make_scope_guard`的意思是，当这个函数执行完后，使用`waitpid`函数销毁子进程

之后会从`ServiceList`中通过pid去查找service，查到后调用`Service::Reap`处理后事

```c++
void Service::Reap(const siginfo_t& siginfo) {
    //当service的参数没有oneshot或者restart时，kill整个进程组
    if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
        KillProcessGroup(SIGKILL, false);
    } else {
        // Legacy behavior from ~2007 until Android R: this else branch did not exist and we did not
        // kill the process group in this case.
        if (SelinuxGetVendorAndroidVersion() >= __ANDROID_API_R__) {
            // The new behavior in Android R is to kill these process groups in all cases.  The
            // 'true' parameter instructions KillProcessGroup() to report a warning message where it
            // detects a difference in behavior has occurred.
            KillProcessGroup(SIGKILL, true);
        }
    }

    // Remove any socket resources we may have created.
    //移除已创建的sockets
    for (const auto& socket : sockets_) {
        auto path = ANDROID_SOCKET_DIR "/" + socket.name;
        unlink(path.c_str());
    }
    //执行回调
    for (const auto& f : reap_callbacks_) {
        f(siginfo);
    }

    //如果进程接收信号异常或被终止的状态异常，并且包含reboot_on_failure标志，重启系统
    if ((siginfo.si_code != CLD_EXITED || siginfo.si_status != 0) && on_failure_reboot_target_) {
        LOG(ERROR) << "Service with 'reboot_on_failure' option failed, shutting down system.";
        trigger_shutdown(*on_failure_reboot_target_);
    }

    //当service参数为exec时，释放相应服务资源
    if (flags_ & SVC_EXEC) UnSetExec();

    if (flags_ & SVC_TEMPORARY) return;

    pid_ = 0;
    flags_ &= (~SVC_RUNNING);
    start_order_ = 0;

    // Oneshot processes go into the disabled state on exit,
    // except when manually restarted.
    //当service参数有oneshot，没有restart和reset时，将service状态置为disable
    if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART) && !(flags_ & SVC_RESET)) {
        flags_ |= SVC_DISABLED;
    }

    // Disabled and reset processes do not get restarted automatically.
    //禁用和重置的服务，都不能自动重启
    if (flags_ & (SVC_DISABLED | SVC_RESET))  {
        NotifyStateChange("stopped");
        return;
    }
    ...
    //将标志置为重启中
    flags_ &= (~SVC_RESTART);
    flags_ |= SVC_RESTARTING;

    // Execute all onrestart commands for this service.
    //执行该service下的所有onrestart命令
    onrestart_.ExecuteAllCommands();

    NotifyStateChange("restarting");
    return;
}
```

service相关的参数可以去`system/core/init/README.md`中自行查看

这个函数检查了一堆service的标志和状态，判断如何处理这个service，如果需要重启，则调用`onrestart_.ExecuteAllCommands()`执行该service下的所有`onrestart`命令，具体的执行过程之前在启动服务那边已经分析过了，这里就不再往下看了

# 总结

至此，整个init进程的启动过程最重要的部分基本都已分析完成，我也是一边从网上搜集资料一边对照着源码磕磕绊绊看过来的，有什么错误或者遗漏的部分欢迎指正，谢谢～
