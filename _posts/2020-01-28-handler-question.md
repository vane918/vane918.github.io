---
layout: post
title:  "关于Handler的几个问题"
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
3. Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗？
4. 子线程有哪些更新UI的方法

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

## 4 Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗

对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。
主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

## 5 子线程有哪些更新UI的方法

- 主线程中定义Handler，子线程通过mHandler发送消息，主线程Handler的handleMessage更新UI。
- 用Activity对象的runOnUiThread方法。
- 创建Handler，传入getMainLooper。
- View.post(Runnable r) 。

### 5.1 runOnUiThread

```java
new Thread(new Runnable() {
  @Override
  public void run() {
    runOnUiThread(new Runnable() {
      @Override
      public void run() {
        //DO UI method      
      }
    });
  }
}).start();


final Handler mHandler = new Handler();
public final void runOnUiThread(Runnable action) {
  if (Thread.currentThread() != mUiThread) {
    mHandler.post(action);//子线程（非UI线程）
  } else {
    action.run();
  }
}
```

进入Activity类里面，可以看到如果是在子线程中，通过mHandler发送的更新UI消息。而这个Handler是在Activity中创建的，也就是说在主线程中创建，所以便和我们在主线程中使用Handler更新UI没有差别。
因为这个Looper，就是ActivityThread中创建的Looper（Looper.prepareMainLooper())。

### 5.2 创建Handler，传入getMainLooper

那么同理，我们在子线程中，是否也可以创建一个Handler，并获取`MainLooper`，从而在子线程中更新UI呢？
首先我们看到，在`Looper`类中有静态对象`sMainLooper`，并且这个`sMainLooper`就是在ActivityThread中创建的`MainLooper`。

```java
private static Looper sMainLooper;  // guarded by Looper.class
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

所以不用多说，我们就可以通过这个`sMainLooper`来进行更新UI操作。

```java
new Thread(new Runnable() {
  @Override
  public void run() {
    Handler handler=new Handler(getMainLooper());
    handler.post(new Runnable() {
      @Override
      public void run() {
        //Do Ui method
      }
    });
  }
}).start();
```

### 5.3 View.post(Runnable r)

post的源码

```java
public boolean post(Runnable action) {
  final AttachInfo attachInfo = mAttachInfo;
  if (attachInfo != null) {
    return attachInfo.mHandler.post(action); //一般情况走这里
  }
  // Postpone the runnable until we know on which thread it needs to run.
  // Assume that the runnable will be successfully placed after attach
  getRunQueue().post(action);
  return true;
}
```

居然也是Handler从中作祟，根据Handler的注释，也可以清楚该Handler可以处理UI事件，也就是说它的Looper也是主线程的`sMainLooper`。这就是说我们常用的更新UI都是通过Handler实现的。另外更新UI 也可以通过`AsyncTask`来实现，`AsyncTask`的线程切换也是通过 Handler实现的。

## 6 Handler内存泄漏案例

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

## 7 总结

1. 只有在创建了ViewRootImpl对象后，在子线程更新ui才会报异常。
2. 创建ViewRootImpl对象是在onResume()中。
3. 线程创建handler对象需要先调用Looper.prepare()将线程的looper对象存入ThreadLocal对象。
4. 主线程设置looper对象是在app启动的时候在ActivityThread的main函数里面执行的。
5. 同一个线程只能由一个looper对象。