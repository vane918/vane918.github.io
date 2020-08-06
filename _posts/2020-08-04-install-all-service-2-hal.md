---
layout: post
title:  "Android添加系统服务访问驱动程序系列:添加HAL层"
date:   2020-08-04 00:00:00
catalog:  true
tags:
    - 系统
    - HAL
    - Android

---



>  基于Android 6.0源码，基于实操在Android系统中如何为驱动程序添加一个HAL层。

## 1. 概述

在[Android添加系统服务访问驱动程序系列:添加驱动程序](https://vanelst.site/2020/07/30/install-all-service-1-driver/)中我们实现了一个简单的读写字符设备驱动 mychar，该篇文章为 驱动 mychar 是实现一个HAL层。
Android 系统的HAL层(硬件抽象层)运行在用户空间中，它向下屏蔽硬件驱动模块的实现细节，向上提供硬件访问服务。通过硬件抽象层，Android 系统分两层来支持硬件设备，其中一层实现在用户空间中，另一层实现在内核空间中。传统的 Linux 系统把对硬件的支持完全实现在内核空间中，即把对硬件的支持完全实现在硬件驱动模块中。
Android 系统为什么要增加HAL层呢？最主要的原因是开源协议的问题。Linux 内核源码必须遵循 GPL协议的，则需要公开内核的源码。如果Android 系统把对硬件的支持完全实现在硬件驱动模块中，那么就必须将这些硬件驱动模块源码公开，这样的话有可能会暴露了硬件的实现细节和参数，损害了设备厂商的利益。另一方面，Android 系统源码遵循的是 Apache License 协议，它允许移动设备厂商添加或者修改 Android 系统的源代码，而不必要公开这些代码。因此，增加了HAL层运行在用户空间中，另一部分对硬件的支持运行在内核空间中，其中，内核空间仍然是以硬件驱动模块的形式来支持，不过它只是提供简单的硬件访问通道；而用户空间的HAL层则封装了硬件的实现细节和参数，这样就可以保护厂家的利益了。
接下来，我们以上一篇实现的mychar 驱动，为该驱动开发一个 HAL 层。本文基于Android 6.0 版本编写，Android 8.0版本及更高的版本在 HAL层有很大的重构，后续有时间再研究 Android 8.0及更高版本的 HAL 层。

## 2. HAL 层模块编写规范

Android 系统的 HAL 层以模块的形式来管理各个硬件访问接口。每一个硬件模块都对应有一个动态链接库文件，这些动态链接库文件的命令需要符合一定的规范。每个 HAL 层模块都使用结构体 `hw_module_t` 来描述，而硬件设备则使用结构体`hw_device_t` 来描述。

### 2.1 HAL 层模块文件命名规范

根据`hardware/libhardware/hardware.c`文件中的定义，

```c
static const char *variant_keys[] = {
    "ro.hardware",  /* This goes first so that it can pick up a different
                       file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};
```

HAL 层模块文件的命名规范为<MODUILE_ID>.variant.so，其中 MODUILE_ID 表示模块的ID，variant 表示 variant_keys中的四个属性之一。系统在加载 HAL 层模块时，会一次按照variant_keys的四个属性取对应的属性值。如果其中一个系统属性存在，则把它作为 variant 的值，然后在检查对应的so文件是否存在，如果存在，则找到加载的 HAL 层模块文件了，否则就继续查找一下个系统属性。如果都不存在，则使用<MODUILE_ID>.default.so 来作为要加载的 HAL 模块文件的名称。

### 2.2  HAL 层模块结构体定义规范

[->hardware/libhardware/include/hardware/hardware.h]

```c
typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;
    uint16_t module_api_version;
    uint16_t hal_api_version;
    /** Identifier of module */
    const char *id;

    /** Name of this module */
    const char *name;

    /** Author/owner/implementor of the module */
    const char *author;

    /** Modules methods */
    struct hw_module_methods_t* methods;

    /** module's dso */
    void* dso;
} hw_module_t;
```

关于结构体 hw_module_t 的规范如下：

- HAL 层中每一个模块都必须自定义一个 HAL 层模块结构体，而且它的第一个成员变量的类型必须为 hw_module_t。
- HAL 层的每一个模块都必须存在一个导出符号 HAL_MODULE_IFNO_SYM，即"HMI"，它指向一个自定义的 HAL 层模块结构体。
- 结构体 hw_module_t 的成员变量 dso 用来保护加载 HAL 层模块后得到的句柄值。用来保存加载对应的动态链接库文件时返回的句柄值，在卸载这个 HAL 层模块时会用到该句柄值。
- 结构体 hw_module_t 的成员变量 methods 定义了一个 HAL 层模块的操作方法列表，其类型为hw_module_methods_t。

[->hardware/libhardware/include/hardware/hardware.h]

```c
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;
```

结构体 hw_module_methods_t 只有一个成员变量，它是一个函数指针，用来打来 HAL 层模块中的硬件设备。参数含义如下：

- module：表示要打开的硬件设备所在的模块
- id：表示要打开的硬件设备的ID
- device：表示一个输出参数，用来描述一个已经打开的硬件设备。

由于一个 HAL 层模块可能包含多个硬件设备，因此在调用结构体hw_module_methods_t 的open 时需要指定其ID。 HAL 层中的硬件设备用结构体 hw_device_t来描述。

[->hardware/libhardware/include/hardware/hardware.h]

```c
typedef struct hw_device_t {
    /** tag must be initialized to HARDWARE_DEVICE_TAG */
    uint32_t tag;
    uint32_t version;
    /** reference to the module this device belongs to */
    struct hw_module_t* module;
    /** Close this device */
    int (*close)(struct hw_device_t* device);
} hw_device_t;
```

关于结构体 hw_device_t 的规范如下：

- HAL 层模块中的每一个硬件设备都必须自定义一个硬件设备结构体，而且它的第一个成员变量必须为 hw_device_t。
- 结构体hw_device_t 的成员变量 tag 的值必须设置为 HARDWARE_DEVICE_TAG，用来标志这是一个 HAL 层中的硬件设备结构体。
- close 是一个函数指针，用来关闭一个硬件设备。

## 3. 编写 HAL 层名模块接口

每个 HAL 层模块在内核中都对应有一个驱动程序，HAL 层模块就是通过这些驱动程序来访问硬件设备的。在[Android添加系统服务访问驱动程序系列:添加驱动程序](https://vanelst.site/2020/07/30/install-all-service-1-driver/)中我们实现了一个简单的读写字符设备驱动 mychar，现在我们为该驱动程序添加一个 HAL 层模块。我们将虚拟硬件设备mychar 在 HAL 层中的模块名称定义为`operatechar`。目录如下：

```c
/hardware/libhardware
---include
    ---hardware
       ---operate_char.h
---modules
    ---operatechar
       ---operate_char.c
       ---Android.mk
```

有三个文件构成：

- operate_char.h：模块的头文件
- operate_char.c：模块的实现文件
- Android.mk：模块的编译脚本文件

下面分别看下这几个文件的实现。

### 3.1operate_char.h

[->/hardware/libhardware/include/hardware/operate_char.h]

```c
#ifndef ANDROID_OPERATE_CHAR_H
#define ANDROID_OPERATE_CHAR_H

#include <stdint.h>
#include <sys/cdefs.h>
#include <sys/types.h>

#include <hardware/hardware.h>
    
__BEGIN_DECLS

#define OPERATE_CHAR_HARDWARE_MODULE_ID "operatechar"
#define OPERATE_CHAR_HARDWARE_DEVICE_ID "operatechar"

/*自定义模块结构体*/
struct opetatechar_module_t {
    struct hw_module_t common;
};

/*自定义设备结构体*/
struct operatechar_device_t {
    struct hw_device_t common;
    int fd;
    int (*close)(void);
    int (*read)(struct operatechar_device_t* dev, char* buffer, int length);
    int (*write)(struct operatechar_device_t* dev, char* buffer);
};

__END_DECLS

#endif // ANDROID_OPERATE_CHAR_H
```

 宏 OPERATE_CHAR_HARDWARE_MODULE_ID 和 OPERATE_CHAR_HARDWARE_DEVICE_ID 分别用来描述模块ID和设备ID。结构体 opetatechar_module_t 用来描述自定义的模块结构体，它的第一个成员变量的类型必须为 hw_module_t。结构体 operatechar_device_t 用来描述虚拟硬件设备 mychar，它的第一个成员变量的类型必须为 hw_device_t。其他三个成员变量，fd 是一个文件描述符，用来描述打开的设备文件/dev/mychar，read 和 write 是函数指针，分别用来读和写虚拟硬件设备mychar。

### 3.2 operate_char.c

[->/hardware/libhardware/modules/operatechar/operate_char.c]

```c
#define  LOG_TAG  "operate_char_hal"
#include <cutils/log.h>
#include <cutils/sockets.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <malloc.h>
#include <errno.h>
#include <hardware/operate_char.h>

#define   OPERATECHAR_DEBUG   1

#if OPERATECHAR_DEBUG
#  define D(...)   ALOGD(__VA_ARGS__)
#else
#  define D(...)   ((void)0)
#endif

#define DEVICE_NAME "/dev/mychar"

/*设备打开与关闭操作*/
static int operatechar_open(const struct hw_module_t* module, const char* id,struct hw_device_t** device);
static int operatechar_close(struct hw_device_t* dev);

/*设备读写操作*/
static int operatechar_write(struct operatechar_device_t* dev, char* buffer);
static int operatechar_read(struct operatechar_device_t* dev, char* buffer, int length);

static int operatechar_open(const struct hw_module_t* module, const char* id, struct hw_device_t** device)
{
    if(!strcmp(id, OPERATE_CHAR_HARDWARE_DEVICE_ID)) {
        struct operatechar_device_t *dev = malloc(sizeof(struct operatechar_device_t));
        if (!dev) {
            D("OPERATECHAR HW failed to malloc memory !!!");
            return -1;
        }
        memset(dev, 0, sizeof(*dev));
        dev->common.tag = HARDWARE_DEVICE_TAG;
        dev->common.version = 0;
        dev->common.module = (struct hw_module_t*)module;
        dev->read = operatechar_read;
        dev->write = operatechar_write;
        dev->close = operatechar_close;

        dev->fd = open(DEVICE_NAME, O_RDWR);
        if (dev->fd < 0) {
            D("failed to open /dev/mychar!");
            free(dev);
            return -EFAULT;
        }
        *device = &(dev->common);
        D("OPERATECHAR HW has been initialized");

        return 0;
    }
    return -EFAULT;
}

static int operatechar_close(struct hw_device_t* dev)
{
    struct operatechar_device_t* operatechar_device = (struct operatechar_device_t*)dev;
    if (operatechar_device) {
        close(operatechar_device->fd);
        free(operatechar_device);
    }
    return 0;
}

/*往/dev/mychar存数据*/
static int operatechar_write(struct operatechar_device_t* dev, char* buffer)
{
    if(!dev) {
        D("Null device pointer.");
        return -EFAULT;
    }
    write(dev->fd, buffer, strlen(buffer));
    D("write data to driver: %s", buffer);
    return 0;
}

/*读取/dev/mychar的数据*/
static int operatechar_read(struct operatechar_device_t* dev, char* buffer, int length)
{
    if(!dev) {
        D("Null device pointer.");
        return -EFAULT;
    }
    if(!buffer) {
        D("Null buffer pointer.");
        return -EFAULT;
    }
    int retval;
    retval = read(dev->fd, buffer, length);
    D("read data from driver: %s", buffer);
    return retval;
}

static struct hw_module_methods_t operatechar_module_methods = {
    .open = operatechar_open,
};

struct opetatechar_module_t HAL_MODULE_INFO_SYM = {
    common: {
        .tag = HARDWARE_MODULE_TAG,
        .version_major = 1,
        .version_minor = 0,
        .id = OPERATE_CHAR_HARDWARE_MODULE_ID,
        .name = "Operatechar HW Module",
        .author = "Vane",
        .methods = &operatechar_module_methods,
    }
};
```

按照 HAL 层模块编写规范，每个 HAL 层模块必须导出一个名称为 HAL_MODULE_INFO_SYM 的符号，它指向一个自定义的 HAL 层模块结构体，而且它的第一个类型为 hw_module_t 的成员变量 tag 的值必须设置为 HARDWARE_MODULE_TAG。除此之外，还初始化了这个 HAL 层模块结构体的版本号、ID、名称、作者和操作方法列表。
一个 HAL 层模块可能会包含多个硬件设备，而这些硬件设备的打开操作都是由函数 operatechar_open 来完成的，因此，函数 operatechar_open 会根据传进来的参数id来判断要打开哪一个硬件设备。
在 HAL 层模块operatechar 中。只有一个虚拟硬件设备mychar，它使用结构体 operatechar_device_t 来描述。因此，函数operatechar_open 发现参数id与虚拟硬件设备mychar 的ID 匹配后，就会分配一个operatechar_device_t 结构体，并且对它的成员变量进行初始化。按照 HAL 层模块编写规范， HAL 层中的硬件设备标签(dev->commom.tag)必须设置为HARDWARE_MODULE_TAG。
初始化完成用来描述虚拟硬件设备mychar 的结构体 operatechar_device_t 之后，我们就可以调用open 函数来打开虚拟硬件设备文件/dev/mychar 了，并且得到文件的描述符保存在结构体operatechar_device_t  的fd 中。operatechar_close 函数是关闭设备文件/dev/mychar，以及释放设备在打开时所分配的资源。

### 3.3 Android.mk

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE_PATH := $(TARGET_OUT_SHARED_LIBRARIES)/hw
LOCAL_CFLAGS += $(common_flags)
LOCAL_LDLIBS += -llog
LOCAL_C_INCLUDES := hardware/libhardware
LOCAL_SHARED_LIBRARIES := liblog libcutils libhardware
LOCAL_SRC_FILES := operate_char.c
LOCAL_MODULE := operatechar.$(TARGET_BOARD_PLATFORM)
LOCAL_MODULE_TAGS := optional
include $(BUILD_SHARED_LIBRARY)
```

这是 HAL 层模块operatechar 的编译脚本文件。$(BUILD_SHARED_LIBRARY)表示要将 operatechar 编译成一个动态链接库文件，名称为 operatechar.xxx.so，xxx是芯片平台，如 MSM8909，并且保存在xxx/system/lib/hw 目录下。

## 3. 处理硬件设备访问权限问题

在 HAL 层模块中，我们是调用 open 函数来打开对应的设备文件 /dev/mychar 的，如果不修改 /dev/mychar 的访问权限，那么应用程序调用operatechar_open 函数打开设备文件 /dev/mychar 就会失败，在[Android添加系统服务访问驱动程序系列:添加驱动程序](https://vanelst.site/2020/07/30/install-all-service-1-driver/)的第6小节 赋予 /dev/mychar 权限已经对其进行了权限修改，即在 /system/core/rootdir/ueventd.rc 文件中末尾添加上下面的语句：

```makefile
/dev/mychar               0660   system     system
```

这表示system 用户可以对/dev/mychar，进行读写操作。

## 4. 编译 HAL 层的operatechar 模块

### 4.1单独编译

source 和lunch 后，使用下面的编译指令可以单独编译 HAL 层的 operatechar 模块。

```
mmm hardware/libhardware/modules/operatechar/
```

就会在 out/target/product/xxx/system/lib/hw 目录下生成一个operatechar.xxx.so文件，如operatechar.MSM8909.so。

### 4.2 集成到Android 系统中

上述4.1采用的方式只能单独编译，如果想将HAL 层的operatechar 模块集成到系统中参与编译，需要以下的修改。
1、修改/device/xxx/common/base.mk文件，如device/qcom/common/base.mk

```makefile
...
#OPERATECHAR
LIBOPERATECHAR_HARDWARE := operatechar.msm8909
PRODUCT_PACKAGES += $(LIBOPERATECHAR_HARDWARE)
```

上面的operatechar.msm8909根据自身的芯片和模块的Android.mk定义的LOCAL_MODULE变量值来，我这里LOCAL_MODULE是LOCAL_MODULE := operatechar.$(TARGET_BOARD_PLATFORM)，TARGET_BOARD_PLATFORM值为msm8909，因此上面的值设备operatechar.msm8909。
然后采用全编的方式就可以将operatechar 编译到系统镜像中了。
关于HAL 层 operatechar 模块的编写就到此为止了，下篇我们针对 operatechar 模块编写硬件访问服务。