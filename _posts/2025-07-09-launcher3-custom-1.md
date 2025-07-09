---
layout: post
title:  "Android 13 Launcher3定制系列1"
date:   2025-07-09 00:00:00
catalog:  true
tags:
    - android
    - launcher3

---

## 1.隐藏hotseat

```java
diff --git a/res/layout/launcher.xml b/res/layout/launcher.xml
index 039d8d3..17b8eb5 100644
--- a/res/layout/launcher.xml
+++ b/res/layout/launcher.xml
@@ -47,6 +47,7 @@
         <!-- DO NOT CHANGE THE ID -->
         <include
             android:id="@+id/hotseat"
+            android:visibility="gone"
             layout="@layout/hotseat" />

         <!-- Keep these behind the workspace so that they are not visible when
diff --git a/src/com/android/launcher3/DeviceProfile.java b/src/com/android/launcher3/DeviceProfile.java
index b276397..102bf69 100644
--- a/src/com/android/launcher3/DeviceProfile.java
+++ b/src/com/android/launcher3/DeviceProfile.java
@@ -548,6 +548,8 @@ public class DeviceProfile {
                     + hotseatBarBottomPaddingPx + (isScalableGrid ? 0 : hotseatExtraVerticalSize)
                     + hotseatBarSizeExtraSpacePx;
         }
+        //隐藏hotseat
+        hotseatBarSizePx = 0;
     }

     private Point getCellLayoutBorderSpace(InvariantDeviceProfile idp) {
```

## 2.隐藏所有应用界面的Serach apps搜索栏

```java
diff --git a/res/layout/search_container_all_apps.xml b/res/layout/search_container_all_apps.xml
index e1646ba..4fbfc58 100644
--- a/res/layout/search_container_all_apps.xml
+++ b/res/layout/search_container_all_apps.xml
@@ -16,6 +16,7 @@
 <com.android.launcher3.allapps.search.AppsSearchContainerLayout
     xmlns:android="http://schemas.android.com/apk/res/android"
     android:id="@id/search_container_all_apps"
+    android:visibility="gone"
     android:layout_width="match_parent"
     android:layout_height="@dimen/all_apps_search_bar_field_height"
     android:layout_centerHorizontal="true"
diff --git a/src/com/android/launcher3/config/FeatureFlags.java b/src/com/android/launcher3/config/FeatureFlags.java
index fdd3615..9317fa1 100644
--- a/src/com/android/launcher3/config/FeatureFlags.java
+++ b/src/com/android/launcher3/config/FeatureFlags.java
@@ -53,7 +53,7 @@ public final class FeatureFlags {
      * and should be modified at a project level.
      */
     //public static final boolean QSB_ON_FIRST_SCREEN = BuildConfig.QSB_ON_FIRST_SCREEN;
-    public static final boolean QSB_ON_FIRST_SCREEN = true;
+    public static final boolean QSB_ON_FIRST_SCREEN = false;

     /**
      * Feature flag to handle define config changes dynamically instead of killing the process.
diff --git a/src_build_config/com/android/launcher3/BuildConfig.java b/src_build_config/com/android/launcher3/BuildConfig.java
index 9a81d3f..8c83bcc 100644
--- a/src_build_config/com/android/launcher3/BuildConfig.java
+++ b/src_build_config/com/android/launcher3/BuildConfig.java
@@ -23,5 +23,5 @@ public final class BuildConfig {
      * Flag to state if the QSB is on the first screen and placed on the top,
      * this can be overwritten in other launchers with a different value, if needed.
      */
-    public static final boolean QSB_ON_FIRST_SCREEN = true;
+    public static final boolean QSB_ON_FIRST_SCREEN = false;
 }
```

## 3.缩短应用列表第一行应用距离顶部的距离

```xml
diff --git a/res/values/dimens.xml b/res/values/dimens.xml
index 8403af4..17eadfe 100644
--- a/res/values/dimens.xml
+++ b/res/values/dimens.xml
@@ -123,8 +123,8 @@
     <dimen name="all_apps_header_pill_corner_radius">12dp</dimen>
     <dimen name="all_apps_header_tab_height">48dp</dimen>
     <dimen name="all_apps_tabs_indicator_height">2dp</dimen>
-    <dimen name="all_apps_header_top_margin">33dp</dimen>
-    <dimen name="all_apps_header_top_padding">36dp</dimen>
+    <dimen name="all_apps_header_top_margin">1dp</dimen>
+    <dimen name="all_apps_header_top_padding">10dp</dimen>
     <dimen name="all_apps_header_bottom_padding">14dp</dimen>
     <dimen name="all_apps_header_top_adjustment">6dp</dimen>
     <dimen name="all_apps_header_bottom_adjustment">4dp</dimen>
```

## 4.应用列表界面白色背景换成和首页背景一致

```java
diff --git a/src/com/android/launcher3/views/ScrimView.java b/src/com/android/launcher3/views/ScrimView.java
index 4c0bfde..be11b46 100644
--- a/src/com/android/launcher3/views/ScrimView.java
+++ b/src/com/android/launcher3/views/ScrimView.java
@@ -70,6 +70,8 @@ public class ScrimView extends View implements Insettable {

     @Override
     public void setBackgroundColor(int color) {
+        //所有应用界面白色背景换成和首页背景一致 by vane
+        color = 0;
         mBackgroundColor = color;
         updateSysUiColors();
         dispatchVisibilityListenersIfNeeded();
```

也可以使用下面的方法：

```java
diff --git a/quickstep/src/com/android/launcher3/uioverrides/states/AllAppsState.java b/quickstep/src/com/android/launcher3/uioverrides/states/AllAppsState.java
index a74774c..6e79f1f 100644
--- a/quickstep/src/com/android/launcher3/uioverrides/states/AllAppsState.java
+++ b/quickstep/src/com/android/launcher3/uioverrides/states/AllAppsState.java
@@ -19,6 +19,7 @@ import static com.android.launcher3.anim.Interpolators.DEACCEL_2;
 import static com.android.launcher3.logging.StatsLogManager.LAUNCHER_STATE_ALLAPPS;

 import android.content.Context;
+import android.graphics.Color;

 import com.android.launcher3.DeviceProfile.DeviceProfileListenable;
 import com.android.launcher3.Launcher;
@@ -111,8 +112,10 @@ public class AllAppsState extends LauncherState {

     @Override
     public int getWorkspaceScrimColor(Launcher launcher) {
-        return launcher.getDeviceProfile().isTablet
-                ? launcher.getResources().getColor(R.color.widgets_picker_scrim)
-                : Themes.getAttrColor(launcher, R.attr.allAppsScrimColor);
+        //by vane:allApps界面和首页显示一样的背景
+        return Color.TRANSPARENT;
+//        return launcher.getDeviceProfile().isTablet
+//                ? launcher.getResources().getColor(R.color.widgets_picker_scrim)
+//                : Themes.getAttrColor(launcher, R.attr.allAppsScrimColor);
     }
 }
diff --git a/quickstep/src/com/android/launcher3/uioverrides/states/OverviewState.java b/quickstep/src/com/android/launcher3/uioverrides/states/OverviewState.java
index 6427e09..50d6d7a 100644
--- a/quickstep/src/com/android/launcher3/uioverrides/states/OverviewState.java
+++ b/quickstep/src/com/android/launcher3/uioverrides/states/OverviewState.java
@@ -19,6 +19,7 @@ import static com.android.launcher3.anim.Interpolators.DEACCEL_2;
 import static com.android.launcher3.logging.StatsLogManager.LAUNCHER_STATE_OVERVIEW;

 import android.content.Context;
+import android.graphics.Color;
 import android.graphics.Rect;
 import android.os.SystemProperties;

@@ -106,7 +107,9 @@ public class OverviewState extends LauncherState {

     @Override
     public int getWorkspaceScrimColor(Launcher launcher) {
-        return Themes.getAttrColor(launcher, R.attr.overviewScrimColor);
+        //by vane:点击最近应用列表按键的背景和首页一致
+//        return Themes.getAttrColor(launcher, R.attr.overviewScrimColor);
+        return Color.TRANSPARENT;
     }

     @Override
```

## 5.应用列表界面app标题颜色换成白色

```java
diff --git a/src/com/android/launcher3/BubbleTextView.java b/src/com/android/launcher3/BubbleTextView.java
index 5fb8925..2231d8c 100644
--- a/src/com/android/launcher3/BubbleTextView.java
+++ b/src/com/android/launcher3/BubbleTextView.java
@@ -205,6 +205,8 @@ public class BubbleTextView extends TextView implements ItemInfoUpdateReceiver,
             setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.allAppsIconTextSizePx);
             setCompoundDrawablePadding(grid.allAppsIconDrawablePaddingPx);
             defaultIconSize = grid.allAppsIconSizePx;
+            //所有应用界面app标题颜色换成白色
+            setTextColor(Color.parseColor("#FFFFFF"));
         } else if (mDisplay == DISPLAY_FOLDER) {
             setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.folderChildTextSizePx);
             setCompoundDrawablePadding(grid.folderChildDrawablePaddingPx);
```

## 6.横屏不隐藏首页应用的标题

```java
diff --git a/src/com/android/launcher3/DeviceProfile.java b/src/com/android/launcher3/DeviceProfile.java
index 102bf69..cbe6854 100644
--- a/src/com/android/launcher3/DeviceProfile.java
+++ b/src/com/android/launcher3/DeviceProfile.java
@@ -34,6 +34,7 @@ import android.graphics.Point;
 import android.graphics.PointF;
 import android.graphics.Rect;
 import android.util.DisplayMetrics;
 import android.view.Surface;

 import com.android.launcher3.CellLayout.ContainerType;
@@ -822,7 +823,9 @@ public class DeviceProfile {

         updateAllAppsContainerWidth(res);
         if (isVerticalBarLayout()) {
-            hideWorkspaceLabelsIfNotEnoughSpace();
+            // 横屏不隐藏首页应用的标题 by vane
+//            hideWorkspaceLabelsIfNotEnoughSpace();
         }
     }
```

## 7.应用列表实现高斯模糊功能

Android Launcher3原生的高斯模糊效果不明显，需要加强高斯模糊。在 Android 12 中，新增了一些用于实现窗口模糊处理效果（例如背景模糊处理和模糊处理后方屏幕）的公共 API，调用 [Window#setBackgroundBlurRadius(int)](https://developer.android.google.cn/reference/android/view/Window?hl=zh-cn#setBackgroundBlurRadius(int)) 设置背景模糊处理半径。

注意:需要使用`adb shell getprop ro.surface_flinger.supports_background_blur`查看是否开启了高斯模糊的功能，为1是已经启动该功能，不启动高斯模糊功能使用对应的API设置高斯模糊是不生效的。google官方说明链接：[窗口模糊处理](https://source.android.google.cn/docs/core/display/window-blurs?hl=zh-cn)

下面在应用列表界面进行高斯模糊，而首页不进行高斯模糊的功能，需要处理这两个界面切换时高斯模糊的动态设置，容易出现bug。

```java
commit 9e1aa4d045ba68352da9f78d59bbcde120072730
Author: vane <vane918@163.com>
Date:   Wed Aug 14 16:05:19 2024 +0800

    feat: 应用列表背景实现高斯模糊效果

diff --git a/quickstep/src/com/android/quickstep/util/BaseDepthController.java b/quickstep/src/com/android/quickstep/util/BaseDepthController.java
index 29ae9a1..8e81c78 100644
--- a/quickstep/src/com/android/quickstep/util/BaseDepthController.java
+++ b/quickstep/src/com/android/quickstep/util/BaseDepthController.java
@@ -21,7 +21,10 @@ import android.util.FloatProperty;
 import android.view.AttachedSurfaceControl;
 import android.view.SurfaceControl;
 
+import com.android.launcher3.CTZShareData;
 import com.android.launcher3.Launcher;
+import com.android.launcher3.LauncherState;
 import com.android.launcher3.R;
 import com.android.launcher3.Utilities;
 import com.android.launcher3.util.MultiPropertyFactory;
@@ -92,6 +95,13 @@ public class BaseDepthController {
         mLauncher = activity;
         mMaxBlurRadius = activity.getResources().getInteger(R.integer.max_depth_blur_radius);
         mWallpaperManager = activity.getSystemService(WallpaperManager.class);
+        CTZShareData.getInstance().setOnToStateChangeListener(new CTZShareData.OnToStateChangeListener() {
+            @Override
+            public void onUpdateStateChange(LauncherState state) {
+                applyDepthAndBlur();
+            }
+        });
     }
 
     protected void setCrossWindowBlursEnabled(boolean isEnabled) {
@@ -104,6 +114,10 @@ public class BaseDepthController {
     }
 
     protected void applyDepthAndBlur() {
+        if (!CTZShareData.getInstance().getNeedSetBlur()) {
+            return;
+        }
         float depth = mDepth;
         IBinder windowToken = mLauncher.getRootView().getWindowToken();
         if (windowToken != null) {
@@ -121,6 +135,13 @@ public class BaseDepthController {
 
         mCurrentBlur = !mCrossWindowBlursEnabled || hasOpaqueBg
                 ? 0 : (int) (depth * mMaxBlurRadius);
+
+        //by vane:解决在ALL_APPS界面点击recent按键后选择应用，然后点击返回键高斯模糊没有的问题
+        if (CTZShareData.getInstance().getToState() == LauncherState.ALL_APPS && mCurrentBlur == 0 && depth == 0) {
+            mCurrentBlur = 23;
+            mDepth = 1.0F;
+        }
+
         SurfaceControl.Transaction transaction = new SurfaceControl.Transaction()
                 .setBackgroundBlurRadius(mSurface, mCurrentBlur)
                 .setOpaque(mSurface, isSurfaceOpaque);
@@ -141,6 +162,16 @@ public class BaseDepthController {
         if (rootSurfaceControl != null) {
             rootSurfaceControl.applyTransactionOnDraw(transaction);
         }
+
+        if(CTZShareData.getInstance().getToState() == LauncherState.NORMAL) {
+            mLauncher.setBlur(0);
+        } else if (CTZShareData.getInstance().getToState() == LauncherState.ALL_APPS) {
+            if (mCurrentBlur == 23){
+                mLauncher.setBlur(150);
+            } else if (mCurrentBlur == 0) {
+                mLauncher.setBlur(0);
+            }
+        }
     }
 
     protected void setDepth(float depth) {
diff --git a/src/com/android/launcher3/CTZShareData.java b/src/com/android/launcher3/CTZShareData.java
new file mode 100644
index 0000000..b7a2a1a
--- /dev/null
+++ b/src/com/android/launcher3/CTZShareData.java
@@ -0,0 +1,72 @@
+package com.android.launcher3;
+
+/**
+ * CTZShareData 类用于在不同模块共享一些变量，如LauncherState。
+ * 它采用单例模式确保整个应用中只有一个实例。
+ * 提供了状态更新的通知机制，以便相关模块可以根据状态变化进行相应的处理。
+ */
+public class CTZShareData {
+
+    //单例模式
+    private static CTZShareData mInstance = null;
+    private CTZShareData() {
+    }
+    public static CTZShareData getInstance() {
+        if (mInstance == null) {
+            mInstance = new CTZShareData();
+        }
+        return mInstance;
+    }
+
+    private LauncherState toState;
+
+    private LauncherState currentState;
+
+    public boolean getNeedSetBlur() {
+        if (toState == LauncherState.BACKGROUND_APP || toState == LauncherState.OVERVIEW)
+            return false;
+
+        return true;
+    }
+
+    public void setToState(LauncherState toState) {
+        this.toState = toState;
+    }
+
+    public void updateState(LauncherState toState) {
+        this.toState = toState;
+        mOnToStateChangeListener.onUpdateStateChange(toState);
+    }
+
+    public LauncherState getToState() {
+        return toState;
+    }
+
+    public void setCurrentState(LauncherState currentState) {
+        this.currentState = currentState;
+    }
+
+    public LauncherState getCurrentState() {
+        return currentState;
+    }
+
+    //toState变量更新监听器
+    public interface OnToStateChangeListener {
+        void onUpdateStateChange(LauncherState toState);
+    }
+
+    private OnToStateChangeListener mOnToStateChangeListener;
+    /**
+     * 设置状态改变监听器。
+     * 该接口主要是方便在不同的模块回调真实的LauncherState，比如在首页上拉进入应用列表，但是如果没有到达进入应用列表，
+     * 原生的流程LauncherState就已经更改为ALL_APPS，只有某个时刻才会更改为NORMAL，此时HOME状态已经改变，但是toState还是ALL_APPS，
+     * 因此需要通知高斯模糊渲染模块，重新渲染。
+     * 目前只有DepthController模块注册该接口。
+     *
+     * @param listener 状态改变监听器接口的实现。这个参数提供了当状态改变时需要执行的操作。
+     */
+    public void setOnToStateChangeListener(OnToStateChangeListener listener) {
+        mOnToStateChangeListener = listener;
+    }
+
+}
diff --git a/src/com/android/launcher3/Launcher.java b/src/com/android/launcher3/Launcher.java
index 2ec4b32..441c103 100644
--- a/src/com/android/launcher3/Launcher.java
+++ b/src/com/android/launcher3/Launcher.java
@@ -110,6 +110,7 @@ import android.view.MotionEvent;
 import android.view.View;
 import android.view.ViewGroup;
 import android.view.ViewTreeObserver.OnPreDrawListener;
+import android.view.WindowManager;
 import android.view.WindowManager.LayoutParams;
 import android.view.accessibility.AccessibilityEvent;
 import android.view.animation.OvershootInterpolator;
@@ -557,6 +561,10 @@ public class Launcher extends StatefulActivity<LauncherState>
             getWindow().setSoftInputMode(LayoutParams.SOFT_INPUT_ADJUST_NOTHING);
         }
         setTitle(R.string.home_screen);
+
+        mHomeAndRecentAppReceiver = new HomeAndRecentAppReceiver();
+        IntentFilter myFilter = new IntentFilter(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
+        registerReceiver(mHomeAndRecentAppReceiver, myFilter, Context.RECEIVER_NOT_EXPORTED);
     }
 
     protected LauncherOverlayManager getDefaultOverlay() {
@@ -1187,6 +1195,139 @@ public class Launcher extends StatefulActivity<LauncherState>
         return Optional.of(LAUNCHER_ALLAPPS_EXIT);
     }
 
+    // by vane: 监听Home键和RecentApp键。
+    // 如监听到RecentApp键后取消毛玻璃效果，解决在应用列表打开应用后点击RecentApp键会出现毛玻璃bug
+    private HomeAndRecentAppReceiver mHomeAndRecentAppReceiver;
+
+    private Runnable pendingBlurRunnable = null;
+
+    public class HomeAndRecentAppReceiver extends BroadcastReceiver {
+
+        private final String SYSTEM_DIALOG_REASON_KEY = "reason";
+        private final String SYSTEM_DIALOG_REASON_RECENT_APPS = "recentapps";
+        private final String SYSTEM_DIALOG_REASON_HOME_KEY = "homekey";
+
+        private String lastProcessedReason = "";
+        private long lastProcessedTime = 0;
+
+        @Override
+        public void onReceive(Context context, Intent intent) {
+            String action = intent.getAction();

+            if (Intent.ACTION_CLOSE_SYSTEM_DIALOGS.equals(action)) {
+                String reason = intent.getStringExtra(SYSTEM_DIALOG_REASON_KEY);
+                if (reason == null) {
+                    return;
+                }
+
+                long currentTime = System.currentTimeMillis();
+                if (lastProcessedReason.isEmpty() || !reason.equals(lastProcessedReason) || currentTime - lastProcessedTime > 250) {
+                    //home key
+                    if (reason.equals(SYSTEM_DIALOG_REASON_HOME_KEY)) {
+                        // Handle HOME key press
+                        if (pendingBlurRunnable != null) {
+                            mWorkspace.removeCallbacks(pendingBlurRunnable);
+                            pendingBlurRunnable = null;
+                        }
+//                        CTZShareData.getInstance().setNeedSetBlur(true);
+                        setBlur(0);
+                    }
+
+                    //recent key
+                    else if (reason.equals(SYSTEM_DIALOG_REASON_RECENT_APPS)) {
+                        // Handle RECENT_APPS key press
+                        setBlur(0);
+                        CTZShareData.getInstance().setToState(mStateManager.getState());
+
+
+                        if (pendingBlurRunnable != null) {
+                            mWorkspace.removeCallbacks(pendingBlurRunnable);
+                            pendingBlurRunnable = null;
+                        }
+
+                        pendingBlurRunnable = new Runnable() {
+                            @Override
+                            public void run() {
+                                enterSplitSelect();
+                            }
+                        };
+
+                        mWorkspace.postDelayed(pendingBlurRunnable, 1000);
+
+                    }
+
+                } else {
+                }
+
+                lastProcessedReason = reason;
+                lastProcessedTime = currentTime;
+            }
+        }
+    }
+
+
+    private int mBlurValue = -1;
+
+    // 高斯模糊开关 by vane
+    private final static boolean CTZ_BLUR_ENABLE = true;
+    public void setBlur(int value) {
+        if (!CTZ_BLUR_ENABLE)
+            return;
+
+        if (getBlur() == value) {
+//            CTZLog.d("setBlur value:" + value + " is same");
+            return;
+        }
+
+        //背景实现毛玻璃效果(高斯模糊) by vane
+        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
+            CTZLog.d("setBlur value:" + value);
+            WindowManager.LayoutParams params = this.getWindow().getAttributes();
+            params.flags |= LayoutParams.FLAG_BLUR_BEHIND;
+            params.setBlurBehindRadius(value);
+            this.getWindow().setAttributes(params);
+
+            mBlurValue = value;
+        }
+    }
+
+    protected int getBlur() {
+        return mBlurValue;
+    }
+
     @Override
     protected void onResume() {
         Object traceToken = TraceHelper.INSTANCE.beginSection(ON_RESUME_EVT,
@@ -1719,6 +1860,7 @@ public class Launcher extends StatefulActivity<LauncherState>
         ACTIVITY_TRACKER.onActivityDestroyed(this);
 
         unregisterReceiver(mScreenOffReceiver);
+        unregisterReceiver(mHomeAndRecentAppReceiver);
         mWorkspace.removeFolderListeners();
         PluginManagerWrapper.INSTANCE.get(this).removePluginListener(this);
 
diff --git a/src/com/android/launcher3/allapps/AllAppsTransitionController.java b/src/com/android/launcher3/allapps/AllAppsTransitionController.java
index 872c4fd..d30abe4 100644
--- a/src/com/android/launcher3/allapps/AllAppsTransitionController.java
+++ b/src/com/android/launcher3/allapps/AllAppsTransitionController.java
@@ -32,6 +32,7 @@ import android.view.HapticFeedbackConstants;
 import android.view.View;
 import android.view.animation.Interpolator;
 
+import com.android.launcher3.CTZShareData;
 import com.android.launcher3.DeviceProfile;
 import com.android.launcher3.DeviceProfile.OnDeviceProfileChangeListener;
 import com.android.launcher3.Launcher;
@@ -215,6 +216,7 @@ public class AllAppsTransitionController
      */
     @Override
     public void setState(LauncherState state) {
+        CTZShareData.getInstance().updateState(state);
         setProgress(state.getVerticalProgress(mLauncher));
         setAlphas(state, new StateAnimationConfig(), NO_ANIM_PROPERTY_SETTER);
         onProgressAnimationEnd();
@@ -227,6 +229,7 @@ public class AllAppsTransitionController
     @Override
     public void setStateWithAnimation(LauncherState toState,
             StateAnimationConfig config, PendingAnimation builder) {
+        CTZShareData.getInstance().setToState(toState);
         if (mLauncher.isInState(ALL_APPS) && !ALL_APPS.equals(toState)) {
             // For atomic animations, we close the keyboard immediately.
             if (!config.userControlled && !FeatureFlags.ENABLE_KEYBOARD_TRANSITION_SYNC.get()) {

```

