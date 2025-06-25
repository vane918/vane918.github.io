---
layout: post
title:  "Android应用、权限与进程模型"
date:   2025-06-25 00:00:00
catalog:  true
tags:
    - android
    - 权限

---

## 摘要

本⽂旨在为 Android 开发者系统性地阐述 Android 平台的应⽤、权限与进程模型。我们将从 PID、UID 等基础概念入手，详细剖析不同类型的应用（用户应用、系统应用、特权应用等）及其在系统中的角色和位置，并深入探讨 Android 的权限分类与授予机制。最后，本文将通过实战工具 `adb dumpsys`，探讨开发者如何在真实设备上验证这些核心概念。

---

## 目录

1.  [基础概念：PID、UID 与应用沙箱](#1-基础概念piduid与应用沙箱)
2.  [Android分区](#2-Android分区)
3.  [Android 应用的角色与身份](#3-android-应用的角色与身份)
4.  [Android 权限体系详解](#4-android-权限体系详解)
5.  [应用与权限的交互矩阵](#5-应用与权限的交互矩阵)
6.  [实战验证：使用 `adb dumpsys` 查看应用信息](#6-实战验证使用-adb-dumpsys-查看应用信息)
7.  [总结与最佳实践](#7-总结与最佳实践)

---

## 1. 基础概念：PID、UID与应用沙箱

要理解 Android 的安全模型，首先必须掌握 Linux 内核的几个基本概念。

### 1.1. 进程 ID (PID - Process ID)

- **定义**: PID 是操作系统为每个正在运行的**进程**分配的唯一数字标识符。它是一个**临时的、动态的**标识。
- **生命周期**: 当一个应用启动时，系统会为其创建一个或多个进程，每个进程都有自己唯一的 PID。当进程结束时，其 PID 会被系统回收。
- **简单理解**: 如果把操作系统看作一个公司，PID 就像是发给每位当天上班员工的**临时工牌号**。

### 1.2. 用户 ID (UID - User ID)

- **定义**: UID 是操作系统用来标识**用户**并控制其对文件和资源访问权限的数字。它是一个**持久的、与身份绑定的**标识。
- **Android 中的应用**: Android 巧妙地利用了这一机制，将每个应用视为一个独立的用户。安装应用时，系统会为其分配一个唯一的 UID（通常以 `u0_a` 开头，如 `u0_a123`）。
- **简单理解**: 在公司的比喻中，UID 就像是每个员工的**永久员工编号**，决定了他能进入哪些部门（访问哪些文件）。

### 1.3. Android 应用沙箱 (Application Sandbox)

Android 的核心安全特性就是应用沙箱。**这个沙箱是基于 UID 实现的**。

- **隔离**: 每个应用都在其独立的进程中运行，并拥有自己唯一的 UID。内核会强制执行安全策略，确保一个应用（一个 UID）无法访问另一个应用（另一个 UID）的私有数据。
- **数据目录**: 应用的私有数据存储在 `/data/data/<package_name>` 目录下，该目录的所有者就是该应用的 UID。其他应用由于 UID 不同，默认无权读写此目录。

## 2.Android分区

`system_ext` 是一个为了更好地**解耦和模块化**系统组件而创建的新分区，它介于纯粹的 `/system` 分区和 `/vendor` 分区之间，主要用于存放**与 AOSP (Android Open Source Project) 紧密相关但又非 AOSP 核心的系统级应用和服务**。

下面我为您详细解析 `system`, `system_ext`, `product`, 和 `vendor` 这几个关键分区的区别和演进逻辑。

### 2.1. 演进的背景：Project Treble 与模块化

在理解 `system_ext` 之前，先回顾一下 Android 分区演进的驱动力：

*   **Project Treble (Android 8.0)**:
    *   **目标**: 将 Android OS 框架与硬件供应商的底层实现（驱动等）分离开，以加快 Android 版本的更新速度。
    *   **实现**: 引入了 `/vendor` 分区，专门存放由芯片制造商（如高通、联发科）提供的、与硬件相关的代码（HALs, drivers）。`/system` 分区则存放纯粹的 AOSP 代码。
    *   **规则**: `/system` 分区的代码可以独立于 `/vendor` 分区进行更新（通过 OTA）。两者之间通过一个稳定的接口 (VNDK) 进行通信。

*   **系统分区的进一步细分 (Android 9.0/10.0)**:
    *   随着 Treble 的成功，Google 发现即使是 `/system` 分区内部，也存在不同类型的组件，它们的来源、功能和更新周期都不同。
    *   例如，有些是纯 AOSP 代码，有些是设备制造商（如三星、小米）为了实现其特色功能而添加的系统应用，还有些是运营商定制的应用。将它们都混在 `/system` 里，依然不够灵活和清晰。
    *   因此，引入了 `product` 和 `system_ext` 分区，将原来的 `/system` 大分区进一步拆解。

### 2.2. 各分区的功能定位

现代 Android（如 Android U）中，这几个分区各自扮演的角色：

#### **`/system` (The Core AOSP)**

*   **所有者**: Google (AOSP)
*   **内容**:
    *   最核心的 Android 操作系统框架和服务 (`framework.jar`, `system_server` 等)。
    *   最基础的、不应被修改的 AOSP 应用（如 `PackageInstaller`, `PermissionController`）。
    *   **定位**: 这是 Android 的“心脏”，理论上所有设备都应该共享这部分代码。它应该保持尽可能的纯净和通用。

#### **`/vendor` (The Chipset/Hardware Layer)**

*   **所有者**: SoCs/芯片供应商 (如 Qualcomm, MediaTek)
*   **内容**:
    *   硬件抽象层 (HAL) 实现。
    *   硬件相关的驱动和库文件。
    *   芯片供应商提供的特定服务。
*   **定位**: 这是 Android 的“躯干和四肢”，负责驱动硬件。它可以独立于 `/system` 进行更新。

#### **`/product` (The OEM/Device-Specific Layer)**

*   **所有者**: OEM/设备制造商 (如 Samsung, Xiaomi, Google Pixel)
*   **内容**:
    *   设备制造商定制的系统应用和服务（如三星的 Bixby, 小米的 MIUI 核心服务）。
    *   设备特有的配置、主题、铃声、字体等。
    *   运营商定制的应用有时也会放在这里。
*   **定位**: 这是设备的“个性和皮肤”。它包含了让一部三星手机不同于一部 Pixel 手机的软件部分。这个分区可以独立于 `/system` 和 `/vendor` 进行更新。

#### **`/system_ext` (The Extended AOSP Layer)**

*   **所有者**: Google (AOSP)，但允许 OEM/SoC 供应商进行扩展。
*   **内容**: 这是最微妙的一个分区，它的定位是：
    *   存放那些**与 AOSP 紧密耦合，但又不属于最核心 AOSP 框架**的系统组件。
    *   这些组件通常是 AOSP 的扩展模块或服务，它们需要访问 `/system` 的私有 API，但又希望和核心框架保持一定的隔离。
    *   一些由 Google 提供、但作为可选项的系统服务可能会放在这里。
    *   OEM 也可以将他们的一些需要与 AOSP 紧密交互的系统服务放在这里，特别是那些原来不得不修改 `/system` 分区才能实现的功能。
*   **定位**: 这是 `/system` 分区的“**亲密伙伴”或“官方扩展包**”。它解决了这样一个难题：有些模块既不像 `/vendor` 那样纯粹是硬件实现，也不像 `/product` 那样是 OEM 的上层定制，而是对 AOSP 核心功能的底层扩展。

### 2.3. `/system_ext/app` 和 `/system_ext/priv-app`

这两个目录的功能与 `/system` 分区下的对应目录完全相同：

*   **`/system_ext/app`**: 存放普通的系统应用。
*   **`/system_ext/priv-app`**: 存放特权应用，这些应用可以被授予高风险的 `privileged` 权限。

从权限模型的角度看，**安装在 `/system_ext` 下的应用与安装在 `/system` 下的应用享有完全相同的地位**。系统会将这两个分区视为一个逻辑整体（有时称为 "System-as-root" 的一部分），共同构成系统级软件。

**一个应用被放在 `/system_ext/priv-app` 而不是 `/system/priv-app`，这更多是出于软件架构和模块化管理的考虑，而不是权限级别的差异。**

### 2.4. 总结与类比

| 分区              | 类比：构建一台电脑                                           | 所有者/负责方              | 核心特征                                             |
| :---------------- | :----------------------------------------------------------- | :------------------------- | :--------------------------------------------------- |
| **`/system`**     | **操作系统内核和核心库** (如 Windows Kernel,核心DLLs)        | Google (AOSP)              | 纯净、通用、核心的 OS 框架。                         |
| **`/vendor`**     | **主板驱动、显卡驱动、声卡驱动**                             | 芯片供应商 (Intel, Nvidia) | 与硬件紧密绑定，负责驱动硬件。                       |
| **`/product`**    | **预装的品牌软件** (如 Dell SupportAssist, HP Omen Gaming Hub) | 设备制造商 (Dell, HP)      | 设备特有的功能、UI 和应用。                          |
| **`/system_ext`** | **系统级的官方扩展工具** (如 .NET Framework, DirectX)        | Google/OEM                 | 紧密依赖 OS 核心，但又作为独立模块存在的系统级服务。 |

**结论：**
`/system_ext` 分区的出现是 Android 系统架构演进的必然结果，它通过更精细的分区划分，实现了更高程度的模块化和解耦。对于应用开发者和权限分析者来说，可以简单地将 `/system` 和 `/system_ext` 视为一个逻辑上的“大系统分区”。在这两个分区下的应用都属于系统应用。

## 3. Android 应用的角色与身份

一个应用在 Android 系统中的“身份”由其**安装位置**和**签名**共同决定。

| 应用类型         | 安装位置                            | 签名要求     | UID               | 核心特征                                   |
| :--------------- | :---------------------------------- | :----------- | :---------------- | :----------------------------------------- |
| **普通用户应用** | `/data/app`                         | 任意有效签名 | 独立的 `u0_aXXX`  | 用户可自由安装卸载，权限受限。             |
| **普通系统应用** | `/system/app`                       | 任意有效签名 | 独立的 `u0_aXXX`  | 用户不可卸载，可默认授予危险权限。         |
| **特权应用**     | `/system/priv-app`                  | 任意有效签名 | 独立的 `u0_aXXX`  | 具备申请高风险**特权权限**的资格。         |
| **平台签名应用** | `/system/app` 或 `/system/priv-app` | **平台签名** | 独立的 `u0_aXXX`  | 自动获得所有**签名权限**，受系统高度信任。 |
| **核心系统组件** | `/system/app` 或 `/system/priv-app` | **平台签名** | **共享的 `1000`** | 与系统核心共享沙箱，拥有最高权限。         |

一个常见的误区是认为所有系统应用都应是 UID=1000。**这是不正确的**。一个应用要获得 UID=1000，必须**同时满足**：

1. **Manifest 声明**: `<manifest android:sharedUserId="android.uid.system">`
2. **平台签名**

获得 UID=1000 意味着：

- **共享沙箱**: 该应用与 system_server 进程处于同一个安全沙箱，可以无限制地访问属于系统核心的数据目录（如 /data/system）。
- **共享进程（可选）**: 默认情况下，应用启动时会运行在一个**独立的进程**里（但 UID 是 1000）。通过在 Manifest 中为组件设置 android:process="system"，可以让该组件**真正地被加载并运行在 system_server 的主进程中**，实现最高效的系统集成。

## 4. Android 权限体系详解

Android 的权限根据其风险和授予方式，可分为以下几类：

| 权限类别                | 保护级别                    | 描述与示例                                                   |
| :---------------------- | :-------------------------- | :----------------------------------------------------------- |
| **1. 普通权限**         | `normal`                    | 风险低，安装时自动授予。<br/>示例: `android.permission.INTERNET` |
| **2. 危险权限**         | `dangerous`                 | 涉及用户隐私，需用户运行时动态授予。<br/>示例: `android.permission.CAMERA`, `android.permission.ACCESS_FINE_LOCATION` |
| **3. 签名权限**         | `signature`                 | 只有与定义该权限的 APK 具有相同签名的应用才能获得。<br/>示例: `android.permission.STATUS_BAR`, `android.permission.REBOOT` |
| **4. 特权权限**         | `signature` \| `privileged` | 风险极高，仅能授予给 `/system/priv-app` 下的特权应用。<br/>示例: `android.permission.WRITE_SECURE_SETTINGS`, `android.permission.INSTALL_PACKAGES` |
| **5. 特殊应用访问权限** | (AppOps Control)            | 需通过 Intent 引导用户到特殊设置页面手动开启。<br/>示例: `android.permission.SYSTEM_ALERT_WINDOW` (悬浮窗), `android.permission.WRITE_SETTINGS` |

## 5. 应用与权限的交互矩阵

下表是本文的核心，它清晰地展示了不同应用类型如何与各类权限进行交互。

| 应用类型         | 普通权限     | 危险权限                   | 特殊应用访问权限               | 签名权限                  | 特权权限                     |
| :--------------- | :----------- | :------------------------- | :----------------------------- | :------------------------ | :--------------------------- |
| **普通用户应用** | **自动授予** | 需运行时**动态申请**       | 需引导用户到**设置页手动开启** | **无法获取** (签名不匹配) | **无法获取**                 |
| **普通系统应用** | **自动授予** | **默认授予 (有重要例外)**¹ | **默认授予**¹                  | **无法获取** (签名不匹配) | **无法获取**                 |
| **特权应用**     | **自动授予** | **默认授予 (有重要例外)**¹ | **默认授予**¹                  | **无法获取** (签名不匹配) | **可以获取** (需白名单授权)² |
| **平台签名应用** | **自动授予** | **默认授予 (有重要例外)**¹ | **默认授予**¹                  | **自动授予** (签名匹配)   | **可以获取** (需白名单授权)² |
| **核心系统组件** | **自动授予** | **默认授予 (有重要例外)**¹ | **自动授予**                   | **自动授予**              | **自动授予** (天生拥有)      |

---

**脚注解释:**

¹ **默认授予 (有重要例外)**: 对于系统/特权应用，大部分危险权限会在安装时被预先授予 (Pre-granted)，免去用户交互。**然而，从 Android 10 开始，为了保护用户隐私，对于涉及位置、麦克风、相机等核心敏感权限，即使是系统应用，在首次访问时也必须弹出对话框请求用户明确授权。** 只有极少数被特殊白名单豁免的系统核心组件才能绕过此流程。

² **白名单授权 (Whitelist Grant)**: 特权应用要获得 `privileged` 权限，除了在 Manifest 中声明，还必须在 `/system/etc/permissions/` 目录下的 XML 文件中被明确“列入白名单”。

**从 Android 10 (Q) 开始，对于涉及用户隐私的核心权限，特别是“位置权限”，Google 改变了系统应用的权限授予策略。即使是预置的系统应用，在首次访问位置信息时，也必须遵循与普通应用类似的用户授权流程。**

下面详细拆解这个变化的原因和具体机制。

### 5.1. 为什么会有这个变化？——用户隐私优先

在 Android 早期版本中，系统应用确实可以“为所欲为”，在后台静默地获取位置等敏感信息。这导致了用户对其隐私安全的担忧。

从 Android 10 (Q) 开始，Google 极大地加强了对用户隐私的保护，将**用户控制权**放在了最高优先级。其核心设计理念是：**无论应用是什么身份（系统或普通），用户都应该明确知道哪个应用在何时、为何访问其敏感数据，并有权批准或拒绝。**

位置信息是隐私中的重中之重，因此成为了这项改革的重点。

### 5.2. 具体的机制变化是什么？

之前的“默认授予 (Pre-granted)”机制依然存在，但它的作用被**降级**了。

*   **旧机制 (Android 9 及之前)**: `PackageManagerService` 在安装系统应用时，会检查其 Manifest 中的危险权限，并直接将它们的状态设置为“已授予”。应用运行时，检查权限即为通过。

*   **新机制 (Android 10及之后，特别是针对位置权限)**:
    1.  **安装时（编译时）授权依然存在**: `PackageManagerService` 依然会将权限标记为“预授予”。**但这不再是最终决定**。它更像是一个“白名单资格”，表示“这个系统应用*有资格*访问位置，但最终决定权在用户”。
    2.  **运行时强制用户确认**: 当这个系统应用**第一次**尝试访问位置信息时，Android 框架会进行一次额外的检查。它会发现虽然权限被“预授予”，但用户从未**主动确认**过。此时，系统会**强制弹出一个权限授权对话框**，与普通应用看到的一样（或者是一个经过简化的版本）。
    3.  **用户选择成为最终依据**: 只有当用户在对话框中选择了“仅在使用中允许”或“始终允许”后，该权限才算真正被激活。用户的这个选择会被系统记录下来，后续访问就不再弹窗。

### 5.3. 如何在代码和配置中体现？

这种行为是由硬编码在 Android 框架中的策略决定的，但 OEM 厂商或系统开发者可以通过特定的白名单来豁免某些极少数、极其核心的应用。

这个豁免列表通常在 `privapp-permissions.xml` 或类似的系统配置文件中定义。除了声明权限本身，还需要添加特定的标志来绕过用户交互。

例如，一个普通的特权应用白名单可能是这样的：

```xml
<privapp-permissions package="com.example.systemapp">
    <permission name="android.permission.ACCESS_FINE_LOCATION"/>
</privapp-permissions>
```

这只会让它获得“预授权”资格，但仍需用户在运行时确认。

而一个被完全豁免的应用，可能需要类似这样的配置（**注意：具体标志可能因 Android 版本和 OEM 而异，此处为示例**）：

```xml
<privapp-permissions package="com.android.systemui">
    <permission name="android.permission.ACCESS_FINE_LOCATION" flags="canBypassUserConfirmation"/>
</privapp-permissions>
```

只有像 SystemUI（状态栏）、核心定位服务等真正离不开后台位置且用户明确感知的系统组件，才会被加入到这种“豁免”名单中。对于绝大多数系统应用（如系统天气、系统地图），Google 的兼容性要求 (CTS) 规定它们必须请求用户授权。

## 6. 实战验证：使用 `adb dumpsys ` 查看应用信息

理论需要实践来验证。`adb dumpsys package` 命令是查看应用身份和权限状态的强大工具。

### 6.1 命令格式

```bash
adb shell dumpsys package <package_name>
```

### 6.2 关键信息解读

以下是 `dumpsys` 输出中需要关注的关键字段：

- `userId`: **应用的 UID**。例如 `userId=10150`。如果是核心组件，则为 `userId=1000`。
- `sharedUser`: 如果应用使用了 `sharedUserId`，这里会显示共享用户的信息，如 `sharedUser=SharedUserSetting{... android.uid.system ...}`。
- `pkgFlags和privateFlags`: **应用身份标志**。
  - `[ SYSTEM ]`: 表示这是一个系统应用（位于 `/system/app` 或 `/system/priv-app`）。
  - `[ PRIVILEGED ]`: 表示这是一个特权应用（位于 `/system/priv-app`）。
- `signatures`: 显示 APK 的签名证书信息。可以通过比对哈希值来判断是否为平台签名。
- `install permissions`: 应用在安装时被授予的权限列表。
- `runtime permissions`: 应用的运行时权限（危险权限）的授予状态。

### 6.3 实例分析：查看系统设置应用

我们以“系统设置”(`com.android.settings`)为例，它通常是一个特权应用。

```bash
adb shell dumpsys package com.android.settings
```

**你可能会看到类似如下的输出（已简化）：**

```
Package [com.android.settings] (2104cc7):
  userId=1000  // 注意：在某些系统中，Settings为了方便会共享system UID，但更常见的是独立UID
  // 假设在另一个系统中，它有独立UID
  // userId=10123
  sharedUser=null // 如果有sharedUserId，这里会显示
  pkgFlags=[ SYSTEM HAS_CODE ALLOW_CLEAR_USER_DATA ] // 关键！SYSTEM
  privateFlags=[ PRIVILEGED SYSTEM_EXT] //PRIVILEGED
  ...
  signatures:
    PackageSignatures{... signatures=[[...]]} // 签名信息
  ...
  install permissions:
    android.permission.ACCESS_NETWORK_STATE: granted=true
    android.permission.MANAGE_USERS: granted=true // 一个特权权限
    android.permission.REBOOT: granted=true, flags=[GRANTED_BY_SIGNATURE] // 一个签名权限
    ...
  runtime permissions:
    ...
```

**分析:**

1.  `pkgFlags` 和`privateFlags`中分别包含 `[ SYSTEM ]` 和 `[ PRIVILEGED ]`，这确认了它作为**特权应用**的身份。
2.  `MANAGE_USERS` 是一个特权权限，因为它被白名单授权，所以 `granted=true`。
3.  `REBOOT` 是一个签名权限。如果此 APK 是平台签名的，它会被自动授予。

通过这个命令，你可以清晰地验证任何一个应用的身份、UID、签名级别和权限状态。

## 7. 总结与最佳实践

- **身份由位置决定**: `/system/app` 赋予应用“默认授予”危险权限的能力；`/system/priv-app` 赋予其申请“特权权限”的资格。
- **信任由签名决定**: **平台签名**是获取 `signature` 权限的钥匙，代表系统对其的最高信任。
- **沙箱由 UID 决定**: `sharedUserId` 是一种合并沙箱的机制，`android.uid.system` 是最高级别的合并，仅应在绝对必要时使用，因为它打破了应用间的核心隔离。

**给开发者的建议**:

- **普通系统工具 (如计算器)**: 放置于 `/system/app`。
- **需要高级功能的应用 (如定制桌面)**: 放置于 `/system/priv-app`，并通过白名单精确授予所需特权权限。
- **需要与系统框架深度交互的应用 (如网络管理工具)**: 使用**平台签名**，放置于 `/system/app` 或 `/system/priv-app`。
- **核心框架扩展**: 只有在构建需要直接修改系统核心状态且性能要求极高的服务时，才审慎考虑使用 `sharedUserId="android.uid.system"`。