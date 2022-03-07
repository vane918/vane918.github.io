---
layout: post
title: "Android 10无法申请UVC摄像头权限的问题分析" 
date:   2022-03-07 00:00:00
catalog:  true
tags:
    - Android
    - USB权限
---



如果需要打开USB设备，如USB摄像头，需要在打开设备前先动态请求USB设备访问权限，如下是USB请求权限流程图

![image-20211018221219906](/images/usb/usb-request-permission.png)

该流程图来自：https://www.cnblogs.com/xiaoxiaing/p/12213750.html

当调用申请权限的API:UsbManager.java的requestPermission()方法时，如果是首次访问该USB设备(插拔USB设备或Android系统重启)，正常情况会弹出授权确认框，当点击确定授权或拒绝授权时，Android系统以广播的形式通知权限申请方。当在Android 10系统上，调用UsbManager.java的requestPermission()没有弹出权限确认框，默认返回授权失败的广播，该问题是AOSP的bug，下面结合源码简单分析下原因。

[xref](http://aospxref.com/android-10.0.0_r47/xref/): /[frameworks](http://aospxref.com/android-10.0.0_r47/xref/frameworks/)/[base](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/)/[services](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/)/[java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/)/[com](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/)/[android](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/)/[server](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/)/[UsbService.java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/UsbService.java)

```java
@Override
377      public void requestDevicePermission(UsbDevice device, String packageName, PendingIntent pi) {
378          final int uid = Binder.getCallingUid();
379          final int userId = UserHandle.getUserId(uid);
380  
381          final long token = Binder.clearCallingIdentity();
382          try {
383              getSettingsForUser(userId).requestPermission(device, packageName, pi, uid);
384          } finally {
385              Binder.restoreCallingIdentity(token);
386          }
387      }
388  
```

[xref](http://aospxref.com/android-10.0.0_r47/xref/): /[frameworks](http://aospxref.com/android-10.0.0_r47/xref/frameworks/)/[base](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/)/[services](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/)/[java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/)/[com](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/)/[android](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/)/[server](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/)/[UsbUserSettingsManager.java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java)

```java
public void requestPermission(UsbDevice device, String packageName, PendingIntent pi, int uid) {
210          Intent intent = new Intent();
211  
212          // respond immediately if permission has already been granted
213          if (hasPermission(device, packageName, uid)) {
214              intent.putExtra(UsbManager.EXTRA_DEVICE, device);
215              intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, true);
216              try {
217                  pi.send(mUserContext, 0, intent);
218              } catch (PendingIntent.CanceledException e) {
219                  if (DEBUG) Slog.d(TAG, "requestPermission PendingIntent was cancelled");
220              }
221              return;
222          }
223          if (isCameraDevicePresent(device)) {
224              if (!isCameraPermissionGranted(packageName, uid)) {
225                  intent.putExtra(UsbManager.EXTRA_DEVICE, device);
226                  intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, false);
227                  try {
228                      pi.send(mUserContext, 0, intent);
229                  } catch (PendingIntent.CanceledException e) {
230                      if (DEBUG) Slog.d(TAG, "requestPermission PendingIntent was cancelled");
231                  }
232                  return;
233              }
234          }
235  
236          requestPermissionDialog(device, null, canBeDefault(device, packageName), packageName, pi,
237                  uid);
238      }
```

问题出在上面224行sCameraPermissionGranted()方法。代码如下

[xref](http://aospxref.com/android-10.0.0_r47/xref/): /[frameworks](http://aospxref.com/android-10.0.0_r47/xref/frameworks/)/[base](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/)/[services](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/)/[java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/)/[com](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/)/[android](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/)/[server](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/)/[usb](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/)/[UsbUserSettingsManager.java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java)

```java
private boolean isCameraPermissionGranted(String packageName, int uid) {
135          int targetSdkVersion = android.os.Build.VERSION_CODES.P;
136          try {
137              ApplicationInfo aInfo = mPackageManager.getApplicationInfo(packageName, 0);
138              // compare uid with packageName to foil apps pretending to be someone else
139              if (aInfo.uid != uid) {
140                  Slog.i(TAG, "Package " + packageName + " does not match caller's uid " + uid);
141                  return false;
142              }
143              targetSdkVersion = aInfo.targetSdkVersion;
144          } catch (PackageManager.NameNotFoundException e) {
145              Slog.i(TAG, "Package not found, likely due to invalid package name!");
146              return false;
147          }
148  
149          if (targetSdkVersion >= android.os.Build.VERSION_CODES.P) {
150              int allowed = mUserContext.checkCallingPermission(android.Manifest.permission.CAMERA);
151              if (android.content.pm.PackageManager.PERMISSION_DENIED == allowed) {
152                  Slog.i(TAG, "Camera permission required for USB video class devices");
153                  return false;
154              }
155          }
156  
157          return true;
158      }
```

[xref](http://aospxref.com/android-10.0.0_r47/xref/): /[frameworks](http://aospxref.com/android-10.0.0_r47/xref/frameworks/)/[base](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/)/[core](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/)/[java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/)/[android](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/)/[app](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/app/)/[ContextImpl.java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/app/ContextImpl.java)

```java
@Override
1857      public int checkCallingPermission(String permission) {
1858          if (permission == null) {
1859              throw new IllegalArgumentException("permission is null");
1860          }
1861  
1862          int pid = Binder.getCallingPid();
1863          if (pid != Process.myPid()) {
1864              return checkPermission(permission, pid, Binder.getCallingUid());
1865          }
1866          return PackageManager.PERMISSION_DENIED;
1867      }
```

由UsbUserSettingsManager.java的149行可知，当targetSdkVersion设定为大于等于28时，allowed的值永远是PERMISSION_DENIED的，因此会返回false。allowed值之所以为PERMISSION_DENIED，是因为在上面UsbService.java中的381行调用了Binder.clearCallingIdentity()导致的。

该bug已有人提交给了google，从android-10.0.0_r30版本已经修复了该问题，但在android-10.0.0_r46又把该修复去掉了，但android11级以后的版本是修复该bug的。具体的bug谈论链接：https://issuetracker.google.com/issues/145082934?pli=1

修复的patch链接：https://android-review.googlesource.com/c/platform/frameworks/base/+/1193928

考虑到一部分人上不了google，下面是修复的patch的具体内容：

```java
From db62d2b4cf4d0914de518e5bca14383a5e0a5f63 Mon Sep 17 00:00:00 2001
From: Emilian Peev <epeev@google.com>
Date: Wed, 13 Nov 2019 15:47:42 -0800
Subject: [PATCH] Use correct calling identity during camera permission check

UVC clients need to acquire the CAMERA permission
before trying to call open. The permission check is
currently done via calls to "checkCallingPermission".
In case the calling identity gets cleared, the check
will always fail. To resolve this, use "checkPermission"
with appropriate identity arguments.

Bug: 144433314
Test: Manual using application,
CtsVerifier USB Device test

Change-Id: I5318fbb4426bec1448ecc398c86ea96500bb3189
Merged-In: I5318fbb4426bec1448ecc398c86ea96500bb3189
---

diff --git a/services/usb/java/com/android/server/usb/UsbHostManager.java b/services/usb/java/com/android/server/usb/UsbHostManager.java
index 00c7548..8122374 100644
--- a/services/usb/java/com/android/server/usb/UsbHostManager.java
+++ b/services/usb/java/com/android/server/usb/UsbHostManager.java
@@ -486,7 +486,7 @@
 
     /* Opens the specified USB device */
     public ParcelFileDescriptor openDevice(String deviceAddress, UsbUserSettingsManager settings,
-            String packageName, int uid) {
+            String packageName, int pid, int uid) {
         synchronized (mLock) {
             if (isBlackListed(deviceAddress)) {
                 throw new SecurityException("USB device is on a restricted bus");
@@ -498,7 +498,7 @@
                         "device " + deviceAddress + " does not exist or is restricted");
             }
 
-            settings.checkPermission(device, packageName, uid);
+            settings.checkPermission(device, packageName, pid, uid);
             return nativeOpenDevice(deviceAddress);
         }
     }
diff --git a/services/usb/java/com/android/server/usb/UsbSerialReader.java b/services/usb/java/com/android/server/usb/UsbSerialReader.java
index 8ca77f0..077d6b9 100644
--- a/services/usb/java/com/android/server/usb/UsbSerialReader.java
+++ b/services/usb/java/com/android/server/usb/UsbSerialReader.java
@@ -93,7 +93,7 @@
                                 UserHandle.getUserId(uid));
 
                         if (mDevice instanceof UsbDevice) {
-                            settings.checkPermission((UsbDevice) mDevice, packageName, uid);
+                            settings.checkPermission((UsbDevice) mDevice, packageName, pid, uid);
                         } else {
                             settings.checkPermission((UsbAccessory) mDevice, uid);
                         }
diff --git a/services/usb/java/com/android/server/usb/UsbService.java b/services/usb/java/com/android/server/usb/UsbService.java
index 4be68b8..13275f3 100644
--- a/services/usb/java/com/android/server/usb/UsbService.java
+++ b/services/usb/java/com/android/server/usb/UsbService.java
@@ -249,6 +249,7 @@
         if (mHostManager != null) {
             if (deviceName != null) {
                 int uid = Binder.getCallingUid();
+                int pid = Binder.getCallingPid();
                 int user = UserHandle.getUserId(uid);
 
                 long ident = clearCallingIdentity();
@@ -256,7 +257,7 @@
                     synchronized (mLock) {
                         if (mUserManager.isSameProfileGroup(user, mCurrentUserId)) {
                             fd = mHostManager.openDevice(deviceName, getSettingsForUser(user),
-                                    packageName, uid);
+                                    packageName, pid, uid);
                         } else {
                             Slog.w(TAG, "Cannot open " + deviceName + " for user " + user
                                     + " as user is not active.");
@@ -350,11 +351,12 @@
     @Override
     public boolean hasDevicePermission(UsbDevice device, String packageName) {
         final int uid = Binder.getCallingUid();
+        final int pid = Binder.getCallingPid();
         final int userId = UserHandle.getUserId(uid);
 
         final long token = Binder.clearCallingIdentity();
         try {
-            return getSettingsForUser(userId).hasPermission(device, packageName, uid);
+            return getSettingsForUser(userId).hasPermission(device, packageName, pid, uid);
         } finally {
             Binder.restoreCallingIdentity(token);
         }
@@ -376,11 +378,12 @@
     @Override
     public void requestDevicePermission(UsbDevice device, String packageName, PendingIntent pi) {
         final int uid = Binder.getCallingUid();
+        final int pid = Binder.getCallingPid();
         final int userId = UserHandle.getUserId(uid);
 
         final long token = Binder.clearCallingIdentity();
         try {
-            getSettingsForUser(userId).requestPermission(device, packageName, pi, uid);
+            getSettingsForUser(userId).requestPermission(device, packageName, pi, pid, uid);
         } finally {
             Binder.restoreCallingIdentity(token);
         }
diff --git a/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java b/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java
index 84add88..e1bfb8a 100644
--- a/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java
+++ b/services/usb/java/com/android/server/usb/UsbUserSettingsManager.java
@@ -127,11 +127,12 @@
      * Check for camera permission of the calling process.
      *
      * @param packageName Package name of the caller.
+     * @param pid Linux pid of the calling process.
      * @param uid Linux uid of the calling process.
      *
      * @return True in case camera permission is available, False otherwise.
      */
-    private boolean isCameraPermissionGranted(String packageName, int uid) {
+    private boolean isCameraPermissionGranted(String packageName, int pid, int uid) {
         int targetSdkVersion = android.os.Build.VERSION_CODES.P;
         try {
             ApplicationInfo aInfo = mPackageManager.getApplicationInfo(packageName, 0);
@@ -147,7 +148,8 @@
         }
 
         if (targetSdkVersion >= android.os.Build.VERSION_CODES.P) {
-            int allowed = mUserContext.checkCallingPermission(android.Manifest.permission.CAMERA);
+            int allowed = mUserContext.checkPermission(android.Manifest.permission.CAMERA, pid,
+                    uid);
             if (android.content.pm.PackageManager.PERMISSION_DENIED == allowed) {
                 Slog.i(TAG, "Camera permission required for USB video class devices");
                 return false;
@@ -157,9 +159,9 @@
         return true;
     }
 
-    public boolean hasPermission(UsbDevice device, String packageName, int uid) {
+    public boolean hasPermission(UsbDevice device, String packageName, int pid, int uid) {
         if (isCameraDevicePresent(device)) {
-            if (!isCameraPermissionGranted(packageName, uid)) {
+            if (!isCameraPermissionGranted(packageName, pid, uid)) {
                 return false;
             }
         }
@@ -171,8 +173,8 @@
         return mUsbPermissionManager.hasPermission(accessory, uid);
     }
 
-    public void checkPermission(UsbDevice device, String packageName, int uid) {
-        if (!hasPermission(device, packageName, uid)) {
+    public void checkPermission(UsbDevice device, String packageName, int pid, int uid) {
+        if (!hasPermission(device, packageName, pid, uid)) {
             throw new SecurityException("User has not given " + uid + "/" + packageName
                     + " permission to access device " + device.getDeviceName());
         }
@@ -206,11 +208,12 @@
                 accessory, canBeDefault, packageName, uid, mUserContext, pi);
     }
 
-    public void requestPermission(UsbDevice device, String packageName, PendingIntent pi, int uid) {
+    public void requestPermission(UsbDevice device, String packageName, PendingIntent pi, int pid,
+            int uid) {
         Intent intent = new Intent();
 
         // respond immediately if permission has already been granted
-        if (hasPermission(device, packageName, uid)) {
+        if (hasPermission(device, packageName, pid, uid)) {
             intent.putExtra(UsbManager.EXTRA_DEVICE, device);
             intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, true);
             try {
@@ -221,7 +224,7 @@
             return;
         }
         if (isCameraDevicePresent(device)) {
-            if (!isCameraPermissionGranted(packageName, uid)) {
+            if (!isCameraPermissionGranted(packageName, pid, uid)) {
                 intent.putExtra(UsbManager.EXTRA_DEVICE, device);
                 intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, false);
                 try {

```

如果是作为app层的开发者想解决该问题，可以将targetSdkVersion设置为小于28，如27，但如果在google play上架，貌似targetSdkVersion必须要大于27。
