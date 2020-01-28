---
layout: post
title:  "关于Handler的两个问题"
date:   2020-01-28 00:00:00
catalog:  true
tags:
    - Handler
    - Message
    - Android源码
---



## 1 概述

先思考几个问题：

1. 更新ui为什么只能在ui线程
2. 线程如何创建handler

## 2 更新ui为什么只能在ui线程

其实并不是所有情况都只能在ui线程中更新ui，例如在onCreate()、onStart()和onResume()方法中启动子线程更新ui，是可以正常更新ui的。但是在其他情况下会报异常：

```
Process: com.vanelst.test, PID: 10598
    android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
        at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:8191)
        at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:1420)
```

意思是只有在创建ui的线程才能更新ui。那为什么在onCreate()方法中开启子线程就能够更新ui呢，问题的本质原因在于更新ui时是否触发了checkThread(),该方法的作用时检测当前的线程是否时创建ui的线程。接下来从Android源码的角度分析TextView的setText()流程。

```java
private void test() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            tv.setText("test");
        }
    }).start();
}
```

### 2.1 TextView的setText()流程

```java
private void setText(CharSequence text, BufferType type,
             boolean notifyBefore, int oldlen) {
  ...
  if (mLayout != null) {
            checkForRelayout();
        }
```

看下checkForRelayout()

```java
@UnsupportedAppUsage
private void checkForRelayout() {
    ...
    requestLayout();
    invalidate();
}
```

接下来看下requestLayout()。

```java
protected ViewParent mParent;
@CallSuper
public void requestLayout() {
    ...
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    ...
}
```

mParent是ViewParent，而ViewRootImpl类是ViewParent的子类。上面的代码首先判断mParent是否为空，也就是ViewRootImpl对象，ViewRootImpl对象是在onResume()方法后才创建的，因此可以解答了前面的问题：onCreate()、onStart()和onResume()方法中启动子线程更新ui，是可以正常更新ui的。

接下来分析的ViewRootImpl的requestLayout() 。

```java
@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

终于看到了报错的地方了，更新ui时最终调用ViewRootImpl的checkThread()方法检测当前的线程是否时ui线程，不是的话抛出异常。

## 3 线程如何创建handler

handler和message通常用开在子线程调用handler的sendMessage()方法，然后在handler的handleMessage()方法中进行ui更新。在ui进程可以直接通过new Handler()的方式创建handler对象，但是在子进程时不可以直接利用new Hnadler()的方式创建handler对象的。

### 3.1 主线程创建handler

我们从源码分析下在主线程new Handler()的流程。

[->frameworks/base/core/java/android/os/Handler.java]

```java
public Handler() {
    this(null, false);
}
public Handler(@Nullable Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

上面的代码mLooper = Looper.myLooper()获取Looper对象，代码如下：

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

sThreadLocal是ThreadLocal对象，存Looper对象，如果sThreadLocal没取到对象，那么mLooper为空，抛出RuntimeException异常。那么主线程是在什么时候存入Looper对象的呢。答案是在启动app的时候。下面简单分析下启动app时sThreadLocal存入looper对象的流程。

[->frameworks/base/core/java/android/app/ActivityThread.java]

```java
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();
    ....
}
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

启动app时，会调用ActivityThread的main函数，然后调用Looper.prepareMainLooper()，先看下prepare()方法。

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

上面的代码显示，sThreadLocal存入looper对象是在prepare()方法中操作的。sThreadLocal的key是线程，value是线程的looper对象，上面的代码显示，同一个线程只能创建一个looper对象，否则会抛出RuntimeException异常。

回到上面的mLooper = Looper.myLooper()继续分析。

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

sThreadLocal取出在app启动时存入主线程的looper对象。

小结：

1. 所有的线程创建Handler对象时，都需要先调用Looper.prepare()方法，将该线程的looper对象存入sThreadLocal中。
2. 同一个线程只有一个looper对象。
3. 主线程的looper对象时在app启动的时候创建的。

### 3.2 子线程创建handler

由上面的分析可知，创建Handler对象，需要首先调用Looper.prepare()将线程的looper对象存入sThreadLocal中，下面的代码就是在子线程中创建Handler对象的代码：

```java
Handler handler;
private void test() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            Looper.prepare();
            handler = new Handler() {
                @Override
                public void handleMessage(@NonNull Message msg) {
                    super.handleMessage(msg);
                    Log.d("wyh","handler");
                }
            };
            Looper.loop();
        }
    }).start();
}
```

Looper.loop()会开启一个死循环监听发往handler的message，handler和message的机制下一篇节讲解。

## 4 Handler内存泄漏案例

在第一个activity中的子线程延时开启另一个activity。

```java
public class MainActivity extends AppCompatActivity {

    private TextView tv;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tv = findViewById(R.id.tv);
        test();
    }

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            Intent intent = new Intent();
            intent.setClassName("com.vanelst.test","com.vanelst.test.SecondActivity");
            startActivity(intent);
        }
    };

    private void test() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                Message message = new Message();
                SystemClock.sleep(10000);
                message.what = 1;
                mHandler.sendMessage(message);
            }
        }).start();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mHandler.removeMessages(1);
        Log.d("wyh","onDestroy");
    }
```

按照上面的代码，当启动activity10s内关闭，虽然回执行onDestroy()方法，调用mHandler.removeMessages(1)，但是依旧会启动SecondActivity。原因是利用sleep()方法进行延时时，销毁MainActivity调用onDestroy()的mHandler.removeMessages(1)，但此刻还没执行mHandler.sendMessage(message)，因此取消不了mHandler的延时消息。解决的方案可以将run的方法改为以下代码：

```
public void run() {
    Message message = new Message();
    message.what = 1;
    mHandler.sendMessageDelayed(message,10000);
}
```

利用mHandler.sendMessageDelayed(message,10000)来延时发送消息。如果非要执行sleep()，可以在onDestroy()方法中将mHandler设为null，然后在sendMessage时判断mHandler非空，非空才执行sendMessage()。

## 5 总结

1. 只有在创建了ViewRootImpl对象后，在子线程更新ui才会报异常。
2. 创建ViewRootImpl对象是在onResume()中。
3. 线程创建handler对象需要先调用Looper.prepare()将线程的looper对象存入ThreadLocal对象。
4. 主线程设置looper对象是在app启动的时候在ActivityThread的main函数里面执行的。
5. 同一个线程只能由一个looper对象。