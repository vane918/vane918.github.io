---
layout: post
title:  "Android O系统启动流程--SystemServer篇一"
date:   2019-11-09 00:00:00
catalog:  true
tags:
    - android
    - 系统启动
    - SystemServer
---



>  - frameworks/base/core/jni/
>  -  \- com_android_internal_os_Zygote.cpp 
>  -  \- AndroidRuntime.cpp
>  - frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
>  -  \-ZygoteServer.java
>  -  \-Zygote.java



## 1 概述

SystemServer由Zygote fork生成的，进程名为`system_server`，该进程承载着framework的核心服务。 上篇

[Android O系统启动流程--zygote篇](http://vanelst.site/2019/11/07/android-startup-zygote/)讲到Zygote启动过程中会调用forkSystemServer()，接下来分析SystemServer是如何启动的。

## 2 forkSystemServer

[->->ZygoteInit.java]

```java
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        ..
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

由于SystemServer是复制Zygote的进程，因此也会包含Zygote的socket，该socket是服务端socket，对于SystemServer没有其他作用，需要先将其关闭；通过handleSystemServerProcess开启SystemServer进程。

## 3 Zygote的forkSystemServer

```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        // Resets nice priority for zygote process.
        resetNicePriority();
        // 调用native方法fork system_server
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

nativeForkSystemServer()方法在AndroidRuntime.cpp中注册的，调用com_android_internal_os_Zygote.cpp中的register_com_android_internal_os_Zygote()方法建立native方法的映射关系，所以接下来进入如下方法。 

```c++
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  //fork子进程
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL, NULL);
  if (pid > 0) {
      // zygote进程，检测system_server进程是否创建
      // The zygote process checks whether the child process has died or not.
      ALOGI("System server process %d has been created", pid);
      gSystemServerPid = pid;
      // There is a slight window that the system server process has crashed
      // but it went unnoticed because we haven't published its pid yet. So
      // we recheck here just to make sure that all is well.
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          //当system_server进程死亡后，重启zygote进程
          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
      }

      // Assign system_server to the correct memory cgroup.
      if (!WriteStringToFile(StringPrintf("%d", pid), "/dev/memcg/system/tasks")) {
        ALOGE("couldn't write %d to /dev/memcg/system/tasks", pid);
      }
  }
  return pid;
}
```

当system_server进程创建失败时，将会重启zygote进程。这里需要注意，对于Android 5.0以上系统，有两个zygote进程，分别是zygote、zygote64两个进程，system_server的父进程，一般来说32位系统其父进程是zygote32进程。

1. 当kill system_server进程后，只重启zygote32和system_server，不重启zygote;
2. 当kill zygote32进程后，只重启zygote32和system_server，也不重启zygote；
3. 当kill zygote进程，则重启zygote、zygote32以及system_server。

5 

```cpp
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jintArray fdsToIgnore,
                                     jstring instructionSet, jstring dataDir) {
  SetSigChldHandler();//设置子进程的signal信号处理函数
  ...
  //fork子进程
  pid_t pid = fork();

  if (pid == 0) {
    //进入子进程
    PreApplicationInit();

    // Clean up any descriptors which must be closed immediately
    //关闭并清除文件描述符
    DetachDescriptors(env, fdsToClose);
    ...
    //对于非system_server子进程，则创建进程组
    if (!is_system_server) {
        int rc = createProcessGroup(uid, getpid());
        if (rc != 0) {
            if (rc == -EROFS) {
                ALOGW("createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?");
            } else {
                ALOGE("createProcessGroup(%d, %d) failed: %s", uid, pid, strerror(-rc));
            }
        }
    }
    //设置设置group
    SetGids(env, javaGids);
    //设置资源limit
    SetRLimits(env, javaRlimits);
    ...
    //设置调度策略
    SetSchedulerPolicy(env);
    ...
    //selinux上下文
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
    if (rc == -1) {
      ALOGE("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
            is_system_server, se_info_c_str, se_name_c_str);
      RuntimeAbort(env, __LINE__, "selinux_android_setcontext failed");
    }

    // Make it easier to debug audit logs by setting the main thread's name to the
    // nice name rather than "app_process".
    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_info_c_str != NULL) {
      //设置线程名为system_server，方便调试
      SetThreadName(se_name_c_str);
    }

    delete se_info;
    delete se_name;
    //设置子进程的signal信号处理函数为默认函数
    UnsetSigChldHandler();
    //等价于调用zygote.callPostForkChildHooks()
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server, instructionSet);
    if (env->ExceptionCheck()) {
      RuntimeAbort(env, __LINE__, "Error calling post fork hooks.");
    }
  } else if (pid > 0) {
    // the parent process
    //进入父进程，即zygote进程
    // We blocked SIGCHLD prior to a fork, we unblock it here.
    if (sigprocmask(SIG_UNBLOCK, &sigchld, nullptr) == -1) {
      ALOGE("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno));
      RuntimeAbort(env, __LINE__, "Call to sigprocmask(SIG_UNBLOCK, { SIGCHLD }) failed.");
    }
  }
  return pid;
}
```

fork()创建新进程，采用copy on write方式，这是linux创建进程的标准方法，会有两次return,对于pid==0为子进程的返回，对于pid>0为父进程的返回。 到此system_server进程已完成了创建的所有工作，接下来开始了system_server进程的真正工作。新创建出来的system_server进程便进入handleSystemServerProcess()方法。 

## 5 handleSystemServerProcess

 [–>ZygoteInit.java] 

```java
private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);
        //设置当前进程名为"system_server"
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            //执行dex优化操作
            performSystemServerDexOpt(systemServerClasspath);
        ...
        if (parsedArgs.invokeWith != null) {
            String[] args = parsedArgs.remainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(args, 0, amendedArgs, 2, args.length);
                args = amendedArgs;
            }
            //启动应用进程
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);

            throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                // 创建类加载器，并赋予当前线程
                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);

                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
            //system_server故进入此分支，初始化zygoteInit
            return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
}
```

## 6 performSystemServerDexOpt

 [–>ZygoteInit.java] 

```java
private static void performSystemServerDexOpt(String classPath) {
    final String[] classPathElements = classPath.split(":");
    final IInstalld installd = IInstalld.Stub
                .asInterface(ServiceManager.getService("installd"));
            ...
            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                final String packageName = "*";
                final String outputPath = null;
                final int dexFlags = 0;
                final String compilerFilter = systemServerFilter;
                final String uuid = StorageManager.UUID_PRIVATE_INTERNAL;
                final String seInfo = null;
                final String classLoaderContext =
                        getSystemServerClassLoaderContext(classPathForElement);
                try {
                    //以system权限，执行dex文件优化
                    installd.dexopt(classPathElement, Process.SYSTEM_UID, packageName,
                            instructionSet, dexoptNeeded, outputPath, dexFlags, compilerFilter,
                            uuid, classLoaderContext, seInfo, false /* downgrade */);
                } catch (RemoteException | ServiceSpecificException e) {
                    // Ignore (but log), we need this on the classpath for fallback mode.
                    Log.w(TAG, "Failed compiling classpath element for system server: "
                            + classPathElement, e);
                }
            }

            classPathForElement = encodeSystemServerClassPath(
                    classPathForElement, classPathElement);
        }
}
```

将classPath字符串中的apk，分别进行dex优化操作。真正执行优化工作通过socket通信将相应的命令参数，发送给installd来完成。

##  7 zygoteInit

 [–>RuntimeInit.java] 

handleSystemServerProcess函数最后会执行zygoteInit。

```java
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        if (RuntimeInit.DEBUG) {
            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();
        //通用的一些初始化
        RuntimeInit.commonInit();
        //启动Binder线程池
        ZygoteInit.nativeZygoteInit();
        //执行SystemServer的main方法
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```

看下applicationInit是如何调用到SystemServer.java的main方法的。

## 8 applicationInit

 [–>RuntimeInit.java] 

```java
protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args = new Arguments(argv);

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Remaining arguments are passed to the start class's static main
        return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

跟进findStaticMain方法。

## 9 findStaticMain

```java
private static Runnable findStaticMain(String className, String[] argv,
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
            //通过反射，获取SystemServer类的main方法
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

 这里传入的className就是`com.android.server.SystemServer` ，最后传入到MethodAndArgsCaller中通过反射启动SystemServer类的main方法。跟进MethodAndArgsCaller。

## 10 MethodAndArgsCaller

```java
static class MethodAndArgsCaller implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
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

最后 通过mMethod.invoke完成了从Zygote经过RuntimeInit最后完成SystemServer的main方法的运行。

