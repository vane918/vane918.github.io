---
layout: post
title:  "关于CountDownTimer调用cancel失效问题分析"
date:   2019-12-16 00:00:01
catalog:  true
tags:
    - android
    - CountDownTimer
---



## 1 概述

在做一个功能的时候用到了CountDownTimer，功能需求如下：开启一个10分钟的CountDownTimer，每隔10s调用一次onTick，在onTick判断添加，如果满足条件，则在onTick中调用cancel()方法取消CountDownTimer倒计时，然后马上调用start()开启新的CountDownTimer倒计时重新倒数。测试发现，在CountDownTimer的回调方法onTick()调用cancel()取消CountDownTimer失败，下面先简单介绍下CountDownTimer的使用。

## 2 CountDownTimer的使用

CountDownTimer是Android提供的一个倒计时功能，使用方法如下：

1. 创建一个类集成CountDownTimer，一般用匿名内部类实现
2. 调用步骤1实例化的CountDownTimer类的start()开启倒计时
3. 调用CountDownTimer的cancel()方法取消倒计时

下面举个例子：

```java
private CountDownTimer mCountDownTimer = new CountDownTimer(60*1000, 10*1000) {
        @Override
        public void onTick(long arg0) {
        }
        
        @Override
        public void onFinish() {
        }
    };
```

CountDownTimer的构造函数有两个参数：millisInFuture：倒计时总时间，单位是ms，上例的总时间是60s；countDownInterval：执行onTick()的时间间隔，单位是ms，上例的时间间隔是10s。

```java
public CountDownTimer(long millisInFuture, long countDownInterval) {
        mMillisInFuture = millisInFuture;
        mCountdownInterval = countDownInterval;
    }
```

下面结合源码分析CountDownTimer。

## 3 CountDownTimer源码分析

[->Z:frameworks/base/core/java/android/os/CountDownTimer.java]

### 3.1 start

```java
/**
     * Start the countdown.
     */
    public synchronized final CountDownTimer start() {
        mCancelled = false;
        if (mMillisInFuture <= 0) {
            onFinish();
            return this;
        }
        mStopTimeInFuture = SystemClock.elapsedRealtime() + mMillisInFuture;
        mHandler.sendMessage(mHandler.obtainMessage(MSG));
        return this;
    }
```

CountDownTimer的start()方法开启一个倒计时，上面的代码显示，start()主要做了两件事：

1. 将取消倒计时标记位mCancelled设为false，表示不取消倒计时
2. 利用Handler发送一个及时倒计时的消息MSG

下面看下start()方法发送消息后的处理。

### 3.2 Handler

```java
private Handler mHandler = new Handler() {

    @Override
    public void handleMessage(Message msg) {

        synchronized (CountDownTimer.this) {
            if (mCancelled) {
                return;
            }

            final long millisLeft = mStopTimeInFuture - SystemClock.elapsedRealtime();

            if (millisLeft <= 0) {
                onFinish();
            } else {
                long lastTickStart = SystemClock.elapsedRealtime();
                onTick(millisLeft);

                // take into account user's onTick taking time to execute
                long lastTickDuration = SystemClock.elapsedRealtime() - lastTickStart;
                long delay;

                if (millisLeft < mCountdownInterval) {
                    // just delay until done
                    delay = millisLeft - lastTickDuration;

                    // special case: user's onTick took more than interval to
                    // complete, trigger onFinish without delay
                    if (delay < 0) delay = 0;
                } else {
                    delay = mCountdownInterval - lastTickDuration;

                    // special case: user's onTick took more than interval to
                    // complete, skip to next interval
                    while (delay < 0) delay += mCountdownInterval;
                }

                sendMessageDelayed(obtainMessage(MSG), delay);
            }
        }
    }
};
```

调用CountDownTimer的start()方法后最后会在handleMessage()方法中处理相关的倒计时逻辑。

1. 判断取消的标记位mCancelled是否为true，是的话表示这次的倒计时结束了
2. 判断该次倒计时剩余时间millisLeft是否小于0，为true则表示倒计时结束，回调onFinish()方法，onFinish()是抽象方法，需要用户实现
3. 如果该次倒计时没结束，则首先调用onTick()方法，onTick()也是抽象方法，需要用户实现。
4. 最后调用sendMessageDelayed()方法进行下一次的计时。

### 3.3 cancel

cancel()方法可以取消倒计时，逻辑很简单，就是将取消倒计时标记位mCancelled设为true，然后取消计时的Handler的延时消息sendMessageDelayed。

```java
public synchronized final void cancel() {
        mCancelled = true;
        mHandler.removeMessages(MSG);
    }
```

### 3.4 onTick

onTick()是抽象方法，用户需要实现，根据初始化CountDownTimer构造函数的时候传入的参数countDownInterval，每隔countDownInterval毫秒都会回调onTick()，用户可以在onTick()方法中做一些统计等操作。

```java
public abstract void onTick(long millisUntilFinished);
```

### 3.5 onFinish

onFinish()也是抽象方法，用户需要实现，根据初始化CountDownTimer构造函数的时候传入的参数millisInFuture，倒计时结束后，最后会回调onFinish()方法，回调的动作是3.2小节Handler中做的，进入handleMessage后首先判断是否到达倒计时的时间millisLeft，是的话则回调了onFinish()方法。

```java
public abstract void onFinish();
```

## 4 CountDownTimer调用cancel失效问题分析

如果在onTick()方法中调用cancel()，和start()，则流程走到了handleMessage()的最后一个else分支，如下：

```java
else {
        onTick(millisLeft);
    ...
        sendMessageDelayed(obtainMessage(MSG), delay);
      }
```

cancel()方法只是将取消标记位mCancelled设为true和取消sendMessageDelayed()发送的延时操作，如果在但是onTick()方法中调用cancel()取消倒计时，然后又马上调用start()开启新的倒计时，那么问题来了，sendMessageDelayed()方法是在onTick()方法后执行的，也就是说cancel()是在sendMessageDelayed()执行前，mHandler.removeMessages(MSG)方法根本没撤回sendMessageDelayed()发送的延时消息，因为还没发送，何来的撤回。因此等到时间间隔到了之后，还是会执行handleMessage()进行计时操作。

由于在onTick()中调用cancel()后又调用start()开启新的倒计时，start()方法又将取消标记位mCancelled设为false，那么第一个倒计时时间间隔到了之后调用handleMessage处理计时，mCancelled标记位由于start()方法将其设为false，因此第一个倒计时会继续执行。

## 5 解决方案

在onTick()方法中发送一个延时消息执行cancel()和start()即可解决问题，原理是等onTick()执行完后执行sendMessageDelayed()，再调用cancel()撤回这次的消息延时。如果单单是取消倒计时，不马上重新开启新的倒计时，在5.0之后的版本是可以在onTick()中直接调用cancel()的，调用cancel()后，等到下次的时间间隔到了会执行handleMessage()，由于cancel()将mCancelled设为true，因此在handleMessage()的最前面就返回return了，故就取消了这次的倒计时。

但是在5.0之前，由于没有取消倒计时标记位mCancelled，因此在onTick()方法执行cancel()取消倒计时就失败了，解决方案也是发一个延时消息执行cancel()，原理是等onTick()执行完后执行sendMessageDelayed()，再调用cancel()撤回这次的消息延时。