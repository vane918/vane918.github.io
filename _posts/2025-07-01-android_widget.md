---
layout: post
title:  "Android Widget开发"
date:   2025-07-01 00:00:00
catalog:  true
tags:
    - android
    - widget

---

## Android Widget概述

Android Widgets是主屏幕自定义中不可或缺的一部分。你可以把它们看作“一览”视图，可以直接在用户的主屏幕上展示应用的最重要的数据和功能。用户可以随意移动小部件，并且如果支持，还可以调整其大小，以便根据个人偏好调整小部件中信息的量。

## 小部件类型

小部件通常分为以下几类：

1. **信息小部件**

   信息小部件通常显示关键信息并跟踪这些信息随时间的变化。例如，天气小部件、时钟小部件或体育比分跟踪小部件。点击这些小部件通常会启动相关应用，并展示小部件信息的详细视图。

2. **集合小部件**

   集合小部件专门展示同类型的多个元素，如相册应用中的图片集合，新闻应用中的文章集合，或通信应用中的邮件或消息集合。集合小部件通常可以纵向滚动。

   集合小部件通常集中在以下用途：

   - 浏览集合
   - 打开集合中的元素到应用的详细视图
   - 与元素互动，例如标记完成状态（在Android 12引入了复合按钮支持）。

3. **控制小部件**

   控制小部件的主要目的是展示常用功能，让用户可以从主屏幕直接触发而不需要打开应用。你可以把它们看作应用的遥控器。例如，家庭控制小部件可以让用户开关房间的灯光。

   与控制小部件互动可能会打开应用的相关详细视图，这取决于小部件的功能是否产生任何数据输出，例如搜索小部件。

4. **混合小部件**

   虽然一些小部件单独代表上述类型之一（信息、集合或控制），但很多小部件是混合类型，结合了不同类型的元素。例如，音乐播放器小部件主要是控制小部件，但它也显示当前播放的曲目，就像信息小部件。


### 小部件的限制

虽然小部件可以看作“迷你应用”，但在Android的小部件存在很多限制，如：

- **手势支持：** 由于小部件位于主屏幕，它们需要与已经存在的导航手势共存。因此，小部件的手势支持有限，只有触摸和垂直滑动手势是可用的。
- **UI元素：** 小部件支持的UI控件很少，如下：
  - **文本控件：** TextView, Button, EditText
  - **图像控件：** ImageView, ImageButton
  - **动画控件：** ViewFlipper
  - **适配器视图：** ListView, GridView
  - **进度条：** ProgressBar
  - **复合按钮：** CheckBox, RadioButton, ToggleButton
  - **布局控件：** FrameLayout, LinearLayout, RelativeLayout。

## Android12新增API

Android12新增了一些小部件的API，使得小部件更强大，新增API如下：

### RemoteViews类

| 方法                                          | 作用                                                 |
| --------------------------------------------- | ---------------------------------------------------- |
| RemoteViews(Map<SizeF, RemoteViews>)          | 根据响应式布局映射表创建目标RemoteViews              |
| addStableView()                               | 向RemoteViews动态添加子View，类似ViewGroup#addView() |
| setCompoundButtonChecked()                    | 针对CheckBox或Switch控件更新选中状态                 |
| setRadioGroupChecked()                        | 针对RadioButton控件更新选中状态                      |
| setRemoteAdapter(int , RemoteCollectionItems) | 直接将数据填充进小组件的ListView                     |
| setColorStateList()                           | 动态更新小组件视图的颜色                             |
| setViewLayoutMargin()                         | 动态更新小组件视图的边距                             |
| setViewLayoutWidth()、setViewLayoutHeight()   | 动态更新小组件视图的宽高                             |
| setOnCheckedChangeResponse()                  | 监听CheckBox等三种状态小组件的状态变化               |



### XML属性

| 属性                              | 作用                                               |
| --------------------------------- | -------------------------------------------------- |
| description                       | 配置小组件在选择器里的补充描述                     |
| previewLayout                     | 配置小组件的预览布局                               |
| reconfigurable                    | 指定小组件的尺寸支持直接调节                       |
| configuration_optional            | 指定小组件的内容可以采用默认设计，无需启动配置画面 |
| targetCellWidth、targetCellHeight | 限定小组件所占的Launcher单元格                     |
| maxResizeWidth、maxResizeHeight   | 配置小组件所能支持的最大高宽尺寸                   |

## Widget小部件架构

Android Widget的架构，设计的核心目标是解决一个关键问题：**如何让一个应用程序（App）的UI片段，安全、高效地运行在另一个应用程序（通常是桌面Launcher）的进程中。**

### Android Widget 架构概览

Android Widget的架构是一个典型的基于**进程间通信（IPC）**和**代理模式**的设计。它横跨了至少三个进程：

1. **Widget提供方进程（App Process）**: 你的应用程序进程。它负责提供Widget的内容、布局和响应逻辑。
2. **Widget宿主进程（Host Process）**: 通常是桌面（Launcher）应用的进程。它负责请求、展示Widget，并将用户的交互事件传递出去。
3. **系统服务进程（System Server Process）**: system_server进程。它是整个架构的中枢和协调者，**AppWidgetService**就运行在这里。

下面，我们来拆解这套架构中的核心组件和它们之间的交互流程。

### 核心组件 (The Key Players)

#### 1. AppWidgetProvider (in App Process)

- **本质**: 一个特殊的BroadcastReceiver。这是你作为Widget开发者最主要的入口。
- **职责**:
  - **生命周期管理**: 接收来自AppWidgetManager的系统广播（Intent），这些广播代表了Widget的生命周期事件，例如：ACTION_APPWIDGET_UPDATE, ACTION_APPWIDGET_ENABLED, ACTION_APPWIDGET_DELETED等。
  - **UI构建**: 在接收到更新事件后，创建或更新一个RemoteViews对象。
  - **逻辑触发**: 它是所有Widget更新逻辑的起点。

#### 2. RemoteViews (A Parcelable UI Description)

- **本质**: 这是Widget架构的**魔法核心**。它**不是一个真正的View**，而是一个可序列化（Parcelable）的对象。它描述了View层次结构的布局信息以及要对这些View执行的操作（比如，设置文本、设置图片、绑定点击事件等）。
- **为什么需要它?**: 你不能直接将一个View对象从你的App进程传递到Launcher进程，因为它们处于不同的内存空间。RemoteViews就是这个问题的解决方案。它将UI的“蓝图”打包，通过Binder IPC从App进程发送到Host进程，再由Host进程在自己的上下文中“inflate”成真实的View对象。
- **局限性**: 出于安全和性能考虑，RemoteViews支持的View类型和可执行的操作是有限的。例如，只支持TextView, ImageView, Button等基础组件，以及ListView, GridView, StackView, AdapterViewFlipper等集合视图。不支持自定义View。

#### 3. AppWidgetManager (The Central Coordinator)

AppWidgetManager是一个典型的Facade模式应用，它对开发者屏蔽了底层的IPC复杂性。它有两个“面孔”：

- **客户端代理 (in App/Host Process)**: 我们在代码中通过AppWidgetManager.getInstance(context)获取到的对象。它是一个代理，负责将我们的请求（如 updateAppWidget()）通过Binder IPC发送给系统服务。
- **系统服务端 (in System Server Process)**: 由 **AppWidgetService** (入口) + **AppWidgetServiceImpl** (核心逻辑) 共同组成。**当我们谈论Widget服务的状态和逻辑时，我们现在指的是 AppWidgetServiceImpl。** 客户端AppWidgetManager通过Binder连接到的，是AppWidgetServiceImpl暴露出的Binder对象。
  - **状态维护**: 维护系统中所有Widget实例的状态信息（哪个Host放置了哪个Provider的哪个widgetId）。
  - **事件分发**: 负责向各个AppWidgetProvider发送生命周期广播。例如，当到了updatePeriodMillis设定的更新时间，AWMs会构建一个ACTION_APPWIDGET_UPDATE的Intent并广播出去。
  - **连接桥梁**: 接收来自App进程的RemoteViews，并将其转发给对应的Host进程。

#### 4. AppWidgetHost (in Host Process)

- **本质**: Launcher中用来管理和承载Widget的组件。
- **职责**:
  - **分配ID**: 当用户从Widget选择器中拖出一个Widget时，AppWidgetHost会调用allocateAppWidgetId()向AppWidgetManager申请一个唯一的appWidgetId。
  - **视图创建**: 持有AppWidgetHostView，这是一个特殊的FrameLayout，专门用于承载从AppWidgetManager接收到的RemoteViews并将其apply()（inflate）成真实的View树。
  - **交互监听**: 监听并处理用户的点击等交互事件（这些事件通常绑定在PendingIntent上）。

#### 5. AppWidgetProviderInfo (The Metadata)

- **本质**: 一个描述Widget元数据（Metadata）的对象。
- **来源**: 系统在解析应用的AndroidManifest.xml和相关的appwidget-provider XML文件后，将这些静态配置信息封装成AppWidgetProviderInfo对象，并由AppWidgetManagerService进行管理。
- **内容**: 包含了Widget的各种静态属性，如：最小/最大尺寸、初始布局、更新周期(updatePeriodMillis)、配置Activity(configure)等。

### 核心交互流程 

让我们把这些组件串起来，看看几个关键场景下的交互流程。

#### 流程1：用户添加一个Widget到桌面

1. **用户操作**: 用户在Launcher中长按，选择并拖动你的Widget到屏幕上。
2. **Host (Launcher)**: AppWidgetHost调用allocateAppWidgetId()。这个请求通过Binder IPC到达AppWidgetManagerService。
3. **System Server (IPC路由)**: 请求通过Binder IPC到达system_server。ServiceManager将请求路由给appwidget服务，也就是AppWidgetService发布的那个Binder对象，该对象实际上是AppWidgetServiceImpl的实例。
4. **System Server (逻辑处理)**: **AppWidgetServiceImpl** 接收到调用，创建一个全局唯一的appWidgetId，记录下所有关联信息，然后将appWidgetId返回给AppWidgetHost。
5. **Host (Launcher)**: Launcher在桌面上创建一个AppWidgetHostView来“占位”。
6. **配置与首次更新**: **AppWidgetServiceImpl** 根据需要，触发配置Activity的启动，或者直接构建ACTION_APPWIDGET_UPDATE广播并发送。
7. **App (Provider)**:
   - 你的AppWidgetProvider的onUpdate()方法被调用。
   - 在onUpdate()中，你创建RemoteViews对象，描述Widget的初始UI。
   - 你调用AppWidgetManager.getInstance(context).updateAppWidget(appWidgetId, remoteViews)。
8. **跨进程UI传递**:
   - AppWidgetManager代理将appWidgetId和RemoteViews通过Binder IPC发送给**AppWidgetServiceImpl**。
   - **AppWidgetServiceImpl 的**找到对应的Host，通过其IAppWidgetHost回调接口，将RemoteViews推送给Host进程。
9. **Host (Launcher)**:
   - AppWidgetHost收到RemoteViews后，调用其AppWidgetHostView的updateAppWidget(remoteViews)方法。
   - AppWidgetHostView内部调用remoteViews.apply()，在Launcher的进程中，根据RemoteViews的描述，创建出真实的View对象并显示出来。

#### 流程2：定时更新 (updatePeriodMillis)

1. **System Server**: AppWidgetManagerService内部有一个基于AlarmManager的定时器。当某个Widget的更新时间到达时，它会触发。
2. **System Server**: AWMs构建一个ACTION_APPWIDGET_UPDATE广播，其中包含所有需要更新的该Provider的appWidgetId数组。
3. **App (Provider)**: AppWidgetProvider的onUpdate()被调用，后续流程与首次更新的第7步开始完全相同。

#### 流程3：用户点击Widget上的按钮

1. **App (Provider)**: 在构建RemoteViews时，你不能设置OnClickListener。取而代之的是，你为一个按钮设置一个PendingIntent。例如: remoteViews.setOnClickPendingIntent(R.id.my_button, pendingIntent)。
2. **Host (Launcher)**: 当AppWidgetHostView加载这个RemoteViews时，它会把这个PendingIntent真正地附加到那个按钮的点击事件上。
3. **用户操作**: 用户点击了按钮。
4. **Host (Launcher)**: Launcher进程中的View系统响应点击事件，并触发执行与之关联的PendingIntent。
5. **系统级操作**: PendingIntent是一种特殊的令牌，它允许另一个应用（Launcher）以你原始应用（App）的身份和权限去执行一个预定义的Intent。系统服务（ActivityManagerService）会解析这个PendingIntent。
6. **App (Provider)**: 根据你创建PendingIntent时指定的Intent（可能是启动一个Activity，发送一个广播，或启动一个Service），你的应用的相应组件会被激活，从而执行刷新数据、再次调用updateAppWidget等操作。

### 流程图

![widget](/images/ui/widget.png)

- **参与者 (Participants)**:
  - **App Process**: 应用进程，包含 AppWidgetProvider、业务逻辑、AppWidgetManager 代理和被 PendingIntent 启动的目标组件。
  - **System Server Process**: 系统核心进程，包含真正的 AppWidgetServiceImpl 逻辑和用于处理 PendingIntent 的 ActivityManagerService。
  - **Host Process**: 桌面（Launcher）进程，包含 AppWidgetHost 和用于展示的 AppWidgetHostView。
- **流程一：Widget 更新 (蓝色背景)**:
  1. AppWidgetServiceImpl 发起更新。
  2. AppWidgetProvider 响应并创建 RemoteViews。
  3. RemoteViews 通过 AppWidgetManager 代理和 AppWidgetServiceImpl 中转。
  4. 最终 RemoteViews 到达 AppWidgetHost。
  5. AppWidgetHostView 将其渲染成界面。
- **流程二：用户交互 (橙色背景)**:
  1. 用户点击 AppWidgetHostView 上的某个元素。
  2. Host 进程触发绑定在该元素上的 PendingIntent。
  3. 系统服务 (ActivityManagerService) 接收并解析 PendingIntent。
  4. 系统以你的应用权限，启动在 PendingIntent 中定义的目标组件（如一个 Service 或 Activity），从而执行后续逻辑。

### 总结与重点

- **解耦**: Widget架构通过AppWidgetManagerService作为中心协调者，将Widget提供方和宿主方彻底解耦。它们之间从不直接通信，只与系统服务交互。
- **IPC核心**: Binder是底层通信的基石，Intent（特别是广播）用于事件通知，RemoteViews是UI描述的载体，PendingIntent是安全回调的机制。
- **省电与高效**: AppWidgetProvider作为BroadcastReceiver，只有在需要更新时才会被唤醒，执行完任务后进程即可被系统回收，非常省电。
- **安全**: RemoteViews的受限设计和PendingIntent的授权机制，保证了Host进程无法为所欲为，只能展示你提供的内容和触发你授权的操作。

理解了这个三方协作、基于IPC和代理的架构模型后，在开发Widget时遇到的各种问题，比如更新不及时、点击无响应、跨进程数据同步等，都能从架构层面找到根源。

## 音乐小部件停止更新案例分析

下面分享一个音乐小部件开发实际开发遇到的"坑"。

### 音乐Widget的UI元素

- 歌曲图片封面
- 歌名
- 歌手
- 播放进度条
- 上一首按钮，暂停/播放按钮，下一首按钮

### **问题：长时间播放音乐时，会出现音乐Widget不更新。**

### 问题分析

```
12774 12956 E JavaBinder: !!! FAILED BINDER TRANSACTION !!!  (parcel size = 264488)
12774 12956 E AppWidgetServiceImpl: Widget host dead: HostId{user:0, app:1000, hostId:1024, pkg:com.android.launcher3}
12774 12956 E AppWidgetServiceImpl: android.os.TransactionTooLargeException: data parcel size 264488 bytes
12774 12956 E AppWidgetServiceImpl: 	at android.os.BinderProxy.transactNative(Native Method)
12774 12956 E AppWidgetServiceImpl: 	at android.os.BinderProxy.transact(BinderProxy.java:584)
12774 12956 E AppWidgetServiceImpl: 	at com.android.internal.appwidget.IAppWidgetHost$Stub$Proxy.updateAppWidget(IAppWidgetHost.java:182)
12774 12956 E AppWidgetServiceImpl: 	at com.android.server.appwidget.AppWidgetServiceImpl.handleNotifyUpdateAppWidget(AppWidgetServiceImpl.java:1996)
12774 12956 E AppWidgetServiceImpl: 	at com.android.server.appwidget.AppWidgetServiceImpl.-$$Nest$mhandleNotifyUpdateAppWidget(Unknown Source:0)
12774 12956 E AppWidgetServiceImpl: 	at com.android.server.appwidget.AppWidgetServiceImpl$CallbackHandler.handleMessage(AppWidgetServiceImpl.java:3759)
12774 12956 E AppWidgetServiceImpl: 	at android.os.Handler.dispatchMessage(Handler.java:106)
12774 12956 E AppWidgetServiceImpl: 	at android.os.Looper.loopOnce(Looper.java:201)
12774 12956 E AppWidgetServiceImpl: 	at android.os.Looper.loop(Looper.java:288)
12774 12956 E AppWidgetServiceImpl: 	at android.os.HandlerThread.run(HandlerThread.java:67)
12774 12956 E AppWidgetServiceImpl: 	at com.android.server.ServiceThread.run(ServiceThread.java:44)
12956 12956 I binder  : 12774:12956 transaction failed 29201/-28, size 264488-304 line 3101
```

log显示是调用updateAppWidget时，发生了android.os.TransactionTooLargeException异常，导致了widget UI不更新。

1. **RemoteViews的本质**: 正如我们之前讨论的，它不是一个View，而是一个Parcelable对象。其内部最核心的成员是一个ArrayList<Action>（在源码中名为mActions）。Action是所有UI操作（如setText, setImageViewBitmap, setOnClickPendingIntent等）的抽象基元。

2. **set...方法的行为**

   : 当调用 remoteViews.setTextViewText(R.id.title, "...")时，RemoteViews的内部实现并不是去修改某个属性，而是执行了类似下面的操作：

   ```java
   addAction(new ReflectionAction(viewId, methodName, BaseReflectionAction.CHAR_SEQUENCE,value));
   
   private void addAction(Action a) {
           
           if (mActions == null) {
               mActions = new ArrayList<>();
           }
           mActions.add(a);
       }
   ```

   可以看到每次都会new一个ReflectionAction对象，然后放在mActions列表中，目的是在非widget界面时，如果remoteViews执行了set动作，可以将这些ui更新动作放到列表缓存，然后再widget可视化后执行ui更新。

   **内存泄漏的形成**:

   - 代码中将mRemoteViews作为一个成员变量，并在每次音乐状态更新时复用它。
   - 每次更新（例如，每秒更新一次播放进度），都会调用mRemoteViews.setTextViewText(...)。
   - 这就导致mActions列表像滚雪球一样，越来越大。播放10分钟的歌曲，每秒更新一次，就累积了600个Action对象。如果每次更新还包括设置专辑封面，那问题就更严重了。
   - RemoteViews的公开API中**没有提供clear()或remove()方法**。这个设计强制我们将其视为一个不可变的构建器。

   **TransactionTooLargeException的爆发**:

   - 当调用appWidgetManager.updateAppWidget(id, mRemoteViews)时，整个mRemoteViews对象（包括它内部那个巨大的mActions列表）会被序列化成一个Parcel。
   - 这个Parcel通过Binder IPC从App进程发送到system_server进程（AppWidgetServiceImpl）。
   - AppWidgetServiceImpl再将这个Parcel通过另一次Binder IPC发送到Host进程（Launcher）。
   - 当序列化后的mRemoteViews大小超过binder传输的阈值时，Binder驱动会抛TransactionTooLargeException。

   ### 解决方案

   **核心原则：在每次需要更新Widget时，都创建一个全新的** **RemoteViews** **实例。**

   **最终解决方案：**
   1.更新歌曲每次都new RemoteViews；

   2.更新进度条复用RemoteViews，
   但是会做调用mAppWidgetManager.updateAppWidget前先检查当前的mRemoteViews大小，如果超过200kb阈值，则重新new一个RemoteViews。
   200*1024是binder源码中定义的parcelSize阈值，如果超过这个阈值，就会导致TransactionTooLargeException。
   测试发现：不是超过了这个阈值就马上会发生TransactionTooLargeException，但是一定会发生TransactionTooLargeException。

   ### 复用RemoteViews 和new RemoteViews 方案对比

| 特性               | 复用 RemoteViews (错误方案)                                | 每次 new RemoteViews (正确方案)                          |
| :----------------- | ---------------------------------------------------------- | -------------------------------------------------------- |
| **工作原理**       | 累积指令集，mActions列表无限增长                           | 每次都发送一个独立的、全新的指令集                       |
| **内存/性能**      | **严重内存泄漏**，最终导致TransactionTooLargeException     | **高效、安全**，每次传输的数据包都很小                   |
| **后台更新**       | 每次都发送一个更大的数据包                                 | 每次都发送一个小数据包，系统只保留最新的                 |
| **返回桌面时**     | 应用一个包含所有历史指令的“超级包”，**凑巧**能显示最终状态 | 应用系统保留的**最后一个**状态包，高效直接               |
| **刷新不及时问题** | 表面上不明显，因为“超级包”包含了最终状态                   | 如果只依赖定时更新，问题会暴露。**需要事件驱动来解决**。 |
| **根本性质**       | **Buggy behavior**, an unintended side effect              | **Designed behavior**, stable and predictable            |