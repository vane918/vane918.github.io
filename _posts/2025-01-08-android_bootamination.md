---
layout: post
title:  "Android开机动画流程"
date:   2025-01-08 00:00:00
catalog:  true
tags:
    - android
    - framework
    - 开机动画

---

## 1.Android开机动画概述

Android开机动画是设备启动过程中用户看到的视觉效果，通常在操作系统加载时播放。它的主要目的是：

1. **用户体验**：通过动态的视觉效果，增强用户的期待感，使设备启动过程更加愉悦。
2. **品牌识别**：不同的Android设备制造商（如三星、华为、小米等）会设计独特的开机动画，以体现品牌特色。
3. **系统状态指示**：开机动画可以让用户知道设备正在启动，避免用户在启动过程中产生不必要的焦虑。

#### 开机动画的特点

- **动态性**：通常包含动画效果，如渐变、旋转或其他视觉特效。
- **时间限制**：开机动画的持续时间通常在几秒钟内，以便尽快进入系统界面。
- **可定制性**：用户和开发者可以通过修改系统文件或使用特定应用程序来自定义开机动画。

#### 技术实现

开机动画通常由以下几个部分组成：

- **图像资源**：包含动画所需的图像文件，通常是PNG或SVG格式。
- **动画文件**：定义动画的播放顺序和效果，可能使用XML或其他格式。
- **系统调用**：在设备启动时，系统会调用这些资源并按照预设的顺序播放动画。

总之，Android开机动画不仅是设备启动过程中的一部分，更是用户与设备互动的第一步，能够有效提升用户的使用体验。

## 2.开机动画涉及到的模块

在Android 13版本中，开机动画涉及多个模块和代码路径。以下是主要模块及其作用的概述：

### 1. **init**

- **代码路径**: `system/core/init/`

- **作用**

  `init`模块负责系统的启动过程，包括启动各种服务和设置系统属性。在开机过程中，它会加载必要的配置文件，如`init.rc`，并启动`bootanimation`服务。

### 2. **bootanimation**

- **代码路径**: `system/media/bootanimation/`

- **作用**

  `bootanimation`模块专门用于处理开机动画的播放。它读取动画的配置文件（通常是`desc.txt`），然后根据该配置播放指定的帧图像序列。该模块是用户在设备开机时看到的动画的核心部分。

### 3. **SurfaceFlinger**

- **代码路径**: `frameworks/native/services/surfaceflinger/`

- **作用**

  `SurfaceFlinger`是Android的图形合成服务，负责将不同的图形层合成到屏幕上。在开机动画期间，`SurfaceFlinger`处理来自`bootanimation`模块的图像，并将其渲染到显示设备上。

### 4. **System Server**

- **代码路径**: `frameworks/base/services/core/java/com/android/server/

- **作用**

  `System Server`是Android系统的核心服务，负责管理各种系统服务。在开机动画期间，它会确保所有必要的服务都已启动，并协调它们的运行。

## 3.启动流程

![startup](/images/bootamination/startup.png)

Android开机动画启动流程时序图如下：

![startup](/images/bootamination/seq.png)

## 4.源码分析

> system/core/rootdir/init.rc
>
> frameworks/native/services/surfaceflinger
>
> frameworks/base/cmds/bootanimation
>
> system/core/init/init.cpp

### 4.1 启动surfaceflinger

surfaceflinger和bootanimation进程的启动是init进程第二阶段中启动的，具体流程是在init.cpp的LoadBootScripts中解析init.rc脚本，init.rc脚本执行到`class_start core`时，会启动classname 为 core 的 Service。

bootanim.rc

```
service bootanim /system/bin/bootanimation
    class core animation
	...
```

surfaceflinger.rc

```
service surfaceflinger /system/bin/surfaceflinger
    class core animation
```

main_surfaceflinger.cpp

```cpp
int main(int, char**) {
    ...
    // initialize before clients can connect
    flinger->init();
    ...
    // run surface flinger in this thread
    flinger->run();

    return 0;
}
```

SurfaceFlinger.cpp

```cpp
// Do not call property_set on main thread which will be blocked by init
// Use StartPropertySetThread instead.
void SurfaceFlinger::init() {
    ...

    if (mStartPropertySetThread->Start() != NO_ERROR) {
        ALOGE("Run StartPropertySetThread failed!");
    }

    ...
    ALOGV("Done initializing");
}
```

StartPropertySetThread.cpp

```cpp
status_t StartPropertySetThread::Start() {
    return run("SurfaceFlinger::StartPropertySetThread", PRIORITY_NORMAL);
}

bool StartPropertySetThread::threadLoop() {
    // Set property service.sf.present_timestamp, consumer need check its readiness
    property_set(kTimestampProperty, mTimestampPropertyValue ? "1" : "0");
    // Clear BootAnimation exit flag
    property_set("service.bootanim.exit", "0");
    property_set("service.bootanim.progress", "0");
    // Start BootAnimation if not started
    property_set("ctl.start", "bootanim");
    // Exit immediately
    return false;
}
```

可以看到在surfaceflinger进程在线程中将系统属性service.bootanim.exit和service.bootanim.progress分别设置为0，顾名思义，这两个属性分别表示开机动画是否已经退出和开机动画的进度。

### 4.2 启动bootanimation

bootanimation进程也是有init.rc通过`class_start core`触发的。

bootanimation_main.cpp

```cpp
int main()
{
    setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);

    bool noBootAnimation = bootAnimationDisabled();
    ALOGI_IF(noBootAnimation,  "boot animation disabled");
    if (!noBootAnimation) {

        sp<ProcessState> proc(ProcessState::self());
        ProcessState::self()->startThreadPool();

        // create the boot animation object (may take up to 200ms for 2MB zip)
        sp<BootAnimation> boot = new BootAnimation(audioplay::createAnimationCallbacks());

        waitForSurfaceFlinger();

        boot->run("BootAnimation", PRIORITY_DISPLAY);

        ALOGV("Boot animation set up. Joining pool.");

        IPCThreadState::self()->joinThreadPool();
    }
    return 0;
}
```

上面的开机动画工作如下：

1. 进程优先级设置

   ```cpp
   setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);
   ```

   - 设置进程优先级为显示优先级（ANDROID_PRIORITY_DISPLAY）

   - 确保开机动画获得足够的系统资源

   - 保证动画流畅性

2. 检查是否需要禁用开机动画

   ```cpp
   bootAnimationDisabled()
   ```

3. 创建开机动画对象

   解压2MB的zip开机动画资源耗时200ms左右。

4. 等待图形系统就绪

   等待 SurfaceFlinger 服务启动完成。SurfaceFlinger 是 Android 的核心图形服务，确保有可用的显示系统才开始播放动画。

5. 启动动画

   - 开始执行开机动画
   - 设置动画线程名称为 "BootAnimation"
   - 使用显示优先级运行

#### 4.2.1检查是否需要禁用开机动画

BootAnimationUtil.cpp

```cpp
bool bootAnimationDisabled() {
    char value[PROPERTY_VALUE_MAX];
    property_get("debug.sf.nobootanimation", value, "0");
    if (atoi(value) > 0) {
        return true;
    }

    property_get("ro.boot.quiescent", value, "0");
    if (atoi(value) > 0) {
        // Only show the bootanimation for quiescent boots if this system property is set to enabled
        if (!property_get_bool("ro.bootanim.quiescent.enabled", false)) {
            return true;
        }
    }

    return false;
}
```

1. bootAnimationDisabled() 函数的作用是判断是否需要禁用开机动画，返回 true 表示禁用，false 表示启用。

2. 有两种情况会禁用开机动画：

   1. 调试模式禁用

       如果设置了debug.sf.nobootanimation属性且大于0，禁用开机动画

   2. 静默启动模式

      在静默启动模式下，除非特别设置了ro.bootanim.quiescent.enabled为true，否则也会禁用开机动画

**关键属性说明：**

- debug.sf.nobootanimation: 调试开关，用于禁用开机动画
- ro.boot.quiescent: 静默启动模式标志
- ro.bootanim.quiescent.enabled: 在静默启动模式下是否允许显示开机动画

#### 4.2.2启动动画

1. 加载动画资源
   - findBootAnimationFile：查找开机动画文件
   - loadAnimation：加载动画资源
2. 初始化图形环境
3. 动画循环播放

##### 查找开机动画文件

**基本流程**

```cpp
void BootAnimation::findBootAnimationFile() {
    // 1. 检查加密状态
    if (!mShuttingDown && encryptedAnimation) {
        if (findBootAnimationFileInternal(encryptedBootFiles)) {
            return;
        }
    }

    // 2. 检查自定义动画
    if (access(custAnim, R_OK) == 0) {
        mZipFileName = custAnim;
        return;
    }

    // 3. 根据不同场景选择查找路径
    if (android::base::GetBoolProperty("sys.init.userspace_reboot.in_progress", false)) {
        // 用户空间重启场景
        findBootAnimationFileInternal(userspaceRebootFiles);
    } else if (mShuttingDown) {
        // 关机场景
        findBootAnimationFileInternal(shutdownFiles);
    } else {
        // 正常启动场景
        const bool playDarkAnim = android::base::GetIntProperty("ro.boot.theme", 0) == 1;
        findBootAnimationFileInternal(bootFiles);
    }
}
```

**查找优先级**

1. 加密启动状态：
   1. /product/media/bootanimation-encrypted.zip
   2. /system/media/bootanimation-encrypted.zip
2. 自定义动画：
   1. persist.sys.customanim.boot
   2. persist.sys.customanim.shutdown
3. 用户空间重启：
   1. /product/media/userspace-reboot.zip
   2. /oem/media/userspace-reboot.zip
   3. /system/media/userspace-reboot.zip
4. 正常启动：
   1. /apex/com.android.bootanimation/etc/bootanimation.zip
   2. /product/media/bootanimation-dark.zip (暗色主题时)
   3. /product/media/bootanimation.zip
   4. /oem/media/bootanimation.zip
   5. /system/media/bootanimation.zip
5. 关机动画：
   1. /product/media/shutdownanimation.zip
   2. /oem/media/shutdownanimation.zip
   3. /system/media/shutdownanimation.zip

#### 4.2.3 自定义开机动画bootanimation.zip

**bootanimation.zip 目录结构**

bootanimation.zip
├── desc.txt           # 动画描述文件
├── part0/             # 第一部分动画
│   ├── 00001.png
│   ├── 00002.png
│   └── ...
└── part1/             # 第二部分动画
    ├── 00001.png
    ├── 00002.png
    └── ...

**desc.txt**

```
1280 720 12
p 1 0 part0
p 0 0 part1
```

基本参数：

1280 720 12
|    |    |
|    |    └── 帧率(fps)
|    └──────── 高度(height)
└──────────── 宽度(width)

动画部分定义：

p 1 0 part0
| | | |
| | | └── 文件夹名称
| | └──── 暂停时间(pause)
| └────── 播放次数(count)
└──────── 播放类型(type)

**参数详解**

A. 播放类型(type)

```
'p': // part - 普通部分
'c': // complete - 播放完成才继续
'f': // fade - 支持淡入淡出效果
```

B. 播放次数(count)

```
0: 无限循环播放
n: 播放n次
```

C. 暂停时间(pause)

```
单位：帧数
0: 无暂停
n: 暂停n帧
```

根据上面的分析，上面的desc.txt行为如下：

播放流程：

以 1280x720 分辨率，12fps 的速率播放

先播放 part0 文件夹中的动画一次

然后无限循环播放 part1 文件夹中的动画。

将bootanimation.zip放在/product/media/目录下即可。

生成bootanimation.zip的方式为存储模式。选择存储模式，不要选择压缩模式，不然会导致不开机等各种问题。bootanimation.zip仅仅是打包了的文件，并没有压缩，切记。



