---
layout: post
title:  "Launcher3架构"
date:   2025-06-13 00:00:01
catalog:  true
tags:
    - android
    - launcher3

---

基于android 13，部分代码涉及到14 15

## Core Components Architecture

![launcher_system](/images/launcher3/launcher_system.png)

![launcher核心UI](/images/launcher3/launcher核心UI.png)

![Launcher_UI_Component_Hierarchy](/images/launcher3/Launcher_UI_Component_Hierarchy.png)

![launcher系统架构图](/images/launcher3/launcher系统架构图.png)

更详细的launcher架构大图请点击：[大图](https://mermaid.live/view#pako:eNqVWf9P20gW_1eQpZXupLZq2tJCdFopBbqNRAtq2CLdcaqMY4h1xo6cCV22qkTbA0qALe3Sctuy24blS1ZXstfbbknJAv9MbIf_4uaL7fHYM07OP82X9z7z5s17b94bP5AUM69KaWlKN-8rBdkCPWODE0YP_L74oqfVaLb3Hjm1qv3jJhkclsuGUlCtv01IfrPnT7dkzejJKECb1cDcnyekvxPaAMatfHLmH7n1qru-aB-su7813eZbMlsqT05bcrHQkwMyUG_JhjytzqgGgPhhrgATfSFSJEe4y9D5AmKCkMC4H0ccs2SjpAHNNILF28fvnbVt57t9--kn-4caw8OXp-f8-fPswgl0kWV52HnNUhU02TN2nTfP6q8EJXeqq2c_bhP5GYHRd3vkzq3MMKQijb9MWl86Bz_bjUYiV2Z4-F5mdDQH-fwm4rQXamdPavbRhrtRE3DmRu9kb391b3gkMzg0iM4q3Merr7xyVo4F3CN3h-7czQ6NQ0a_iXhazaZdqTp7j9v7TzkrDmfH7uWGhocGxvCCtItlfrpof3h2Nr_srPzC8KpGnqdf5ijpoZUIRcAUmPrXWbf5z1bzk8jMv84OmDNF04Amjg7LJ2dE6frMx03rH6WirKgMpn242zreslePONDoC7ggbdDGem00A94Y14Cq68PynFlGrkk7WKdrL8-qv9uNJ_aH-fh5FEwLKGWQMfLjWn5aBQOmAWC8IK4rnMS4b06cd0v2i1VoZUil9c9RH0TfKHSn_F1NvQ_xgjZm33kNxWrvL0Ic--2R-6HJ5c0aeU2RgWl5_EEfYUAA6BbO6pK7c8RbXKBZaChfUsGSiKgmRerGZGJFdZIg2E5HUw_MatCSpxmL8pyUb06IGkqKzzNoU9e2Pzzmsngn5jcpAzkwLg_atGXqerAWHaD8_GhNIMzimGxB7V2XCUKojwE2TtzjuvumDs3OefcsBnC9DIBpUDaIER3CMKvLZy_qYbAOdhOoDZ-ar5Kk_QeEmK0LytBOE5SCiaNb6t5wMsUiG4nI1cC3m4yuQ_pSYMieQfCGo5eNIBJ4rHdUZU7Ro4DhUYxHkJ5utqtCJHo1M5aXMEuRnQo0_eX26ZLTqIpNEuLkVNlSCp6wzvqW-3Fb4AKd9YfPj7PhbjnD0nStkfCiDGb3hjMC4BXLms7Cp1bzFfd6RN9NE5RUGTmg1yKKf4lM5PTfztohz31vmHoeHyBpYFd9tYRulp3PAuKsgrNB2mGYyAXFCROTk7o6pn4DvFNlB0LWh_khnLP1vsNpe9tMvDOIlP_XfUE35rl-WNDuz--GbspAM3DYwhfG73W7UhOYcWayBCxZAWEm5FWcYaztEJboGjeL5WKww3ENFDKWZeKEgD-Dj6Cx59YP7J2P9h-f7aWj9rMtmMfEkyWsvdKNsq7nCioO-dEhnCfgDIWklZ3zBN5WyYXNF7dL7qhgnPOLJ6zOy_84a3VU4v20IkpaB2Ug34Jloo6ONkTPrbZ8QqaPDxKXGEJ-TDduaQC7aKiHeTGXvfiDvbAb1a-_0Khlzmp5pjT1h0IQB_-yH9ecZ-utkzft-QUuEAqDkZLRH6K-yylZssasbGmyAQbVWU1R4eJTmo5Q-BMIq10_sXeWzhbWYKpAwgqDGAWK8dvH39vLa2EUQY0anGUWqDNZY8osBWcJPSp6Fng3HiGS32viFaELVucDTnF1EeKPjWEgv9Q4q36GWRLvavTYvRbVfeu06jz6VRSyCRPtMCFbwBo6Z-JFHgp3nDq8AI2vSOKjUVUk0NItJBBxJUyg95TZdeGLHZFZyXeqJNrAyrj-3SWc73QMOYbpQMv3t06OSjLlOEc8ZtqVt-0nx-RZTBQzb8DbumzhF5kwOWMrTMWFEu7cXAkqzqu4YP1AgHmlUM4sW7iCp51QCXS0nlD8MJVPtOzhLjZSRBlfyVvN69Hl4LUHfbiLGofIGdQuHpBITjFdp3SEOEKgTOKrAmUGfnPTLAESbGiferpdP201OJkHFAVewZl8ngkbvGEcvk4WnMOmXXmXGD4CEWAOX4KVu5qHubyqzZLiQzhJhXUebds7a853u84Gt-ogAN5L5E1VL2JkziiF9B5vK784yyudspuwBklaw1FHJx6ONN1qyY9zFLF70_kKYkG3pY64vGJXTgW2M2aWlQJTGkZG8BW_v-c8bySUgKRyyt3XiiqvzIxMRCtMp_kcpsdi-ODCwaINa3Bf5O2NP8HczER0981ze71TfRLZeLgmjGygMx9fsu7P8LYJtCn06AWjBg0C86_dt7uCgwxzhFTEG8YuQbCwXrhPkiiNR0lzKCWNjWFN49LDS4m5OSkOiaYfV7wWlaG9_wIG7a7SEN5maNERlqzDZkhcNoEgjeBcl4e79sIhSjNFj-JA072Xa0LIbAXNakDDV2nQ9lUAi2guEyppB2SYGaD01W_TR2X3j-9hJcDP_g1txhcoNoYRKjV3o0mWjfxFsmCU81cN9TDXwab7fq_V-C9naf_XGfIDWHsj32dH8F37EV4ctVaj0mqswHSWld2chlHSl9rZ3LVPNzkC8p7sFEUtlbRJpNY5CvDu7PWWu73GwWBUFWYeVHV1OlI18ebxXvwFWs2fo3_3iEIIo67iN07PWjN5uUiqw8R5ZgE_Q3kMq6POhXmi4CSmJa3cMUQF5kvSVN8uRQQx8xMRhmxNRBKxKOGS1JJE7kx-CrdPf4IJhr3wG_Ro9tewJxHzO5dDEf4TxpkO3hw4c35uzUOlkgcCt3_dc5uLcYHZnyXsszmh6PTqyfx6NhKe6Wlyz1s6Ui3Fy6mAXEQgfn7lZFYsWOSvNO9g4mVSnCZ-6QpfFIXvzxE9hN4zBRoS-2lcQP49xntMI_S8zCR-3uFSg9UuXR-SSeekGdWakbW8lJYeINYJCRSga0xIadjMq1NyWQcT0oTxEJLKZWDm5gxFSgOrrJ6TLLM8XZDSU7Jegr1yMQ-lGNRkGMFnfJJpC0GTdlE2_mqawZQFXRi9rpcNIKX7rl7DBFL6gfSNlE6l-i7096d6r_T296UuX-xNwdk5KX352qULfZf6L13pvdabgrNXH56TvsWQFy_0py6mUldT_Vcv9_X1X-m79vB_AkicWA)

![launcher_ui_architecture](/images/launcher3/launcher_ui_architecture.png)

**Launcher3 代码库围绕以下关键组成部分组织**：

1. **Launcher Activity**

   Launcher 类是主活动类，作为所有其他组件的容器。它处理生命周期事件、状态转换，并协调所有用户交互。

2. **Workspace**

   Workspace 类表示可滚动的主屏幕，用户可以在其中放置应用、文件夹和小部件。它由多个 CellLayout 实例组成，每个实例代表一个单独的主屏幕页面。

3. **DragLayer**

   DragLayer 是一个特殊的视图组，处理启动器中的所有拖放操作。它位于所有其他组件之上，协调项目在启动器不同部分之间的移动。

4. **Hotseat (Dock)**

   Hotseat 是屏幕底部（或在某些布局中的侧面）区域，包含常用应用程序。它在所有主屏幕页面中始终可见。

5. **All Apps View**

   ActivityAllAppsContainerView 以可滚动的网格显示所有已安装的应用程序，通常通过从主屏幕向上滑动访问。

6. **DropTargetBar**

   DropTargetBar 在拖动操作期间出现，并为删除或卸载应用程序等操作提供目标。

### Launcher Item Types

The launcher manages several types of items that can be placed on the home screen:

| Item Type    | Class                   | Description                                          |
| ------------ | ----------------------- | ---------------------------------------------------- |
| App Shortcut | `WorkspaceItemInfo`     | Icon that launches an application                    |
| Folder       | `FolderInfo`            | Container that holds multiple app shortcuts          |
| Widget       | `LauncherAppWidgetInfo` | Interactive application widget                       |
| App Pair     | `AppPairInfo`           | Shortcut that launches two apps in split-screen mode |

### 主要组件和职责

1. **基本结构**：

   - 作为桌面启动器的主 Activity
   - 管理工作区、应用抽屉、Hotseat（底部快捷栏）等主要 UI 组件
   - 处理应用启动、桌面部件、文件夹和各种交互

2. **状态管理**：

   - 通过 `StateManager` 管理桌面的不同状态（正常、全部应用、分屏等）
   - 处理状态转换和动画效果

3. **UI 组件**：

   - `Workspace`：主桌面工作区，管理多个屏幕和部件

   - `Hotseat`：底部固定快捷方式区域

   - `DragLayer`：处理拖拽操作的层

   - `AllAppsContainerView`

     ：全部应用视图

4. **数据模型**：

   - `LauncherModel`：管理桌面数据，包括应用图标、部件等
   - `ModelWriter`：负责数据写入
   - `PopupDataProvider`：提供弹出菜单数据

5. **事件处理**：

   - 处理触摸、拖拽、点击等用户交互
   - 管理应用启动和返回
   - 处理系统事件（如 Home 键、最近任务键）

### 主要功能模块

1. **桌面管理**：
   - 管理多页桌面 (`Workspace`)
   - 处理页面滑动和过渡效果
   - 管理图标和部件的布局 (`CellLayout`)
2. **应用程序管理**：
   - 显示全部应用 (`AllAppsContainerView`)
   - 搜索和过滤应用
   - 应用启动和交互
3. **部件管理**：
   - 添加和配置桌面部件 (`AppWidgetHostView`)
   - 调整部件大小和位置
   - 部件数据更新
4. **文件夹功能**：
   - 创建和管理文件夹
   - 文件夹图标和预览
   - 文件夹内容排序和管理
5. **拖放操作**：
   - 拖拽图标和部件 (`DragController`)
   - 处理放置目标 (`DropTarget`)
   - 拖拽动画和反馈

### 目录结构及关键子模块

launcher3 目录下包含多个子目录，每个都负责特定功能：

1. **allapps**：全部应用相关的类
2. **anim**：动画相关类
3. **dragndrop**：拖放功能实现
4. **folder**：文件夹相关类
5. **model**：数据模型
6. **widget**：部件相关类
7. **touch**：触摸和手势处理
8. **accessibility**：无障碍功能
9. **util**：工具类
10. **views**：自定义视图

## 关键流程

1. **启动流程**：
   - `onCreate()` 初始化基本组件
   - 设置工作区和应用抽屉
   - 注册广播接收器和监听器
2. **加载桌面项目**：
   - 通过`LauncherModel`异步加载数据
   - `bindItems()` 将数据绑定到视图
   - 处理图标、部件和文件夹的显示
3. **状态转换**：
   - 正常桌面、全部应用、部件选择等状态之间的切换
   - 状态转换时的动画和布局调整
4. **应用启动**：
   - 处理点击事件
   - 启动应用程序活动
   - 显示启动反馈
5. **分屏功能**：
   - 分屏模式的进入和退出
   - 应用选择和排布

Launcher.java 作为核心控制器，协调这些组件和流程，管理用户与桌面的交互，并维护桌面状态。这个文件包含了大量的方法来处理各种场景和事件，是理解整个 Launcher3 框架的关键。



Launcher UI核心组件如下:

1. **Launcher**: 作为整个启动器体验的入口点和控制器
2. **DragLayer**: 一个包含所有主要用户界面组件并处理拖动操作的 ViewGroup
3. **Workspace**: 显示用户放置应用程序图标、文件夹和小部件的主屏幕页面
4. **Hotseat**: 屏幕底部的停靠区，包含常用应用程序
5. **AllAppsContainerView**: 显示所有已安装应用程序的应用抽屉
6. **CellLayout**: 工作区Workspace内的独立主屏幕页面
7. **ShortcutAndWidgetContainer**: 在 CellLayout 中容纳应用图标、文件夹和小部件

### 状态管理(State Management)

Launcher 使用状态管理系统来处理不同用户界面状态之间的转换:

| State         | Description                         |
| ------------- | ----------------------------------- |
| NORMAL        | 常规主屏幕显示(即首页Workspace界面) |
| ALL_APPS      | 应用抽屉                            |
| SPRING_LOADED | 正在进行拖动操作                    |
| OVERVIEW      | 进入最近任务列表的状态              |

StateManager状态管理器协调这些状态之间的转换，管理动画并确保用户界面一致性。.

## Icon System Architecture

![icon](/images/launcher3/icon.png)

图标系统负责加载、缓存和显示应用程序图标:

1. **IconCache**: 管理加载和缓存应用图标以提高性能
2. **IconProvider**: 请求来自多个来源的图标接口
3. **IconFactory**: 创建具有适当尺寸和处理的图标可绘制对象
4. **IconCacheUpdateHandler**: 处理应用安装/更新以刷新图标缓存

特殊图标类型包括:

- **AdaptiveIcon**: 标准的Android自适应图标，包含前景层和背景层
- **ThemedIconDrawable**: 适应系统主题的单色图标
- **ClockDrawableWrapper**: 动态时钟图标显示当前时间
- **BadgedIconFactory**: Creates icons with badges (work profile, notifications, etc.)

## Widget System Architecture

![widget](/images/launcher3/widget.png)

Widge小部件系统管理主屏幕上小部件的选择、预览和放置:

1. **WidgetsContainerView**: 显示小部件选择器界面
2. **WidgetCell**: 在选择器中表示单个小部件
3. **WidgetPreviewLoader**: 生成小部件的预览图像
4. **LauncherAppWidgetHost**: 管理与 Android  widget framework小部件框架的通信
5. **LauncherAppWidgetHostView**: 在主屏幕上托管和渲染小部件

Widgets小部件遵循特定的生命周期:

1. 用户打开小部件选择器
2. 用户选择并拖动一个小部件
3. 用户将小部件放在主屏幕上
4. Launcher创建一个 LauncherAppWidgetHostView
5. 小部件绑定到 LauncherAppWidgetHost
6. 小组件在主屏幕上显示并正常运行

## Launcher State Management

![launcher状态管理](/images/launcher3/launcher状态管理.png)

The launcher uses a state management system to handle different UI states:

1. **NORMAL**: The default state showing the workspace and hotseat
2. **ALL_APPS**: Shows the all apps grid
3. **SPRING_LOADED**: Activated during drag operations, makes the workspace more receptive to drops
4. **EDIT_MODE**: Allows rearranging and customizing the workspace

The `StateManager` class handles transitions between these states, applying the appropriate animations and visual changes.

## Drag and Drop System

One of the core features of the launcher is its drag and drop system, which allows users to rearrange items on the home screen.

![launcher拖拽](/images/launcher3/launcher拖拽.png)

The drag and drop system has these key components:

1. **DragController**: Orchestrates the entire drag operation, from beginning to end
2. **DragLayer**: Provides the visual layer where drag operations take place
3. **DragView**: Visual representation of the item being dragged
4. **DropTarget**: Interface implemented by views that can accept dropped items

当用户长按某个项目时，系统创建一个表示该项目的DragView并将其附加到DragLayer上。当用户移动手指时，DragView会跟随。当用户松开手指时，系统确定适当的放置目标并执行相应的操作。

