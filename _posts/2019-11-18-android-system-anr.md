---
layout: post
title:  "Android ANR问题总结(非原创)"
date:   2019-11-18 00:00:00
catalog:  true
tags:
    - android
    - ANR
---

# 1 概述

 ANR(Application Not responding)，是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。 

- ANR：Application Not Responding，即应用无响应
- 为用户在主线程长时间被阻塞
- Android 系统自身提供的一种检测机制

# 2  ANR分类

ANR包含如下几种类型：

-  inputDispatching Timeout: 输入事件分发超时**5s**，包括按键分发事件的超时。==主要类型 
-  BroadcastQueue Timeout：比如前台广播在**10s**内执行完成 
-  ContentProvider Timeout：内容提供者执行超时， 在publish过超时**10s** （读写权限导致的） 
- Service Timeout:服务在**20s**内未执行完成；===小概率类型

# 3 产生ANR原因

产生ANR原因，如下几种：

- 主线程耗时操作， 例如，主线程阻塞（block）、挂起、死锁、死循环、执行耗时操作等 
- 主线程被其它线程Block
- 系统级响应阻塞
- 系统或进程自身可用内存紧张
- CPU资源被抢占， 例如，其他进程的 CPU 占用率过高、某一时刻系统 CPU 负载过高等，  都会导致当前进程无法抢到 CPU 时间片 

对于这些ANR，先给大家的推荐一下大致分析思路和相关日志，通常发生ANR时，首先去查找对应Trace日志，看看主线程是否在处理该广播或被阻塞，如果发现上述现象，那么恭喜你，已经很接近答案了。但如果发现堆栈完全处于空闲状态，那么很不幸，就需要扩大参考面了，需要结合log日志进行分析，日志包括logcat, kernel日志，cpuinfo以及meminfo等，参考顺序从前向后。

## 3.1 分析logcat思路

首先在日志中搜索（“anr in”，“low_memory”, “slow_operation”）等关键字，通过该类关键字主要是查看系统Cpu负载，如果是发现应用进程CPU明显过高，那么很有可能是该进程抢占CPU过多导致，系统调度不及时,误认为应用发生了超时行为。

从 LOG 上可以提取到的有效信息：

-> 举例说明：

```java
04-01 13:12:11.572 I/InputDispatcher( 220): Application is not responding:Window{2b263310com.android.email/com.android.email.activity.SplitScreenActivitypaused=false}.
5009.8ms since event, 5009.5ms since waitstarted
04-0113:12:11.572 I/WindowManager( 220): Input event 
dispatching timedout sending 
tocom.android.email/com.android.email.activity.SplitScreenActivity

// 1. 发生 ANR 的时间和生成 trace.txt 的时间
04-01 13:12:14.123 I/Process( 220): Sending signal. PID: 21404 SIG: 
04-01 13:12:14.123 I/dalvikvm(21404):threadid=4: reacting to 
signal 3 
// 2. ANR 发生的地点
04-0113:12:15.872 E/ActivityManager( 220): ANR in 
com.android.email(com.android.email/.activity.SplitScreenActivity)
04-0113:12:15.872 E/ActivityManager( 220): 
// 3. ANR 的类型
Reason:keyDispatchingTimedOut
// 4. ANR 发生前 CPU 的负载情况
04-0113:12:15.872 E/ActivityManager( 220): Load: 8.68 / 8.37 / 8.53
04-0113:12:15.872 E/ActivityManager( 220): CPUusage from 4361ms to 699ms ago 
04-0113:12:15.872 E/ActivityManager( 220): 5.5%21404/com.android.email: 1.3% user + 4.1% kernel / faults:
10 minor
04-0113:12:15.872 E/ActivityManager( 220): 4.3%220/system_server: 2.7% user + 1.5% kernel / faults: 11
minor 2 major
04-0113:12:15.872 E/ActivityManager( 220): 0.9%52/spi_qsd.0: 0% user + 0.9% kernel
04-0113:12:15.872 E/ActivityManager( 220): 0.5%65/irq/170-cyttsp-: 0% user + 0.5% kernel
04-0113:12:15.872 E/ActivityManager( 220): 0.5%296/com.android.systemui: 0.5% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 100%TOTAL: 4.8% user + 7.6% kernel + 87% iowait
// 5. ANR 发生后 CPU 的负载情况
04-0113:12:15.872 E/ActivityManager( 220): CPUusage from 3697ms to 4223ms 
04-0113:12:15.872 E/ActivityManager( 220): 25%21404/com.android.email: 25% user + 0% kernel / faults: 191 minor
04-0113:12:15.872 E/ActivityManager( 220): 16% 21603/__eas(par.hakan: 16% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 7.2% 21406/GC: 7.2% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 1.8% 21409/Compiler: 1.8% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 5.5%220/system_server: 0% user + 5.5% kernel / faults: 1 minor
04-0113:12:15.872 E/ActivityManager( 220): 5.5% 263/InputDispatcher: 0% user + 5.5% kernel
04-0113:12:15.872 E/ActivityManager( 220): 32%TOTAL: 28% user + 3.7% kernel
```

- ANR 发生的时间、地点以及类型
  类型为 keyDispatchingTimedOut，即按键事件输入响应超时，发生在 com.android.email 应用中
- CPU 负载情况
  1. 如果 CPU 占用率接近 100%，说明当前设备很忙，有可能是 CPU 饥饿导致了ANR
  2. 如果 IOWait 很高，说明主线程正在进行 IO 操作
  3. 如果 CPU 占用率很低，则说明当前主线程被 BLOCK 了

 从上面的 LOG 可以看出，total CPU 占用率达到 100%，并且 io wait 占用 87%，说明当前主线程正在进行 io 操作。 

## 3.2 分析 traces 文件思路

一般情况下，如果ANR发生，对应的应用会收到SIGQUIT异常终止信号，dalvik虚拟机就会自动在/data/anr/目录下生成trace.txt文件，这个文件记录了在发生ANR时刻系统各个线程的执行状态，获取这个文件是不需要root权限的，因此首先需要做的就是通过adb pull命令将这个文件导出并等待分析。 产生ANR的原因有很多，比如CPU使用过高、事件没有得到及时的响应、死锁等 。

 trace 文件中会标明 Thread 的状态，如： 

```java
DALVIK THREADS (12):
// 当前线程名 - "main"
// 线程优先级 - "prio=5"
// 线程id - "tid=1"
// 线程状态 - "Sleeping"
"main" prio=5 tid=1 Runnable
```

 其他常用的状态定义在 Thread.java 文件中： 

```java
ThreadState (defined at "dalvik/vm/thread.h")
THREAD_UNDEFINED = -1, / makes enum compatible with int32_t /
THREAD_ZOMBIE = 0, / 线程死亡，终止运行 /
THREAD_RUNNING = 1, / 线程可运行或正在运行 /
THREAD_TIMED_WAIT = 2, / 执行了带有超时参数的 wait、sleep 或 join 函数 /
THREAD_MONITOR = 3, / 线程阻塞 /
THREAD_WAIT = 4, / 执行了无超时参数的 wait 函数 /
THREAD_INITIALIZING= 5, / 新建，正在初始化，为其分配资源 /
THREAD_STARTING = 6, / 新建，正在启动 /
THREAD_NATIVE = 7, / 正在执行 JNI 本地函数 /
THREAD_VMWAIT = 8, / 正在等待 VM 资源 /
THREAD_SUSPENDED = 9, / suspended, usually by GC or debugger /
```

 举例分析： 

```java
----- pid 609 at 2019-02-16 05:00:10 -----
Cmd line: com.android.systemui
...
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x748ef1f0 self=0xb0851000
  | sysTid=609 nice=0 cgrp=default sched=0/0 handle=0xb4a3f494
  | state=S schedstat=( 392030973719 182772499142 1025298 ) utm=33261 stm=5941 core=2 HZ=100
  | stack=0xbe5fa000-0xbe5fc000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/609/stack)
  native: #00 pc 00019d74  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d11d  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+88)
  native: #02 pc 00063a99  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 0037cb2b  /system/lib/libhwui.so (android::uirenderer::renderthread::DrawFrameTask::postAndWait()+174)
  native: #04 pc 0037ca59  /system/lib/libhwui.so (android::uirenderer::renderthread::DrawFrameTask::drawFrame()+24)
  at android.view.ThreadedRenderer.nSyncAndDrawFrame(Native method)
  at android.view.ThreadedRenderer.draw(ThreadedRenderer.java:823)
  at android.view.ViewRootImpl.draw(ViewRootImpl.java:3316)
  at android.view.ViewRootImpl.performDraw(ViewRootImpl.java:3120)
  at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:2489)
  at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:1465)
  at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:7188)
  at android.view.Choreographer$CallbackRecord.run(Choreographer.java:949)
  at android.view.Choreographer.doCallbacks(Choreographer.java:761)
  at android.view.Choreographer.doFrame(Choreographer.java:696)
  at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:935)
  at android.os.Handler.handleCallback(Handler.java:873)
  at android.os.Handler.dispatchMessage(Handler.java:99)
  at android.os.Looper.loop(Looper.java:193)
  at android.app.ActivityThread.main(ActivityThread.java:6718)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

 以上的 trace 文件来自 systemui ANR，可以看出在 main 线程执行 NATIVE 方法的过程中，nSyncAndDrawFrame GPU 渲染的过程发生超时。 

## 3.3 分析kernel思路

在此类日志中直接搜索lowmemorykiller, 如果存在则查看发生时间和ANR时间是否大致对应，相差无几的话，可以从该日志中看到操作系统层面当前内存情况，Free Memory说明的是空闲物理内存，File Free说明的则是文件Cache，也就是应用或系统从硬盘读取文件，使用结束后，kernel并没有这正释放这类内存，加以缓存，目的是为了下次读写过程加快速度。当然，发现Free和Other整体数值都偏低时，Kernel会进行一定程度的内存交换，导致整个系统卡顿。同时这类现象也会体现在log日志“slow_operation”中，即系统进程的调度也会收到影响。

## 3.4 分析cpuinfo思路

这类日志一目了然，可以清晰的看到哪类进程CPU偏高，如果存在明显偏高进程，那么ANR和此进程抢占CPU有一定关系。当然，如发现Kswapd，emmc进程在top中，则说明遇到系统内存压力或文件IO开销。

## 3.5 分析meminfo思路

分析该类日志，主要是看哪类应用或系统占用内存偏高，如果应用内存占用比较正常，系统也没有发生过度内存使用，那么则说明系统中缓存了大量进程，并没有及时释放导致系统整体内存偏低。

## 3.6 综合分析

综合分析当时系统环境，例如电量（低电可能会引起手机限频，限核等等），手机温度（温度过高也可能会引起限频），以及操作频率（例如执行monkey测试）等等；
上面说了这么多，下面结合实例进行分析。

# 4 实例分析

## 4.1 实例一：主线程进行耗时操作，或被进程内其它线程阻塞

### 4.1.1 分析

观察Trace 主线程堆栈，发现主线程在申请内存过程中被block，等待GC结束，但通过堆栈进一步发现其GC并没有发生在该线程，也就是说在其他线程在执行GC动作，而主线程在申请内存过程中需要等待GC完成，再进一步申请内存。

```java
"main" prio=5 tid=1 WaitingForGcToComplete

  native: #00 pc 0000000000019980  /system/lib64/libc.so (syscall+28)
  native: #01 pc 000000000013a62c  /system/lib64/libart.so (_ZN3art17ConditionVariable4WaitEPNS_6ThreadE+136)
  native: #02 pc 0000000000237f14  /system/lib64/libart.so (_ZN3art2gc4Heap19WaitForGcToCompleteENS0_7GcCauseEPNS_6ThreadE+1376)
  native: #03 pc 000000000024798c  /system/lib64/libart.so (_ZN3art2gc4Heap22AllocateInternalWithGcEPNS_6ThreadENS0_13AllocatorTypeEmPmS5_S5_PPNS_6mirror5ClassE+168)
  native: #04 pc 000000000050394c  /system/lib64/libart.so (artAllocObjectFromCodeRosAlloc+1412)
  native: #05 pc 00000000001215d0  /system/lib64/libart.so (art_quick_alloc_object_rosalloc+64)
  native: #06 pc 00000000018e72f0  /system/framework/arm64/boot.oat (Java_android_widget_TextView__0003cinit_0003e__Landroid_content_Context_2Landroid_util_AttributeSet_2II+1156)
  at android.widget.TextView.<init>(TextView.java:727)
  at android.widget.TextView.<init>(TextView.java:682)
  at android.widget.TextView.<init>(TextView.java:678)
  at java.lang.reflect.Constructor.newInstance!(Native method)
```

再看看其它线程状态，进一步查找发现，下面任务正在执行GC

```java
"LeuiRunningState:Background" prio=5 tid=28 WaitingPerformingGc

"AsyncTask #6" prio=5 tid=20 WaitingPerformingGc
```

综上可以得出大致结论，Tid=28,20线程执行GC,导致主线程申请内存被Block.  但是进一步思考，应用GC是常有的事，但是为何这次需要这么长时间呢，带着疑问我们看看进程的内存使用情况：

```java
Total number of allocations 9887486
Total bytes allocated 732MB
Total bytes freed 476MB
Free memory 5KB
Free memory until GC 5KB
Free memory until OOME 5KB
Total memory 256MB
Max memory 256MB
```

上面发现，应用已使用256Mb, 距离OOM只有5K，内存对象超过998万个，也就是说GC过程需要扫描这些对象的巨大部分，导致耗时很久，另外内存距离OOM只有5kb，说明有内存泄漏，或内存使用不合理。

### 4.1.2 结论

综上，对于这个问题得出结论，应用进程内存存在泄漏或使用不当，导致GC时间过程，产生ANR.

## 4.2 实例二：应用内部线程逻辑依赖关系导致超时，触发ANR

### 4.2.1 分析

观察Trace 主线程堆栈，发现主线程在Binder通信过程被Block.

```java
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 obj=0x75f0eaa8 self=0x7fad046a00
  | sysTid=4298 nice=-6 cgrp=default sched=0/0 handle=0x7fb1d18fe8
  | state=S schedstat=( 79488910537 19985244611 169915 ) utm=6564 stm=1384 core=0 HZ=100
  | stack=0x7fc237c000-0x7fc237e000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/4298/stack)
  native: #00 pc 00000000000683d0  /system/lib64/libc.so (__ioctl+4)
  native: #01 pc 00000000000723f8  /system/lib64/libc.so (ioctl+100)
  native: #02 pc 000000000002d584  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+164)
  native: #03 pc 000000000002e050  /system/lib64/libbinder.so (_ZN7android14IPCThreadState15waitForResponseEPNS_6ParcelEPi+104)
  native: #04 pc 000000000002e2c4  /system/lib64/libbinder.so (_ZN7android14IPCThreadState8transactEijRKNS_6ParcelEPS1_j+176)
  native: #05 pc 0000000000025654  /system/lib64/libbinder.so (_ZN7android8BpBinder8transactEjRKNS_6ParcelEPS1_j+64)
  native: #06 pc 00000000000e0928  /system/lib64/libandroid_runtime.so (???)
  native: #07 pc 000000000139ba24  /system/framework/arm64/boot.oat (Java_android_os_BinderProxy_transactNative__ILandroid_os_Parcel_2Landroid_os_Parcel_2I+200)
  at android.os.BinderProxy.transactNative(Native method)
  at android.os.BinderProxy.transact(Binder.java:503)
  at android.nfc.INfcAdapter$Stub$Proxy.setAppCallback(INfcAdapter.java:529)
  at android.nfc.NfcActivityManager.requestNfcServiceCallback(NfcActivityManager.java:339)
  at android.nfc.NfcActivityManager.setNdefPushMessageCallback(NfcActivityManager.java:309)
```

进一步查找此线程在和哪个进程进行通信，搜索关键字“setAppCallback”（Android命名习惯，客户端和服务端函数命名基本相同），在Nfc的Binder_3线程响应了客户端请求，但在处理过程中被线程1阻塞，顺着再看看线程1状态

```java
"Binder_3" prio=5 tid=17 Blocked

  | group="main" sCount=1 dsCount=0 obj=0x12ddf0a0 self=0x7fa670f000

  | sysTid=3183 nice=-6 cgrp=default sched=0/0 handle=0x7f93c30440

  | state=S schedstat=( 3041465858 2637156615 16961 ) utm=168 stm=136 core=3 HZ=100

  | stack=0x7f93b34000-0x7f93b36000 stackSize=1013KB

  | held mutexes=

  at com.android.nfc.P2pLinkManager.setNdefCallback(P2pLinkManager.java:420)

  waiting to lock <0x0bed0520> (a com.android.nfc.P2pLinkManager) held by thread 1

  at com.android.nfc.NfcService$NfcAdapterService.setAppCallback(NfcService.java:1679)

  at android.nfc.INfcAdapter$Stub.onTransact(INfcAdapter.java:178)

  at android.os.Binder.execTransact(Binder.java:453)

"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 obj=0x75f0eaa8 self=0x7fad046a00
  | sysTid=2706 nice=0 cgrp=default sched=0/0 handle=0x7fb1d18fe8
  | state=S schedstat=( 115355173189 36125520701 224819 ) utm=8594 stm=2941 core=0 HZ=100
  | stack=0x7fc237c000-0x7fc237e000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/2706/stack)
  native: #00 pc 00000000000683d0  /system/lib64/libc.so (__ioctl+4)
  native: #01 pc 00000000000723f8  /system/lib64/libc.so (ioctl+100)
  native: #02 pc 000000000002d584  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+164)
  native: #03 pc 000000000002e050  /system/lib64/libbinder.so (_ZN7android14IPCThreadState15waitForResponseEPNS_6ParcelEPi+104)
  native: #04 pc 000000000002e2c4  /system/lib64/libbinder.so (_ZN7android14IPCThreadState8transactEijRKNS_6ParcelEPS1_j+176)
  native: #05 pc 0000000000025654  /system/lib64/libbinder.so (_ZN7android8BpBinder8transactEjRKNS_6ParcelEPS1_j+64)
  native: #06 pc 00000000000e0928  /system/lib64/libandroid_runtime.so (???)
  native: #07 pc 000000000139ba24  /system/framework/arm64/boot.oat (Java_android_os_BinderProxy_transactNative__ILandroid_os_Parcel_2Landroid_os_Parcel_2I+200)
  at android.os.BinderProxy.transactNative(Native method)
  at android.os.BinderProxy.transact(Binder.java:503)
  at android.nfc.IAppCallback$Stub$Proxy.createBeamShareData(IAppCallback.java:113)
  at com.android.nfc.P2pLinkManager.prepareMessageToSend(P2pLinkManager.java:558)

locked <0x0bed0520> (a com.android.nfc.P2pLinkManager)
```

通过主线程，又发现正进程Binder通信，同时被block,搜索关键字“createBeamShareData”，发现又回到浏览器线程，Binder_6线程响应此请求，同时也处于Waiting状态

```java
"Binder_6" prio=5 tid=12 Waiting

  | group="main" sCount=1 dsCount=0 obj=0x12c13a00 self=0x7f52850e00

  | sysTid=23857 nice=0 cgrp=default sched=0/0 handle=0x7f694ff440

  | state=S schedstat=( 705897380 828401158 3677 ) utm=45 stm=25 core=1 HZ=100

  | stack=0x7f69403000-0x7f69405000 stackSize=1013KB

  | held mutexes=

  at java.lang.Object.wait!(Native method)

  - waiting on <0x08a80433> (a java.lang.Object)

  at java.lang.Thread.parkFor$(Thread.java:1220)

  - locked <0x08a80433> (a java.lang.Object)

  at sun.misc.Unsafe.park(Unsafe.java:299)

  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:158)

  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:810)

  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:970)

  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1278)

  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:203)

  at com.android.browser.NfcHandler.createNdefMessage(NfcHandler.java:92)

  at android.nfc.NfcActivityManager.createBeamShareData(NfcActivityManager.java:377)

  at android.nfc.IAppCallback$Stub.onTransact(IAppCallback.java:53)

  at android.os.Binder.execTransact(Binder.java:453)
```

为什么Binder_6处于Waiting状态？这就需要大家结合Read the Fuck Code的精神研究逻辑了，事后发现此线程的事件放在了主线程执行，当执行完毕后接收通知，停止waiting.
至此，我们找到了一条完整的链路，（浏览器主线程---->NFC Binder_3---->NFC主线程---->浏览器 Binder_6---->浏览器主线程），大家到此看到了根本原因，死锁！！！

### 4.2.2 结论

综上，对于这个问题得出结论，应用通信过程中发生死锁导致ANR，后面只需解锁即可。
上面两类问题，相对简单，大家遇到时也多半能自行分析解决，下面两类涉及到较多系统或其它因素，问题比较隐晦，但是按照一定分析思路，静下来分析，多数时候还是能找到原因或给出优化方案的。

## 4.3 实例三：系统内存过低，kernel进行内存交换过程会引起整个系统运行缓慢(卡顿)

### 4.3.1 分析

观察Trace 主线程堆栈，发现主线程处于Suspend状态；发生此类问题一般是两种情况，一种是进程自身过于繁忙，每次分配时间片都不够用，调度器强制把它置换成休眠了，另一种是系统比较繁忙，低优先级集成得不到时间片；带着这样的疑问，继续看：

```java
"main" prio=5 tid=1 Suspended

  | group="main" sCount=1 dsCount=0 obj=0x745518a0 self=0x7f86254a00

  | sysTid=21916 nice=0 cgrp=default sched=0/0 handle=0x7f8b30efc8

  | state=S schedstat=( 311762801762 96254728754 409881 ) utm=25610 stm=5566 core=0 HZ=100

  | stack=0x7fd023c000-0x7fd023e000 stackSize=8MB

  | held mutexes=

  at java.util.regex.Splitter.fastSplit(Splitter.java:73)

  at java.lang.String.split(String.java:1410)

  at java.lang.String.split(String.java:1392)

  at android.content.res.theme.LeResourceHelper.getResName(LeResourceHelper.java:193)

  at android.content.res.Resources.loadDrawable(Resources.java:2624)

  at android.content.res.Resources.getDrawable(Resources.java:862)

  at android.content.Context.getDrawable(Context.java:458)

  at android.widget.ImageView.resolveUri(ImageView.java:813)
```

这个时候可以看看应用逻辑是否会存在繁忙操作不停抢占时间片，另一方面可以看看对应日志，通过logcat发现如下信息

```java
11-17 09:49:41.392  1532  1574 E ActivityManager: ANR in com.android.systemui

11-17 09:49:41.392  1532  1574 E ActivityManager: PID: 21916

11-17 09:49:41.392  1532  1574 E ActivityManager: Reason: Broadcast of Intent { act=android.intent.action.TIME_TICK flg=0x50000014 mCallingUid=1000 (has extras) }

11-17 09:49:41.392  1532  1574 E ActivityManager: Load: 22.72 / 20.06 / 15.54  /分别对应1分钟/5分钟/15分钟/

11-17 09:49:41.392  1532  1574 E ActivityManager: CPU usage from 3ms to 24033ms later:

11-17 09:49:41.392  1532  1574 E ActivityManager:   60% 134/kswapd0: 0% user + 60% kernel

11-17 09:49:41.392  1532  1574 E ActivityManager:   32% 1532/system_server: 7.4% user + 25% kernel / faults: 31214 minor 423 major
```

系统整体负载很重，常规下负载在10左右；另外发现kswapdCPU占用率极高，通过这两项可以得到系统内存偏低，不停kill进程并发生内存交换，是不是这样的呢？我们再搜索一下其它关键字Slow operation：

```java
11-17 09:42:25.292  1532  1572 W ActivityManager: Slow operation: 2440ms so far, now at startProcess: returned from zygote!

11-17 09:42:25.357  1532  1572 W ActivityManager: Slow operation: 2505ms so far, now at startProcess: done updating battery stats

11-17 09:42:25.357  1532  1572 I am_proc_start: [0,30188,10088,com.letv.android.usagestats,service,com.letv.android.usagestats/.UsageStatsReportService]

11-17 09:42:25.357  1532  1572 W ActivityManager: Slow operation: 2505ms so far, now at startProcess: building log message

11-17 09:42:25.357  1532  1572 I ActivityManager: Start proc 30188:com.letv.android.usagestats/u0a88 for service com.letv.android.usagestats/.UsageStatsReportService

11-17 09:42:25.357  1532  1572 W ActivityManager: Slow operation: 2505ms so far, now at startProcess: starting to update pids map

11-17 09:42:25.357  1532  1572 W ActivityManager: Slow operation: 2505ms so far, now at startProcess: done updating pids map

11-17 09:42:25.385  1532  1572 W ActivityManager: Slow operation: 2534ms so far, now at startProcess: done starting proc!
```

发现普通系统函数执行一次就耗费了2S以上，足见系统卡顿。现在我们继续延着内存方向确认，看看meminfo日志吧

```java
Total PSS by process:

  3441530 kB: com.android.mms (pid 2518 / activities)

  229272 kB: mediaserver (pid 763)
```

通过PSS发现，SMS进程内存占用超过3G！对，第一反应就是内存泄漏，普通应用甚至系统内存占用根本不可能达到这么多。如果大家有时间可以看看kernel日志，搜索lowmemoryKiller，发生问题时间内一定有大量的进程被kill.

### 4.3.2 结论

综上，对于这个问题得出结论，应用在Native层发生内存泄漏(不要问我为什么不是Java层发生这么多内存泄漏@@)。导致系统整体内存吃紧，又因为其本身Persist属性，具有很高优先级（-12），LMK不会将其Kill.只能不停Kill其它应用，并进程内存交换，类似问题参见XIIIM-8358

## 4.4 实例四：Binder资源耗尽，导致通信请求难以及时响应

### 4.4.1 分析

该类问题和内存过低相似，查看主线程堆栈基本正常

```java
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 obj=0x76261710 self=0x7f82646a00
  | sysTid=3084 nice=0 cgrp=default sched=0/0 handle=0x7f874adfe8
  | state=S schedstat=( 83808100322 29188718104 264083 ) utm=5716 stm=2664 core=1 HZ=100
  | stack=0x7ff0f87000-0x7ff0f89000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/3084/stack)
  native: #00 pc 00000000000682e4  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 000000000001f3a4  /system/lib64/libc.so (epoll_pwait+32)
  native: #02 pc 000000000001be88  /system/lib64/libutils.so (_ZN7android6Looper9pollInnerEi+144)
  native: #03 pc 000000000001c268  /system/lib64/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+80)
  native: #04 pc 00000000000d3088  /system/lib64/libandroid_runtime.so (_ZN7android18NativeMessageQueue8pollOnceEP7_JNIEnvP8_jobjecti+48)
  native: #05 pc 000000000000554c  /system/framework/arm64/boot.oat (Java_android_os_MessageQueue_nativePollOnce__JI+144)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:324)
  at android.os.Looper.loop(Looper.java:135)
```

当处于这种状态时，我们直奔主题，分析log日志，按照logcat, kernel, cpuinfo, meminfo等依次分析：

```java
11-08 23:51:44.088  1514  1554 E ActivityManager: ANR in com.android.phone

11-08 23:51:44.088  1514  1554 E ActivityManager: PID: 3084

11-08 23:51:44.088  1514  1554 E ActivityManager: Reason: Broadcast of Intent { act=com.android.internal.telephony.data-restart-trysetup.default flg=0x10000014 mCallingUid=1001 (has extras) }

11-08 23:51:44.088  1514  1554 E ActivityManager: Load: 9.92 / 9.81 / 10.02

11-08 23:51:44.088  1514  1554 E ActivityManager: CPU usage from 0ms to 6497ms later:

11-08 23:51:44.088  1514  1554 E ActivityManager:   108% 3084/com.android.phone: 101% user + 6.7% kernel / faults: 12120 minor 179 major

11-08 23:51:44.088  1514  1554 E ActivityManager:   66% 1514/system_server: 16% user + 49% kernel / faults: 20836 minor 88 major

11-08 23:51:44.088  1514  1554 E ActivityManager:   13% 13013/ca.bellmedia.cp24: 5.3% user + 8.4% kernel / faults: 3216 minor 39 major
```

通过上面的log日志，发现发生ANR进程本身CPU占用比较高，再搜索"slow operation"，“low_memory” 等关键字，都没有出现在log日志中，而lowmemorykiller也以较合理的频率出现在dmesg日志中，所以基本排除是内存过低导致；所以下面延着CPU方向继续分析。
log日志无法找到更多线索，同时思考既然主线程状态正常，那么高cpu一定是其它线程引起的，那就反馈trace继续分析，查看phone进程的其它线程发现，几乎所有binder线程都处于waiting状态，只有Binder_2在工作：

```java
"Binder_1" prio=5 tid=40 TimedWaiting

"Binder_3" prio=5 tid=40 TimedWaiting

"Binder_4" prio=5 tid=40 TimedWaiting

"Binder_5" prio=5 tid=39 TimedWaiting

"Binder_6" prio=5 tid=40 TimedWaiting

"Binder_7" prio=5 tid=40 TimedWaiting

"Binder_8" prio=5 tid=40 TimedWaiting

。。。。

"Binder_2" prio=5 tid=8 Native

  | group="main" sCount=1 dsCount=0 obj=0x12c9b0a0 self=0x7f7be14400
  | sysTid=3107 nice=0 cgrp=default sched=0/0 handle=0x7f8131d440
  | state=R schedstat=( 515275891171 40426859698 234033 ) utm=49200 stm=2327 core=2 HZ=100
  | stack=0x7f81221000-0x7f81223000 stackSize=1013KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/3107/stack)
  native: #00 pc 0000000000070f20  /system/lib64/libsqlite.so (???)
  native: #01 pc 000000000007420c  /system/lib64/libsqlite.so (sqlite3_step+652)
  native: #02 pc 00000000000ba4a4  /system/lib64/libandroid_runtime.so (???)
  native: #03 pc 00000000000ba514  /system/lib64/libandroid_runtime.so (???)
  native: #04 pc 00000000003bc578  /system/framework/arm64/boot.oat (Java_android_database_sqlite_SQLiteConnection_nativeExecuteForChangedRowCount__JJ+140)
  at android.database.sqlite.SQLiteConnection.nativeExecuteForChangedRowCount(Native method)
  at android.database.sqlite.SQLiteConnection.executeForChangedRowCount(SQLiteConnection.java:732)
  at android.database.sqlite.SQLiteSession.executeForChangedRowCount(SQLiteSession.java:754)
  at android.database.sqlite.SQLiteStatement.executeUpdateDelete(SQLiteStatement.java:64)
  at android.database.sqlite.SQLiteDatabase.delete(SQLiteDatabase.java:1499)
  at com.android.providers.telephony.SmsProvider.delete(SmsProvider.java:899)
  at android.content.ContentProvider$Transport.delete(ContentProvider.java:339)
  at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:206)
  at android.os.Binder.execTransact(Binder.java:453)
```

进一步分析该线程状态：state=R 说明其处于工作态。通过查看线程堆栈逻辑，发现正常情况下有log打印，借此再次返回到log日志，发现如下信息：

```java
11-08 23:51:14.512  3084  3289 W SQLiteConnectionPool: The connection pool for database '/data/user/0/com.android.providers.telephony/databases/mmssms.db' has been unable to grant a connection to thread 111 (Binder_3) with flags 0x1 for 30.000002 seconds.

11-08 23:51:14.512  3084  3289 W SQLiteConnectionPool: Connections: 1 active, 0 idle, 0 available.

11-08 23:51:14.512  3084  3289 W SQLiteConnectionPool:

11-08 23:51:14.512  3084  3289 W SQLiteConnectionPool: Requests in progress:

11-08 23:51:14.512  3084  3289 W SQLiteConnectionPool:   executeForChangedRowCount started 30008ms ago - running, sql="DELETE FROM sms WHERE (thread_id=2) AND (locked=0 AND date<1452658564000)"

11-08 23:51:14.513  3084  3613 W SQLiteConnectionPool: The connection pool for database '/data/user/0/com.android.providers.telephony/databases/mmssms.db' has been unable to grant a connection to thread 141 (Binder_5) with flags 0x1 for 30.009 seconds.

11-08 23:51:14.513  3084  3613 W SQLiteConnectionPool: Connections: 1 active, 0 idle, 0 available.
```

说明在Binder_3和Binder_6线程执行Sql之前，已经有其它线程执行时间超过30S仍未结束。继续搜集log发现，有15个Binder线程处于Waiting状态，而那个正在执行的则为Binder-2，耗时30S以上。

### 4.4.2 结论

综上了该进程高CPU的原因：Binder_2线程执行Sql操作时间过长，进一步引起其它所有Binder线程被block，导致系统广播发送无法及时通过Binder传递给主线程，误触发系统认为Phone进程处理广播超时。

## 4.5 实例五：高CPU过度抢占时间片，导致其它应用或任务难以及时调度

### 4.5.1 分析

该类问题主线程多半是处于空闲或Suspend状态，后者表示系统分配的CPU时间片无法满足当前需要便被强行切换，而引起该类现象的要么是底层系统动作，要么是其它任务高优先级任务抢占CPU行为；

```java
"main" prio=5 tid=1 Suspended

  | group="main" sCount=2 dsCount=0 obj=0x75285af8 self=0x7f87a46a00

  | sysTid=9251 nice=-6 cgrp=default sched=0/0 handle=0x7f8c5f7fe8

  | state=S schedstat=( 50580737351 8433337317 81975 ) utm=4561 stm=497 core=1 HZ=100

  | stack=0x7ff8105000-0x7ff8107000 stackSize=8MB

  | held mutexes=

  at java.util.Arrays.checkOffsetAndCount(Arrays.java:1722)

  at java.nio.CharBuffer.wrap(CharBuffer.java:90)

  at java.nio.CharBuffer.wrap(CharBuffer.java:68)

  at android.text.TextDirectionHeuristics$TextDirectionHeuristicImpl.isRtl(TextDirectionHeuristics.java:149)

  at android.text.BoringLayout.isBoring(BoringLayout.java:477)

  at android.widget.TextView.onMeasure(TextView.java:7096)

  at android.view.View.measure(View.java:19138)

  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6064)

  at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1465)

  at android.widget.LinearLayout.measureHorizontal(LinearLayout.java:1112)

  at android.widget.LinearLayout.onMeasure(LinearLayout.java:632)

  at android.view.View.measure(View.java:19138)
```

当Trace上无法继续分析时，便需要分析日志了，搜索关键字“anr in”，发现

```java
11-26 11:47:16.514  1457  1490 E ActivityManager: ANR in com.android.browser (com.android.browser/.MainActivity)

11-26 11:47:16.514  1457  1490 E ActivityManager: PID: 9251

11-26 11:47:16.514  1457  1490 E ActivityManager: Reason: Input dispatching timed out (Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.  Wait queue length: 10.  Wait queue head age: 8974.9ms.)

11-26 11:47:16.514  1457  1490 E ActivityManager: Load: 10.97 / 10.71 / 10.0

11-26 11:47:16.514  1457  1490 E ActivityManager: CPU usage from 0ms to 10480ms later:

11-26 11:47:16.514  1457  1490 E ActivityManager:   114% 9251/com.android.browser: 65% user + 48% kernel / faults: 10870 minor 11 major

11-26 11:47:16.514  1457  1490 E ActivityManager:   108% 1457/system_server: 33% user + 74% kernel / faults: 9584 minor 11 major
```

浏览器自身CPU占用较高，至于System_server占用比较多，尤其是当大家看到“ CPU usage from 0ms to 10480ms later”已经kernel部分（74% kernel /）占用较多的情况下，不要再轻易怀疑是system_server高CPU导致，其高CPU的真正原因是需要dump各进程信息而已。
顺着"ANR in"之前的日志，我们继续向上看，发现该应该进行了大量且频繁的GC操作

```java
11-26 11:47:05.204  1457  1467 I art     : Background partial concurrent mark sweep GC freed 842(578KB) AllocSpace objects, 455(85MB) LOS objects, 8% free, 169MB/185MB, paused 2.140ms total 245.072ms

11-26 11:47:10.493  9251 31938 W art     : Suspending all threads took: 131.446ms

11-26 11:47:10.598  9251 31938 W art     : Suspending all threads took: 88.134ms

11-26 11:47:10.699  9251 31938 W art     : Suspending all threads took: 93.939ms

11-26 11:47:10.795  9251 31938 W art     : Suspending all threads took: 75.051ms

11-26 11:47:10.821  9251 31938 W art     : Suspending all threads took: 14.536ms

11-26 11:47:10.956  9251 31938 W art     : Suspending all threads took: 114.243ms

11-26 11:47:11.101  9251 31938 W art     : Suspending all threads took: 121.775ms

11-26 11:47:11.254  9251 31938 W art     : Suspending all threads took: 93.763ms

.....
```

而根据GC类型（Background partial concurrent）来看，应该是有任务在不停的申请和使用大量内存，带着这样的想法，需要再此返回到Trace日志，分析相关线程状态，在大量的对比分析筛选之后，很幸运的发现了如下线程（该线程只有采集TraceView才会出现），并且处于R状态。对TraceView了解的同事都知道，该任务会引起关联进程非常大CPU消耗，并且异常卡顿（主线程得不到及时响应）。

```java
"Sampling Profiler" daemon prio=9 tid=162 Native

  | group="system" sCount=1 dsCount=0 obj=0x13102220 self=0x7f5a82f800

  | sysTid=31938 nice=-6 cgrp=default sched=0/0 handle=0x7f643ff440

  | state=R schedstat=( 22112458218 4449717737 10001 ) utm=2021 stm=190 core=0 HZ=100
```

### 4.5.2 结论

综上找到了该进程高CPU的原因：采集TraceView线程需要申请大量内存不断触发进程内部GC，并且自身任务属于高耗时操作，从未导致主线程得不到及时调度和响应，触发ANR。

## 4.6  实例六：IO写文件超时

### 4.6.1分析

```java
04-01 13:12:11.572 I/InputDispatcher( 220): Application is not responding:Window{2b263310com.Android.email/com.android.email.activity.SplitScreenActivitypaused=false}.  5009.8ms since event, 5009.5ms since waitstarted

04-0113:12:11.572 I/WindowManager( 220): Input event dispatching timedout sending tocom.android.email/com.android.email.activity.SplitScreenActivity

04-01 13:12:14.123 I/Process(  220): Sending signal. PID: 21404 SIG: 3---发生ANR的时间和生成trace.txt的时间

04-01 13:12:14.123 I/dalvikvm(21404):threadid=4: reacting to signal 3 

……

04-0113:12:15.872 E/ActivityManager(  220): ANR in com.android.email(com.android.email/.activity.SplitScreenActivity)

04-0113:12:15.872 E/ActivityManager(  220): Reason:keyDispatchingTimedOut

04-0113:12:15.872 E/ActivityManager(  220): Load: 8.68 / 8.37 / 8.53

04-0113:12:15.872 E/ActivityManager(  220): CPUusage from 4361ms to 699ms ago ----CPU在ANR发生前的使用情况

 

04-0113:12:15.872 E/ActivityManager(  220):   5.5%21404/com.android.email: 1.3% user + 4.1% kernel / faults: 10 minor

04-0113:12:15.872 E/ActivityManager(  220):   4.3%220/system_server: 2.7% user + 1.5% kernel / faults: 11 minor 2 major

04-0113:12:15.872 E/ActivityManager(  220):   0.9%52/spi_qsd.0: 0% user + 0.9% kernel

04-0113:12:15.872 E/ActivityManager(  220):   0.5%65/irq/170-cyttsp-: 0% user + 0.5% kernel

04-0113:12:15.872 E/ActivityManager(  220):   0.5%296/com.android.systemui: 0.5% user + 0% kernel

04-0113:12:15.872 E/ActivityManager(  220): 100%TOTAL: 4.8% user + 7.6% kernel + 87% iowait

04-0113:12:15.872 E/ActivityManager(  220): CPUusage from 3697ms to 4223ms later:-- ANR后CPU的使用量======later是标志

04-0113:12:15.872 E/ActivityManager(  220):   25%21404/com.android.email: 25% user + 0% kernel / faults: 191 minor

04-0113:12:15.872 E/ActivityManager(  220):    16% 21603/__eas(par.hakan: 16% user + 0% kernel

04-0113:12:15.872 E/ActivityManager(  220):    7.2% 21406/GC: 7.2% user + 0% kernel

04-0113:12:15.872 E/ActivityManager(  220):    1.8% 21409/Compiler: 1.8% user + 0% kernel

04-0113:12:15.872 E/ActivityManager(  220):   5.5%220/system_server: 0% user + 5.5% kernel / faults: 1 minor

04-0113:12:15.872 E/ActivityManager(  220):    5.5% 263/InputDispatcher: 0% user + 5.5% kernel

04-0113:12:15.872 E/ActivityManager(  220): 32%TOTAL: 28% user + 3.7% kernel
```

### 4.6.2 结论

从LOG可以看出ANR的类型，CPU的使用情况，如果CPU使用量接近100%，说明当前设备很忙，有可能是CPU饥饿导致了ANR
如果CPU使用量很少，说明主线程被BLOCK了
如果IOwait很高，说明ANR有可能是主线程在进行I/O操作造成的

## 4.7 案例七： 死锁 

### 4.7.1 分析

```java
DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)
"main" prio=5 tid=1 MONITOR
  | group="main" sCount=1 dsCount=0 obj=0x2ba9c460 self=0x8e820
  | sysTid=861 nice=0 sched=0/0 cgrp=[fopen-error:2] handle=716342112
  | schedstat=( 0 0 0 ) utm=464 stm=65 core=0
  at com.android.server.am.ActivityManagerService.isUserAMonkey(ActivityManagerService.java:~6546)
  - waiting to lock <0x2c1141c8> (a com.android.server.am.ActivityManagerService) held by tid=59 (Binder Thread #6)
  at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:1273)
  at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:1545)
  at android.os.Binder.execTransact(Binder.java:338)
  at com.android.server.SystemServer.init1(Native Method)
  at com.android.server.SystemServer.main(SystemServer.java:808)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:511)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:784)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:551)
  at dalvik.system.NativeStart.main(Native Method)
```

trace中的waiting to lock <0x2c1141c8> 说明这个主线程在等待锁一个object 0x2c1141c8 (通常就是synchronized操作，这里就是com.android.server.am.ActivityManagerService类型的一个object),但被tid=59占住了， 再看看 tid=59的栈帧： 

```java
"Binder Thread #6" prio=5 tid=59 MONITOR
  | group="main" sCount=1 dsCount=0 obj=0x2c3bd838 self=0x34c5d8
  | sysTid=1120 nice=0 sched=0/0 cgrp=[fopen-error:2] handle=3460688
  | schedstat=( 0 0 0 ) utm=168 stm=48 core=0
  at com.android.server.am.BatteryStatsService.noteStopWakelock(BatteryStatsService.java:~114)
  - waiting to lock <0x2c117d50> (a com.android.internal.os.BatteryStatsImpl) held by tid=13 (ProcessStats)
  at com.android.server.PowerManagerService.noteStopWakeLocked(PowerManagerService.java:798)
  at com.android.server.PowerManagerService.releaseWakeLockLocked(PowerManagerService.java:1015)
  at com.android.server.PowerManagerService.releaseWakeLock(PowerManagerService.java:967)
  at android.os.PowerManager$WakeLock.release(PowerManager.java:319)
  at android.os.PowerManager$WakeLock.release(PowerManager.java:300)
  at com.android.server.am.ActivityStack.activityIdleInternal(ActivityStack.java:3254)
  at com.android.server.am.ActivityManagerService.activityIdle(ActivityManagerService.java:3953)
  at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:362)
  at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:1545)
  at android.os.Binder.execTransact(Binder.java:338)
  at dalvik.system.NativeStart.run(Native Method)
```

### 4.7.2 结论

tid为何没有释放锁object 0x2c1141c8呢？因为它在等到锁 object 0x2c117d50(一个com.android.internal.os.BatteryStatsImpl类型的对象)。如果大家有较丰富的捉虫经验的话，看到这， 想必都清楚了，持锁时又请求锁，极大的可能就是死锁了。

## 4.8 实例六：日志不全，缺少Trace或其它日志

### 4.8.1 分析

遇到这类问题是比较郁闷的，这个时候智能拿现有的信息进行分析，尝试找出问题或改进方向，例如缺少Trace.但是其它日志相对齐全。
例如在event日志中找到了应用ANR的大概时间点：10-14 00:40:26.010650

```java
10-14 00:40:26.010650  1132  1172 I am_anr  : [0,19746,android.process.media,952680005,Broadcast of Intent { act=android.intent.action.MEDIA_SCANNER_SCAN_FILE dat=file:///sdcard/AutoSmoke_UI30/testSwitchLetvView_20161014_003533/1476376700108.png 在flg=0x10 cmp=com.android.providers.media/.MediaScannerReceiver }]
```

在sys_log中发现ANR时进程CPU信息

```java
10-14 00:40:57.052274  1132  1172 E ANRManager: ANR in android.process.media, time=304722739
10-14 00:40:57.052274  1132  1172 E ANRManager: Reason: Broadcast of Intent { act=android.intent.action.MEDIA_SCANNER_SCAN_FILE dat=file:///sdcard/AutoSmoke_UI30/testSwitchLetvView_20161014_003533/1476376700108.png flg=0x10 cmp=com.android.providers.media/.MediaScannerReceiver }
10-14 00:40:57.052274  1132  1172 E ANRManager: Load: 37.88 / 25.54 / 20.22
10-14 00:40:57.052274  1132  1172 E ANRManager: Android time :[2016-10-14 00:40:56.95] [304754.500]
10-14 00:40:57.052274  1132  1172 E ANRManager: CPU usage from 17448ms to 0ms ago:
10-14 00:40:57.052274  1132  1172 E ANRManager:   117% 19252/com.letv.android.letvlive: 80% user + 36% kernel / faults: 684 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   110% 11620/mediaserver: 64% user + 45% kernel / faults: 23 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   41% 378/logd: 19% user + 21% kernel / faults: 17 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   22% 573/mobile_log_d: 17% user + 5.3% kernel / faults: 1123 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   18% 19286/com.letv.android.letvlive:cde: 11% user + 6.9% kernel / faults: 6029 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   18% 422/adbd: 2.1% user + 15% kernel / faults: 1722 minor
10-14 00:40:57.052274  1132  1172 E ANRManager:   17% 18392/logcat: 7.4% user + 10% kernel
```

从上面日志可以看到有两个进程CPU占用率偏高，且系统长时间CPU负载很重（Load: 37.88 / 25.54 / 20.22），尤其是ANR之前1分钟的负载达到37；由此我们可以大概率的猜测这次ANR事故是由CPU过高导致其它任务调度不及时导致，到底是不是呢？还是如其他同事认为的内存原因引起呢？下面我们继续看对应时间点的Kernel日志，关键字”lowmemorykiller“，得到如下信息：

```java
<6>[302600.931727]  (4)[10628:Cam@AuxSensorCo]lowmemorykiller: Killing 'android.browser' (28649), adj 18, score_adj 1000,

<6>[302600.931727]    to free 72464kB on behalf of 'Cam@AuxSensorCo' (10628) because

<6>[302600.931727]    cache 1000628kB is below limit 322560kB for oom_score_adj 0

<6>[302600.931727]    Free memory is 235708kB above reserved

<6>[303901.663086]  (6)[16560:Cam@AuxSensorCo]lowmemorykiller: Killing 'roid.emojistore' (15854), adj 18, score_adj 1000,

<6>[303901.663086]    to free 75636kB on behalf of 'Cam@AuxSensorCo' (16560) because

<6>[303901.663086]    cache 1292884kB is below limit 322560kB for oom_score_adj 0

<6>[303901.663086]    Free memory is 285336kB above reserved

<6>[302623.705248]  (2)[10970:Cam@AuxSensorCo]lowmemorykiller: Killing 'ews:pushservice' (6186), adj 13, score_adj 764,

<6>[302623.705248]    to free 62140kB on behalf of 'Cam@AuxSensorCo' (10970) because

<6>[302623.705248]    cache 992668kB is below limit 322560kB for oom_score_adj 0

<6>[302623.705248]    Free memory is 81320kB above reserved
```

- cache项 ：为kernel端的文件缓存cache，为了提高IO访问速度，底层系统会有选择的缓存一些文件；
- limit： 内存（文件缓存）的最低内存限制322560kB，当内存和文件缓存同时低于这个阀值，LMK变开始寻找低优先级进程查杀。
- score_adj：从上层设置到kernel经过转换后的进程优先级，adj--> score_adj; score_adj为1000，则说明被查杀的进程优先级很低。
- Free memory：当前空闲物理内存。
- [302623.705248]：Kernel开机时间戳

通过以上日志分析可以得出结论：系统可用内存（Free+Cache）整体维持在1G左右，属于良好。查杀进程间隔时间较长，不会对系统负载带来太多开销。
分析完以上日志，基本排除了内存问题引起的ANR，接下来再回到log日志，分析ANR高CPU进程的相关日志，看看能否有进一步挖掘。在log日志中，高亮进程PID(11620)，结果发现在很长一段时间内存，该进程有几十万的日志输出，此时心里或许有了希望，这么频繁的输出，且含有很多相同日志，那就说明该进程产生了大量循环，而大量循环也是高CPU的常见起因。

```java
10-14 00:40:46.035707 11620 19687 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8d70, omx=0xa3b9dfe0, i=5
10-14 00:40:46.035791 11620 19687 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xA3B9DFE0)
10-14 00:40:46.036599 11620 11620 D MtkOmxMVAMgr: [0xb3cca9f0] [ION][FreeBuffer] entry=0xa3bcf3c0, va=0xd30d7000, pa=0x47600000,size=0x180000, srcFd=0xFFFFFFFF, fd=0xFFFFFFFF, bufHdr=0xA3B9CAE0
10-14 00:40:46.037036 11620 11620 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8d28, omx=0xa3b9cae0, i=4
10-14 00:40:46.037125 11620 11620 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xA3B9CAE0)
10-14 00:40:46.037907 11620 11655 D MtkOmxMVAMgr: [0xb3cca9f0] [ION][FreeBuffer] entry=0xabbfc4e0, va=0xd3557000, pa=0x47000000,size=0x180000, srcFd=0xFFFFFFFF, fd=0xFFFFFFFF, bufHdr=0xA3B9C0C0
10-14 00:40:46.038281 11620 11655 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8ce0, omx=0xa3b9c0c0, i=3
10-14 00:40:46.038364 11620 11655 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xA3B9C0C0)
10-14 00:40:46.039097 11620 11657 D MtkOmxMVAMgr: [0xb3cca9f0] [ION][FreeBuffer] entry=0xa3bcf240, va=0xd3f80000, pa=0x46c00000,size=0x180000, srcFd=0xFFFFFFFF, fd=0xFFFFFFFF, bufHdr=0xA3B9C120
10-14 00:40:46.039734 11620 11657 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8c98, omx=0xa3b9c120, i=2
10-14 00:40:46.039829 11620 11657 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xA3B9C120)
10-14 00:40:46.041510 11620 11653 D MtkOmxMVAMgr: [0xb3cca9f0] [ION][FreeBuffer] entry=0xa3bcf6f0, va=0xdb528000, pa=0x46600000,size=0x180000, srcFd=0xFFFFFFFF, fd=0xFFFFFFFF, bufHdr=0xA3B9DF20
10-14 00:40:46.041966 11620 11653 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8c50, omx=0xa3b9df20, i=1
10-14 00:40:46.042057 11620 11653 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xA3B9DF20)
10-14 00:40:46.043345 11620 11654 D MtkOmxMVAMgr: [0xb3cca9f0] [ION][FreeBuffer] entry=0xa3bcf120, va=0xdb828000, pa=0x43200000,size=0x180000, srcFd=0xFFFFFFFF, fd=0xFFFFFFFF, bufHdr=0xABBC4420
10-14 00:40:46.043756 11620 11654 D MtkOmxVdecEx: [0xe1eb7800] RemoveInputBuf frm=0xe1eb8c08, omx=0xabbc4420, i=0
10-14 00:40:46.043841 11620 11654 D MtkOmxVdecEx: [0xe1eb7800] FB in (0xABBC4420)
10-14 00:40:46.044026 11620 11654 D MtkOmxVdecEx: [0xe1eb7800] MtkOmxVdec::FreeBuffer all input buffers have been freed!!! signal mInPortFreeDoneSem(1)
```

### 4.8.2 结论

至此，导出进一步结论，应用发生ANR主要是上面两个进程高CPU引起调度不及时。至于进程高CPU的进一步原因，则需要相关模块Owner结合日志进一步分析，论证。

# 5 如何避免ANR

- UI线程尽量只做跟UI相关的工作
- 耗时的工作（比如数据库操作，I/O，连接网络或者别的有可能阻碍UI线程的操作）把它放入单独的线程处理
- 尽量用Handler来处理UIthread和别的thread之间的交互
- 使用AsyncTask处理耗时IO操作
- 使用Thread或者HandlerThread时，调用Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)设置优先级，否则仍然会降低程序响应，因为默认Thread的优先级和主线程相同
- 使用Handler处理工作线程结果，而不是使用Thread.wait()或者Thread.sleep()来阻塞主线程
- Activity的onCreate和onResume回调中尽量避免耗时的代码
- BroadcastReceiver中onReceive代码也要尽量减少耗时，建议使用IntentService处理，或者通过 handlerThread 分发执行

# 6 总结

 通过以上6类ANR实例剖析，可以看出，除了正常Receiver处理耗时操作引起的ANR之外，也会有其它因素引发此类问题，例如总体内存偏低导致交换（kswap），CPU过高导致调度不及时，Binder资源被耗尽无法及时通讯等等，发生此类问题线索较为隐晦，需要大家汇总多个日志反复对比；但是好在这类问题发生时，系统都有关键log日志输出，可以利用关键字多角度深入分析，综合对比，这类问题多数时候是可以得出有效结论，并给出优化(解决)方案；对应日志确实不足的，只能借助于测试帮忙复现，并提供更多有效日志了。除此之外，也需要对相关系统知识有更多了解，例如LMK, 进程调能，Binder通信机制。方能在分析，解决此类问题过程中，有更多的参考和衡量。

参考文章

> [ANR问题简析](https://blog.csdn.net/qzh123456/article/details/78737791)
>
> [Android 系统 ANR 分析详解](https://blog.csdn.net/qq_19923217/article/details/88342448)
>
> [Android 性能优化（五）ANR 秒变大神](https://blog.csdn.net/WHB20081815/article/details/70245594)