---
layout: post
title:  "Android及Linux系统下UVC创建两个video节点的原因及解决方案"
date:   2023-04-20 00:00:00
catalog:  true
tags:
    - android
    - v4l2
    - uvc
---

# 背景

在某些采用Linux内核的系统如Android和Ubuntu，插入一个UVC(usb video camera)设备会创建两个video设备节点，如/dev/video0和/dev/video1，但只有一个video设备节点是可以获取camera流数据，另外一个只能查询camera信息而已。针对该问题，手上有firefly RK3399的开发板和Android7.1的源码，下面深入源码分析该问题。

# 1. V4L2驱动框架概述

V4L2(video for linux two)是linux为视频设备提供的一套标准接口，用于控制和处理视频设备的数据。简单来说即linux为上层应用提供的操作视频设备的驱动。

## V4L2驱动框架的基本结构

V4L2驱动框架的基本结构包括以下几个模块：

1. 应用程序层(Application Layer)：应用程序通过系统调用打开摄像头设备，然后使用V4L2库函数进行视频捕获、处理等操作。

2. V4L2核心模块(V4L2 Core)：作为V4L2的核心模块，主要负责以下功能：

   - 管理摄像头设备节点；

   - 定义设备驱动程序的远程过程调用(RPC)接口函数；

   - 实现设备的控制接口。

1. V4L2子设备模块(V4L2 Subdevice)：作为子设备驱动程序，主要负责以下功能：

   - 支持设备的特定操作；

   - 支持特定视频格式和功能；

   - 支持特定设备的控制接口。

1. 设备驱动程序(Device Driver)：作为设备的驱动程序，主要负责以下功能：

   - 收集并处理摄像头采集到的数据；

   - 控制摄像头设备硬件；

   - 将收集到的数据发送给V4L2核心模块进行处理。

## V4L2驱动框架在应用中的具体实现

在实际应用中，应用程序可以基于V4L2的API编写程序，通过API来控制硬件采集数据并对视频数据进行处理。在V4L2驱动框架下，应用程序通常会调用如下函数：

- open：打开一个视频设备，获取视频设备的描述符。
- ioctl：使用ioctl函数来实现对视频设备的控制和管理，例如获取设备信息、设置视频格式、启动/停止视频设备等等。
- mmap：使用mmap函数映射视频缓冲区，来实现应用程序与驱动程序之间的数据传递。
- read：读取视频数据。
- write：写入视频数据。

# 2. UVC插入过程分析

UVC设备本质是一个USB设备，USB设备支持热插拔，当插入一个UVC设备后，具体的流程时怎么样的。

插入UVC设备后，dmesg命令可以看到一条内核log：

`uvcvideo: Found UVC 1.00 device USB 2.0 Camera (05a3:9230)`

在kernel的驱动目录/kernel/drivers下搜索“Found UVC”，结果如下：

```
/kernel/drivers$ find . -name *.c | xargs grep -nr --color "Found UVC"
./media/usb/uvc/uvc_driver.c:2066:      uvc_printk(KERN_INFO, "Found UVC %u.%02x device %s (%04x:%04x)\n",
```

该log是uvc_driver.c的`uvc_probe`函数打印的，`uvc_probe`是探测到UVC插入后会自动调用的函数。

## 2.1 uvc_probe

```c
static int uvc_probe(struct usb_interface *intf,
		     const struct usb_device_id *id)
{
	struct usb_device *udev = interface_to_usbdev(intf);
	struct uvc_device *dev;
	const struct uvc_device_info *info =
		(const struct uvc_device_info *)id->driver_info;
	u32 quirks = info ? info->quirks : 0;
	...
	uvc_printk(KERN_INFO, "Found UVC %u.%02x device %s (%04x:%04x)\n",
		dev->uvc_version >> 8, dev->uvc_version & 0xff,
		udev->product ? udev->product : "<unnamed>",
		le16_to_cpu(udev->descriptor.idVendor),
		le16_to_cpu(udev->descriptor.idProduct));
	...
	/* Register video device nodes. */
	if (uvc_register_chains(dev) < 0)
		goto error;

	...
}
```

## 2.2 uvc_register_chains

```c
static int uvc_register_chains(struct uvc_device *dev)
{
	struct uvc_video_chain *chain;
	int ret;
	list_for_each_entry(chain, &dev->chains, list) {
		ret = uvc_register_terms(dev, chain);
		if (ret < 0)
			return ret;
    ...
}
```

核心是`uvc_register_terms`函数。

## 2.3 uvc_register_terms

```c
/*
 * Register all video devices in all chains.
 */
static int uvc_register_terms(struct uvc_device *dev,
	struct uvc_video_chain *chain)
{
	struct uvc_streaming *stream;
	struct uvc_entity *term;
	int ret;
	list_for_each_entry(term, &chain->entities, chain) {
		if (UVC_ENTITY_TYPE(term) != UVC_TT_STREAMING)
			continue;

		stream = uvc_stream_by_id(dev, term->id);
		if (stream == NULL) {
			uvc_printk(KERN_INFO, "No streaming interface found "
				   "for terminal %u.", term->id);
			continue;
		}

		stream->chain = chain;
		ret = uvc_register_video(dev, stream);
		if (ret < 0)
			return ret;

		/* Register a metadata node, but ignore a possible failure,
		 * complete registration of video nodes anyway.
		 */
		uvc_meta_register(stream);

		term->vdev = &stream->vdev;
	}

	return 0;
}
```

该函数的功能点有两个：

1. 调用`uvc_register_video`注册video设备节点。
2. 调用`uvc_meta_register`注册一个元数据节点。

### 2.3.1 uvc_register_video

```c
static int uvc_register_video(struct uvc_device *dev,
		struct uvc_streaming *stream)
{
	int ret;

	/* Initialize the streaming interface with default parameters. */
	ret = uvc_video_init(stream);
	if (ret < 0) {
		uvc_printk(KERN_ERR, "Failed to initialize the device (%d).\n",
			   ret);
		return ret;
	}

	if (stream->type == V4L2_BUF_TYPE_VIDEO_CAPTURE)
		stream->chain->caps |= V4L2_CAP_VIDEO_CAPTURE
			| V4L2_CAP_META_CAPTURE;
	else
		stream->chain->caps |= V4L2_CAP_VIDEO_OUTPUT;

	uvc_debugfs_init_stream(stream);

	/* Register the device with V4L. */
	return uvc_register_video_device(dev, stream, &stream->vdev,
					 &stream->queue, stream->type,
					 &uvc_fops, &uvc_ioctl_ops);
}
```

该函数的功能如下：

1. 调用`uvc_video_init`函数初始化`stream`的默认参数。
2. 根据`stream`的类型设置`stream->chain->caps`，即标识该流支持的视频（`V4L2_CAP_VIDEO_CAPTURE`）或元数据捕获功能（`V4L2_CAP_META_CAPTURE`）。
3. 调用`uvc_debugfs_init_stream`函数初始化`stream`的DebugFS节点（仅用于调试目的）。
4. 调用`uvc_register_video_device`函数将`stream`注册为V4L2设备，传递了相关的参数，包括：

- `dev`：UVC设备实例；
- `stream`：UVC流实例；
- `&stream->vdev`：V4L2设备实例；
- `&stream->queue`：V4L2缓冲队列；
- `stream->type`：流的类型；
- `&uvc_fops`：文件操作实例，包含设备打开、关闭、读写等函数；
- `&uvc_ioctl_ops`：设备IOCTL操作的指针，包含了设备的控制接口。

最终调用`uvc_register_video_device`函数将`stream`注册为V4L2设备。先回到`uvc_register_terms`函数，`uvc_register_terms`函数调用完`uvc_register_video`后接着调用`uvc_meta_register`函数。

### 2.3.2 uvc_meta_register

该函数在uvc_metadata.c文件中，文件的注释如下：

```
/*
 *      uvc_metadata.c  --  USB Video Class driver - Metadata handling
 *
 *      Copyright (C) 2016
 *          Guennadi Liakhovetski (guennadi.liakhovetski@intel.com)
 *
 *      This program is free software; you can redistribute it and/or modify
 *      it under the terms of the GNU General Public License as published by
 *      the Free Software Foundation; either version 2 of the License, or
 *      (at your option) any later version.
 */
```

上面的信息表明uvc_metadata.c是UVC驱动用来处理元数据的，提交的信息是2016年。接着分析`uvc_meta_register`函数。

```c
int uvc_meta_register(struct uvc_streaming *stream)
{
	struct uvc_device *dev = stream->dev;
	struct video_device *vdev = &stream->meta.vdev;
	struct uvc_video_queue *queue = &stream->meta.queue;

	stream->meta.format = V4L2_META_FMT_UVC;

	/*
	 * The video interface queue uses manual locking and thus does not set
	 * the queue pointer. Set it manually here.
	 */
	vdev->queue = &queue->queue;
	return uvc_register_video_device(dev, stream, vdev, queue,
					 V4L2_BUF_TYPE_META_CAPTURE,
					 &uvc_meta_fops, &uvc_meta_ioctl_ops);
}

```

该函数功能是注册一个`uvc_metadata`的设备节点，最终也是调用`uvc_register_video_device`函数来进一步将metadata注册为V4L2设备。

### 2.3.3 uvc_register_video_device

```c
int uvc_register_video_device(struct uvc_device *dev,
			      struct uvc_streaming *stream,
			      struct video_device *vdev,
			      struct uvc_video_queue *queue,
			      enum v4l2_buf_type type,
			      const struct v4l2_file_operations *fops,
			      const struct v4l2_ioctl_ops *ioctl_ops)
{
	int ret;

	/* Initialize the video buffers queue. */
	ret = uvc_queue_init(queue, type, !uvc_no_drop_param);
	if (ret)
		return ret;

	/* Register the device with V4L. */

	/*
	 * We already hold a reference to dev->udev. The video device will be
	 * unregistered before the reference is released, so we don't need to
	 * get another one.
	 */
	vdev->v4l2_dev = &dev->vdev;
	vdev->fops = fops;
	vdev->ioctl_ops = ioctl_ops;
	vdev->release = uvc_release;
	vdev->prio = &stream->chain->prio;
	if (type == V4L2_BUF_TYPE_VIDEO_OUTPUT)
		vdev->vfl_dir = VFL_DIR_TX;
	else
		vdev->vfl_dir = VFL_DIR_RX;

	switch (type) {
	case V4L2_BUF_TYPE_VIDEO_CAPTURE:
	default:
		vdev->device_caps = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING;
		break;
	case V4L2_BUF_TYPE_VIDEO_OUTPUT:
		vdev->device_caps = V4L2_CAP_VIDEO_OUTPUT | V4L2_CAP_STREAMING;
		break;
	case V4L2_BUF_TYPE_META_CAPTURE:
		vdev->device_caps = V4L2_CAP_META_CAPTURE | V4L2_CAP_STREAMING;
		break;
	}

	strlcpy(vdev->name, dev->name, sizeof vdev->name);

	/*
	 * Set the driver data before calling video_register_device, otherwise
	 * the file open() handler might race us.
	 */
	video_set_drvdata(vdev, stream);
	ret = video_register_device(vdev, VFL_TYPE_GRABBER, -1);
	if (ret < 0) {
		uvc_printk(KERN_ERR, "Failed to register %s device (%d).\n",
			   v4l2_type_names[type], ret);
		return ret;
	}

	kref_get(&dev->ref);
	return 0;
}
```

该函数的功能是将UVC流实例`stream`注册为V4L2设备（`vdev`），并将其添加到V4L2设备列表中。

具体实现如下：

1. 调用`uvc_queue_init`函数初始化`queue`。
2. 设置`vdev`的各种参数，包括V4L2设备实例、文件操作实例、IOCTL操作指针、释放函数指针、优先级指针、以及流的类型（输入/输出）等。
3. 根据流的类型设置V4L2设备的capability，包括视频/元数据捕获功能以及流媒体传输功能。
4. 将UVC设备的名称复制到设备实例的名称中。
5. 调用`video_set_drvdata`函数将流实例的指针存储为V4L2设备实例的driver data。
6. 调用`video_register_device`函数将V4L2设备实例注册到V4L2设备列表中，同时指定设备类型为"VFL_TYPE_GRABBER"。
7. 如果注册成功，则增加UVC设备实例的引用计数。
8. 如果注册失败，则返回错误值。

函数`uvc_register_video_device`最终是调用了`video_register_device`函数来完成V4L2设备注册，该函数将会为该设备创建对应的设备节点。

### 2.3.4 video_register_device

```c
/* Register video devices. Note that if video_register_device fails,
   the release() callback of the video_device structure is *not* called, so
   the caller is responsible for freeing any data. Usually that means that
   you call video_device_release() on failure. */
static inline int __must_check video_register_device(struct video_device *vdev,
		int type, int nr)
{
	return __video_register_device(vdev, type, nr, 1, vdev->fops->owner);
}
```

```c
/**
 *	__video_register_device - register video4linux devices
 *	@vdev: video device structure we want to register
 *	@type: type of device to register
 *	@nr:   which device node number (0 == /dev/video0, 1 == /dev/video1, ...
 *             -1 == first free)
 *	@warn_if_nr_in_use: warn if the desired device node number
 *	       was already in use and another number was chosen instead.
 *	@owner: module that owns the video device node
 *
 *	The registration code assigns minor numbers and device node numbers
 *	based on the requested type and registers the new device node with
 *	the kernel.
 *
 *	This function assumes that struct video_device was zeroed when it
 *	was allocated and does not contain any stale date.
 *
 *	An error is returned if no free minor or device node number could be
 *	found, or if the registration of the device node failed.
 *
 *	Zero is returned on success.
 *
 *	Valid types are
 *
 *	%VFL_TYPE_GRABBER - A frame grabber
 *
 *	%VFL_TYPE_VBI - Vertical blank data (undecoded)
 *
 *	%VFL_TYPE_RADIO - A radio card
 *
 *	%VFL_TYPE_SUBDEV - A subdevice
 *
 *	%VFL_TYPE_SDR - Software Defined Radio
 */
int __video_register_device(struct video_device *vdev, int type, int nr,
		int warn_if_nr_in_use, struct module *owner)
{
	int i = 0;
	int ret;
	int minor_offset = 0;
	int minor_cnt = VIDEO_NUM_DEVICES;
	const char *name_base;

	/* A minor value of -1 marks this video device as never
	   having been registered */
	vdev->minor = -1;

	/* the release callback MUST be present */
	if (WARN_ON(!vdev->release))
		return -EINVAL;
	/* the v4l2_dev pointer MUST be present */
	if (WARN_ON(!vdev->v4l2_dev))
		return -EINVAL;

	/* v4l2_fh support */
	spin_lock_init(&vdev->fh_lock);
	INIT_LIST_HEAD(&vdev->fh_list);

	/* Part 1: check device type */
	switch (type) {
	case VFL_TYPE_GRABBER:
		name_base = "video";
		break;
	case VFL_TYPE_VBI:
		name_base = "vbi";
		break;
	case VFL_TYPE_RADIO:
		name_base = "radio";
		break;
	case VFL_TYPE_SUBDEV:
		name_base = "v4l-subdev";
		break;
	case VFL_TYPE_SDR:
		/* Use device name 'swradio' because 'sdr' was already taken. */
		name_base = "swradio";
		break;
	default:
		printk(KERN_ERR "%s called with unknown type: %d\n",
		       __func__, type);
		return -EINVAL;
	}	   
	vdev->vfl_type = type;
	vdev->cdev = NULL;
	if (vdev->dev_parent == NULL)
		vdev->dev_parent = vdev->v4l2_dev->dev;
	if (vdev->ctrl_handler == NULL)
		vdev->ctrl_handler = vdev->v4l2_dev->ctrl_handler;
	/* If the prio state pointer is NULL, then use the v4l2_device
	   prio state. */
	if (vdev->prio == NULL)
		vdev->prio = &vdev->v4l2_dev->prio;

	/* Part 2: find a free minor, device node number and device index. */
#ifdef CONFIG_VIDEO_FIXED_MINOR_RANGES
	/* Keep the ranges for the first four types for historical
	 * reasons.
	 * Newer devices (not yet in place) should use the range
	 * of 128-191 and just pick the first free minor there
	 * (new style). */
	switch (type) {
	case VFL_TYPE_GRABBER:
		minor_offset = 0;
		minor_cnt = 64;
		break;
	case VFL_TYPE_RADIO:
		minor_offset = 64;
		minor_cnt = 64;
		break;
	case VFL_TYPE_VBI:
		minor_offset = 224;
		minor_cnt = 32;
		break;
	default:
		minor_offset = 128;
		minor_cnt = 64;
		break;
	}
#endif

	/* Pick a device node number */
	mutex_lock(&videodev_lock);
	nr = devnode_find(vdev, nr == -1 ? 0 : nr, minor_cnt);
	printk(KERN_WARNING "%s vanelst nr:%d\n", __func__, nr);
	if (nr == minor_cnt)
		nr = devnode_find(vdev, 0, minor_cnt);
	if (nr == minor_cnt) {
		printk(KERN_ERR "could not get a free device node number\n");
		mutex_unlock(&videodev_lock);
		return -ENFILE;
	}
#ifdef CONFIG_VIDEO_FIXED_MINOR_RANGES
	/* 1-on-1 mapping of device node number to minor number */
	i = nr;
#else
	/* The device node number and minor numbers are independent, so
	   we just find the first free minor number. */
	for (i = 0; i < VIDEO_NUM_DEVICES; i++)
		if (video_device[i] == NULL)
			break;
	if (i == VIDEO_NUM_DEVICES) {
		mutex_unlock(&videodev_lock);
		printk(KERN_ERR "could not get a free minor\n");
		return -ENFILE;
	}
#endif
	vdev->minor = i + minor_offset;
	vdev->num = nr;
	devnode_set(vdev);

	/* Should not happen since we thought this minor was free */
	WARN_ON(video_device[vdev->minor] != NULL);
	vdev->index = get_index(vdev);
	video_device[vdev->minor] = vdev;
	mutex_unlock(&videodev_lock);

	if (vdev->ioctl_ops)
		determine_valid_ioctls(vdev);

	/* Part 3: Initialize the character device */
	vdev->cdev = cdev_alloc();
	if (vdev->cdev == NULL) {
		ret = -ENOMEM;
		goto cleanup;
	}
	vdev->cdev->ops = &v4l2_fops;
	vdev->cdev->owner = owner;
	ret = cdev_add(vdev->cdev, MKDEV(VIDEO_MAJOR, vdev->minor), 1);
	if (ret < 0) {
		printk(KERN_ERR "%s: cdev_add failed\n", __func__);
		kfree(vdev->cdev);
		vdev->cdev = NULL;
		goto cleanup;
	}

	/* Part 4: register the device with sysfs */
	vdev->dev.class = &video_class;
	vdev->dev.devt = MKDEV(VIDEO_MAJOR, vdev->minor);
	vdev->dev.parent = vdev->dev_parent;
	dev_set_name(&vdev->dev, "%s%d", name_base, vdev->num);
	ret = device_register(&vdev->dev);
	if (ret < 0) {
		printk(KERN_ERR "%s: device_register failed\n", __func__);
		goto cleanup;
	}
	/* Register the release callback that will be called when the last
	   reference to the device goes away. */
	vdev->dev.release = v4l2_device_release;

	if (nr != -1 && nr != vdev->num && warn_if_nr_in_use)
		printk(KERN_WARNING "%s: requested %s%d, got %s\n", __func__,
			name_base, nr, video_device_node_name(vdev));

	/* Increase v4l2_device refcount */
	v4l2_device_get(vdev->v4l2_dev);

#if defined(CONFIG_MEDIA_CONTROLLER)
	/* Part 5: Register the entity. */
	if (vdev->v4l2_dev->mdev &&
	    vdev->vfl_type != VFL_TYPE_SUBDEV) {
		vdev->entity.type = MEDIA_ENT_T_DEVNODE_V4L;
		vdev->entity.name = vdev->name;
		vdev->entity.info.dev.major = VIDEO_MAJOR;
		vdev->entity.info.dev.minor = vdev->minor;
		ret = media_device_register_entity(vdev->v4l2_dev->mdev,
			&vdev->entity);
		if (ret < 0)
			printk(KERN_WARNING
			       "%s: media_device_register_entity failed\n",
			       __func__);
	}
#endif
	/* Part 6: Activate this minor. The char device can now be used. */
	set_bit(V4L2_FL_REGISTERED, &vdev->flags);

	return 0;

cleanup:
	mutex_lock(&videodev_lock);
	if (vdev->cdev)
		cdev_del(vdev->cdev);
	video_device[vdev->minor] = NULL;
	devnode_clear(vdev);
	mutex_unlock(&videodev_lock);
	/* Mark this video device as never having been registered. */
	vdev->minor = -1;
	return ret;
}
EXPORT_SYMBOL(__video_register_device);
```

该函数的作用是注册一个视频设备并创建相应的字符设备节点/dev/video。

函数的参数：

- 指向`video_device`结构体的指针`vdev`
- 设备类型`type`
- 设备编号`nr`
- 是否在使用时发出警告的标志`warn_if_nr_in_use`
- 指向模块的指针`owner`

函数的具体功能如下：

1. 初始化vdev结构体中的一些字段。
2. 根据设备类型选择相应的设备名称，并根据设备类型选择相应的`minor`偏移量和`minor`计数器。
3. 调用`devnode_find`函数查找可用的设备节点号，并将其分配给vdev结构体中的`num`字段。
4. 为该视频设备分配一个唯一的minor号，并将其设置为vdev结构体中的minor字段。
5. 调用`cdev_alloc`函数为该视频设备分配一个字符设备结构体，并将其初始化。
6. 调用`cdev_add`函数将该字符设备结构体添加到内核中，并注册该设备节点/dev/video。
7. 将该视频设备与`sysfs`注册，并在最后一个引用该设备的引用被释放时调用v4l2_device_release函数。
8. 增加v4l2_device的引用计数。
9. 如果CONFIG_MEDIA_CONTROLLER宏已定义，则该函数还将该视频设备注册为媒体实体。
10. 将V4L2_FL_REGISTERED标志设置为1，表示该视频设备已成功注册。

创建设备节点/dev/video是调用cdev_add函数实现的，该函数逻辑如下：

1. 将字符设备结构体的owner字段设置为vdev结构体中的owner字段。
2. 将字符设备结构体的ops字段设置为vdev结构体中的fops字段。
3. 调用`cdev_device_add`函数将该字符设备结构体添加到内核中，并注册该设备节点/dev/video。
4. 如果添加成功，则将字符设备结构体的设备节点号保存在vdev结构体中的num字段中。

### 2.3.5 分配/dev/video号

分配/dev/video号是由下面的函数决定的：

```c
/* Try to find a free device node number in the range [from, to> */
static inline int devnode_find(struct video_device *vdev, int from, int to)
{
	return find_next_zero_bit(devnode_bits(vdev->vfl_type), to, from);
}
```

`find_next_zero_bit`函数的逻辑如下：

1. `devnode_bits`(vdev->vfl_type)返回一个unsigned long类型的位图，表示该设备类型已经被分配的设备节点号。
2. to表示从哪一位开始查找，from表示查找的结束位置。
3. 函数从to开始向后查找，直到找到第一个为0的位。
4. 如果找到，则返回该位的位置；否则，返回from。

总的来说，/dev/video后面数字，是根据系统中已经存在的设备节点号来分配的。调用`devnode_find`函数查找可用的设备节点号，并将其分配给vdev结构体中的num字段。如果使用CONFIG_VIDEO_FIXED_MINOR_RANGES宏，则将该minor号与设备节点号一一映射；否则，将该minor号与视频设备数组video_device中的第一个空闲位置关联。

### 2.3.6 如何区分UVC的两个video设备节点

如果一个UVC设备创建两个/dev/video设备节点，这两个设备一个是用于获取流数据，一个是获取UVC的元数据，如何区分呢。需要根据v4l2_dev的index来区分。即/sys/class/video4linux/videox/index，x对应/dev/videox。

v4l2_dev结构体中的index字段是用于标识该v4l2_dev结构体的唯一编号。每个v4l2_dev结构体都有一个唯一的index编号，用于在全局范围内标识该结构体。在v4l2设备驱动中，一个v4l2设备通常会包含多个子设备，每个子设备都有一个对应的v4l2_dev结构体。因此，通过v4l2_dev结构体中的index字段，可以方便地区分不同的子设备，并对它们进行管理和操作。

在v4l2设备驱动中，通常会使用一个全局的v4l2_dev结构体数组来管理所有的v4l2设备，每个v4l2设备都有一个唯一的index编号，用于在该数组中查找对应的v4l2_dev结构体。

为新的video_device结构体分配一个唯一的index号是由`get_index`函数实现的。

```c
static int get_index(struct video_device *vdev)
{
	/* This can be static since this function is called with the global
	   videodev_lock held. */
	static DECLARE_BITMAP(used, VIDEO_NUM_DEVICES);
	int i;

	bitmap_zero(used, VIDEO_NUM_DEVICES);

	for (i = 0; i < VIDEO_NUM_DEVICES; i++) {
		if (video_device[i] != NULL &&
		    video_device[i]->v4l2_dev == vdev->v4l2_dev) {
			set_bit(video_device[i]->index, used);
		}
	}

	return find_first_zero_bit(used, VIDEO_NUM_DEVICES);
}
```

函数的逻辑如下：

1. 该函数是在videodev_lock互斥锁的保护下执行的，因此可以安全地访问video_device数组。
2. 首先，该函数通过调用DECLARE_BITMAP宏声明了一个名为used的位图，用于记录已经被占用的index号。
3. 然后，遍历video_device数组，如果该video_device结构体的v4l2_dev字段与当前video_device结构体的v4l2_dev字段相同，则将该结构体的index号对应的位设置为1。
4. 最后，调用`find_first_zero_bit`函数在used位图中查找第一个为0的位，即可得到一个可用的index号。

因此，`get_index`函数返回的值是一个唯一的、尚未被占用的index号，可以用于新的video_device结构体。

由上面的`uvc_register_terms`函数可知首先是注册用于获取流数据的v4l2_dev，然后再注册用于获取元数据的v4l2_dev，因此获取流数据的v4l2_dev的index是0，用于获取元数据的v4l2_dev的index是1。

因此可以根据/sys/class/video4linux/videox/index来判断哪个video才是用于获取流数据的。

## 2.4 小结

插入UVC设备后，UVC驱动的探测函数`uvc_probe`会执行，然后调用`uvc_register_terms`函数，该函数分别调用`uvc_register_video`和`uvc_meta_register`，前者用于注册video设备节点用于获取UVC的流数据，后者用于注册一个获取UVC元数据的节点，这两个函数最终都是调用`__video_register_device`函数，该函数功能是注册一个视频设备并创建相应的字符设备节点/dev/video。

# 3 获取UVC元数据功能

该功能是2016年新增的功能。主要目的是因为一些 UVC 摄像机在其有效负载头(payload headers)中包含元数据。而该功能则是获取这些元数据。对应uvc metadata.c文件。

该功能会在系统中创建另一个 /dev/video 设备节点。
这是因为元数据信息经由另一个视频设备节点传递，为了避免与实际视频流产生任何冲突或数据损坏而创建的。
创建另一个 /dev/video 设备节点确保元数据信息与实际视频流分开接收和处理。这使得系统能够高效地处理元数据和视频流，而不会产生任何干扰。

下面是该功能提交者Guennadi Liakhovetski与review者Laurent Pinchart围绕该patch的讨论。

> Laurent Pinchart:
> I think so, only for the cameras that can produce metadata.
>
> 我认为是这样，仅适用于可以产生元数据的相机。
>
> Guennadi Liakhovetski:
> Every UVC camera produces metadata, but most cameras only have standard fields there.
> Whether we should stream standard header fields from the metadata node will be discussed later.
> If we do decide to stream standard header fields, then every USB camera gets metadata nodes. If
> we decide not to include standard fields, how do we know whether the camera has any private
> fields in headers without streaming from it? Do you want a quirk for such cameras?
>
> 每个 UVC 相机都会产生元数据，但大多数相机在那里只有标准字段。我们是否应该从元数据节点流式传输标准头字段将在后面讨论。如果我们决定流式传输标准标头字段，那么每个 USB 摄像头都会获得元数据节点。如果我们决定不包含标准字段，我们如何知道相机的标头中是否有任何私有字段而不从中流出？你想要这种相机的怪癖吗？
>
> Laurent Pinchart:
> Unless they can be detected in a standard way that's probably the best solution. Please
> remember that the UVC specification doesn't allow vendors to extend headers in a
> vendor-specific way. This is an abuse of the specification, so a quirk is probably the best option.
>
> 除非可以用标准方式检测它们，否则这可能是最好的解决方案。请记住，UVC 规范不允许供应商以供应商特定的方式扩展标头。这是对规范的滥用，所以一个怪癖可能是最好的选择。
>
> Guennadi Liakhovetski:
> Does it not? Where is that specified? I didn't find that anywhere.
>
> 不是吗？哪里规定的？我没有在任何地方找到它。
>
> Laurent Pinchart:
> It's the other way around, it document a standard way to extend the payload header. Any option
> you pick risks being incompatible with future revisions of the specification. For instance the
> payload header's bmHeaderInfo field can be extended through the use of the EOF bit, but a
> future version of the specification could also extend it, in a way that would make a
> vendor-specific extension be mistaken for the standard extension.
> Now, you could add data after the standard payload without touching the bmHeaderInfo field,
> but I'm still worried it could be broken by future versions of the specification.
>
> 相反，它记录了扩展有效负载标头的标准方法。您选择的任何选项都有可能与规范的未来修订版不兼容。例如，有效载荷标头的 bmHeaderInfo 字段可以通过使用 EOF 位进行扩展，但未来版本的规范也可以扩展它，这种方式会使特定于供应商的扩展被误认为是标准扩展。
>
> 现在，您可以在不触及 bmHeaderInfo 字段的情况下在标准负载之后添加数据，但我仍然担心它可能会被规范的未来版本破坏。
>
> Guennadi Liakhovetski:
> Exactly - using "free" space in payload headers is not a part of the spec, but is also not prohibited
> by it. As for future versions - cameras specify which version of the spec they implement, and
> existing versions cannot change. If a camera decides to upgrade and in future UVC versions there
> won't be enough space left for the private data, only then the camera manufacturer will have a
> problem, that they will have to solve. The user-space software will have to know, that UVC 5.1
> metadata has a different format, than UVC 1.5, that's true.
>
> 没错—在有效负载标头中使用“自由”空间不是规范的一部分，但也不被禁止。至于未来的版本——相机会指定它们实施的规范版本，现有版本不能更改。如果相机决定升级，并且在未来的 UVC 版本中将没有足够的空间用于私人数据，那么相机制造商就会遇到问题，他们必须解决这个问题。用户空间软件必须知道，UVC 5.1 元数据的格式与 UVC 1.5 不同，这是事实。
>
> Laurent Pinchart:
>
> I agree that the specification doesn't explicitly forbid it, but it's a very grey area. An extension
> mechanism should be standardized by the USB-IF UVC working group. I'd propose it myself if they
> hadn't decided to kick me out years ago :-)
>
> 我同意规范没有明确禁止它，但这是一个非常灰色的区域。扩展机制应由 USB-IF UVC 工作组标准化。如果他们几年前没有决定把我踢出去，我会自己提议的:-)
>
> Guennadi Liakhovetski:
> How about a module parameter with a list of VID:PID pairs? The problem with the quirk is, that
> as vendors produce multiple cameras with different PIDs they will have to push patches for each
> such camera.
>
> 带有 VID:PID 对列表的模块参数怎么样？问题在于，随着供应商生产具有不同 PID 的多个摄像头，他们将不得不为每个此类摄像头推送补丁。
>
> Laurent Pinchart:
> I'd like something that works out of the box for end-users, at least in most cases. There's already
> a way to set quirks through a module parameter, and I think I'd accept a patch extending that it
> make it VID:PID dependent. That's an acceptable solution for testing, but should not be
> considered as the way to go for production.
> How many such devices do you expect ?
>
> 我想要一些对最终用户来说开箱即用的东西，至少在大多数情况下是这样。已经有一种通过模块参数设置怪癖的方法，我想我会接受一个补丁扩展它使它依赖于 VID:PID。这是一个可接受的测试解决方案，但不应被视为用于生产的方式。
>
> 您觉得有多少这样的设备？
>
> Guennadi Liakhovetski:
> Ok, that helps already, sure.
> No idea, significantly more than 2, let's say :) But well, you already can count a few RealSense
> USB / UVC cameras on the market.Concerning metadata for isochronous endpoints. I actually
> don't know what to do with it. I don't have any such cameras with non-standard metadata.
> For the standard metadata it would probably be enough to get either the first or the last or the
> middle payload. Collecting all of them seems redundant to me. Maybe I could for now only
> enable metadata nodes for bulk endpoints. Would that be acceptable?
>
> 好的，这确实有帮助。不知道，比方说，明显超过 2 个 :) 但是，您已经可以数数市场上的一些 RealSense USB / UVC 摄像头。关于等时端点的元数据。我实际上不知道该怎么办。我没有任何带有非标准元数据的此类相机。对于标准元数据，获取第一个或最后一个或中间的有效负载可能就足够了。收集所有这些对我来说似乎是多余的。也许我现在只能为批量端点启用元数据节点。那可以接受吗？

# 4 解决方案

如果ubuntu或Android系统合入2016年获取UVC元数据的patch，则插入一个UVC会生成两个/dev/video设备节点。该问题其实不是bug，是一个功能，只不过市面上很少有UVC设备会在payload headers携带UVC厂家私有数据，有不少Android开发板会将该功能屏蔽掉，毕竟生成两个/dev/video会对上层应用造成一定的混淆(需要根据index过滤掉元数据的video节点)，那么如何屏蔽掉该功能呢。

由上面2.3小节`uvc_register_terms`可知，`uvc_meta_register`是注册元数据设备的，因此将`uvc_meta_register`函数注释掉即可。
