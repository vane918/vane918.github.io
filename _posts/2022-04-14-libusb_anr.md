---
layout: post
title:  "ANR案例分析"
date:   2022-04-14 00:00:00
catalog:  true
tags:
    - android
    - ANR
---

# 背景

UVC camera压测冷启动时，在关闭camera时导致ANR。

# 分析traces文件

```
----- pid 2821 at 1970-02-21 07:37:10 -----
Cmd line: com.angzv.camera
...
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 obj=0x73edb530 self=0xe6005400
  | sysTid=2821 nice=-8 cgrp=default sched=0/0 handle=0xe8cc9534
  | state=S schedstat=( 220940919719 9167305615 904553 ) utm=9225 stm=12869 core=6 HZ=100
  | stack=0xff7b8000-0xff7ba000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/2821/stack)
  native: #00 pc 000176d8  /system/lib/libc.so (syscall+28)
  native: #01 pc 0004780d  /system/lib/libc.so (_ZL33__pthread_mutex_lock_with_timeoutP24pthread_mutex_internal_tbPK8timespec+520)
  native: #02 pc 0005fe24  /data/app/com.angzv.camera-1/lib/arm/libANG.so (libusb_lock_events+32)
  native: #03 pc 00059ed4  /data/app/com.angzv.camera-1/lib/arm/libANG.so (libusb_close+128)
  native: #04 pc 000543cd  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #05 pc 000326fd  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #06 pc 0002bd09  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #07 pc 00023a63  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #08 pc 00025ba5  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #09 pc 0000c8d1  /data/app/com.angzv.camera-1/lib/arm/libOpenNI2.so (???)
  native: #10 pc 0001211b  /data/app/com.angzv.camera-1/lib/arm/libOpenNI2.so (???)
  native: #11 pc 0001978d  /data/app/com.angzv.camera-1/oat/arm/base.odex (Java_org_openni_NativeMethods_oniDeviceClose__J+88)
  at org.openni.NativeMethods.oniDeviceClose(Native method)
  at org.openni.Device.close(Device.java:4)
  at com.angstrong.camera.ANGCamera.softReset(ANGCamera.java:11)
  - locked <0x099add06> (a java.lang.Object)
  at com.angstrong.camera.ANGCamera.access$3400(ANGCamera.java:1)
  at com.angstrong.camera.ANGCamera$UsbDeviceReceiver.onReceive(ANGCamera.java:19)
  at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:1122)
  at android.os.Handler.handleCallback(Handler.java:751)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:154)
  at android.app.ActivityThread.main(ActivityThread.java:6121)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:889)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:779)
...
```

line 1 表明发生ANR的时间和发生ANR的进程：2821，line 2 表明发生ANR的包名：com.angzv.camera。

根据pid 2821，找到sysTid=2821所在的块（line 4-36），由native: #03 pc 00059ed4和native: #02 pc 0005fe24可以用addr2line工具找到对应的代码，具体如下：

```
addr2line.exe -e D:\xxx\libANG.so 00059ed4
D:/xxx/libusb/core.c:1435

addr2line.exe -e D:\xxx\libANG.so 0005fe24
D:/xxx/libusb/io.c:1701
```

native: #03 pc 00059ed4 对应的源码是：

core.c

libusb_close的libusb_lock_events()函数

```c
void API_EXPORTED libusb_close(libusb_device_handle *dev_handle) {
...

	/* take event handling lock */
	libusb_lock_events(ctx);	// XXX crash
	{
		...
	}
	/* Release event handling lock and wake up event waiters */
	libusb_unlock_events(ctx);
}
```

native: #02 pc 0005fe24对应的源码是：

io.c

usbi_mutex_lock(&ctx->events_lock)函数。

```c
void API_EXPORTED libusb_lock_events(libusb_context *ctx) {

	USBI_GET_CONTEXT(ctx);
	usbi_mutex_lock(&ctx->events_lock);
	ctx->event_handler_active = 1;
}
```

初步分析可以知道线程**2821**在等待ctx->events_lock锁，但该锁一直被其他线程持有，而**2821**线程是主线程，因此超过了一定时间后主线程没返回，发生了ANR,因此**工作的难点就是找到哪个线程持有该锁**。

在libusb源码里有很多地方都会持有events_lock，因此逐个排查工作量很大，因此换个思路继续分析。分析ANR出了traces.txt文件外，logcat文件也是重要的部分。该案例中，可以从logcat文件中得出线程**2926**在执行硬复位时，没有返回，这是个重要的线索。

在traces文件中尝试搜索**2926**关键字，可以搜到下面的信息：

```
"thread_work" prio=5 tid=19 Native
  | group="main" sCount=1 dsCount=0 obj=0x32c58820 self=0xc6c0d600
  | sysTid=2926 nice=0 cgrp=default sched=0/0 handle=0xca0ff920
  | state=S schedstat=( 16910575424 2376872626 103811 ) utm=649 stm=1042 core=5 HZ=100
  | stack=0xc9ffd000-0xc9fff000 stackSize=1038KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/2926/stack)
  native: #00 pc 000176dc  /system/lib/libc.so (syscall+32)
  native: #01 pc 00046b6d  /system/lib/libc.so (_ZL24__pthread_cond_timedwaitP23pthread_cond_internal_tP15pthread_mutex_tbPK8timespec+102)
  native: #02 pc 000604e0  /data/app/com.angzv.camera-1/lib/arm/libANG.so (libusb_handle_events_timeout_completed+992)
  native: #03 pc 00060bc0  /data/app/com.angzv.camera-1/lib/arm/libANG.so (libusb_handle_events_completed+36)
  native: #04 pc 00061204  /data/app/com.angzv.camera-1/lib/arm/libANG.so (libusb_control_transfer+316)
  native: #05 pc 00054bff  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #06 pc 00032df5  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #07 pc 00033ef1  /data/app/com.angzv.camera-1/lib/arm/libANG.so (???)
  native: #08 pc 0001978d  /data/app/com.angzv.camera-1/oat/arm/base.odex (Java_org_openni_NativeMethods_oniHardResetCamera__J+88)
  at org.openni.NativeMethods.oniHardResetCamera(Native method)
  at org.openni.Device.hardResetCamera(Device.java:1)
  - locked <0x0880f163> (a org.openni.Device)
  at com.angstrong.camera.ANGCamera.hardReset(ANGCamera.java:5)
  at com.angstrong.camera.ANGCamera.hardResetDevice(ANGCamera.java:2)
  at com.angzv.camera.utils.CameraAPI.hardResetDevice(CameraAPI.java:1)
  at com.angzv.camera.activity.stable.ColdZVResetActivity.b(ColdZVResetActivity.java:6)
  at c.c.a.a.e.b.run(lambda:-1)
  at android.os.Handler.handleCallback(Handler.java:751)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:154)
  at android.os.HandlerThread.run(HandlerThread.java:61)
```

上面的traces信息由line 19 可知 线程**2926**持有锁**0x0880f163** ，一直未释放。根据上面的pc可知执行硬复位的流程如下：

libusb_control_transfer->libusb_fill_control_transfer->libusb_submit_transfer->sync_transfer_wait_for_completion

​	libusb_handle_events_completed->libusb_handle_events_timeout_completed->libusb_wait_for_event

```
usbi_cond_timedwait(&ctx->event_waiters_cond,
   &ctx->event_waiters_lock, &timeout)
```

由于libusb_control_transfer传入的超时时间是0，0表示永远等待事件的返回，但event_waiters_cond一直都没有满足，导致了堵塞在usbi_cond_timedwait中，从而导致没释放锁**event_waiters_lock**：**0x0880f163**。

根据**0x0880f163**，可以找到下面的traces信息：

```
"ang_ito_read" prio=5 tid=18 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x32e07310 self=0xcb3de900
  | sysTid=2899 nice=0 cgrp=default sched=0/0 handle=0xca700920
  | state=S schedstat=( 6452388606 123246025 40949 ) utm=503 stm=142 core=5 HZ=100
  | stack=0xca5fe000-0xca600000 stackSize=1038KB
  | held mutexes=
  at org.openni.Device.getFirmwareInfo(Device.java:-1)
  - waiting to lock <0x0880f163> (a org.openni.Device) held by thread 19
  at com.angstrong.camera.ANGCamera.getCameraState(ANGCamera.java:2)
  at com.angstrong.camera.ANGCamera.b(ANGCamera.java:7)
  at c.b.a.c.run(lambda:-1)
  at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:428)
  at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:278)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:273)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1133)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:607)
  at java.lang.Thread.run(Thread.java:761)
...
```

可以看到线程**2899** ang_ito_read堵塞了，由上面的line 8 可知堵塞的原因是在等待**0x0880f163**锁。

目前初步了解到信息如下：

线程**2926**由于执行hardReset硬复位，持有了**0x0880f163** 锁，但未释放，导致线程**2899**在申请**0x0880f163** 锁时Blocked。但主线程2821等待的是**events_lock**锁，但现场2926持有的是**event_waiters_lock**，只可能是某些地方先持有了events_lock，然后再持有event_waiters_lock，由于未释放event_waiters_lock，从而导致events_lock也没释放。

在libusb源码里面查找嵌套events_lock和event_waiters_lock的逻辑块，发现如下代码块存在该嵌套锁，如下：

android_usbfs.c

```c
static int op_handle_events(struct libusb_context *ctx, struct pollfd *fds,
      POLL_NFDS_TYPE nfds, int num_ready) {
   ...
      if (pollfd->revents & POLLERR) {
         usbi_remove_pollfd(HANDLE_CTX(handle), hpriv->fd);
         usbi_mutex_lock(&ctx->events_lock); 
         usbi_handle_disconnect(handle);
         usbi_mutex_unlock(&ctx->events_lock);
        ...
         continue;
      }

      do {
         r = reap_for_handle(handle);
      } while (r == 0);
      if (r == 1 || r == LIBUSB_ERROR_NO_DEVICE)
         continue;
      else if (r < 0)
         goto out;
   }

   r = 0;
out:
   usbi_mutex_unlock(&ctx->open_devs_lock);
   return r;
}
```

line 6先持有了events_lock锁，然后执行usbi_handle_disconnect。

io.c

usbi_handle_disconnect

```c
void usbi_handle_disconnect(struct libusb_device_handle *handle) {

   struct usbi_transfer *cur;
   struct usbi_transfer *to_cancel;

   usbi_dbg("device %d.%d",
      handle->dev->bus_number, handle->dev->device_address);

   /* terminate all pending transfers with the LIBUSB_TRANSFER_NO_DEVICE
    * status code.
    *
    * this is a bit tricky because:
    * 1. we can't do transfer completion while holding flying_transfers_lock
    *    because the completion handler may try to re-submit the transfer
    * 2. the transfers list can change underneath us - if we were to build a
    *    list of transfers to complete (while holding lock), the situation
    *    might be different by the time we come to free them
    *
    * so we resort to a loop-based approach as below
    *
    * This is safe because transfers are only removed from the
    * flying_transfer list by usbi_handle_transfer_completion and
    * libusb_close, both of which hold the events_lock while doing so,
    * so usbi_handle_disconnect cannot be running at the same time.
    *
    * Note that libusb_submit_transfer also removes the transfer from
    * the flying_transfer list on submission failure, but it keeps the
    * flying_transfer list locked between addition and removal, so
    * usbi_handle_disconnect never sees such transfers.
    */

   while (1) {
      usbi_mutex_lock(&HANDLE_CTX(handle)->flying_transfers_lock);
      to_cancel = NULL;
      list_for_each_entry(cur, &HANDLE_CTX(handle)->flying_transfers, list, struct usbi_transfer)
         if (USBI_TRANSFER_TO_LIBUSB_TRANSFER(cur)->dev_handle == handle) {
            to_cancel = cur;
            break;
         }
      usbi_mutex_unlock(&HANDLE_CTX(handle)->flying_transfers_lock);

      if (!to_cancel)
         break;

      usbi_dbg("cancelling transfer %p from disconnect",
          USBI_TRANSFER_TO_LIBUSB_TRANSFER(to_cancel));

      usbi_backend->clear_transfer_priv(to_cancel);
      usbi_handle_transfer_completion(to_cancel, LIBUSB_TRANSFER_NO_DEVICE);
   }

}
```

io.c

usbi_handle_transfer_completion

```c
int usbi_handle_transfer_completion(struct usbi_transfer *itransfer,
   enum libusb_transfer_status status) {

   struct libusb_transfer *transfer =
      USBI_TRANSFER_TO_LIBUSB_TRANSFER(itransfer);
   struct libusb_context *ctx = TRANSFER_CTX(transfer);
   struct libusb_device_handle *handle = transfer->dev_handle;
   uint8_t flags;
   int r = 0;

   /* FIXME: could be more intelligent with the timerfd here. we don't need
    * to disarm the timerfd if there was no timer running, and we only need
    * to rearm the timerfd if the transfer that expired was the one with
    * the shortest timeout. */

   usbi_mutex_lock(&ctx->flying_transfers_lock);
   {
      list_del(&itransfer->list);
      if (usbi_using_timerfd(ctx))
         r = arm_timerfd_for_next_timeout(ctx);
   }
   usbi_mutex_unlock(&ctx->flying_transfers_lock);
   if (usbi_using_timerfd(ctx) && (r < 0))
      return r;

   if (status == LIBUSB_TRANSFER_COMPLETED
         && transfer->flags & LIBUSB_TRANSFER_SHORT_NOT_OK) {
      int rqlen = transfer->length;
      if (transfer->type == LIBUSB_TRANSFER_TYPE_CONTROL)
         rqlen -= LIBUSB_CONTROL_SETUP_SIZE;
      if (rqlen != itransfer->transferred) { // XXX itransfer->transferred is almost always zero on iso transfer mode...
         usbi_dbg("interpreting short transfer as error");
         LOGI("interpreting short transfer as error:rqlen=%d,transferred=%d", rqlen, itransfer->transferred);
         status = LIBUSB_TRANSFER_ERROR;
      }
   }

   flags = transfer->flags;
   transfer->status = status;
   transfer->actual_length = itransfer->transferred;  // XXX therefore transfer->actual_length is also almost always zero on iso transfer mode
   usbi_dbg("transfer %p has callback %p", transfer, transfer->callback);
   if LIKELY(transfer->callback)
      transfer->callback(transfer);
   /* transfer might have been freed by the above call, do not use from
    * this point. */
   if (flags & LIBUSB_TRANSFER_FREE_TRANSFER)
      libusb_free_transfer(transfer);
   usbi_mutex_lock(&ctx->event_waiters_lock);
   {
      usbi_cond_broadcast(&ctx->event_waiters_cond);
   }
   usbi_mutex_unlock(&ctx->event_waiters_lock);
   libusb_unref_device(handle->dev);
   return LIBUSB_SUCCESS;
}
```

上面的line 48显示会申请event_waiters_lock锁，该锁在执行硬复位时已经申请持有，但由于硬复位时内核层没返回，导致硬复位函数没返回，从而导致持有的event_waiters_lock没释放，导致堵塞在上面line 48行的usbi_mutex_lock(&ctx->event_waiters_lock)，最终导致上面的android_usbfs.c的op_handle_events函数申请的usbi_mutex_lock(&ctx->events_lock)没释放。因此，当调用libusb_close时，一直申请不了events_lock锁，最终导致了ANR.

# 结论

该ANR案例的本质是申请的锁没释放，导致后续再次申请该锁失败，导致了ANR。该案例涉及两个锁：events_lock和event_waiters_lock，由于event_waiters_lock锁没释放，间接导致了events_lock锁没释放。当再次申请events_lock锁时，发生了堵塞，从而产生了ANR。该案例之所以没释放event_waiters_lock锁，是因为调用硬复位时，是通过调用ioctrl系统调用，但该系统调用一直没返回，从而导致没释放event_waiters_lock锁。

解决的方案是硬复位设置超时机制，从而避免异常时没释放event_waiters_lock锁。
