---
layout: post
title:  "Android消息机制-Handler源码分析"
date:   2020-01-29 00:00:00
catalog:  true
tags:
    - Handler
    - Message
    - 消息机制
    - Android源码
---

## 1 概述

Android的消息机制简单来说指的是Handler与MessageQueue、Message和Looper完成一个体系机制，它的使用很简单，Handler消息机制用于同进程的线程间通信。通过它可以轻松的将一个任务切换到Handler所在的线程去执行，我们用的最多的地方也就是Handler刷新UI的功能，其实，它还有很多用途，比如子线程中进行消耗的I/O操作，读取文件等，而我们用它来刷新UI，是因为子线程无法刷新UI，可以通过它来切换回主线程执行，因此，本质来说，它只是常用来刷新UI而已。

## 2 消息机制

先放张消息机制的原理简图：

![handler](/images/handler/handler.png)

- **Message**：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
- **MessageQueue**：消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
- **Handler**：消息辅助类，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
- **Looper**：不断循环执行(`Looper.loop`)，按分发机制将消息分发给目标处理者。

再放张handler消息机制的架构图

![handler1](/images/handler/handler1.jpg)

- **Looper**有一个MessageQueue消息队列；
- **MessageQueue**有一组待处理的Message；
- **Message**中有一个用于处理消息的Handler；
- **Handler**中有Looper和MessageQueue。

由上篇[关于Handler的两个问题](http://vanelst.site/2020/01/28/handler-question/)可知，启动app时ActivityThread会调用Looper.preare()方法，将主线程的looper对象存入ThreadLocal对象中，因此在主线程可以直接通过new Handler()的方式创建handler对象。我们简单对上图的消息机制进行梳理下。

1. handler调用sendMessage()方法，会调用enqueueMessage()往消息队列MessageQueue存入消息
2. looper对象中有个loop()方法，该方法会轮询根据消息策略，如是否延时发送等，从消息队列中取出消息
3. 消息Message对象根据先进先出规则，由消息队列的next()方法实现取出
4. 最终调用线程的handler对象的dispatchMessage()方法，最终会回到handler的handlerMessage()方法实现最终的消息消费。

下面根据源码来详细分析handler消息机制的原理。

## 3 消息机制源码分析

分析handler消息机制抓住两个主要方向就比较好把握了：发送消息和消费消息。首先在activity的子线程创建一个handler。

首先分析主线程创建looper对象的流程。

### 3.1 主线程创建looper对象

首先看下流程图

![mainthread-looper](/images/handler/mainthread-looper.png)

[->frameworks//base/core/java/android/app/ActivityThread.java]

```java
public static void main(String[] args) {
  ...
  Looper.prepareMainLooper();
  ...
}
```

app启动时调用ActivityThread.java的main()方法，然后调用Looper的prepareMainLooper()。

[->frameworks//base/core/java/android/os/Looper.java]

```java
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

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

prepare()方法可知，app第一次创建的时候，会将主线程作为key，looper对象作为value，存入ThreadLocal对象中。接下来看下初始化Looper()对象的代码。

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

Looper对象初始化创建了一个消息队列MessageQueue，由于mQueue是Looper的全局变量，因此该消息队列和looper是绑定的。上面的Looper构造函数是private的，因此不能在其他类中通过new Looper()的方式来创建Looper对象，只能通过prepare()方法创建Looper对象。

### 3.2 主线程创建Handler对象

![createhandler](/images/handler/createhandler.png)

首先在Activity中重写了Handler的handleMessage()方法。

```java
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
        handler.sendEmptyMessage(1);
    }
};
```

new Handler()做了什么事情呢？源码走起。

```java
public Handler() {
    this(null, false);
}
public Handler(@Nullable Callback callback, boolean async) {
  ...
  mLooper = Looper.myLooper();
  ...
  mQueue = mLooper.mQueue;
  ...
}
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

Looper对象是从sThreadLocal取的，在app启动的时候在ActivityThread的main()调用prepare()存入了主线程的looper了，因此主线程直接创建Handler对象是能够拿到Looper对象的。接下来重点分析handler如何发消息和接收消息的。

### 3.3 Handler发送消息

发送消息可以用handler的sendEmptyMessage、sendMessage、sendEmptyMessageDelayed、sendEmptyMessageAtTime、sendMessageAtTime。这几个方法最终都会调用Handler的enqueueMessage()方法将消息存入消息队列中。

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
    if (mAsynchronous) { 
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

enqueueMessage()最终调用MessageQueue的enqueueMessage()方法将消息压入消息队列MessageQueue中。消息队列是一种先进先出的数据结构。

```java
boolean enqueueMessage(Message msg, long when) {
  // 每一个普通Message必须有一个target
  if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
   if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
      if (mQuitting) {//正在退出时，回收msg，加入到消息池
        IllegalStateException e = new IllegalStateException(
              msg.target + " sending message to a Handler on a dead thread");
        Log.w(TAG, e.getMessage(), e);
        msg.recycle();
        return false;
    }
    msg.markInUse();
    msg.when = when;
    Message p = mMessages;
    boolean needWake;
    if (p == null || when == 0 || when < p.when) {
        //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
        // New head, wake up the event queue if blocked.
        msg.next = p;
        mMessages = msg;
        needWake = mBlocked;//当阻塞时需要唤醒
    } else {
        // Inserted within the middle of the queue.  Usually we don't have to wake
        // up the event queue unless there is a barrier at the head of the queue
        // and the message is the earliest asynchronous message in the queue.
        //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
        //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
        needWake = mBlocked && p.target == null && msg.isAsynchronous();
        Message prev;
        //将新来的msg消息插入到消息队列尾部
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
            if (needWake && p.isAsynchronous()) {
                needWake = false;
            }
        }
        msg.next = p; // invariant: p == prev.next
        prev.next = msg;
    }
    ...
    }
```

`MessageQueue`是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

### 3.4 Handler消费消息

![handlermessage](/images/handler/handlermessage.png)

在app启动时在ActivityThread的main()方法中启动了Looper.loop()方法，下面重点分析loop()方法。

```java
public static void loop() {
  //获取Looper对象
  final Looper me = myLooper();
  if (me == null) {
    throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
  }
  //获取消息队列对象  
  final MessageQueue queue = me.mQueue;
  // Make sure the identity of this thread is that of the local process,
  // and keep track of what that identity token actually is.
  Binder.clearCallingIdentity();
  //确保在权限检查时基于本地进程，而不是调用进程。
  final long ident = Binder.clearCallingIdentity();
  //进入loop的主循环方法
  for (;;) {
    //等待消息，可能会堵塞，底层采用linux的进程间机制，因此没消息主线程就休眠，有消息就唤醒，因此
    //死循环也不会产生ANR
    Message msg = queue.next(); // might block
    //没有消息，则退出循环
    if (msg == null) {
      // No message indicates that the message queue is quitting.
      return;
    }
    ...
   try {
     //用于分发Message
     msg.target.dispatchMessage(msg);
     ...
     //将Message放入消息池 
     msg.recycleUnchecked();
    }
}
```

loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

- 读取MessageQueue的下一条Message；
- 把Message分发给相应的target；
- 再把分发后的Message回收到消息池，以便重复利用。

首先看下MessageQueue读取消息的代码

```java
Message next() {
  // Return here if the message loop has already quit and been disposed.
  // This can happen if the application tries to restart a looper after quit
  // which is not supported.
  final long ptr = mPtr;
  //当消息循环已经退出，则直接返回
  if (ptr == 0) {
    return null;
  }
  // 循环迭代的首次为-1
  int pendingIdleHandlerCount = -1; // -1 only during first iteration
  int nextPollTimeoutMillis = 0;
  for (;;) {
    if (nextPollTimeoutMillis != 0) {
      Binder.flushPendingCommands();
    }
    //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
    nativePollOnce(ptr, nextPollTimeoutMillis);
    synchronized (this) {
      // Try to retrieve the next message.  Return if found.
      final long now = SystemClock.uptimeMillis();
      Message prevMsg = null;
      Message msg = mMessages;
      //当消息的Handler为空时，则查询异步消息
      if (msg != null && msg.target == null) {
        // Stalled by a barrier.  Find the next asynchronous message in the queue.
        //当查询到异步消息，则立刻退出循环
        do {
          prevMsg = msg;
          msg = msg.next;
        } while (msg != null && !msg.isAsynchronous());
      }
      if (msg != null) {
        if (now < msg.when) {
          // Next message is not ready.  Set a timeout to wake up when it is ready.
          //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
          nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)                  
        } else {
          // Got a message.
          // 获取一条消息，并返回
          mBlocked = false;
          if (prevMsg != null) {
            prevMsg.next = msg.next;
          } else {
            mMessages = msg.next;
          }
          msg.next = null;
          if (DEBUG) Log.v(TAG, "Returning message: " + msg);
          //设置消息的使用状态，即flags |= FLAG_IN_USE
          msg.markInUse();
          //成功地获取MessageQueue中的下一条即将要执行的消息
          return msg;
        }
      } else {
        // No more messages.
        //没有消息
        nextPollTimeoutMillis = -1;
      }
      // Process the quit message now that all pending messages have been handled.
      //消息正在退出，返回null
      if (mQuitting) {
        dispose();
        return null;
      }

      // If first time idle, then get the number of idlers to run.
      // Idle handles only run if the queue is empty or if the first message
      // in the queue (possibly a barrier) is due to be handled in the future.
      //当消息队列为空，或者是消息队列的第一个消息时
      if (pendingIdleHandlerCount < 0
          && (mMessages == null || now < mMessages.when)) {
        pendingIdleHandlerCount = mIdleHandlers.size();
      }
      if (pendingIdleHandlerCount <= 0) {
        // No idle handlers to run.  Loop and wait some more.
        //没有idle handlers 需要运行，则循环并等待。
        mBlocked = true;
        continue;
      }

      if (mPendingIdleHandlers == null) {
        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
      }
      mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
    }

    // Run the idle handlers.
    // We only ever reach this code block during the first iteration.
    //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
    for (int i = 0; i < pendingIdleHandlerCount; i++) {
      final IdleHandler idler = mPendingIdleHandlers[i];
      mPendingIdleHandlers[i] = null; // release the reference to the handler
      boolean keep = false;
      try {
        //idle时执行的方法
        keep = idler.queueIdle();
      } catch (Throwable t) {
        Log.wtf(TAG, "IdleHandler threw exception", t);
      }
      if (!keep) {
        synchronized (this) {
          mIdleHandlers.remove(idler);
        }
      }
    }
    // Reset the idle handler count to 0 so we do not run them again.
    //重置idle handler个数为0，以保证不会再次重复运行
    pendingIdleHandlerCount = 0;
    // While calling an idle handler, a new message could have been delivered
    // so go back and look again for a pending message without waiting.
    //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message
    nextPollTimeoutMillis = 0;
  }
}
```

`nativePollOnce`是阻塞操作，其中`nextPollTimeoutMillis`代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。

当处于空闲时，往往会执行`IdleHandler`中的方法。当nativePollOnce()返回后，next()从`mMessages`中提取一个消息。

接下来分析消息分到到activity的handler处理。

```java
public void dispatchMessage(@NonNull Message msg) {
  if (msg.callback != null) {
    //当Message存在回调方法，回调msg.callback.run()方法；
    handleCallback(msg);
  } else {
    if (mCallback != null) {
      //当Handler存在Callback成员变量时，回调方法handleMessage()；
      if (mCallback.handleMessage(msg)) {
        return;
      }
    }
    //Handler自身的回调方法handleMessage()
    handleMessage(msg);
  }
}
```

**分发消息流程：**

1. 当`Message`的回调方法不为空时，则回调方法`msg.callback.run()`，其中callBack数据类型为Runnable,否则进入步骤2；
2. 当`Handler`的`mCallback`成员变量不为空时，则回调方法`mCallback.handleMessage(msg)`,否则进入步骤3；
3. 调用`Handler`自身的回调方法`handleMessage()`，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。

对于很多情况下，消息分发后的处理方法是第3种情况，即Handler.handleMessage()，一般地往往通过覆写该方法从而实现自己的业务逻辑。

## 4 总结

再次看下消息机制的原理简图

![handler](/images/handler/handler.png)

- Handler通过sendMessage()等方法发送Message到MessageQueue队列；
- Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target(即handler)来处理；
- 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
- 将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作。