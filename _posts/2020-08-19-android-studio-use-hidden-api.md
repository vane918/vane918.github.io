---
layout: post
title:  "Android Studio使用系统隐藏API的方法"
date:   2020-08-19 00:00:00
catalog:  true
tags:
    - 系统
    - Android

---



>  本文主要介绍了如何使用 Android Studio 进行App开发时调用 Android 的 internal API 和 hidden API，同时也介绍如何在Android Studio 进行App开发时调用定制的Android 系统的API。

## 1. 概述

Android Studio 是Google基于 IntelliJ IDEA 且适用于开发 Android 应用的官方集成开发环境 (IDE)。使用Android Studio开发App时，一般会选择指定的编译版本，选择编译版本之后，项目会使用SDK中对应版本的android.jar(路径：***/sdk/platforms/android-version/android.jar)进行开发。
Android 有两种类型的 API 不能通过 SDK 访问。 

- com.android.internal 包中的 API，称之为 internal API
- 被标记为 @hide 属性的类和方法，这是被隐藏的 API，称之为 hidden API

这两种类型的API不能通过SDK来进行访问的原因是sdk/platforms/android-version/android.jar被进行了处理。在android.jar中，默认是移除了所有的被@hide标识的方法或者类，以及internal包下的类，所以在开发的时候这两种类型的API无法被访问到。
为什么我们进行反射就能够调用这些类和方法呢？
在Android设备上，使用的并不是跟SDK中相同的jar包，当应用在设备上运行时，它会加载 /system/framework/framework.jar ，framework.jar和android.jar的唯一的区别就是它没有移除 internal API 和 hidden API，这就说明了为什么可以通过反射调用，因为我们开发的SDK中不包含这些API，所以我们无法进行显式的调用，当我们利用反射，程序在设备上运行的时候，其实是可以找到对应的方法进行调用的。
本文主要介绍了如何使用 Android Studio 进行App开发时调用 Android 的 internal API 和 hidden API，同时也介绍如何在Android Studio 进行App开发时调用定制的Android 系统的API。

## 2.  如何使用系统隐藏API

### 2.1 整体思路

Android Studio 使用Android Studio 提供的SDK开发时系统有意屏蔽了一些类和方法，在应用开发的时候不让开发者进行使用，但是这些类和方法是确实存在的。开发者在Android Studio中使用系统隐藏API的方法是将Android Studio 使用的 android.jar替换为制作的android.jar就可以了（制作的android.jar包含隐藏的API）。

### 2.2 详细步骤

调用 Android 的隐藏 API的步骤如下：
1)	修改module 的compileSdkVersion 和 targetSdkVersion
2)	找到Android源码编译出来的framework.jar文件
3)	制作android.jar文件 
4)	将制作的android.jar文件替换Android Studio 使用的 android.jar。
下面具体介绍每个步骤的使用说明。以Android 8.1版本为例说明。

#### 2.2.1修改module的compileSdkVersion 和 targetSdkVersion

修改xxx/build.gradle(例如app/build.gradle)文件的compileSdkVersion和targetSdkVersion为27。如下：

![gradle](/images/others/gradle.png)

修改compileSdkVersion和targetSdkVersion为27后，需要根据项目的情况修改对应的依赖库版本。

- 如果项目中使用了com.android.support依赖库，则com.android.support的版本不能超过27，建议设置为com.android.support:appcompat-v7:27.1.1，如下图所示：

  ![support](/images/others/support.png)

- 如果项目中使用了androidx依赖库，则需要将androidx依赖库用com.android.support替换，如下图所示：

  ![androidxx](/images/others/androidxx.png)

  替换为

  ![support2](/images/others/support2.png)

  还需要修改gradle.properties文件的android.useAndroidX和android.enableJetifier，将其值设置为false。

#### 2.2.2找到Android源码编译生成的framework.jar文件

1. 制作的android.jar文件步骤如下：
   首先创建一个文件夹，命名为new-android，将 Android Studio 使用的 android.jar文件拷贝到 创建的new-android文件夹中。
   快速找到Android Studio 使用的 android.jar 文件的方法是鼠标左键选中Extenal Libraries的android.jar，然后鼠标右击，在弹出来的菜单中选中Show in Explorer，就会跳转到android.jar所在的文件夹中。
2. 将new-android文件夹中的android.jar文件解压到当前文件夹。
3. 将out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar文件拷贝到new-android文件夹中，然后将classes.jar解压到当前文件夹，如果弹出“确认文件替换”窗口，则点击“全部选是”。
4. 删除new-android文件夹中的android.jar 和 classes.jar 文件
5. 在cmd 程序中使用下面的指令对new-android文件夹进行jar 打包，命名为android.jar。
   jar cvf android.jar -C ***/new-android/ .
   例如new-android文件夹放在E:\ new-android，则jar 打包命令为：
   E:\>jar cvf android.jar -C new-android/ .
   打包后的android.jar在E:\>目录下。
6. 将步骤5打包生成的android.jar 替换SDK中的android.jar。建议替换之前先备份SDK的android.jar，以便后续使用。

完成上面的步骤后，就可以在Android Studio中调用 android 的隐藏API了。
