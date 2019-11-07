---
layout: post
title:  "Android O系统启动流程--zygote篇"
date:   2019-11-07 00:00:00
catalog:  true
tags:
    - android
    - 系统启动
    - zygote
---

```
system/core/init
 - init.cpp
 - init.rc
 - service.cpp
 - builtins.cpp
 - frameworks/base/cmds/app_process/app_main.cpp
 - frameworks/base/core/jni/AndroidRuntime.cpp
 - frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
 - ZygoteServer.java
 - Zygote.java
```



## 1 概述

在Android系统中，JavaVM(Java虚拟机)、应用程序进程以及运行系统的关键服务的SystemServer进程都是由Zygote进程来创建的，我们也将它称为孵化器。它通过fock(复制进程)的形式来创建应用程序进程和SystemServer进程，由于Zygote进程在启动时会创建JavaVM，因此通过fock而创建的应用程序进程和SystemServer进程可以在内部获取一个JavaVM的实例拷贝。 

## 2 init.zygoteXX.rc

从之前分析的init篇中我们知道，在不同的平台（32、64及64_32）上，init.rc将包含不同的zygote.rc文件。在system/core/rootdir目录下，有init.zygote32_64.rc、init.zyote64.rc、 init.zyote32.rc、init.zygote64_32.rc。

- init.zygote32.rc：zygote 进程对应的执行程序是 app_process (纯 32bit 模式)
- init.zygote64.rc：zygote 进程对应的执行程序是 app_process64 (纯 64bit 模式)
- init.zygote32_64.rc：启动两个 zygote 进程 (名为 zygote 和 zygote_secondary)，对应的执行程序分别是 app_process32 (主模式)、app_process64
- init.zygote64_32.rc：启动两个 zygote 进程 (名为 zygote 和 zygote_secondary)，对应的执行程序分别是 app_process64 (主模式)、app_process32

定义这么多种情况主要是因为Android 5.0以后开始支持64位程序，为了兼容32位和64位才这样定义。不同的zygote.rc内容大致相同，主要区别体现在启动的是32位，还是64位的进程。init.zygote32_64.rc和init.zygote64_32.rc会启动两个进程，且存在主次之分。
这里拿32位处理器为例，init.zygote64_32.rc的代码如下所示：

```c++
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

## 3 Zygote触发

在分析init进程时，我们知道init进程启动后，会解析init.rc文件，然后创建和加载service字段指定的进程。zygote进程就是以这种方式，被init进程加载的。 

[->init.rc]

```c++
import /init.${ro.zygote}.rc// ${ro.zygote}由厂商定义，与平台相关
...
on late-init
    ...
    # Now we can start zygote for devices with file based encryption
    trigger zygote-start
...
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=unsupported
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary
```

 关键字start对应的处理函数为do_start，定义在builtins.cpp中 。

[->builtins.cpp]

```c++
static int do_start(const std::vector<std::string>& args) {
    //找到zygote service对应信息
    Service* svc = ServiceManager::GetInstance().FindServiceByName(args[1]);
    if (!svc) {
        LOG(ERROR) << "do_start: Service " << args[1] << " not found";
        return -1;
    }
    //启动对应的进程
    if (!svc->Start())
        return -1;
    return 0;
}
```

 最后，我们来看看service.cpp中定义Start函数： 

[->service.cpp]

```c++
bool Service::Start() {
    ...
    pid_t pid = -1;
    if (namespace_flags_) {
        pid = clone(nullptr, nullptr, namespace_flags_ | SIGCHLD, nullptr);
    } else {
        //从init进程中，fork出zygote进程
        pid = fork();
    }

    if (pid == 0) {
        ...
        //当前fork的子进程替换为app_process
        if (execve(strs[0], (char**) &strs[0], (char**) ENV) < 0) {
            PLOG(ERROR) << "cannot execve('" << strs[0] << "')";
        }
        ...
    }
    ...
}
```

 Start函数主要是fork出一个新进程，然后执行service对应的二进制文件，并将参数传递进去。init.zygote32rc对应的可执行程序是/system/bin/app_process，对应的代码是 app_main.cpp。 

## 4 app_process的main函数  

[->app_main.cpp]

```c++
int main(int argc, char* const argv[])
{
    ...
    //AppRuntime定义于app_main.cpp中，继承自AndroidRuntime
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
    ...
    // Parse runtime arguments.  Stop at first unrecognized option.
    // 通过app_main可以启动zygote、system-server及普通apk进程
    // 这个可以通过init.rc来配置
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    // app_process的名称改为zygote
    String8 niceName;
    // 启动apk进程时，对应的类名
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    //开始解析输入参数
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            //init.zygote32.rc中定义了该字段，表示启动zygote进程
            zygote = true;
            //记录app_process进程名的nice name，即zygote32(平台相关)
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            //init.zygote.rc中定义了该字段， 启动zygote后会启动system-server
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            //表示启动制定进程
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            //可以自己指定进程名
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            //与--application配置，启动指定的类
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    //准备参数
    Vector<String8> args;
    if (!className.isEmpty()) {
        //启动普通进程
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        //创建dalvikCache所需的目录，并定义权限
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            //增加参数, 默认启动zygote后，就会启动system server
            args.add(String8("start-system-server"));
        }
        //获取平台对应的abi信息
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
        //参数需要制定abi
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            //将main函数未处理的参数都递交给zygote main处理
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        //将app_process的进程名，替换为nice name
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        //调用Runtime的start函数, 启动ZygoteInit
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        //启动zygote没有进入这个分支
        //但这个分支说明，通过配置init.rc文件，其实是可以不通过zygote来启动一个进程
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

## 5  **AndroidRuntime的start函数**  

由于AppRuntime继承自AndroidRuntime，且没有重写start方法， 因此zygote的流程进入到了AndroidRuntime.cpp。
接下来，我们来看看AndroidRuntime的start函数的流程。

### 5.1  **创建Java虚拟机** 

[->AndroidRuntime.cpp]

```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);

    JNIEnv* env;
    //创建虚拟机，其中大多数参数由系统属性决定
    //最终，startVm利用JNI_CreateJavaVM创建出虚拟机
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    //回调AppRuntime的onVmCreated函数
    //对于zygote进程的启动流程而言，无实际操作
    onVmCreated(env);
    ...
}
```

### 5.2  **注册JNI函数**  

 初始化JVM后，接下来就会调用startReg函数。 

```c++
/*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
```

```c++
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    //定义Android创建线程的func
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}
```

从上述代码可以看出，startReg函数中主要是通过register_jni_procs来注册JNI函数。  startReg首先是设置了Android创建线程的处理函数，然后创建了一个200容量的局部引用作用域，用于确保不会出现`OutOfMemoryException`，最后就是调用`register_jni_procs`进行JNI注册。 
其中，gRegJNI是一个全局数组，该数组的定义类似于： 

```c++
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_android_util_SeempLog),
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit),
    ...
```

 REG_JNI对应的宏定义及RegJNIRec结构体的定义为： 

```c++
#ifdef NDEBUG
    #define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };
#else
    #define REG_JNI(name)      { name, #name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
        const char* mName;
    };
#endif
```

根据宏定义可以看出，宏REG_JNI将得到函数名。定义RegJNIRec数组时，函数名被赋值给RegJNIRec结构体， 
于是每个函数名被强行转换为函数指针。 因此，register_jni_procs的参数就是一个函数指针数组， 数组的大小和JNIEnv。

```c++
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
```

结合前面的分析，容易知道register_jni_procs函数， 实际上就是调用函数指针对应的函数，以进行实际的JNI函数注册。 

举个例子：

```c++
int register_android_util_SeempLog(JNIEnv* env)
{
    jclass clazz = env->FindClass("android/util/SeempLog");
    if (clazz == NULL) {
        return -1;
    }

    return AndroidRuntime::registerNativeMethods(env, "android/util/SeempLog", gMethods,
            NELEM(gMethods));
}
```

 可以看到，这实际上是自己定义JNI函数并进行动态注册的标准写法。 

### 5.3  **反射启动ZygoteInit**  

[-> AndroidRuntime.cpp]

继续往下看start函数。

```c++
/*
* Start VM.  This thread becomes the main thread of the VM, and will
* not return until the VM exits.
*/
//替换string为实际路径
//例如：将"com.android.internal.os.ZygoteInit"
//替换为"com/android/internal/os/ZygoteInit"
char* slashClassName = toSlashClassName(className);

//找到class文件
jclass startClass = env->FindClass(slashClassName);
if (startClass == NULL) {
    ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    /* keep going */
} else {
    //通过反射找到ZygoteInit的main函数
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
        ALOGE("JavaVM unable to find main() in '%s'\n", className);
        /* keep going */
    } else {
        //调用ZygoteInit的main函数
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
    }
    ...
}
...
```

可以看到，在AndroidRuntime的最后，将通过反射调用ZygoteInit的main函数。 至此，zygote进程进入了java世界。  

## 6 进入Java层

[->ZygoteInit.java]

```java
public static void main(String argv[]) {
    //创建ZygoteServer对象
    ZygoteServer zygoteServer = new ZygoteServer();

    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    // 调用native函数，确保当前没有其它线程在运行
    ZygoteHooks.startZygoteNoThreadCreation(); 

    // Zygote goes into its own process group.
    try {
        Os.setpgid(0, 0);
    } catch (ErrnoException ex) {
        throw new RuntimeException("Failed to setpgid(0,0)", ex);
    }

    try {
        ...
        // enable DDMS
        RuntimeInit.enableDdms();
        // Start profiling the zygote initialization.
        // 开始性能统计
        SamplingProfilerIntegration.start();

        boolean startSystemServer = false;
        String socketName = "zygote";
        String abiList = null;
        boolean enableLazyPreload = false;
        //解析参数，得到上述变量的值
        for (int i = 1; i < argv.length; i++) {
            ...
        }

        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }

        //注册server socket
        zygoteServer.registerZygoteSocket(socketName);

        // In some configurations, we avoid preloading resources and classes eagerly.
        // In such cases, we will preload things prior to our first fork.
        if (!enableLazyPreload) {
            ...
            //默认情况，预加载信息
            preload(bootTimingsTraceLog);
        ...
        } else {
            //如注释，延迟预加载
            //变更Zygote进程优先级为NORMAL级别
            //第一次fork时才会preload
            Zygote.resetNicePriority();
        }

        // Finish profiling the zygote initialization.
        // 结束对zygote的性能统计
        // 实际上是想统计预加载的性能
        SamplingProfilerIntegration.writeZygoteSnapshot();

        // Do an initial gc to clean up after startup
        ...
        //如果预加载了，很有必要GC一波
        gcAndFinalize();
        ...

        //以下均是安全相关的内容

        // Zygote process unmounts root storage spaces.
        // unmount root storage spaces，
        // 那么其它进程就无法访问了
        Zygote.nativeUnmountStorageOnInit();

        // Set seccomp policy
        // 加载seccomp的过滤规则
        // 所有 Android 软件都使用系统调用（简称为 syscall）与 Linux 内核进行通信
        // 内核提供许多特定于设备和SOC的系统调用，让用户空间进程（包括应用）可以直接与内核进行交互
        // 不过，其中许多系统调用Android未予使用或未予正式支持
        // 通过seccomp，Android可使应用软件无法访问未使用的内核系统调用
        // 由于应用无法访问这些系统调用，因此，它们不会被潜在的有害应用利用
        // 该过滤器安装到zygote进程中，由于所有Android应用均衍生自该进程
        // 因而会影响到所有应用
        Seccomp.setPolicy();

        //允许有其它线程了
        ZygoteHooks.stopZygoteNoThreadCreation();

        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }
        ...
        //zygote进程进入无限循环，处理请求
        zygoteServer.runSelectLoop(abiList);

        zygoteServer.closeServerSocket();
    } catch (Zygote.MethodAndArgsCaller caller) {
        //通过反射调用新进程函数的地方
        //后续介绍新进程启动时，再介绍
        caller.run();
    } catch (Throwable ex) {
        ...
        zygoteServer.closeServerSocket();
        throw ex;
    }
}
```

上面是ZygoteInit的main函数的主干部分，除了安全相关的内容外， 最主要的工作就是注册server socket、预加载、启动system server 及进入无限循环处理请求消息。 
接下来，我们进一步分析这几个步骤对应的工作。

### 6.1  **创建server socket**  

[->ZygoteServer.java]

```java
/**
 * Registers a server socket for zygote command connections
 *
 * @throws RuntimeException when open fails
 */
void registerServerSocket(String socketName) {
        if (mServerSocket == null) {
            int fileDesc;
            //此处的socket name，就是zygote
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                //在init.zygote.rc被加载时，指定了名为zygote的socket
                //在进程被创建时，就会创建对应的文件描述符，并加入到环境变量中
                //因此，此时可以取出对应的环境变量
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                //获取zygote socket的文件描述符
                fd.setInt$(fileDesc);
                //将socket包装成一个server socket
                mServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
}
```

### 6.2  **预加载** 

[->ZygoteInit.java]

```java
static void preload(TimingsTraceLog bootTimingsTraceLog) {
        Log.d(TAG, "begin preload");
        bootTimingsTraceLog.traceBegin("BeginIcuCachePinning");
        //Pin ICU Data, 获取字符集转换资源等
        beginIcuCachePinning();
        bootTimingsTraceLog.traceEnd(); // BeginIcuCachePinning
        bootTimingsTraceLog.traceBegin("PreloadClasses");
        //读取文件system/etc/preloaded-classes，然后通过反射加载对应的类
        //一般由厂商来定义，有时需要加载数千个类，启动慢的原因之一
        preloadClasses();
        bootTimingsTraceLog.traceEnd(); // PreloadClasses
        bootTimingsTraceLog.traceBegin("PreloadResources");
        //负责加载一些常用的系统资源
        preloadResources();
        bootTimingsTraceLog.traceEnd(); // PreloadResources
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadAppProcessHALs");
        nativePreloadAppProcessHALs();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadOpenGL");
        //图形相关的
        preloadOpenGL();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        //一些必要库
        preloadSharedLibraries();
        //语言相关的字符信息
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        endIcuCachePinning();
        //安全相关的
        warmUpJcaProviders();
        Log.d(TAG, "end preload");

        sPreloadComplete = true;
    }
```

为了让系统实际运行时更加流畅，在zygote启动时候，调用preload函数进行了一些预加载操作。 
Android 通过zygote fork的方式创建子进程。zygote进程预加载这些类和资源，在fork子进程时，仅需要做一个复制即可。 这样可以节约子进程的启动时间。 同时，根据fork的copy-on-write机制可知，有些类如果不做改变，甚至都不用复制， 子进程可以和父进程共享这部分数据，从而省去不少内存的占用。

### 6.3  **启动SystemServer进程**  

[->->ZygoteInit.java]

```java
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        //准备capabilities参数
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
            OsConstants.CAP_WAKE_ALARM
        );
        /* Containers run without this capability, so avoid setting it in that case */
        if (!SystemProperties.getBoolean(PROPERTY_RUNNING_IN_CONTAINER, false)) {
            capabilities |= posixCapabilitiesAsBits(OsConstants.CAP_BLOCK_SUSPEND);
        }
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            //将上面准备的参数，按照ZygoteConnection的风格进行封装
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            //处理32_64和64_32的情况
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            // fork时会copy socket，system server需要主动关闭
            zygoteServer.closeServerSocket();
            return handleSystemServerProcess(parsedArgs);
        }

        return null;
}
```

上面的代码显示，调用`forkSystemServer`来进行启动SystemServer进程。

[->Zygote.java]

```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        // Resets nice priority for zygote process.
        resetNicePriority();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        // Enable tracing as soon as we enter the system_server.
        if (pid == 0) {
            Trace.setTracingEnabled(true, debugFlags);
        }
        VM_HOOKS.postForkCommon();
        return pid;
}
```

最后是通过调用JNI函数nativeForkSystemServer来启动SystemServer进程的。

### 6.4  **处理请求信息** 

回到ZygoteInit.java的main方法， 创建出SystemServer进程后，zygote进程调用ZygoteServer中的函数`runSelectLoop`。

[->ZygoteServer.java]

```java
Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        //首先将server socket加入到fds
        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
             //每次循环，都重新创建需要监听的pollFds
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                //关注事件到来
                pollFds[i].events = (short) POLLIN;
            }
            try {
                //等待事件到来
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                //server socket最先加入fds， 因此这里是server socket收到数据
                if (i == 0) {
                    //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                    //则创建ZygoteConnection对象,并添加到fds。
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    //加入到peers和fds, 即下一次也开始监听
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        //其它通信连接收到数据，runOnce执行对应命令
                        final Runnable command = connection.processOneCommand(this);

                        if (mIsForkChild) {
                            // We're in the child. We should always have a command to run at this
                            // stage if processOneCommand hasn't called "exec".
                            if (command == null) {
                                throw new IllegalStateException("command == null");
                            }

                            return command;
                        } else {
                            // We're in the server - we should never have any commands to run.
                            if (command != null) {
                                throw new IllegalStateException("command != null");
                            }

                            // We don't know whether the remote side of the socket was closed or
                            // not until we attempt to read from it from processOneCommand. This shows up as
                            // a regular POLLIN event in our regular processing loop.
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                //对应通信连接不再需要执行其它命令，关闭并移除，即下一次不再监听
                                peers.remove(i);
                                fds.remove(i);
                             }
                            ...
}
```

 Zygote采用高效的I/O多路复用机制，保证在没有客户端连接请求或数据处理时休眠，否则响应客户端的请求。 

## 7 总结

 Zygote启动过程的调用流程图： 

![restart-service](/images/startup/zygote_start.PNG)

1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
6. zygote完毕大部分工作，接下来再通过forkSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。