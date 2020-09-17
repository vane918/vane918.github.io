---
layout: post
title:  "Activity启动耗时统计方案(非原创)"
date:   2020-09-15 00:00:00
catalog:  true
tags:
    - Activity
    - 优化
    - Android
---



## 1. 概述

  Activity的启动速度是很多开发者关心的问题，当页面跳转耗时过长时，App 就会给人一种非常笨重的感觉。在遇到某个页面启动过慢的时候，开发的第一直觉一般是 onCreate 执行速度太慢了，然后在 onCreate 方法前后记录下时间戳计算出耗时。不过有时候即使把 onCreate 方法的耗时优化了，效果仍旧不明显。实际上影响到 Activity 启动速度的原因是多方面的，需要从 Activity 的启动流程入手，才能找到真正问题所在。 

## 2. Activity启动流程

如果要给 Activity 的“启动”做一个定义的话，个人觉得应该是：从调用 startActivity 到 Activity 可被操作为止，代表启动成功。所谓的可被操作，是指可接受各种输入事件，比如手势、键盘输入之类的。换个角度来说，也可以看成是主线程处于空闲状态，能执行后续进入的各种 Message。

Activity 的启动可以分为三个步骤，以 ActivityA 启动 ActivityB 为例，三步骤分别为：

1. 以 ActivityA 调用 startActivity，到 ActivityA 成功 pause 为止
2. ActivityB 成功初始化，到执行完 resume 为止
3. ActivityB 向 WSM 注册窗口，到第一帧绘制完成为止

Activity 启动涉及到 App 进程与 ActivityManagerService（AMS）、WindowManagerService（WMS）的通信，网上关于这个流程的文章很多，这边就不再具体描述了，只列一下关键方法的调用链路。

### 2.1 ActiivtyA Pause流程

 当 ActivityA 使用 startActivity 方法启动 ActivityB 时，执行函数链路如下 

```java
ActivityA.startActivity->
Instrumentation.execStartActivity->
ActivityManagerNative.getDefault.startActivity->
ActivityManagerService.startActivityAsUser->
ActivityStarter.startActivityMayWait->
ActivityStarter.startActivityLocked->
ActivityStarter.startActivityUnchecked->
ActivityStackSupervisor.resumeFocusedStackTopActivityLocked->
ActivityStack.resumeTopActivityUncheckedLocked->
ActivityStack.resumeTopActivityInnerLocked->
ActivityStack.startPausingLocked->
ActivityThread$$ApplicationThread.schedulePauseActivity->
ActivityThread.handlePauseActivity->
 └ActivityA.onPause
ActivityManagerNative.getDefault().activityPaused
```

 当 App 请求 AMS 要启动一个新页面的时候，AMS 首先会 pause 掉当前正在显示的 Activity，当然，这个 Activity 可能与请求要开启的 Activity 不在一个进程，比如点击桌面图标启动 App ,当前要暂停的 Activity 就是桌面程序 Launcher。在onPause 内执行耗时操作是一种很不推荐的做法，从上述调用链路可以看出，如果在 onPause 内执行了耗时操作，会直接影响到 ActivityManagerNative.getDefault().activityPaused() 方法的执行，而这个方法的作用就是通知 AMS，当前Activity 已经已经成功暂停，可以启动新Activity 了。 

### 2.2 ActivityB Launch流程

 在 AMS 接收到 App 进程对于 activityPaused 方法的调用后，执行函数链路如下 

```
ActivityManagerService.activityPaused->
ActivityStack.activityPausedLocked->
ActivityStack.completePauseLocked->
ActivityStackSupervisor.resumeFocusedStackTopActivityLocked->
ActivityStackSupervisor.resumeFocusedStackTopActivityLocked->
ActivityStack.resumeTopActivityUncheckedLocked->
ActivityStack.resumeTopActivityInnerLocked->
ActivityStackSupervisor.startSpecificActivityLocked->
 └1.启动新进程：ActivityManagerService.startProcessLocked 暂不展开
 └2.当前进程：ActivityStackSupervisor.realStartActivityLocked->
ActivityThread$$ApplicationThread.scheduleLaunchActivity->
Activity.handleLaunchActivity->
 └Activity.onCreate
 └Activity.onRestoreInstanceState
 └handleResumeActivity
   └Activity.onStart->
   └Activity.onResume->
   └WindowManager.addView->
```

 AMS 在经过一系列方法调用后，通知 App 进程正式启动一个 Actviity，注意如果要启动 Activity 所在进程不存在，比如点击桌面图标第一次打开应用，或者 App 本身就是多进程的，要启动的新页面处于另外一个进程，那就需要走到`ActivityManagerService.startProcessLocked` 流程，等新进程启动完毕后再通知 AMS，这里不展开。按照正常流程，会依次走过Activity生命周期内的 onCreate、onRestoreInstanceState、onStart、onResume 方法，这一步的耗时基本也可以看成就是这四个方法的耗时，由于这四个方法是同步调用的，所以可以通过以 onCreate 方法为起点，onResume 方法为终点，统计出这一步骤的总耗时。 

### 2.3 ActivityB Render流程

 在 ActivityB 执行完 onResume 方法后，就可以显示该 Activity了，调用流程如下 

```
WindowManager.addView->
WindowManagerImpl.addView->
ViewRootImpl.setView->
ViewRootImpl.requestLayout->
 └ViewRootImpl.scheduleTraversals->
 └Choreographer.postCallback->
WindowManagerSerivce.add
```

 这一步的核心实际上是 `Choreographer.postCallback`，向 Choreographer 注册了一个回调，当 Vsync 事件到来时，就会执行下面的回调进行 ui 的渲染 

```
ViewRootImpl.doTraversal->
ViewRootImpl.performTraversals->
 └ViewRootImpl.relayoutWindow
 └ViewRootImpl.performMeasure
 └ViewRootImpl.performLayout
 └ViewRootImpl.performDraw
ViewRootImpl.reportDrawFinished
```

 这里分别执行了 performMeasure、performLayout、performDraw，实际上就是对应到 DecorView 的测量、布局、绘制三个流程。由于 Android 的 UI 是个树状结构，作为根 View 的 DecorView 的测量、布局、绘制，会调用到所有子 View 相应的方法，因此，这一步的总耗时就是所有子 View 在测量、布局、绘制中的耗时之和，如果某个子 View 在这三个方法中如果进行了耗时操作，就会拖慢整个 UI 的渲染，进而影响 Activity 第一帧的渲染速度。 

## 3. 耗时统计方案

知道了 Actviity 启动流程的三个步骤和对应的方法耗时统计方法，那该如何设计一个统计方案呢？在这之前，可以先看看系统提供的耗时统计方法。

### 3.1 系统耗时统计

打开 Android Studio 的 Logcat ,输入过滤关键字 ActivityManager ,在启动一个 Actviity 后就能看到如下日志

`ActivityManager: Displayed com.vane.appstartupdemo/.ActivityB: +59ms`

 末尾的+59ms便是启动该 Activity 的耗时，这个日志是 Android 系统在 AMS 端直接输出的。
 简单来说，上述日志是通过`ActivityRecord.reportLaunchTimeLocked`方法打印出来的 

```java
ActivityRecord.java
   
private void reportLaunchTimeLocked(final long curTime) {
    ......
    final long thisTime = curTime - displayStartTime;
    final long totalTime = stack.mLaunchStartTime != 0
            ? (curTime - stack.mLaunchStartTime) : thisTime;
    if (SHOW_ACTIVITY_START_TIME) {
        Trace.asyncTraceEnd(TRACE_TAG_ACTIVITY_MANAGER, "launching: " + packageName, 0);
        EventLog.writeEvent(AM_ACTIVITY_LAUNCH_TIME,
                userId, System.identityHashCode(this), shortComponentName,
                thisTime, totalTime);
        StringBuilder sb = service.mStringBuilder;
        sb.setLength(0);
        sb.append("Displayed ");
        sb.append(shortComponentName);
        sb.append(": ");
        TimeUtils.formatDuration(thisTime, sb);
        if (thisTime != totalTime) {
            sb.append(" (total ");
            TimeUtils.formatDuration(totalTime, sb);
            sb.append(")");
        }
        Log.i(TAG, sb.toString());
    }
```

 其中 `displayStartTime` 是在 `ActivityStack.setLaunchTime()` 方法中设置的，具体调用链路： 

```
ActivityStackSupervisor.startSpecificActivityLocked->
   └ActivityStack.setLaunchTime
ActivityStackSupervisor.realStartActivityLocked->
ActivityThread$$ApplicationThread.scheduleLaunchActivity->
Activity.handleLaunchActivity->
ActivityThread$$ApplicationThread.scheduleLaunchActivity->
Activity.handleLaunchActivity->
```

在`ActivityStackSupervisor.startSpecificActivityLocked`方法中调用了`ActivityStack.setLaunchTime()`，而`startSpecificActivityLocked`方法最终会走到 App 端的 `Activity.onCreate` 方法，所以统计开始的时间实际上就是App 启动中的第二步开始的时间。
而 `ActivityRecord.reportLaunchTimeLocked`方法自身的调用链如下：

```
ViewRootImpl.reportDrawFinished->
Session.finishDrawing->
WindowManagerService.finishDrawingWindow->
WindowSurfacePlacer.requestTraversal->
WindowSurfacePlacer.performSurfacePlacement->
WindowSurfacePlacer.performSurfacePlacementLoop->
RootWindowContainer.performSurfacePlacement->
WindowSurfacePlacer.handleAppTransitionReadyLocked->
WindowSurfacePlacer.handleOpeningApps->
AppWindowToken.updateReportedVisibilityLocked->
AppWindowContainerController.reportWindowsDrawn->
ActivityRecord.onWindowsDrawn->
ActivityRecord.reportLaunchTimeLocked
```

在启动流程第三步 UI 渲染完成后，App 会通知 WMS，紧接着 WMS 执行一系列和切换动画相关的方法后，调用到`ActivityRecord.reportLaunchTimeLocked`，最终打印出启动耗时。
由上述流程可以看到，系统统计并没有把 ActivityA 的 pause 操作耗时计入 Activity 启动耗时中。不过，如果我们在ActivityA 的 onPause 中做一个 Thread.sleep(2000) 操作，会很神奇地看到系统打印的耗时也跟着变了

`ActivityManager: Displayed com.vane.appstartupdemo/.ActivityB: +1s571ms`

 这次启动耗时变成了1s571ms，明显是把 onPause 的时间算进去了，但是却小于 onPause 内休眠的2秒。其实，这是由于 AMS 对于 pause 操作的超时处理导致的，在 `ActivityStack.startPausingLocked`方法中，会执行到`schedulePauseTimeout`方法 

```java
ActivityThread.Java
    
private static final int PAUSE_TIMEOUT = 500;
private void schedulePauseTimeout(ActivityRecord r) {
    final Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
    msg.obj = r;
    r.pauseTime = SystemClock.uptimeMillis();
    mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Waiting for pause to complete...");
}
...  
private class ActivityStackHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case PAUSE_TIMEOUT_MSG: {
                ActivityRecord r = (ActivityRecord)msg.obj;
                // We don't at this point know if the activity is fullscreen,
                // so we need to be conservative and assume it isn't.
                Slog.w(TAG, "Activity pause timeout for " + r);
                synchronized (mService) {
                    if (r.app != null) {
                        mService.logAppTooSlow(r.app, r.pauseTime, "pausing " + r);
                    }
                    activityPausedLocked(r.appToken, true);
                }
            } break;
```

 这个方法的作用在于，如果过了500ms，上一个要暂停 Activity 的进程还没有回调 `activityPausedLocked` 方法，AMS就会自己调用 activityPausedLocked 方法，继续之后的 Launch 流程。所以过了500ms之后，AMS 就会通知 App 进程启动ActivityB 的操作，然而此时 App 进程仍旧被 onPause 的 Thread.sleep 阻塞着，所以只能再等待1.5s才能继续操作，因此打印出来的时间是 2s-0.5s+ 正常的耗时。 

### 3.2 三种耗时

 说完了系统的统计方案，接下去介绍下应用内的统计方案。根据前面的介绍，若想自己实现 Activity 的启动耗时统计功能，只需要以 startActivity 执行为起始点，以第一帧渲染为结束点，就能得出一个较为准确的耗时。不过，这种统计方式无法帮助我们定位具体的问题，当遇到一个页面启动较慢时，我们可能需要知道它具体慢在哪里。而且，由于启动过程中涉及到大量的系统进程耗时和 App 端 Framework 层的方法耗时，这块耗时又是难以对其进行干涉的，所以接下去会把统计的重点放在通过编码能影响到的耗时上，按照启动流程的三个步骤，划分为三种耗时。 

#### 3.2.1 Pause耗时

尽管启动 Activity 的起点是 startActivity 方法，但是从调用这个方法开始，到 onPause 被执行到为止，其实都是 App 端Framework 层与 AMS 之间的交互，所以这里把第一阶段 Pause 的耗时统计放在 onPause 方法开始时候。这一块的统计也很简单，只需要计算一下 onPause 方法的耗时就足够了。有些同学可能会疑惑：是否 onStop 也要计入 Pause 耗时。并不需要，onStop 操作其实是在主线程空余时才会执行的,在 `Activity.handleResumeActivity` 方法中，会执行`Looper.myQueue().addIdleHandler(new Idler())`方法，Idler 定义如下

```java
ActivityThread.java
    
private class Idler implements MessageQueue.IdleHandler {
    @Override
    public final boolean queueIdle() {
        ......
        am.activityIdle(a.token, a.createdConfig, 
        ......
        return false;
    }
}
```

addIdleHandler 表示会放入一个低优先级的任务，只有在线程空闲的时候才去执行，而 am.activityIdle 方法会通知AMS找到处于 stop 状态的 Activity，通过Binder回调`ActivityThread.scheduleStopActivity`，最终执行到 onStop。而这个时候，UI 第一帧已经渲染完毕。

#### 3.2.2 Launch耗时

Launch 耗时可以通过 onCreate、onRestoreInstanceState、onStart、onResume 四个函数的耗时相加得出。在这四个方法中，onCreate 一般是最重的那个方法，因为很多变量的初始化都会放在这里进行。另外，onCreate 方法中还有个耗时大户是 LayoutInfalter.infalte 方法，调用 setContentView 执行到这个方法，对于一些复杂布局的第一次解析，会消耗大量时间。由于这四个方法是同步顺序执行的，单独把某些操作从 onCreate 移到 onResume 之类的并没有什么意义，Launch耗时只关心这几个方法的总耗时。

#### 3.2.3 Render耗时

从 onResume 执行完成到第一帧渲染完成所花费的时间就是 Render 耗时。Render 耗时可以用三种方式计算出来。
第一种，IdleHandler：

```java
Activity.java
    
@Overrid
protected void onResume() {
    super.onResume();
    final long start = System.currentTimeMillis();
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            Log.d(TAG, "onRender cost:" + (System.currentTimeMillis() - start));
            return false;
        }
    });
}
```

前面说过 IdleHandler 只会在线程处于空闲的时候被执行。
第二种方法，DecorView 的两次 post：

```java
Activity.java
    
@Override
protected void onResume() {
    super.onResume();
    final long start = System.currentTimeMillis();
    getWindow().getDecorView().post(new Runnable() {
        @Override
        public void run() {
            new Hanlder().post(new Runnable() {
                @Override
                public void run() {
                    Log.d(TAG, "onPause cost:" + (System.currentTimeMillis() - start));
                }
            });
        }
    });
}

View.java

public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().post(action);
    return true;
}

void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    ......
    // Transfer all pending runnables.
    if (mRunQueue != null) {
        mRunQueue.executeActions(info.mHandler);
        mRunQueue = null;
    }
    ......
}

ViewRootImpl.java

private void performTraversals() {
    ......
    // host即DecorView
    host.dispatchAttachedToWindow(mAttachInfo, 0);
    .......
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    .......
    performLayout(lp, mWidth, mHeight);
    .......
    performDraw();
    .......
}
```

通过`getWindow().getDecorView()`获取到 DecorView 后，调用 post 方法，此时由于 DecorView 的 attachInfo 为空，会将这个 Runnable 放置 runQueue 中。runQueue 内的任务会在`ViewRootImpl.performTraversals`的开始阶段被依次取出执行，我们知道这个方法内会执行到 DecorView 的测量、布局、绘制操作，不过 runQueue 的执行顺序会在这之前，所以需要再进行一次 post 操作。第二次的post操作可以继续用 DecorView().post 或者其普通 Handler.post(),并无影响。此时mAttachInfo 已不为空，DecorView().post 也是调用了 mHandler.post()。

第三种方法：new Handler 的两次 post：

```java
Activity.java
 
@Override 
    protected void onResume() {
    super.onResume();
    final long start = System.currentTimeMillis();
    new Handler.post(new Runnable() {
        @Override
        public void run() {
            getWindow().getDecorView().post(new Runnable() {
                @Override
                public void run() {
                    Log.d(TAG, "onPause cost:" + (System.currentTimeMillis() - start));
                }
            });
        }
    });
}
```

 乍看一下第三种方法和第二种方法区别不大，实际上原理大不相同。这是因为`ViewRootImpl.scheduleTraversals`方法会往主线程队列插入一个屏障消息，代码如下所示： 

```java
ViewRootImpl.java
     void scheduleTraversals() {
        ......
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
       ......
    }
}
```

 屏障消息的作用在于阻塞在它之后的同步消息的执行，当我们在 onResume 方法中执行第一次 new Handler().post 方法，向主线程消息队列放入一条消息时，从前面的内容可以知道 onResume 是在`ViewRootImpl.scheduleTraversals`方法之前执行的，所以这条消息会在屏障消息之前，能被正常执行；而第二次 post 的消息就在屏障消息之后了，必须等待屏障消息被移除掉才能执行。屏障消息的移除操作在 `ViewRootImpl.doTraversal`方法 

```java
 ViewRootImpl.java
    
void doTraversal() {
        .......
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        .......
        performTraversals();
        .......
    }
}
```

 在这之后就将执行`performTraversals`方法，所以移除屏障消息后，等待`performTraversals`执行完毕，就能正常执行第二次 post 操作了。在这个地方，其实有个小技巧可以只进行一次 post 操作，就是在第一次 post 的时候进行一次小的延迟： 

```java
Activity.java
 
@Override
protected void onResume() {
    super.onResume();
    final long start = System.currentTimeMillis();
    new Handler.postDelay(new Runnable() {
        @Override
        public void run() {
           Log.d(TAG, "onPause cost:" + (System.currentTimeMillis() - start));
        }
    },10);
}
```

通过添加一点小延迟，可以把消息的执行时间延迟到屏障消息之后，这条消息就会被屏障消息阻塞，直到屏障消息被移除时才执行了。不过由于系统函数执行时间不可控，这种方式并不保险。

### 3.3 应用内统计方案

耗时统计是非常适合使用 AOP 思想来实现的功能。我们当然不希望在每个 Activity 的 onPause、onCreate、onResume等方法中进行手动方法统计，第一这会增加编码量，第二这对代码有侵入，第三对于第三方 sdk 内的 Activity 代码，无法进行修改。使用 AOP，表示需要找到一个切入点,这个切入点是 Activity 生命周期回调的入口。这里推荐两三方案。

#### 3.3.1 Hook Instrumentation

Hook Instrumentation 是指通过反射将 ActivtyThread 内的 Instrumentation 对象替换成我们自定义的 Instrumentation 对象。在插件化方案中，Hook Instrumentation 是种很常见的方式。由于所有 Activity 生命周期的回调都要经过Instrumentation 对象，因此通过 Hook Instrumentation 对象，可以很方便地统计出 Actvity 每个生命周期的耗时。以启动流程第一阶段的 Pause 耗时为例，可以这么修改 Instrumentation：

```java
public class TestInstrumentation extends Instrumentation {
    private static final String TAG="TestInstrumentation";
    private static final Instrumentation mBase;
    
    public TestInstrumentation(Instrumentation base){
        mBase = base;
    }
    .......
   
    @Override
    public void callActivityOnPause(Activity activity) {
        long startTime = System.currentTimeMillis();
        mBase.callActivityOnPause(activity);
        Log.d(TAG,"onPause cost:"+(System.currentTimeMillis()-startTime));
    }
 
    .......
}
```

而 Render 耗时，可以在 `callActivityOnResume`方法最后，通过 Post Message 的方式进行统计。
Hook Instrumentation 是种很理想的解决方案，唯一的问题是太多人喜欢 Hook 它了。由于很多功能，比如插件化都喜欢Hook Instrumentation，为了不影响他们的使用，不得不重写大量的方法执行mBase.xx()。
如果 Instrumentation 是个接口，能够使用动态代理就更理想了。

#### 3.3.2 Hook Looper-Printer

Hook Looper 是种比较取巧的方案，做法是通过`Looper.getMainLooper().setMessageLogging(Printer)`方法设置一个日志对象

```java
public static void loop() {
    ......
    for (;;) {
        ......
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        ......
        try {
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        .......
 
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
    .......
    }
}
```

在 Looper 执行消息前后，如果 Printer 对象不为空，就会各输出一段日志，而我们知道 Activity 的生命周期回调的起点其实都是 ActviityThread 内的 mH 这个 Handler，通过解析日志，就能知道当前 msg 是否是相应的生命周期任务，解析大致流程如下：

1. 匹配“>>>>> Dispatching to”和“<<<<< Finished to ”，区分msg开始和结束节点
2. 匹配 msg.target 是否等于“android.app.ActivityThread$H”，确定是否为生命周期调消息
3. 匹配 msg.what，确定当前消息码，不同生命周期回调对应不同消息码，比如LAUNCH_ACTIVITY = 100、PAUSE_ACTIVITY = 101
4. 统计开始节点和结束节点之前的耗时，就能得出响应生命周期的耗时。同样的，Render 耗时需要在 Launch 结束时，通过 Post Message 的方式得出。

这个方案的优点是不需要通过反射等方式，修改系统对象，所以安全性很高。但是通过该方法只能区分 Pause、Launch、Render 三个步骤的相应耗时，无法细分 Launch 方法中各个生命周期的耗时，因为是以每个消息的执行为统计单位，而 Launch 消息实际上同时包含了 onCreate、onStart、onResume等的回调。更致命的一点是在 Android P 中，系统对生命周期的处理做了一次大的重构，不再细分 Pause、Launch、Stop、Finish 等消息，统一使用EXECUTE_TRANSACTION=159 来处理，而具体生命周期的处理则是用多态的方式实现。所以该方案无法兼容 Android P及以上版本

#### 3.3.3 Hook ActivityThread$H

每当 AMS 通过 Binder 调用到到 App 端时，会根据不同的调用方法转化成不同的消息放入 ActivityThread$H 这个 Handler 中，因此，只要 Hook 住了 ActivityThread$H，就能得到所有生命周期的起点。
另外，Handler 事实上可以设置一个 mCallback 字段（需要通过反射设置），在执行 dispatchMessage 方法时，如果mCallback 不为空，则优先执行 mCallback

```java
Handler.java
    
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

 因此，可以通过反射获取 ActivityThread 中的H对象，将 mCallback 修改为自己实现的 Handler.Callback 对象，实现消息的拦截,而不需要替换 Hanlder 对象 

```java
class ProxyHandlerCallback implements Handler.Callback {
 
    //设置当前的callback，防止其他sdk也同时设置了callback被覆盖
    public final Handler.Callback mOldCallback;
    public final Handler mHandler;
 
    ProxyHandlerCallback(Handler.Callback oldCallback, Handler handler) {
        mOldCallback = oldCallback;
        mHandler = handler;
    }
 
    @Override
    public boolean handleMessage(Message msg) {
        // 处理消息开始，同时返回消息类型，主要为了兼容Android P，把159消息转为101(Pause)和100(Launch)
        int msgType = preDispatch(msg);
        // 如果旧的callback返回true,表示已经被它拦截，而它内部必定调用了Handler.handleMessage,直接返回
        if (mOldCallback != null && mOldCallback.handleMessage(msg)) {
            postDispatch(msgType);
            return true;
        }
        // 直接调用handleMessage执行消息处理
        mHandler.handleMessage(msg);
        // 处理消息结束
        postDispatch(msgType);
        // 返回true,表示callback会拦截消息，Hanlder不需要再处理消息因为我们上一步已经处理过了
        return true;
    }
    .......
}
```

 为了统计 mHandler.handleMessage(msg) 方法耗时，Callback 的 handleMessage 方法会返回 true。preDispatch 和 postDispatch 的处理和 Hook Looper 流程差不多，不过增加了 Android P 下，消息类行为159时的处理 。 
和 Hook Looper 一样，Hook Hanlder 也有个缺点是无法分别获取 Launch 中各个生命周期的耗时。 

## 总结

最后做下总结：

1. Activity 的启动分为 Pause、Launch 和 Render 三个步骤，在启动一个新 Activity 时，会先 Pause 前一个正在显示的Activity，再加载新 Activity，然后开始渲染，直到第一帧渲染成功，Activity 才算启动完毕
2. 可以利用 Logcat 查看系统输出的 Activity 启动耗时，系统会统计 Activity Launch+Render 的时间做为耗时时间，而系统最多允许 Pause 操作超时500ms，到时见就会自己调用Pause完成方法进行后续流程。
3. 可以使用 Hook Instrumentation、Hook Looper、Hook Handler三种方式实现 AOP 的耗时统计，其中 Hook Looper 方式无法兼容 Android P。

转自：[Android Activity启动耗时统计方案](https://segmentfault.com/a/1190000020262028)