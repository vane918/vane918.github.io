---
layout: post
title:  "Android添加系统服务访问驱动程序系列:添加framework层服务"
date:   2020-08-05 00:00:00
catalog:  true
tags:
    - 系统
    - framework
    - Android

---



>  基于Android 6.0源码，基于实操在Android系统中如何添加一个framework层的服务访问驱动程序。

## 1. 概述

在[Android添加系统服务访问驱动程序系列:添加HAL层](https://vanelst.site/2020/08/04/install-all-service-2-hal/)中我们为 驱动 mychar 是实现一个HAL层模块 operatechar。开发好 HAL 层模块后，我们需要在framework 层实现一个硬件访问服务 operate_char。硬件访问服务operate_char通过HAL 层模块的operatechar 为上层应用程序提供硬件设备的读写操作。
Android 系统的硬件访问服务operate_char 跟Android 系统的AMS 、PMS等系统核心服务一样运行在系统进程System 中，而使用 operate_char 服务则运行在另外的进程中，因此需要 Binder来进行进程间通信。接下来我们分别详细讲解实现硬件访问服务operate_char的步骤。

## 2. 定义硬件访问服务接口

Android 系统提供了一种描述语言来定义具有跨进程访问能力的服务接口，这种描述语言称为Android接口描述语言(AIDL)。以AIDL定义的服务接口文件是以adil为后缀的，在编译时，编译系统会将它们转换成Java 文件，然后在对它们进程编译。在本节中我们将使用AIDL来定义硬件访问服务接口 IOperateCharService。

[->frameworks/base/core/java/android/operatechar/IOperateCharService.aidl]

```java
package android.operatechar;
/**
* {@hide}
*/
interface IOperateCharService {
    String read(int maxLength);
    void write(String mString);
}
```

IOperateCharService 服务接口只定义了两个成员函数，分别是 read 和 write ，作用分别是对虚拟设备节点 /dev/mychar 进行读写。(/dev/mychar有内核驱动程序创建，详情见[Android添加系统服务访问驱动程序系列:添加驱动程序](https://vanelst.site/2020/07/30/install-all-service-1-driver/))
在下面的文件中修改：

[->/frameworks/base/Android.mk]

```java
LOCAL_SRC_FILES += \
...
core/java/android/operatechar/IOperateCharService.aidl \
```

## 3. 实现硬件访问服务

[->/frameworks/base/services/core/java/com/android/server/operatechar/OperateCharService.java]

```java
package com.android.server.operatechar;

import android.content.Context;
import android.os.Handler;
import android.operatechar.IOperateCharService;
import android.os.Looper;
import android.os.Message;
import android.os.Process;
import android.util.Slog;
import android.os.RemoteException;

public class OperateCharService extends IOperateCharService.Stub {
    private static final String TAG = "OperateCharService";
    private Context mContext;
    private int mNativePointer;

    public OperateCharService(Context context) {
        super();
        Slog.i(TAG, "Start to init OperateChar Service");
        mContext = context;
        mNativePointer = init_native();
        if(mNativePointer ==0) {
            Slog.i(TAG, "Failed to init OperateChar Service");
        }
    }

    protected void finalize() throws Throwable {
        if(mNativePointer ==0) {
            return;
        }
        finalize_native(mNativePointer);
        super.finalize();
    }

    public String read(int maxLength) throws RemoteException
    {
        if(mNativePointer ==0) {
            Slog.e(TAG, "OperateChar Service is not init");
            return null;
        }
        int length;
        byte[] buffer = new byte[maxLength];
        length = read_native(mNativePointer, buffer);
        try {
            return new String(buffer, 0, length, "UTF-8");
        } catch (Exception e) {
            Slog.e(TAG, "read buffer error!");
            return null;
        }
    }

    public void write(String mString) throws RemoteException
    {
        if(mNativePointer ==0) {
            Slog.e(TAG, "OperateChar Service is not init");
        }
        byte[] buffer = mString.getBytes();
        write_native(mNativePointer, buffer);
    }

    private static native int init_native();
    private static native void finalize_native(int ptr);
    private static native int read_native(int ptr, byte[] buffer);
    private static native void write_native(int ptr, byte[] buffer);
}
```

硬件访问服务 OperateCharService 继承了 IOperateCharService.Stub类，并且实现了IOperateCharService 接口的 read 和 write 函数。

## 4. 实现硬件访问服务的JNI方法

[->frameworks/base/services/core/jni/com_android_server_operatechar_OperateCharService.cpp]

```cpp
#define LOG_TAG "OperateCharServiceJNI"

#include "jni.h"
#include "JNIHelp.h"
#include "android_runtime/AndroidRuntime.h"

#include <utils/misc.h>
#include <utils/Log.h>
#include <hardware/hardware.h>
#include <hardware/operate_char.h>

#include <stdio.h>

namespace android
{

operatechar_device_t* operatechar_dev;

static inline int operatechar_device_open(const hw_module_t* module, struct operatechar_device_t** device) {
    return module->methods->open(module, OPERATE_CHAR_HARDWARE_DEVICE_ID, (struct hw_device_t**)device);
}

static jint init_native(JNIEnv *env, jobject /* clazz */)
{
    int err;
    opetatechar_module_t* module;
    operatechar_device_t* dev = NULL;

    err = hw_get_module(OPERATE_CHAR_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
    if (err == 0) {
        if(operatechar_device_open(&(module->common), &dev) ==0) {
            ALOGE("Device mychar is open.");
            return (jint)dev;
        }
        ALOGE("Failed to open device mychar!!!");
        return 0;
    }
    ALOGE("Failed to get HAL mychar!!!");
    return 0;
}

static void finalize_native(JNIEnv *env, jobject /* clazz */, int ptr)
{
    operatechar_device_t* dev = (operatechar_device_t*)ptr;
    if (dev == NULL) {
        return;
    }

    dev->close();
    free(dev);
}

static int read_native(JNIEnv *env, jobject /* clazz */, int ptr, jbyteArray buffer)
{
    operatechar_device_t* dev = (operatechar_device_t*)ptr;
    if(!dev) {
        ALOGE("Device mychar is not open.");
        return 0;
    }
    int length;
    jbyte* my_byte_array;
    my_byte_array = env->GetByteArrayElements(buffer, NULL);
    length = dev->read(dev, (char*) my_byte_array, env->GetArrayLength(buffer));
    ALOGI("read data from hal: %s", (char *)my_byte_array);
    env->ReleaseByteArrayElements(buffer, my_byte_array, 0);
    return length;
}

static void write_native(JNIEnv *env, jobject /* clazz */, int ptr, jbyteArray buffer)
{
    operatechar_device_t* dev = (operatechar_device_t*)ptr;
    if(!dev) {
        ALOGE("Device mychar is not open.");
        return;
    }
    jbyte* my_byte_array;
    my_byte_array = env->GetByteArrayElements(buffer, NULL);
    dev->write(dev, (char*) my_byte_array);
    ALOGI("write data to hal: %s", (char *)my_byte_array);
    env->ReleaseByteArrayElements(buffer, my_byte_array, 0);
}

static JNINativeMethod method_table[] = {
    { "init_native", "()I", (void*)init_native },
    { "finalize_native", "(I)V", (void*)finalize_native },
    { "read_native", "(I[B)I", (void*)read_native },
    { "write_native", "(I[B)V", (void*)write_native }
};

int register_android_server_operatechar_OperateCharService(JNIEnv *env)
{
    return jniRegisterNativeMethods(env, "com/android/server/operatechar/OperateCharService",
            method_table, NELEM(method_table));

};
}
```

在init_native 函数中，首先通过Android HAL 层提供的hw_get_module 函数来加载模块ID为OPERATE_CHAR_HARDWARE_MODULE_ID 的HAL 层模块，最终返回一个hw_module_t接口给init_native函数，这个hw_module_t接口实际上指向的是自定义的一个HAL 层模块对象，即一个opetatechar_module_t对象。接着调用函数operatechar_device_open 开打开设备ID为OPERATE_CHAR_HARDWARE_DEVICE_ID 的硬件设备，而后者又是通过前面获取的hw_module_t 接口的操作方法列表中的open函数来打开指定的硬件设备的。HAL 层operatechar模块中的open函数设置为operatechar_open，这个函数最终返回一个operatechar_device_t接口，最后将operatechar_device_t接口转换成一个整型句柄值返回给调用者。
method_table 是一个JNI 方法表，将init_native等函数注册为相应的函数。最后通过jniRegisterNativeMethods 函数把JNI 方法表的函数注册到Java 虚拟机中。
还需要修改frameworks/base/services/core/jni/onload.cpp文件，在里面添加com_android_server_operatechar_OperateCharService函数的声明和调用。

[->frameworks/base/services/core/jni/onload.cpp]

```c++
namespace android {
...
int register_android_server_operatechar_OperateCharService(JNIEnv* env);
};
extern "C" jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
...
register_android_server_operatechar_OperateCharService(env);
}
```

onload.cpp 文件实现在 libandroid_servers 模块中，当系统加载 libandroid_servers 模块时，就会调用实现在onload.cpp文件中的JNI_Onload 函数，这样就可以将前面定义的JNI 方法注册到Java虚拟机中了。
最后还需要修改frameworks/base/services/core/jni/Android.mk，将新添加的jni文件加入编译。

```makefile
LOCAL_SRC_FILES += \
...
$(LOCAL_REL_DIR)/com_android_server_operatechar_OperateCharService.cpp \
...
```

## 5. 实现硬件访问服务客户端

前面实现的OperateCharService服务与Android系统的其他核心服务如AMS都是运行在system_server 进程中的，其他进程如app想访问该服务，需要通过Binder进行进程间通信，因此该小节实现OperateCharService服务的客户端：OperateCharManager。

[->frameworks/base/core/java/android/operatechar/OperateCharManager.java]

```java
package android.operatechar;

import android.content.Context;
import android.os.RemoteException;
import android.operatechar.IOperateCharService;
import android.util.Slog;

public class OperateCharManager
{
    private static final String TAG = "OperateCharManager";

    IOperateCharService mService;

    public String read(int maxLength) {
        try {
            return mService.read(maxLength);
        } catch (RemoteException e) {
            Slog.e(TAG, "read error!");
            return null;
        }
    }

    public void write(String mString) {
        try {
            mService.write(mString);
        } catch (RemoteException e) {
            Slog.e(TAG, "write error!");
        }
    }

    public OperateCharManager(Context context, IOperateCharService service) {
        mService = service;
    }
}
```

## 6. 启动访问硬件服务

[->frameworks/base/core/java/android/app/SystemServiceRegistry.java]

```java
import android.operatechar.OperateCharManager;
import android.operatechar.IOperateCharService;
...
final class SystemServiceRegistry {
...
    static {
    ...
    registerService(Context.OPERATECHAR_SERVICE, OperateCharManager.class,
                 new CachedServiceFetcher<OperateCharManager>() {
             @Override
             public OperateCharManager createService(ContextImpl ctx) {
                 IBinder b = ServiceManager.getService(Context.OPERATECHAR_SERVICE);
                 IOperateCharService service = IOperateCharService.Stub.asInterface(b);
                 if (service == null) {
                     return null;
                 }
                 return new OperateCharManager(ctx, service);
             }});
    }
    ...

    

```

registerService方法在 Android 系统起来后注册了 OperateCharService 服务，并且将 OperateCharManager 与OperateCharService 服务关联了起来，意味着 APP 调用 OperateCharManager 的 read 或 write 方法都会跨进程调用 OperateCharService 服务的 read 或 write 方法，然后通过 JNI 调用到 HAL 层，再到 内核的驱动程序访问 设备节点 /dev/mychar。

[->frameworks/base/services/java/com/android/server/SystemServer.java]

```java
import com.android.server.operatechar.OperateCharService;
...
public final class SystemServer {
...
    private void startOtherServices() {
    ...
        try {
            Slog.i(TAG, "OperateChar Service");
            operatechar = new OperateCharService(context);
            Slog.i(TAG, "Add OperateChar Service");
            ServiceManager.addService(Context.OPERATECHAR_SERVICE, operatechar);
            Slog.i(TAG, "OperateChar Service Succeed!");
        } catch (Throwable e) {
            Slog.e(TAG, "Failure starting OperateChar Service", e);
        }
    ...
```

Android 系统起来后，会调用 SystemServer.java 的相关方法启动Android系统的核心服务，在 startOtherServices 方法中会启动优先级低一点的系统服务，因此可以在 startOtherServices 方法中将 OperateCharService 启动并且添加到 ServiceManager 进行管理。

## 7. 添加Selinux策略权限

到目前为止已经实现了访问硬件服务的整套流程了，最后还需要修改 Selinux 策略，如果不修改，即使/dev/mychar设备节点有读写权限，但由于Selinux策略，访问硬件服务依然是不能访问/dev/mychar设备节点的。
由于我使用的是qcom高通的芯片平台，因此在高通平台下相关的文件进行修改。

[->device/qcom/sepolicy/common/device.te]

```
type mychar_device, dev_type;
```

定义一个mychar_device类型，其类型继承于dev_type。

[->device/qcom/sepolicy/common/file_contexts]

```
/dev/mychar                                     u:object_r:mychar_device:s0
```

将设备节点 /dev/mychar 的类型转为mychar_device，这一步的目的是只针对/dev/mychar设备节点开放相应的Selinux权限。

[->device/qcom/sepolicy/common/system_server.te]

```
allow system_server mychar_device:chr_file { open read write ioctl };
```

允许 system_server 进程对 mychar_device 类型的设备节点有 open read write ioctl 的操作权限。访问硬件服务是运行在system_server 进程中的。

[->device/qcom/sepolicy/common/service_contexts]

```
operate_char                              u:object_r:operate_char_service:s0
```

operate_char 就是 添加的访问硬件服务 OperateCharService，将 operate_char 转为 operate_char_service 类型。

[-> device/qcom/sepolicy/common/service.te]

```
type operate_char_service, app_api_service, system_server_service, service_manager_type;
```

如果还有报错，可以在logcat抓取的log搜avc关键字，查看缺少什么权限，就开放什么权限就行，

## 8. 更新系统API和编译

由于在framework 中添加了系统的 public 级别的方法，因此在全编译前需要更新API，执行指令如下：

```
make update-api
```

然后再执行全编译指令。

## 9. 编写APP测试验证

核心代码如下：

```java
import android.operatechar.OperateCharManager;
...
// 获取OperateCharManager
OperateCharManager oc = (OperateCharManager)getSystemService(Context.OPERATECHAR_SERVICE);
try {
    oc.write("Hello OperateChar");
    Log.d("vane", "Service returned: " + om.read(32));
 }
 catch (Exception e) {
    Log.d("vane", "FAILED to call service");
    e.printStackTrace();
 }
```

## 10. 总结

到此为止，已经将这个系列完成了，调用链如下：

APP调用->访问硬件服务客户端OperateCharManager->访问硬件服务OperateCharService->HAL 层模块operatechar->内核驱动程序mychar。

 该系列主要参考罗升阳的《Android系统源代码情景分析》的第二章：硬件抽象层。