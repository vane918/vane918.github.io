---
layout: post
title:  "Android开机启动之FallbackHome和Launcher启动"
date:   2025-06-13 00:00:00
catalog:  true
tags:
    - android
    - framework
    - 开机启动

---

## 1.FallbackHome的启动

![fallbackhome](/images/startup/fallbackhome.png)

Android系统起来后，首先启动的Activity并不是launcher，而是FallbackHome，FallbackHome是Settings的一部分。

FallbackHome主要就是因为涉及整个android系统的加密等原因，系统在还没有完全解锁前，不可以启动Launcher，因为Launcher中明显和各个第三方应用耦合较多（比如桌面可能显示着一堆的各个应用的Widget），如果直接Launcher作为FallbackHome启动，相对就会要求Launcher依赖的应用也是支持直接解密类型，那就肯定不现实。所以就先启动了FallbackHome一个什么也不显示的界面来作为启动真正Launcher的一个过渡。

### 1.1 系统是如何选择哪个应用作为HOME应用的？

根据下面的信息：

- Action: Intent.ACTION_MAIN
- Category: Intent.CATEGORY_HOME

### 1.2 为什么系统优先选择FallbackHome，而不是Launcher作为HOME应用？

1. **系统启动阶段与Direct Boot (直接启动模式)**

   - 当Android设备启动时，会先进入一个受限的环境，称为**Direct Boot模式**。在这个模式下，用户数据分区（通常是/data/user/0）是加密的，直到用户输入锁屏凭据（PIN、密码、图案）后才会被解密。
   - 只有标记为android:directBootAware="true"的应用组件（Activities, Services, Receivers, Providers）才能在Direct Boot模式下运行。这些应用只能访问设备加密存储（Device Encrypted storage），不能访问凭据加密存储（Credential Encrypted storage）。

2. **FallbackHome 的角色**

   - FallbackHome是一个非常轻量级的、系统内置的Home应用。
   - **关键点：FallbackHome被标记为android:directBootAware="true"。**
   - 它的主要目的是在系统启动早期，当用户尚未解锁设备，或者常规的Launcher（如Launcher3）因故无法启动时，提供一个临时的、基础的桌面界面。这确保了即使用户数据未解密，系统也至少有一个可以显示的“家”。

3. **Launcher3 (或第三方Launcher) 的情况**

   - 标准的Launcher应用（如AOSP的Launcher3或用户安装的第三方Launcher）通常不是directBootAware的。
   - 原因是Launcher需要访问用户的应用列表、小部件配置、壁纸偏好等，这些数据都存储在凭据加密存储中，只有在用户解锁设备后才能访问。
   - 如果Launcher不是directBootAware，它就无法在Direct Boot模式下运行。

4. **PMS 如何解析 CATEGORY_HOME Intent**

   当系统需要启动一个Home应用时（通常由ActivityManagerService发起），它会构建一个Intent，通常包含：

   - Action: Intent.ACTION_MAIN
   - Category: Intent.CATEGORY_HOME

   然后，PMS会使用这个Intent来查询所有匹配的Activity。这个解析过程会考虑以下因素：

   - **a. android:directBootAware 属性 (在Direct Boot模式下至关重要)**
     - **启动初期 (Direct Boot模式下)：** 当系统处于Direct Boot模式（用户未解锁）时，PMS在解析CATEGORY_HOME Intent时，会**只考虑那些android:directBootAware="true"的Activity**。
     - 由于FallbackHome是directBootAware的，而Launcher3（通常）不是，所以在设备刚刚启动、用户还未解锁时，FallbackHome是唯一（或少数几个）符合条件的候选者。因此，系统会启动FallbackHome。
   - **b. android:priority 属性**
     - android:priority用于intent-filter，表示处理特定Intent的优先级。数值越大，优先级越高。
     - **对于CATEGORY_HOME的解析：**
       - **在Direct Boot模式下：** 如果有多个directBootAware="true"的Home应用，priority可以作为选择的依据之一。FallbackHome通常会有一个合理的优先级以确保它被选中。
       - **用户解锁后：** 当用户解锁设备，凭据加密存储可用时，所有Launcher（无论是否directBootAware）都成为潜在候选者。此时，系统会：
         1. **检查用户是否设置了默认Launcher。** 如果用户已经选择了一个默认Launcher（比如Launcher3或Nova Launcher），系统会优先启动这个用户选择的Launcher，priority的影响会减弱或消失。
         2. **如果没有默认Launcher且有多个匹配项：** 系统通常会弹出一个选择器让用户选择，或者根据其他规则（如系统预置Launcher的优先级可能高于用户安装的）。priority在这里可以作为一种排序或默认提示的参考。
         3. **FallbackHome的低优先级：** FallbackHome通常会有一个相对较低的priority（例如，您提到的-1000可能适用于此场景，或者它本身的优先级就低于常规Launcher），这样一旦常规Launcher可用，系统就不会再倾向于选择FallbackHome。
   - **c. 组件启用状态 (enabled 属性和 PackageManager.setComponentEnabledSetting())**
     - 只有组件状态为enabled的Activity才会被考虑。
   - **d. 用户偏好 (Preferred Activity)**
     - 一旦用户选择了某个Launcher作为“默认”或“始终使用”，这个选择会被系统记录下来。后续解析CATEGORY_HOME时，PMS会直接选择这个用户偏好的Launcher，除非它被卸载或禁用。

### **1.3 FallbackHome如何和launcher3关联的**

开机先启动设置中的FallbackHome，然后再启动真正的Launcher3——是Android系统设计中的一个重要容错和引导机制。下面解释一下FallbackHome和真正的Launcher（如Launcher3）是如何关联和工作的：

**1. FallbackHome 的目的和角色**

*   **安全网/容错机制**：
    *   **Launcher损坏或缺失**：如果用户安装的默认Launcher（如Launcher3）因为某些原因（例如，被卸载、数据损坏、崩溃）无法启动，系统需要一个备用的“家”界面来保证用户至少能进行基本操作（比如进入设置修复问题）。
    *   **早期启动阶段**：在系统启动的非常早期，完整的 `PackageManagerService` 可能还没有完全准备好所有应用信息，或者默认的Launcher依赖的某些服务尚未就绪。此时启动一个轻量级的FallbackHome可以更快地展示UI。
*   **首次开机/恢复出厂设置引导**：
    *   在全新的设备或恢复出厂设置后，系统通常会进入设置向导（Setup Wizard）。这个向导本身可能被注册为一个临时的Home应用，或者FallbackHome被用来承载这个向导的一部分。一旦向导完成，系统才会切换到真正的Launcher。
*   **轻量级**：FallbackHome通常是系统设置应用（`com.android.settings`）中的一个非常简单的Activity（例如`com.android.settings.FallbackHome`）。它功能有限，主要目的是提供一个最低限度的可操作界面。

**2. FallbackHome 和 Launcher3 的关联机制**

这种关联和切换主要依赖于Android的Intent解析机制、Activity优先级以及系统启动流程中的特定检查点。

*   **Intent解析与优先级**：
    *   Launcher应用（包括FallbackHome和Launcher3）都会在其`AndroidManifest.xml`中声明一个Intent Filter，表明它们可以处理`android.intent.action.MAIN`和`android.intent.category.HOME`这两个Action和Category。
    *   `PackageManagerService` (PMS) 负责解析这些Intent。当系统需要启动Home应用时，它会查询PMS所有能处理`CATEGORY_HOME`的Activity。
    *   真正的Launcher（如Launcher3）通常具有比FallbackHome更高的优先级（通过`android:priority`属性设置，或者系统内部有默认的排序逻辑）。
    *   FallbackHome的`AndroidManifest.xml`中通常会有一个较低的优先级，或者它只在没有其他更高优先级的HOME Activity可用时才被选中。

*   **系统启动流程中的切换**：
    *   **早期启动 (Pre-`systemReady`)**:
        *   在`SystemServer`启动的早期阶段，尤其是在`ActivityManagerService` (AMS) 的 `systemReady()` 方法被完全调用之前，系统可能需要显示一个界面。如果此时PMS还不能准确地解析出用户首选的、功能完备的Launcher，或者首选Launcher还未准备好，AMS可能会选择启动FallbackHome。这就是您看到的第一个调用栈中 `systemReady()` 内部触发 `startHomeActivity:com.android.settings` 的情况。
    *   **`systemReady()` 完成后**:
        *   当`SystemServer`中的关键服务（包括PMS）都准备就绪后，AMS会调用其`systemReady()`方法。这个方法内部或之后，系统会更可靠地尝试启动“正确”的Home应用。
        *   此时，PMS能够准确地告诉AMS哪个是用户配置的（或默认的）优先级最高的Launcher (如Launcher3)。
        *   AMS/ATMS会再次尝试启动Home。如果之前启动的是FallbackHome，并且现在真正的Launcher可用，系统会启动真正的Launcher。这可能会导致FallbackHome被`finish()`掉，或者新的Launcher覆盖在其之上。
    *   **Setup Wizard 完成**:
        *   如果FallbackHome或一个专门的Setup Wizard Activity是作为初始Home运行的，当用户完成设置流程后，这个Activity会调用`finish()`。当一个Task的根Activity结束且没有其他Activity在该Task中时，系统会尝试重新解析并启动`CATEGORY_HOME`的Intent，此时就会启动真正的Launcher。
    *   **Activity栈管理**:
        *   当一个Activity（比如FallbackHome或SetupWizard）`finish()`后，`ActivityManagerService`会检查当前的Activity栈。如果栈顶的Task是Home Task并且变空了，它会触发启动新的Home Activity的逻辑。这就是您看到的第二个调用栈中，`ActivityRecord.completeFinishing` -> ... -> `Task.resumeNextFocusableActivityWhenRootTaskIsEmpty` -> `RootWindowContainer.resumeHomeActivity` -> `startHomeActivity:com.android.launcher3` 的流程。

**总结起来，FallbackHome和Launcher3的关联可以看作是一个多阶段的启动和容错策略：**

1.  **初始/紧急阶段**：系统可能先启动FallbackHome，因为它简单、可靠，并且在系统服务未完全就绪时也能启动。这确保了用户在任何情况下都有一个可用的界面。
2.  **系统就绪/正常阶段**：一旦系统完全启动，`PackageManagerService`能够准确识别出用户首选的或系统默认的Launcher3，并且Launcher3自身也准备就绪，系统就会启动Launcher3。
3.  **切换机制**：
    *   可能是系统在`systemReady`后主动重新评估并启动正确的Home。
    *   也可能是临时的Home（如FallbackHome或SetupWizard）在完成其使命后主动`finish()`，从而触发系统寻找并启动下一个合适的Home应用（即Launcher3）。

这种设计确保了Android设备启动过程的健壮性，即使用户的默认Launcher出现问题，或者在首次设置过程中，系统依然能够提供基本的用户交互。

## 2.FallbackHome的退出

> com.android.settings.FallbackHome
>

```java
private void maybeFinish() {
        if (getSystemService(UserManager.class).isUserUnlocked()) {
            final Intent homeIntent = new Intent(Intent.ACTION_MAIN)
                    .addCategory(Intent.CATEGORY_HOME);
            final ResolveInfo homeInfo = getPackageManager().resolveActivity(homeIntent, 0);
            if (Objects.equals(getPackageName(), homeInfo.activityInfo.packageName)) {
                Log.d(TAG, "User unlocked but no home; let's hope someone enables one soon?");
                mHandler.sendEmptyMessageDelayed(0, 500);
            } else {
                Log.d(TAG, "User unlocked and real home found; let's go!");
                getSystemService(PowerManager.class).userActivity(
                        SystemClock.uptimeMillis(), false);
                finish();
            }
        }
    }
```

监听Intent.ACTION_USER_UNLOCKED解锁广播，接到广播后调用maybeFinish。

总结来说，maybeFinish() 方法的目的是：

- 确保用户已经解锁了设备。
- 检查当前运行的应用是否是系统指定的默认主屏幕应用。
- 判断PMS中获取到的Home类型的Activity还是FallbackHome时，则会等待一段时间后重试检查。
- 如果当前应用不是自己(FallbackHome)，则会结束自身，以便让真正的 Home 应用接管界面。

## 3.Launcher的启动

![startlauncher](/images/startup/startlauncher.png)

设备解锁后，FallbackHome结束自身后，调用到AMS的activity pause等流程，最终在某个流程重新startHomeActivity，此时的home Intent是launcher3，最终会启动launcher3.



