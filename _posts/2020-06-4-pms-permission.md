---
layout: post
title:  "Android安全机制之Permission权限应用篇"
date:   2020-06-4 00:00:00
catalog:  true
tags:
    - 权限
    - 安全机制
    - Android
---



## 1. 概述

 Android 是一个权限分隔的操作系统，这是利用 Linux 已有的权限管理机制，通过为每一个应用程序分配不同的 UID 和 GID ， 从而使得不同的应用程序之间的私有数据和访问达到隔离的目的。与此同时， Android 还 在此基础上进行扩展，提供了 permission 机制，它主要是用来对应用程序可以执行的某些具体操作进行权限细分和访问控制。
permission 机制的目的是保护Android用户的隐私。Android应用程序必须申请访问敏感用户数据（如联系人和短信）以及某些系统功能（如摄像头和互联网）的权限。根据功能的不同，系统可能会自动授予权限，也可能会提示用户批准请求。Android安全体系结构的中心设计要点是，默认情况下，任何应用程序都没有权执行任何对其他应用程序、操作系统或用户产生不利影响的操作。这包括读取或写入用户的私人数据（例如联系人或电子邮件）、读取或写入其他应用程序的文件、执行网络访问、使设备保持唤醒状态等等。

## 2. 权限类型

Permission机制的权限按保护级别可分为正常权限、签名权限、危险权限和特殊权限。按照请求的时机可分为安装时请求和运行时请求。危险权限和特殊权限在API 级别23或更高级别需要运行时请求，正常权限和签名权限在安装时授权。

### 2.1 保护级别

权限分为几个保护级别，保护级别会影响是否需要运行时权限请求。

- 正常权限：涵盖应用需要访问其沙盒外部数据或资源，但对用户隐私或其他应用操作风险很小的区域。例如，设置时区的权限就是正常权限。如果应用声明其需要正常权限，系统会自动向应用授予该权限，并且用户无法撤消这些权限。如需当前正常权限的完整列表，请参阅[正常权限](https://developer.android.google.cn/guide/topics/security/normal-permissions?hl=zh-cn)。

-  危险权限涵盖应用需要涉及用户隐私信息的数据或资源，或者可能对用户存储的数据或其他应用的操作产生影响的区域。例如，能够读取用户的联系人属于危险权限。如果应用声明其需要危险权限，则用户必须明确向应用授予该权限。必须需要用户批准许可后，应用程序才能使用该许可的功能。要使用危险权限，应用程序必须提示用户在运行时授予权限。

-  签名权限：系统会在安装时授予这些应用程序权限，前提是请求该签名权限的应用程序与定义该权限的应用程序需要使用相同的证书签名。该类型权限的用法简单来说一个应用程序自定义一个权限，另外一个拥有相同签名的应用程序就可以请求该自定义的权限。

-  特殊权限：有一些权限其行为方式与正常权限及危险权限都不同。比如`SYSTEM_ALERT_WINDOW` 和 `WRITE_SETTINGS` 特别敏感，因此大多数应用不应该使用它们。如果某应用需要其中一种权限，必须在清单中声明该权限，并且发送请求用户授权的 intent，然后系统跳转到相应的界面，示意用户授权。

  如 WRITE_SETTINGS 权限，如果应用程序的目标是 API 级别23或更高级别，则应用程序用户必须通过权限管理屏幕向该应用程序明确授予此权限。该应用通过发送带有操作 `Settings.ACTION_MANAGE_WRITE_SETTINGS` 的 intent 来请求用户的批准。该应用可以通过调用 `Settings.System.canWrite()` 来检查其是否具有此授权。

### 2.2 权限组

所有危险的 Android 系统权限都属于权限组。如果设备运行的是 Android 6.0（API 级别 23），并且应用的 `targetSdkVersion` 是 23 或更高版本，则当用户请求危险权限时系统会发生以下行为：

- 如果应用请求其清单中列出的危险权限，而应用目前在权限组中没有任何权限，则系统会向用户显示一个对话框，描述应用要访问的权限组。对话框不描述该组内的具体权限。例如，如果应用请求 `READ_CONTACTS` 权限，系统对话框只说明该应用需要访问设备的联系信息。如果用户批准，系统将向应用授予其请求的权限。
- 如果应用请求其清单中列出的危险权限，而应用在同一权限组中已有另一项危险权限，则系统会立即授予该权限，而无需与用户进行任何交互。例如，如果某应用已经请求并且被授予了 `READ_CONTACTS` 权限，然后它又请求 `WRITE_CONTACTS`，系统将立即授予该权限。

简单来说， 如果应用程序申请了一个组的权限，那么这个组的其他权限也一并申请了。 
任何权限都可属于一个权限组，包括正常权限和应用定义的权限。但权限组仅当是危险权限时才影响用户体验。可以忽略正常权限的权限组。
如果设备运行的是 Android 5.1（API 级别 22）或更低版本，并且应用的 **targetSdkVersion** 是 22 或更低版本，则系统会在安装时要求用户授予权限。再次强调，系统只告诉用户应用需要的权限组，而不告知具体权限。

**表 1.** 危险权限和权限组。

| 权限组       | 权限                                                         |
| :----------- | :----------------------------------------------------------- |
| `CALENDAR`   | `READ_CALENDAR` `WRITE_CALENDAR`                             |
| `CAMERA`     | `CAMERA`                                                     |
| `CONTACTS`   | `READ_CONTACTS` `WRITE_CONTACTS` `GET_ACCOUNTS`              |
| `LOCATION`   | `ACCESS_FINE_LOCATION` `ACCESS_COARSE_LOCATION`              |
| `MICROPHONE` | `RECORD_AUDIO`                                               |
| `PHONE`      | `READ_PHONE_STATE` `CALL_PHONE` `READ_CALL_LOG` `WRITE_CALL_LOG` `ADD_VOICEMAIL` `USE_SIP` `PROCESS_OUTGOING_CALLS` |
| `SENSORS`    | `BODY_SENSORS`                                               |
| `SMS`        | `SEND_SMS` `RECEIVE_SMS` `READ_SMS` `RECEIVE_WAP_PUSH` `RECEIVE_MMS` |
| `STORAGE`    | `READ_EXTERNAL_STORAGE` `WRITE_EXTERNAL_STORAGE`             |

## 3. 申请权限方式

 每款 Android 应用程序都在访问受限的沙盒中运行。如果应用程序需要使用其自己的沙盒外的资源或信息，则必须请求相应权限。要声明应用程序需要某项权限，可以在应用清单中列出该权限，Android 6.0 及更高版本对于危险权限则需要在运行时请求用户批准每项权限。 

### 3.1 向清单添加权限

 对于所有 Android 版本，要声明应用需要某项权限，请在应用清单中添加<uses-permission>元素，作为顶级<manifest>元素的子项。例如，需要访问互联网的应用需在清单中添加以下代码行： 

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
            package="com.example.snazzyapp">
        <uses-permission android:name="android.permission.INTERNET"/>
        <!-- other permissions go here -->
        <application ...>
            ...
        </application>
    </manifest>
```

 系统在声明权限之后的行为取决于权限的敏感程度。有些权限被视为普通权限（即不会给用户隐私或设备操作带来太大风险的权限），因此系统会在安装应用时立即授予这些权限。还有些则被视为危险权限（即可能影响用户隐私或设备正常操作的权限），因此需要用户明确向应用程序授予相应访问权限。 如需详细了解普通权限和危险权限，请参阅[保护级别](https://developer.android.google.cn/guide/topics/permissions/overview#normal-dangerous)。 

### 3.2 检查权限

 如果设备搭载的是 Android 5.1.1（API 级别 22）或更低版本，或者应用在任何版本的 Android 上运行时其**targetSdkVersion** 是 22 或更低版本，系统将在安装时自动请求用户向应用授予所有危险权限 。
如果设备搭载的是 Android 6.0（API 级别 23）或更高版本，并且应用的 **targetSdkVersion** 是 23 或更高版本，如果应用需要一项危险权限，那么每次执行需要该权限的操作时，都必须检查自己是否具有该权限。用户可随时从任何应用撤消权限。 
用户在安装时不会收到任何应用权限的通知。应用程序必须在运行时请求用户授予危险权限。当应用请求权限时，用户会看到一个系统对话框（如图 1 左图所示），告知用户应用正在尝试访问的权限组。该对话框包括**拒绝**和**允许**按钮。如果用户拒绝权限请求，当应用下次请求该权限时，该对话框将包含一个复选框，选中它即表示用户不想再收到权限提示 。 如果用户选中**不再询问**复选框并点按**拒绝**，当应用程序以后尝试请求相同权限时，系统不会再提示用户授予权限，除非用户在设置中手动授予权限。
要检查应用程序是否具有某项权限，调用 `ContextCompat.checkSelfPermission()` 方法。例如，以下代码段展示了如何检查 Activity 是否具有向日历写入数据的权限：

```java
 if (ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.WRITE_CALENDAR)
            != PackageManager.PERMISSION_GRANTED) {
        // Permission is not granted
    }
```

 如果应用具有此权限，该方法将返回 `PERMISSION_GRANTED`，并且应用可以继续操作。如果应用不具备此权限，该方法将返回 `PERMISSION_DENIED`，且应用必须明确要求用户授予权限。 

### 3.3 请求权限

当应用从 `checkSelfPermission()` 收到 `PERMISSION_DENIED` 时，需要提示用户授予该权限。Android 提供了几种可用来请求权限的方法（如 `requestPermissions()`），如下面的代码段所示。调用这些方法时，会显示一个对话框供用户选择是否授权。

 **注意：不要在用户打开应用程序时就检查或请求权限，而要等到用户选择或打开需要特定权限的功能时再检查或请求权限。** 

```java
// Here, thisActivity is the current activity
    if (ContextCompat.checkSelfPermission(thisActivity,
            Manifest.permission.READ_CONTACTS)
            != PackageManager.PERMISSION_GRANTED) {

        // Permission is not granted
        // Should we show an explanation?
        if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
                Manifest.permission.READ_CONTACTS)) {
            // Show an explanation to the user *asynchronously* -- don't block
            // this thread waiting for the user's response! After the user
            // sees the explanation, try again to request the permission.
        } else {
            // No explanation needed; request the permission
            ActivityCompat.requestPermissions(thisActivity,
                    new String[]{Manifest.permission.READ_CONTACTS},
                    MY_PERMISSIONS_REQUEST_READ_CONTACTS);

            // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
            // app-defined int constant. The callback method gets the
            // result of the request.
        }
    } else {
        // Permission has already been granted
    }
```

### 3.4 处理权限请求响应

 当用户响应您应用的权限请求时，系统会调用应用的 `onRequestPermissionsResult()` 方法，在调用过程中向其传递用户响应。应用必须重写该方法，以查明是否已向其授予相应权限。在回调过程中传递的请求代码与传递给 `requestPermissions()` 的请求代码相同。例如，如果应用请求 `READ_CONTACTS` 访问权限，则它可能采用以下回调方法： 

```java
@Override
    public void onRequestPermissionsResult(int requestCode,
            String[] permissions, int[] grantResults) {
        switch (requestCode) {
            case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
                // If request is cancelled, the result arrays are empty.
                if (grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    // permission was granted, yay! Do the
                    // contacts-related task you need to do.
                } else {
                    // permission denied, boo! Disable the
                    // functionality that depends on this permission.
                }
                return;
            }
            // other 'case' lines to check for other
            // permissions this app might request.
        }
    }
```

### 3.5 处理权限请求遭拒情况

如果用户拒绝了权限请求，应用程序应该帮助用户了解拒绝授予权限的影响。具体而言，应用应该让用户知道因缺少权限而无法使用的功能。在处理这种情况时，请牢记以下最佳做法：

- **引导用户的注意力。**在应用界面中突出显示因为应用没有必要的权限而受限的功能所在的具体部分。以下列举了几个例子说明您可以采取的做法：
  - 在该功能的结果或数据会出现的位置显示一条消息。
  - 显示一个包含错误图标并带有相应错误颜色的不同按钮。
- **内容要具体**。显示的消息不要空泛；而要指出因为应用没有必要的权限而无法使用的具体功能。
- **不要阻止界面显示。**换言之，不要显示全屏警告消息，让用户根本无法继续使用应用程序。

在某些情况下，系统可能会自动拒绝权限，而无需用户执行任何操作。（同样，系统也可能会自动授予权限。）请千万不要对系统的自动行为做出任何假设。每当应用需要使用的功能需要权限时，您都应该检查应用是否仍被授予该权限。
如需在请求应用权限时提供最佳用户体验，另请参阅[应用权限最佳做法](https://developer.android.google.cn/training/permissions/usage-notes)。