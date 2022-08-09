---
layout: post
title:  "libusb线程竞争导致的crash案例分析"
date:   2022-08-04 00:00:00
catalog:  true
tags:
    - android
    - libusb
    - crash
---

# 背景

USB camera压测模式切换时导致libusb crash。

# 分析traces文件

```
08-04 16:49:43.081 29652 29652 I crash_dump32: performing dump of process 29543 (target tid = 29611)
08-04 16:49:43.100 29652 29652 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
08-04 16:49:43.101 29652 29652 F DEBUG   : Build fingerprint: 'rockchip/Point-of-sales/rk3288_Android10:10/QD4A.200805.003/eng.harris.20210617.212156:userdebug/release-keys'
08-04 16:49:43.101 29652 29652 F DEBUG   : Revision: '0'
08-04 16:49:43.101 29652 29652 F DEBUG   : ABI: 'arm'
08-04 16:49:43.102 29652 29652 F DEBUG   : Timestamp: 2022-08-04 16:49:43+0800
08-04 16:49:43.102 29652 29652 F DEBUG   : pid: 29543, tid: 29611, name: ang-usb-event  >>> com.angzv.camera <<<
08-04 16:49:43.102 29652 29652 F DEBUG   : uid: 10069
08-04 16:49:43.102 29652 29652 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
08-04 16:49:43.102 29652 29652 F DEBUG   : Cause: null pointer dereference
08-04 16:49:43.102 29652 29652 F DEBUG   :     r0  00000001  r1  00000004  r2  9f467f07  r3  00000003
08-04 16:49:43.102 29652 29652 F DEBUG   :     r4  0000002a  r5  9c0257c4  r6  00000000  r7  687a0f6b
08-04 16:49:43.102 29652 29652 F DEBUG   :     r8  6ad2fe60  r9  6b153100  r10 6ad2fe60  r11 684ae0c0
08-04 16:49:43.102 29652 29652 F DEBUG   :     ip  684adbb8  sp  684ae0a0  lr  68773198  pc  687731a0
08-04 16:49:43.105 29652 29652 F DEBUG   : 
08-04 16:49:43.105 29652 29652 F DEBUG   : backtrace:
08-04 16:49:43.106 29652 29652 F DEBUG   :       #00 pc 000631a0  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
08-04 16:49:43.106 29652 29652 F DEBUG   :       #01 pc 0006185c  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (usbi_handle_transfer_completion+320) (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
08-04 16:49:43.106 29652 29652 F DEBUG   :       #02 pc 00066c64  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
08-04 16:49:43.106 29652 29652 F DEBUG   :       #03 pc 0006258c  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
08-04 16:49:43.106 29652 29652 F DEBUG   :       #04 pc 00061f38  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (libusb_handle_events_timeout_completed+672) (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
08-04 16:49:43.106 29652 29652 F DEBUG   :       #05 pc 000548bf  /data/app/com.angzv.camera-LxbVNe7n1XX2bmRYACs-eA==/lib/arm/libANG.so (BuildId: 3f49b92cc9366e1f4f62c80ec95e40f9a4009e95)
```

根据上述的#00 pc 000631a0可以使用addr2line ndk自带的工具分析指向奔溃的代码如下：

```
addr2line.exe -e D:\xxx\libANG.so 000631a0
D:/xxx/libusb/sync.c:39
```

```c
static void LIBUSB_CALL sync_transfer_cb(struct libusb_transfer *transfer)
{
	int *completed = transfer->user_data;
	*completed = 1;
	usbi_dbg("actual_length=%d", transfer->actual_length);
	/* caller interprets result and frees transfer */
}
```

指向的代码是*completed = 1;即completed指针是0x0，但还对其指向的地址操作赋值为1。

sync_transfer_cb()函数是进行libusb_control_transfer()和libusb_bulk_transfer()函数传输时的回调函数，这两个函数分别对应控制传输和批量传输，都是同步函数，如libusb_control_transfer()的源码如下：

```c
int API_EXPORTED libusb_control_transfer(libusb_device_handle *dev_handle,
	uint8_t bmRequestType, uint8_t bRequest, uint16_t wValue, uint16_t wIndex,
	unsigned char *data, uint16_t wLength, unsigned int timeout)
{
	struct libusb_transfer *transfer = libusb_alloc_transfer(0);
	unsigned char *buffer;
	int completed = 0;
	int r;

	if (UNLIKELY(!transfer)) {
		return LIBUSB_ERROR_NO_MEM;
	}

	buffer = (unsigned char*) malloc(LIBUSB_CONTROL_SETUP_SIZE + wLength);
	if (UNLIKELY(!buffer)) {
		libusb_free_transfer(transfer);
		return LIBUSB_ERROR_NO_MEM;
	}

	libusb_fill_control_setup(buffer, bmRequestType, bRequest, wValue, wIndex, wLength);
	if ((bmRequestType & LIBUSB_ENDPOINT_DIR_MASK) == LIBUSB_ENDPOINT_OUT)
		memcpy(buffer + LIBUSB_CONTROL_SETUP_SIZE, data, wLength);

	libusb_fill_control_transfer(transfer, dev_handle, buffer,
		sync_transfer_cb, &completed, timeout);
	transfer->flags = LIBUSB_TRANSFER_FREE_BUFFER;
	r = libusb_submit_transfer(transfer);
	if (UNLIKELY(r < 0)) {
		libusb_free_transfer(transfer);
		return r;
	}

	sync_transfer_wait_for_completion(transfer);

	...

	libusb_free_transfer(transfer);

	return r;
}
```

具体的逻辑如下：

1. libusb_alloc_transfer()申请libusb_transfer内存
2. libusb_fill_control_transfer()填充libusb_transfer
3. libusb_submit_transfer()提交libusb_transfer
4. sync_transfer_wait_for_completion()堵塞等待传输的返回数据，由sync_transfer_cb()回调
5. libusb_free_transfer()释放申请的libusb_transfer内存

出现问题时是回调函数sync_transfer_cb()中的*completed = 1，completed 是上面步骤2libusb_fill_control_transfer时填充的，除非执行回调函数sync_transfer_cb()时刚好执行完libusb_free_transfer()，否则completed 不可能为NULL，libusb_free_transfer()函数如下：

```c
void API_EXPORTED libusb_free_transfer(struct libusb_transfer *transfer) {

	struct usbi_transfer *itransfer;
	if (UNLIKELY(!transfer))
		return;

	if (transfer->flags & LIBUSB_TRANSFER_FREE_BUFFER && transfer->buffer)
		free(transfer->buffer);

	itransfer = LIBUSB_TRANSFER_TO_USBI_TRANSFER(transfer);
	usbi_mutex_destroy(&itransfer->lock);
	free(itransfer);
	transfer->user_data = NULL;	// XXX
}
```

上面的transfer->user_data = NULL即为将completed 置空。现在来分析下什么情况下在执行sync_transfer_cb()时刚好执行完libusb_free_transfer()。

单从libusb_control_transfer()源码来分析在执行sync_transfer_cb()时completed 是不可能为NULL的，因为只有执行完sync_transfer_cb()，sync_transfer_wait_for_completion()堵塞函数才会返回，才能继续往下执行libusb_free_transfer(transfer)，这个时候completed 才为NULL。

但如果是两个线程在同时执行libusb_control_transfer()和libusb_bulk_transfer()呢？

加入线程A执行到了libusb_free_transfer()的transfer->user_data = NULL，然后CPU切换到线程B执行sync_transfer_cb()回调函数，执行int *completed = transfer->user_data;不就就有可能将transfer->user_data置为NULL了吗？但仔细看回调函数sync_transfer_cb(struct libusb_transfer *transfer)的参数是一个libusb_transfer 指针，该指针是在libusb_control_transfer()时调用libusb_alloc_transfer()申请的，因此不同的线程回调的sync_transfer_cb(struct libusb_transfer *transfer)函数的transfer是不同的，因此线程A执行libusb_free_transfer()释放的transfer和线程B执行sync_transfer_cb()时的transfer是不同的内存。

但如果存在下面的情况呢？

线程A执行到libusb_free_transfer()的free(itransfer)函数，然后切换到线程B执行libusb_control_transfer()，然后调用libusb_alloc_transfer()申请libusb_transfer内存，如果系统将线程A刚释放的itransfer又重新分配给线程B，线程B继续执行后面的代码，这个时候CPU调度到线程A继续执行上次的代码，即从transfer->user_data = NULL，这个时候transfer其实已经又重新分配给线程B了，因此CPU调度到线程B时执行sync_transfer_cb()时，由于transfer->user_data已经被线程A置为NULL，因此对transfer->user_data写入1时，就发生了段错误。

对比测试机(Android10和Android7.1和Android5.1)发现，只有在Android10系统下发现该问题，通过加log发现Android7.1和Android5.1系统不会将线程A执行libusb_control_transfer()后释放的transfer重新分配给线程B执行的libusb_bulk_transfer()，因此不会发生上面假设的情况，也就不会发生段错误。

流程图如下：

![libusb](/images/usb/libusb.png)

# 小结

该案例发生crash是因为使用了已经释放的内存，根因是线程A和线程B在操作相同的transfer指针，解决方案1是解除线程A与线程B之间针对相同的transfer指针的资源争夺，在进入竞争临界态前加互斥锁即可。方案2将libusb_free_transfer()函数的transfer->user_data = NULL放在free(itransfer)前即可。
