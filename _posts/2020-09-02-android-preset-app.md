---
layout: post
title:  "Android系统预置APP"
date:   2020-09-02 00:00:00
catalog:  true
tags:
    - 系统
    - Android

---



>  本文主要介绍了如何在Android系统源码上预置APP，包括无源码APP和有源码APP两种情况。

## 1. 概述

Android 系统预置 APP 分为两种，一种是直接预置 APK，一种是预置带有源码的 APP。 

## 2.  预置 APK

以 AppDemo.apk 示例，在 vendor 目录下创建文件夹apk，然后在apk目录下新建名为 AppDemo的文件(一般第三方apk最好放在vendor目录下)，放入 AppDemo.apk，再新建 Android.mk，内容如下：

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := < your app folder name >
# 签名
LOCAL_CERTIFICATE := < desired key >
# 指定 src 目录 
LOCAL_SRC_FILES := < app apk filename >
# 将apk编译到/system/priv-app
LOCAL_PRIVILEGED_MODULE := true
LOCAL_MODULE_CLASS := APPS
# 该模块的后缀，不用定义
#LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
include $(BUILD_PREBUILT)
```

**解释：**

- LOCAL_PATH := $(call my-dir)

  每个 Android.mk 文件必须以定义 LOCAL_PATH 为开始，它用于在开发 tree 中查找源文件。

- include $(CLEAR_VARS)

  CLEAR_VARS 变量由 Build System 提供，并指向一个指定的 GNU Makefile，由它负责清理很多 LOCAL_xxx。

  例如：LOCAL_MODULE, LOCAL_SRC_FILES, LOCAL_STATIC_LIBRARIES 等等，但不清理 LOCAL_PATH。

- LOCAL_MODULE_TAGS ：= user eng tests optional

可选定义，表示在什么版本情况下编译该版本，默认 optional

```
- user: 指该模块只在 user 版本下才编译
- eng: 指该模块只在 eng 版本下才编译
- tests: 指该模块只在 tests 版本下才编译
- optional:指该模块在所有版本下都编译
```

- LOCAL_MODULE

  模块名，可不用定义，默认 = $(LOCAL_PACKAGE_NAME)，不能和既有模块相同，如果该变量未设置，则使用 LOCAL_PACKAGE_NAME，如果再没有，就会编译失败。

- LOCAL_CERTIFICATE

  使用的APK签名。

  testkey：普通 APK，默认情况下使用。

  platform：使用Android系统源码中的系统签名，路径在build/make/target/product/security/platform.x509.pem和  build/make/target/product/security//platform.pk8，分别是公钥和私钥。这种方式编译出来的 APK 所在进程的 UID 为 system。

  shared：该 APK 需要和 home/contacts 进程共享数据，可以参考 Launcher。

  media：该 APK 是 media/download 系统中的一环，可以参考 Gallery。

- LOCAL_MODULE_CLASS

指定模块的类型，可不用定义。

```makefile
# 编译 apk 文件
LOCAL_MODULE_CLASS := APPS
# 编译 jar 包
LOCAL_MODULE_CLASS := JAVA_LIBRAYIES
# 定义动态库文件
LOCAL_MODULE_CLASS := SHARED_LIBRAYIES
# 编译可执行文件
LOCAL_MODULE_CLASS := EXECUTABLES
```

- include $(BUILD_PACKAGE)

表示生成一个 APK，它可以是多种类型

```makefile
BUILD_PACKAGE（既可以编apk，也可以编资源包文件，但是需要指定LOCAL_EXPORT_PACKAGE_RESOURCES:=true)
BUILD_JAVA_LIBRARY（java共享库）
BUILD_STATIC_JAVA_LIBRARY（java静态库）
BUILD_EXECUTABLE（执行文件）
BUILD_SHARED_LIBRARY（native共享库）
BUILD_STATIC_LIBRARY（native静态库）
```

### 完整示例

AppDemo.apk 对应如下：

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := AppDemo
# 系统签名
LOCAL_CERTIFICATE := platform
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
LOCAL_MODULE_CLASS := APPS
#LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
include $(BUILD_PREBUILT)
```

### 更改 device.mk 文件

/build/target/board/lunch的版本/device.mk 文件，我编的是 aosp_x86-eng，所以增加或者更新 /build/target/board/generic_x86/device.mk：

```makefile
PRODUCT_PACKAGES += \
		Shadowsocks \
```

使用 mmm 命令来编译指定的模块：

```makefile
mmm vendor/apk/AppDemo
```

编译好模块后，还要重新打包一下 system.img 文件：

```makefile
make snod 
```

完成后就可以烧录了。

## 3. 预置有源码 APP

预置有源码 APP 可能会涉及 jar 包和 so 库等。我们先在 /packages/apps 新建名为 MyTestProject 的文件夹，在新建 MyTestProject /libs、MyTestProject /res、MyTestProject /src，分别将 MyTestProject 下 jar 包和 so 库拷到 MyTestProject /libs 和 MyTestProject /libs/armeabi，将 MyTestProject /app/src/main/res 拷到 MyTestProject /res，将 MyTestProject /app/src/main/java 下文件拷到 MyTestProject /src 下。

### 引用第三方 jar 包

假设，我们当前目录下的 libs 有 AndroidUtil.jar包，我们想引用它，需要做两个步骤：

第一步、 声明我们 jar 包所在的目录

```makefile
LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := AndroidUtil:libs/AndroidUtil.jar
```

这行代码的意思大概可以理解成这样，声明一个变量 AndroidUtil，它的 value 是 libs/AndroidUtil.jar

```makefile
include $(CLEAR_VARS)
LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := AndroidUtil:libs/AndroidUtil.jar
include $(BUILD_MULTI_PREBUILT)
```

第二步、 引用我们声明 jar 包的变量

```makefile
include $(CLEAR_VARS)
# 省略其他
LOCAL_STATIC_JAVA_LIBRARIES := \
		AndroidUtil
# 省略其他
include $(BUILD_PACKAGE)
```

### 引用 so 库

假设，我们当前目录下的 libs/armeabi 有 libBaiduMapSDK1.so、libBaiduMapSDK1.so，libs/arm64-v8a 有 libBaiduMapSDK1.so、libBaiduMapSDK1.so，我们想引用它，有两种方法，可以在根目录 Android.mk 引用 so 库，也可以在 libs 下再建个 Android.mk 配置好 so 库，然后 include，推荐第二种方式。

libs/Android.mk

```makefile
#====================================================
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_SUFFIX := .so
LOCAL_MODULE := libBaiduMapSDK1
LOCAL_MODULE_CLASS := SHARED_LIBRARIES
LOCAL_SRC_FILES_arm :=libs/armeabi/$(LOCAL_MODULE).so
LOCAL_SRC_FILES_arm64 :=libs/arm64-v8a/$(LOCAL_MODULE).so
LOCAL_MODULE_TARGET_ARCHS:= arm arm64
LOCAL_MULTILIB := both
include $(BUILD_PREBUILT)
#====================================================

#====================================================
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_SUFFIX := .so
LOCAL_MODULE := libBaiduMapSDK2
LOCAL_MODULE_CLASS := SHARED_LIBRARIES
LOCAL_SRC_FILES_arm :=libs/armeabi/$(LOCAL_MODULE).so
LOCAL_SRC_FILES_arm64 :=libs/arm64-v8a/$(LOCAL_MODULE).so
LOCAL_MODULE_TARGET_ARCHS:= arm arm64
LOCAL_MULTILIB := both
include $(BUILD_PREBUILT)
```

引用 so 库

```makefile
include $(CLEAR_VARS)
# 省略其他
LOCAL_JNI_SHARED_LIBRARIES :=  \
		libBaiduMapSDK1 \
		libBaiduMapSDK2
# 省略其他
include $(BUILD_PACKAGE)

##########引用第三方 so 库##########
include $(LOCAL_PATH)/libs/Android.mk
```

### 完整示例

```makefile
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional

LOCAL_PACKAGE_NAME := TestName

LOCAL_CERTIFICATE := platform

# 引入系统资源文件
LOCAL_USE_AAPT2 := true

# Java文件
LOCAL_SRC_FILES := $(call all-java-files-under, src)

# 资源文件，可选定义，推荐不定义
#LOCAL_RESOURCE_DIR = \
#        $(LOCAL_PATH)/res \
#        frameworks/support/v7/appcompat/res \
#        frameworks/support/design/res

# 可以使用系统 hide api
LOCAL_PRIVATE_PLATFORM_APIS := true

# 导入系统依赖
LOCAL_STATIC_ANDROID_LIBRARIES := \
        android-support-design \
        android-support-v4 \
        android-support-v7-appcompat \
        android-support-v7-recyclerview 

LOCAL_STATIC_JAVA_LIBRARIES := \
		AndroidUtil

LOCAL_JNI_SHARED_LIBRARIES :=  \
		libBaiduMapSDK1 \
		libBaiduMapSDK2

# R资源生成别名，--extra-packages 是为资源文件设置别名：意思是通过该应用包名+R，com.android.test1.R 和 com.android.test2.R 都可以访问到资源
LOCAL_AAPT_FLAGS := --auto-add-overlay
LOCAL_AAPT_FLAGS += --extra-packages android.support.v4
LOCAL_AAPT_FLAGS += --extra-packages android.support.v7.appcompat
LOCAL_AAPT_FLAGS += --extra-packages android.support.design
LOCAL_AAPT_FLAGS += --extra-packages android.support.v7.recyclerview

# 制定编译的工程，不要使用代码混淆的工具进行代码混淆
LOCAL_PROGUARD_ENABLED := disabled
# 指定不需要混淆的native方法与变量的proguard.flags文件
LOCAL_PROGUARD_FLAG_FILES := proguard.flags

include $(BUILD_PACKAGE)


##########引用第三方 jar 包##########
include $(CLEAR_VARS)

LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := AndroidUtil:libs/AndroidUtil.jar

include $(BUILD_MULTI_PREBUILT)

##########引用第三方 so 库##########
include $(LOCAL_PATH)/libs/Android.mk
```

### 问题

**1、LOCAL_PRIVATE_PLATFORM_APIS 和 LOCAL_SDK_VERSION 有什么区别？**

LOCAL_PRIVATE_PLATFORM_APIS := true
设置后，会使用 sdk 的 hide 的 api 来编译。

LOCAL_SDK_VERSION 这个编译配置，就会使编译的应用不能访问 hide 的 api，有时一些系统的 class 被 import 后编译时说找不到这个类，就是这个原因造成的。

**2、如果直接用 mmm 编译然后 adb install -r xxx.apk 大概会出现如下错误：**

```
Failed to install out/target/product/p212/system/app/xxx/xxx.apk: Failure [INSTALL_FAILED_INVALID_APK: Package couldn't be installed in /data/app/com.droidlogic.mboxlauncher-1: Package /data/app/com.droidlogic.mboxlauncher-1/base.apk code is missing]
```

解决方法：

在对应 app 的 Android.mk 文件中加入

```makefile
LOCAL_DEX_PREOPT := false
```

关闭 dex 优化来提高调试过程，把编译后的 APK 直接替换安装 adb install -r XXX.apk，不然 APK 得 Push 到 system/app，重启设备。

**3、在 Android Studio Gradle 方式中通过 implementation 方式加载的三方库，并没有下载 jar 文件放到 libs 文件夹下啊，该如何集成？**

其实 jar 包有被下载到项目的 External Libraries 目录下，找到引用的 jar 包，点右键 Show in Files，就能得到了 jar 包的文件地址，然后把它拷到 libs 文件夹下，就能像别的 jar 包一样处理了。

另外在 External Libraries 目录还能看到隐藏的 jar，比如 retrofit，其实它有引用 okhttp，okhttp 又引用了 okio，这些也是需要的，一并拷到 libs 文件夹下。