---
layout: post
title:  "Android O系统启动流程--ActivityManagerService篇"
date:   2019-11-12 00:00:00
catalog:  true
tags:
    - android
    - 系统启动
    - SystemServer
    - AMS
---



>  - frameworks/base/services/java/com/android/server/SystemServer.java
>    - SystemServiceManager.java
>  - frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
>  - frameworks/base/core/java/android/app/ActivityThread.java
>    - ContextImpl.java
>    - LoadedApk.java

## 1 概述

ActivityManagerService（以下简称为 AMS）是 Android 中最核心的系统服务之一，AMS 
最重要的功能有两个：
1. 对应用程序进程的管理：应用程序进程的创建、销毁和优先级的调整
2. 对应用程序进程中的四大组件进行管理

以下是对AMS的启动流程进行的分析。

## 2  AMS启动过程

由[Android O系列启动流程--SystemServer篇二](http://vanelst.site/2019/11/11/android-startup-SystemServer2/)可知，AMS是在SystemServer.java的startBootstrapServices()方法中启动的，如下所示：

```java
private void startBootstrapServices() {
        ...
        // Activity manager runs the show.
        traceBeginAndSlog("StartActivityManager");
        //启动AMS服务
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        //设置AMS的系统服务管理器
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        //设置AMS的APP安装器
        mActivityManagerService.setInstaller(installer);
        traceEnd();
        ...
        // //初始化AMS相关的PMS
        mActivityManagerService.initPowerManagement();
        ...
        //设置SystemServer
        mActivityManagerService.setSystemProcess();
}
```

![AMS-startup](/images/startup/AMS-startup.png)

### 2.1 启动AMS服务

由上面的代码可知，mActivityManagerService对象是有ActivityManagerService.java的内部类Lifecycle创建的。代码如下：

```java
public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        @Override
        public void onCleanupUser(int userId) {
            mService.mBatteryStatsService.onCleanupUser(userId);
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }
```

 通过 ActivityManagerService.Lifecycle 这个类，如上所示，ActivityManagerService.Lifecycle 类很简单，继承 SystemService，是一个 ActivityManagerService 的包装类。 

### 2.2 AMS创建

[->ActivityManagerService.java]

```java
public ActivityManagerService(Context systemContext) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector();
        
        // 设置 System Context 对象，此 systemContext 就是在 SystemServer 和 ActivityThread           //中创建和使用的 System Context 对象
        mContext = systemContext;
        //默认为FACTORY_TEST_OFF
        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

        mPermissionReviewRequired = mContext.getResources().getBoolean(
                com.android.internal.R.bool.config_permissionReviewRequired);
        //创建名为"ActivityManager"的前台线程，并获取mHandler
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        mUiHandler = mInjector.getUiHandler(this);

        mConstants = new ActivityManagerConstants(this, mHandler);

        /* static; one-time init here */
        if (sKillHandler == null) {
            sKillThread = new ServiceThread(TAG + ":kill",
                    THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
        }
        //创建 BroadcastQueue 前台广播对象，处理超时时长是 10s
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        //创建 BroadcastQueue 后台广播对象，处理超时时长是 60s
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        //创建 ActiveServices 对象，用于管理 ServiceRecord 对象
        //其中非低内存手机mMaxStartingBackground为8
        mServices = new ActiveServices(this);
        //创建 ProviderMap 对象，用于管理 ContentProviderRecord 对象
        mProviderMap = new ProviderMap(this);
        mAppErrors = new AppErrors(mUiContext, this);

        // TODO: Move creation of battery stats service outside of activity manager service.
        // 初始化 /data/system 目录
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        // 初始化电池状态信息，进程状态 和 应用权限管理
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
        // 创建进程统计服务，信息保存在目录/data/system/procstats
        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
        // 启动 Android 权限检查服务，注册对应的回调接口
        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });

        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));
        // User 0是第一个，也是唯一的一个开机过程中运行的用户
        mUserController = new UserController(this);

        mVrController = new VrController(this);

        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
            ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mTempConfig.setToDefaults();
        mTempConfig.setLocales(LocaleList.getDefault());
        mConfigurationSeq = mTempConfig.seq = 1;
        //创建 ActivityStackSupervisor 对象，是 AMS 中 ActivityRecord 和 TaskRecord 管理和调度         //的重要类
        mStackSupervisor = createStackSupervisor();
        mStackSupervisor.onConfigurationChanged(mTempConfig);
        mKeyguardController = mStackSupervisor.mKeyguardController;
        mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
        mTaskChangeNotificationController =
                new TaskChangeNotificationController(this, mStackSupervisor, mHandler);
         // 创建 ActivityStarter 对象，用于启动 Activity
        mActivityStarter = new ActivityStarter(this, mStackSupervisor);
        // 最近使用的 RecentTasks
        mRecentTasks = new RecentTasks(this, mStackSupervisor);
        // 创建一个用于更新 CPU 信息的线程,名为"CpuTracker"
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                synchronized (mProcessCpuTracker) {
                    mProcessCpuInitLatch.countDown();
                    mProcessCpuTracker.init();
                }
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        ////更新CPU状态
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };
        // 将此 AMS 对象添加到 Watchdog 中
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }
```

在 AMS 的构造方法中主要做了以下事情：

1. 初始化一些对象属性，包括 Context、ActivityThread、ServiceThread、MainHandler、ActivityManagerConstants 等对象
2. 创建和管理四大组件相关的类对象，包括 BroadcastQueue、ActiveServices、ProviderMap、ActivityStackSupervisor、RecentTasks 和 ActivityStarter 等对象
   创建一个 CPU 监控线程 mProcessCpuThread

### 2.3 AMS.start

由2.1小节可知，Lifecycle创建完AMS后，在它的回调方法`onStart()`方法中调用了AMS的`start()`方法，下面看下`start()`。

```java
private void start() {
        //在启动 CPU 监控线程之前，移除所有的进程组
        removeAllProcessGroups();
        // 启动CpuTracker线程
        mProcessCpuThread.start();
        // 启动电池统计服务
        mBatteryStatsService.publish();
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        //创建LocalService，并添加到LocalServices
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
        // Wait for the synchronized block started in mProcessCpuThread,
        // so that any other acccess to mProcessCpuTracker from main thread
        // will be blocked during mProcessCpuTracker initialization.
        try {
            mProcessCpuInitLatch.await();
        } catch (InterruptedException e) {
            Slog.wtf(TAG, "Interrupted wait during start", e);
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Interrupted wait during start");
        }
    }

```

在 `start()` 方法中，主要完成了以下两个事：

1. 启动 CPU 监控线程，在启动 CPU 监控线程之前，首先将进程复位
2. 注册电池状态服务和权限管理服务

### 2.4 AMS.setSystemProcess

由startBootstrapServices()方法可知，AMS会调用setSystemProcess()方法，  初始化发布一些服务信息。如：meminfo、dbinfo、cpuInfo等，并创建system进程的ActivityThread并设置一些基本进程信息。发布meminfo、dbinfo、cpuInfo等服务是为了能够对设备情况进行实时的dump操作。 

```java
public void setSystemProcess() {
        try {
            //将AMS服务注册到了ServiceManager中
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                //创建系统进程信息
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;//设置系统进程持久化属性为true
                app.pid = MY_PID;//设置pid
                app.maxAdj = ProcessList.SYSTEM_ADJ;//最大adj为-900，系统进程
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                updateLruProcessLocked(app, false, null);//更新lru列表
                updateOomAdjLocked();//更新oom优先级
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
```

在setSystemProcess方法中，首先将自己AMS服务注册到了ServiceManager中，然后又注册了权限服务等其他的系统服务如下所示：

| 服务名      | 类名                   | 功能           |
| :---------- | :--------------------- | :------------- |
| activity    | ActivityManagerService | AMS            |
| procstats   | ProcessStatsService    | 进程统计       |
| meminfo     | MemBinder              | 内存           |
| gfxinfo     | GraphicsBinder         | 图像信息       |
| dbinfo      | DbBinder               | 数据库         |
| cpuinfo     | CpuBinder              | CPU            |
| permission  | PermissionController   | 权限           |
| processinfo | ProcessInfoService     | 进程服务       |
| usagestats  | UsageStatsService      | 应用的使用情况 |

想要查看这些服务的信息，可通过`dumpsys <服务名>`命令。比如查看CPU信息命令`dumpsys cpuinfo`。

然后通过先前创建的Context，得到PMS服务，检索framework-res的Application信息，然后将它配置到系统的ActivityThread中。为了能让AMS同样可以管理调度系统进程，也创建了一个关于系统进程的ProcessRecord对象，ProcessRecord对象保存一个进程的相关信息。然后将它保存到mPidsSelfLocked集合中方便管理。
但是AMS具体是如何将检索到的framework-res的application信息，配置到ActivityThread中的，需要继续分析ActivityThread的installSystemApplicationInfo方法。

#### 2.4.1 ActivityThread.installSystemApplicationInfo 

[->ActivityThread.java]

```java
public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        synchronized (this) {
            getSystemContext().installSystemApplicationInfo(info, classLoader);
            getSystemUiContext().installSystemApplicationInfo(info, classLoader);

            // give ourselves a default profiler
            mProfiler = new Profiler();
        }
    }
```

 这个方法中最终调用上面创建的SystemContext的installSystemApplication方法，那就接着看ConxtextImpl的installSystemApplication方法。 

2.4.2 ConxtextImpl.installSystemApplication

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        mPackageInfo.installSystemApplicationInfo(info, classLoader);
    }
```

 它最终调用了mPackageInfo的installSystemApplication方法， 加载名为“android”的package 。mPackageInfo就是在调用createSystemContext方法创建Context对象的时候传进来的LoadedApk，里面保存了一个应用程序的基本信息。 

#### 2.4.3 LoadedApk.installSystemApplicationInfo

[->LoadedApk.java]

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        assert info.packageName.equals("android");
        mApplicationInfo = info;
        mClassLoader = classLoader;
    }
```

 将framework-res.apk的application信息保存到了mApplication变量中。 

我们返回去看下第一次创建LoadedApk的时候,使用了一个参数的构造方法，该构造方法中mApplication变量是直接new了一个ApplicationInfo的对象，该applicationInfo并没有指向任何一个应用，为什么开始的时候不直接指定，反而到现在再来重新做一次呢？
这个是因为创建系统进程的context的时候AMS和PMS都还没有起来，那时候没有办法指定它的application,现在AMS,PMS都起来之后再来赋值就可以了。

#### 2.4.4 setSystemProcess小结

setSystemProcess主要就是设置系统集成的一些信息，在这里设置了系统进程的Application信息，创建了系统进程的ProcessRecord对象将其保存在进程集合中，方便AMS管理调度。

### 2.5 startOtherServices

[->SystemServer.java]

由SystemServer.java的run()可知，调用完startBootstrapServices()后，会调用startOtherServices()方法，接下来分析startOtherServices()方法里面有关AMS的操作。

```java
private void startOtherServices() {
  ...
  //安装系统Provider
  mActivityManagerService.installSystemProviders();
  ...

  //phase480 && 500
  mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
  mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
  ...
  
  mActivityManagerService.systemReady(() -> {
            //phase550
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_ACTIVITY_MANAGER_READY);
            ...
            //初始化webVew
            mWebViewUpdateService.prepareWebViewInSystemServer();
            ...
            //开启SystemUI apk
            traceBeginAndSlog("StartSystemUI");
            try {
                startSystemUi(context, windowManagerF);
            } catch (Throwable e) {
                reportWtf("starting System UI", e);
            }
            ...
            //开启看门狗
            traceBeginAndSlog("StartWatchdog");
            Watchdog.getInstance().start();
            traceEnd();

            // Wait for all packages to be prepared
            mPackageManagerService.waitForAppDataPrepared();

            //phase600
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
            traceEnd();
            ...
            //执行一系列服务的systemRunning方法
        }, BOOT_TIMINGS_TRACE_LOG);
}
```

#### 2.5.1 AMS.installSystemProviders

[->ActivityManagerService.java]

Android系统中有很多配置信息都需要保存，这些信息是保存在SettingsProvider中，而这个SettingsProvider也是运行在SystemServer进程中的，由于SystemServer进程依赖SettingsProvider，放在一个进程中可以减少进程间通信的效率损失。
下面就来分析下如何将SettingsProvider.apk也加载到SystemServer进程中。

```java
public final void installSystemProviders() {
        List<ProviderInfo> providers;
        synchronized (this) {
            //找到名称为”System”的进程，就是上一步创建的processRecord对象
            ProcessRecord app = mProcessNames.get("system", SYSTEM_UID);
            //找到所有和system进程相关的ContentProvider
            providers = generateApplicationProvidersLocked(app);
            if (providers != null) {
                for (int i=providers.size()-1; i>=0; i--) {
                    ProviderInfo pi = (ProviderInfo)providers.get(i);
                    //再次确认进程为system的provider,把不是该进程provider移除
                    if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                        Slog.w(TAG, "Not installing system proc provider " + pi.name
                                + ": not system .apk");
                        //移除非系统的provider
                        providers.remove(i);
                    }
                }
            }
        }
        if (providers != null) {
            //把provider安装到系统的ActivityThread中
            mSystemThread.installSystemProviders(providers);
        }

        mConstants.start(mContext.getContentResolver());
        // 创建核心Settings Observer，用于监控Settings的改变。
        mCoreSettingsObserver = new CoreSettingsObserver(this);
        mFontScaleSettingObserver = new FontScaleSettingObserver();

        // Now that the settings provider is published we can consider sending
        // in a rescue party.
        RescueParty.onSettingsProviderPublished(mContext);

        //mUsageStatsService.monitorPackages();
    }
```

找到名称为system的进程对象，就是SystemServer进程，然后根据进程对象去查询所有有关的ContentProvider，调用系统进程的主线程ActivityThread安装所有相关的ContentProvider。

在所有初始化完成之后，会调用 `systemReady()` 方法 。

#### 2.5.1 AMS.systemReady

```java
public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
        traceLog.traceBegin("PhaseActivityManagerReady");
        synchronized(this) {
            if (mSystemReady) {
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }
           //初始化Doze模式的controller
            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);
            mAssistUtils = new AssistUtils(mContext);
            mVrController.onSystemReady();
            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mRecentTasks.onSystemReadyLocked();
            mAppOpsService.systemReady();
            //设置systemReady为true
            mSystemReady = true;
        }
    
        ArrayList<ProcessRecord> procsToKill = null;
        //收集那些在AMS之前启动的进程
        synchronized(mPidsSelfLocked) {
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                if (!isAllowedWhileBooting(proc.info)){
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }
        //将那些在AMS之前启动的进程杀死，有的进程不能再AMS之前启动
        synchronized(this) {
            if (procsToKill != null) {
                for (int i=procsToKill.size()-1; i>=0; i--) {
                    ProcessRecord proc = procsToKill.get(i);
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    removeProcessLocked(proc, true, false, "system update done");
                }
            }
            mProcessesReady = true;
        }

        Slog.i(TAG, "System now ready");
        //从settingsProvider的设置总初始化部分变量
        retrieveSettings();
        final int currentUserId;
        synchronized (this) {
            currentUserId = mUserController.getCurrentUserIdLocked();
            readGrantedUriPermissionsLocked();
        }
        //调用callback方法，该方法在systemServer代码中实现
        if (goingCallback != null) goingCallback.run();
        traceLog.traceBegin("ActivityManagerStartApps");
        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
                Integer.toString(currentUserId), currentUserId);
        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                Integer.toString(currentUserId), currentUserId);
        mSystemServiceManager.startUser(currentUserId);

        synchronized (this) {
            // Only start up encryption-aware persistent apps; once user is
            // unlocked we'll come back around and start unaware apps
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

            // Start up initial activity.
            mBooting = true;
            // Enable home activity for system user, so that the system can always boot. We don't
            // do this when the system user is not setup since the setup wizard should be the one
            // to handle home activity in this case.
            if (UserManager.isSplitSystemUser() &&
                    Settings.Secure.getInt(mContext.getContentResolver(),
                         Settings.Secure.USER_SETUP_COMPLETE, 0) != 0) {
                ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
                try {
                    AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                            UserHandle.USER_SYSTEM);
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }
            }
            //启动HomeActivity，也就是launcher程序
            startHomeActivityLocked(currentUserId, "systemReady");
            
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            mUserController.sendUserSwitchBroadcastsLocked(-1, currentUserId);
        }
    }
```

SystemReady方法也是比较长，大致可以分为四个部分

1. 在systemReady的时候初始化了deviceIdleControlle等对象
2. 移除并杀死了那些不该在AMS之前启动的进程
3. 执行了参数传入的回调函数，里面主要做了初始化webVew、开启SystemUI、看门狗和 调用其他服务的systemready方法
4. 启动了Launcer界面
5. 启动那些persistent配置为1的进程

## 3 总结

AMS服务启动主要分为几个步骤。

1.  创建了SystemServer进程的运行环境，包括一个ActivityThread主线程，一个和系统进程相关的Context对象。
2.  调用AMS的构造方法和start方法，对AMS必要的内容进行初始化
3.  将函数AMS注册到ServiceManager中，同时对systemServer进程也创建了一个ProcessRecord对象，并设置Context的appliation为framework-res的application对象
4.  将settingsProvider加载到系统进程systemServer中
5.  调用systemReady方法做一些启动前的就绪工作，并启动了HomeActivity和SystemUI