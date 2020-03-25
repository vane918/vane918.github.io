---
layout: post
title:  "安卓PMS模块之安装APK(1)"
date:   2020-03-17 00:00:00
catalog:  true
tags:
    - PMS
    - Android
---



## 1. 概述

 一个 Android 应用安装到手机上大致分为四种情形：

- 系统应用，在设备首次启动时完成安装
- 通过 adb install 命令安装
- 系统自带的应用市场安装，如三星的应用商店
- 第三方应用安装或双击安装包，会启动系统应用引导安装

 无论采用哪种安装方式，最终的安装过程都会走到 PackageManagerService，由这个类来完成一系列的工作。 Android系统关于APK包的管理是由PackageManagerService系统服务(简称PMS)和相关的辅助类一起完成的，负责提供包括安装、优化、查询和卸载等功能。在本篇文章中，主要在分析第三方应用安装或双击安装包进行apk安装的流程。

## 2 PMS模块管理APK相关知识点

### 2.1 数据结构

在PMS模块中，核心的几个数据结构类关系图如下：

![pms-structure](images/pms/pms/pms-structure.png)

-  PackageParser.Package对应一个apk完整的原始数据 ， 对应的packages.xml文件里面的<package>标签。  其mExtras变量指向PackageSetting对象。
-  PackageSetting包含一个PackageParser.Package对象实例，它还包括apk相关配置数据，比如apk内部哪些component是被disable等。 
-  Settings包含了PackageSetting对象列表，也就是说它包含了系统所有apk数据
- SharedUserSetting用来描述具有相同的sharedUserId的应用信息，它的成员变量packages保存了所有具有相同sharedUserId的应用信息引用。这些应用的签名时相同的，所有只需要在成员变量signatures中保存一份。通过这个对象，Android运行时很容易检索到某个应用拥有相同的sharedUserId的其他应用。其中应用的签名保存在ShardUserSetting类的成员变量signatures中。

### 2.2 APK相关目录和文件 

APK涉及到的目录如下：

- /system/priv-app：系统应用安装路径，Android 4.4+ 开始出现，区分系统应用权限，拥有 SignatureOrSystem 权限，此目录下的 service 具有保活能力
- /system/app：系统应用安装路径，权限略低于 priv-app 目录下的应用，放置比如厂商内置应用
- /data/app：用户应用安装路径，应用安装时将 apk 复制到此目录下
- /data/data：用户应用数据存放路径，存在沙箱隔离
- /data/dalvik-cache：存放应用的dex 文件
- /data/system：存放应用安装相关文件

APK涉及到的文件如下：

- packages.xml 是一个应用的注册表，在解析应用时创建，有变化时更新，记录系统权限，各应用信息，如name, codePath, flag, version, userid，下次开机时直接读取并添加到内存列表
- pakcages-back.xml：packages.xml的备用文件
- package.list 指定应用的默认存储位置，userid 等

## 3 APK安装流程

### 3.1 APK安装#installStage

> \frameworks\base\services\core\java\com\android\server\pm\PackageManagerService.java

先上一个installStage的流程图：

![installStage](images/pms/pms/installStage.png)

```java
void installStage(String packageName, File stagedDir, String stagedCid,
        IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
    String installerPackageName, int installerUid, UserHandle user,
        Certificate[][] certificates) {
    // VerificationInfo中主要是用来存储权限验证需要的信息
    final VerificationInfo verificationInfo = new VerificationInfo(
            sessionParams.originatingUri, sessionParams.referrerUri,
            sessionParams.originatingUid, installerUid);
    final Message msg = mHandler.obtainMessage(INIT_COPY);
    final int installReason = fixUpInstallReason(installerPackageName, installerUid,
            sessionParams.installReason);
    // 创建InstallParams对象，准备安装包所需要的安装数据
    final InstallParams params = new InstallParams(origin, null, observer,
            sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
            verificationInfo, user, sessionParams.abiOverride,
            sessionParams.grantedRuntimePermissions, certificates, installReason);
    msg.obj = params;
    // 向mHandler发送数据
    mHandler.sendMessage(msg);
}
```

 在 installStage里会基于传入的参数构造一个 InstallParams 对象，这个对象中包含安装包的所有数据，然后将这个对象作为消息内容，通过 mHandler 发送一个类型为 INIT_COPY 的消息。 

### 3.2 INIT_COPY消息

```java
void doHandleMessage(Message msg) {
    switch (msg.what) {
        case INIT_COPY: {
            // 第一步 取出InstallParams
            HandlerParams params = (HandlerParams) msg.obj;
            // 获取当前等待安装的数量
            int idx = mPendingInstalls.size();
            // 第二步 分两种情况
            // mBound用于标识是否已经绑定到服务
            if (!mBound) {
                // 第三步 
                // 如果还没有绑定，那调用connectToService绑定到实际的服务
                if (!connectToService()) {
                     // 绑定失败直接返回
                    return;
                } else {
                    // 如果绑定成功，将请求添加到mPendingInstalls中，mPendingInstalls是一个ArrayList对象
                    mPendingInstalls.add(idx, params);
                }
            } else {
                // 如果之前已经绑定过服务，同样将新的请求到mPendingIntalls中，等待处理
                mPendingInstalls.add(idx, params);
                if (idx == 0) {
                    // 如果是第一个安装请求，则直接发送事件MCS_BOUND触发处理流程
                    mHandler.sendEmptyMessage(MCS_BOUND);
                }
            }
            break;
```

该方法分为三部分：

1. 取出参数params，这个params就是之前传入的InstallParams
2. 获取等待安装队列的个数，并根据mBound的值进行不同的处理。mBound为true，表示已经绑定，mBound为false表示未绑定。第一次调用则mBound为默认值为false。
3. 如果是第一次调用，则调用connectToService()方法，如果不是第一次调用，且已经绑定则将params添加到mPendingInstalls(等待安装队列)的最后一个位置。如果是第一个安装请求，则发送MCS_BOUND事件触发接下来的流程

connectToService方法与 DefaultContainerService绑定服务， 这个服务用于检查和复制文件，位于 com.android.defcontainer 进程，通过 IMediaContainerService 与PMS 通信。

### 3.3 MCS_BOUND消息

```java
case MCS_BOUND: {
    // 获取msg中的IMediaContainerService服务对象
    if (msg.obj != null) {
        mContainerService = (IMediaContainerService) msg.obj;
    }
    // mContainerService为null，则返回
    if (mContainerService == null) {
        ...
     } else {
     }
    // 如果mPendingInstalls中存在未处理的安装请求
     } else if (mPendingInstalls.size() > 0) {
         HandlerParams params = mPendingInstalls.get(0);
        // 获取第一个安装请求
         if (params != null) {
             // 调用startCopy开始处理APK的安装
             if (params.startCopy()) {
                     // 处理完成，移除安装请求
                     if (mPendingInstalls.size() > 0) {
                         mPendingInstalls.remove(0);
                     }
                     if (mPendingInstalls.size() == 0) {
                         // 如果没有已经没有安装请求了，而且DefaultContainerService还处于绑定中
                         if (mBound) {
                             // 发送消息，解除DefaultContainerService绑定
                              removeMessages(MCS_UNBIND);
                              Message ubmsg = obtainMessage(MCS_UNBIND);
                              sendMessageDelayed(ubmsg, 10000);
                           }
                       } else {
                           // mPendingInstalls中还有未处理的安装请求，继续发送MCS_BOUND消息处理
                               mHandler.sendEmptyMessage(MCS_BOUND);
                        }
                }
            } else {
        }
        break;
```

上面的MCS_BOUND逻辑简单来说，就是与DefaultContainerConnection连接后，如果有安装请求，则调用startCopy来开始拷贝请求的apk，执行完后，会检查是否还有安装请求，如果没有，则解除DefaultContainerService的绑定。如果还有安装请求，则再次发送MCS_BOUND消息进行安装请求处理。先重点分析startCopy方法。

### 3.4 拷贝APK#startCopy

> \frameworks\base\services\core\java\com\android\server\pm\PackageManagerService.java

```java
final boolean startCopy() {
    boolean res;
    try {
        // 第一步
        // MAX_RETRIES=4，代表有4次尝试安装的机会，如果超过四次就放弃这个安装请求
        if (++mRetries > MAX_RETRIES) {
            mHandler.sendEmptyMessage(MCS_GIVE_UP);
            handleServiceError();
            return false;
        } else {
            // 调用handleStartCopy抽象方法
            handleStartCopy();
            res = true;
        }
    } catch (RemoteException e) {
          mHandler.sendEmptyMessage(MCS_RECONNECT);
          res = false;
     }
     // 第二步
     handleReturnCode();
     return res;
}
```

1. 判断尝试次数是否大于四次，这里有两种情况： 
   - 如果超过4次，则发送一个MCS_GIVE_UP消息放弃本次安装
   - 如果没超过4次， 则调用handleStartCopy方法进行APK拷贝文件的操作。如果在handleStartCopy方法调用的时候产生了异常则发送MCS_RECONNECT消息进行重连。
2. 上面完成之后，调用handleReturnCode处理拷贝文件后的操作

 handleStartCopy的具体实现在InstallParams中 。

### 3.5 开始处理拷贝APK#handleStartCopy

```java
public void handleStartCopy() throws RemoteException {
    int ret = PackageManager.INSTALL_SUCCEEDED;
    // 第一步
    //如果apk文件已经暂存，则往下执行，apk文件暂存是在PackageInstaller调用PackageInstallerSession完成的
    // 根据参数设置确定应用的安装位置
    if (origin.staged) {
        if (origin.file != null) {
            installFlags |= PackageManager.INSTALL_INTERNAL;
            installFlags &= ~PackageManager.INSTALL_EXTERNAL;
        } else if (origin.cid != null) {
            installFlags |= PackageManager.INSTALL_EXTERNAL;
            installFlags &= ~PackageManager.INSTALL_INTERNAL;
        } else {
            throw new IllegalStateException("Invalid stage location");
        }
    }
    // 安装到SD卡
    final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
    // 安装到内部存储
    final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
    // 安装临时存储
    final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
    PackageInfoLite pkgLite = null;
    // 不能同时安装在内部存储以及SD卡中
    if (onInt && onSd) {
        ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
        // Instant APP不能安装到SD卡中
    } else if (onSd && ephemeral) {
        ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
    } else {
        // 通过mContainerService获取最小APK安装信息
        // 解析packagePath对应的安装包，获取解析的出来的"轻"安装包pkg
        pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                packageAbiOverride);//【3.5.1】
        // 如果空间不够，尝试释放缓存
        if (!origin.staged && pkgLite.recommendedInstallLocation
                == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
            final StorageManager storage = StorageManager.from(mContext);
            final long lowThreshold = storage.getStorageLowBytes(
                    Environment.getDataDirectory());
            // 计算安装container的大小
            final long sizeBytes = mContainerService.calculateInstalledSize(
                    origin.resolvedPath, isForwardLocked(), packageAbiOverride);
    
            try {
                // 释放缓存
                mInstaller.freeCache(null, sizeBytes + lowThreshold, 0, 0);
                pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath,
                        installFlags, packageAbiOverride);
            } catch (InstallerException e) {
                Slog.w(TAG, "Failed to free cache", e);
            }
            // 空间依旧不够
            if (pkgLite.recommendedInstallLocation
                    == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                pkgLite.recommendedInstallLocation
                    = PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
            }
        }
    }
    // 第二步
    //ret值为安装成功
    if (ret == PackageManager.INSTALL_SUCCEEDED) {
        // 判断返回值，确认应用最后的安装路劲
        int loc = pkgLite.recommendedInstallLocation;
        if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
            ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
        } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
            ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
        } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
            ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
        } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
            ret = PackageManager.INSTALL_FAILED_INVALID_APK;
        } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
            ret = PackageManager.INSTALL_FAILED_INVALID_URI;
        } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
            ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
        } else {
            //获取最终的安装路径,主要判断终端上是否有已经安装过该APK，同一个APK一般只能用新版的替换旧版
            loc = installLocationPolicy(pkgLite);//【3.5.2】
            if (loc == PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE) {
                ret = PackageManager.INSTALL_FAILED_VERSION_DOWNGRADE;
            } else if (!onSd && !onInt) {
                if (loc == PackageHelper.RECOMMEND_INSTALL_EXTERNAL) {
                    installFlags |= PackageManager.INSTALL_EXTERNAL;
                    installFlags &= ~PackageManager.INSTALL_INTERNAL;
                } else if (loc == PackageHelper.RECOMMEND_INSTALL_EPHEMERAL) {
                    installFlags |= PackageManager.INSTALL_INSTANT_APP;
                    installFlags &= ~(PackageManager.INSTALL_EXTERNAL
                            |PackageManager.INSTALL_INTERNAL);
                } else {
                    installFlags |= PackageManager.INSTALL_INTERNAL;
                    installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                }
            }
        }
    }
    // 第三步
    // 确认完安装路径后，调用createInstallArgs来创建一个安装参数对象
    final InstallArgs args = createInstallArgs(this);//【3.5.3】
    mArgs = args;
    // 第四步
    if (ret == PackageManager.INSTALL_SUCCEEDED) {
        // 如果是为所有用户安装，就调用设备拥有者来校验安装此应用
        UserHandle verifierUser = getUser();
        if (verifierUser == UserHandle.ALL) {
            verifierUser = UserHandle.SYSTEM;
        }
        final int requiredUid = mRequiredVerifierPackage == null ? -1
                : getPackageUid(mRequiredVerifierPackage, MATCH_DEBUG_TRIAGED_MISSING,
                        verifierUser.getIdentifier());
    
        final int optionalUid = mOptionalVerifierPackage == null ? -1
                : getPackageUid(mOptionalVerifierPackage, MATCH_DEBUG_TRIAGED_MISSING,
                        verifierUser.getIdentifier());
        final int installerUid =
                verificationInfo == null ? -1 : verificationInfo.installerUid;
        // 判断这个apk是否需要包验证
        if (!origin.existing && (requiredUid != -1 || optionalUid != -1)
                && isVerificationEnabled(
                        verifierUser.getIdentifier(), installFlags, installerUid)) {
            ...
        } else {
            // 调用安装参数对象的copyApk方法执行拷贝安装
            ret = args.copyApk(mContainerService, true);//【3.5.4】
        }
    }
    mRet = ret;
}    
```

1. 根据origin.staged、origin.file和origin.cid确定installFlags和ret的值，通过mContainerService获取最小APK安装信息pkgLite
2. 根据步骤一获取的pkgLite确认安装路径
3. 创建安装参数对象args
4. 这里根据是否需要验证分为两种情况

- 需要验证：需要构建一个Intent对象verifaction，并设置相应的参数，然后发送一个广播进行包验证。在广播的onReceive中，则发送了CHECK_PENDING_VERIFICATION消息
- 不需要验证：直接调用InstallArgs的copyApk方法进行包拷贝。

#### 3.5.1 获取少量APK包信息#getMinimalPackageInfo

> \frameworks\base\packages\DefaultContainerService\src\com\android\defcontainer\DefaultContainerService.java

```java
public PackageInfoLite getMinimalPackageInfo(String packagePath, int flags,
            String abiOverride) {
    // 第一步
    final Context context = DefaultContainerService.this;
    final boolean isForwardLocked = (flags & PackageManager.INSTALL_FORWARD_LOCK) != 0;

    //创建轻量级包信息对象
    PackageInfoLite ret = new PackageInfoLite();
    if (packagePath == null) {
        ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_APK;
        return ret;
    }
    // 第二步
    final File packageFile = new File(packagePath);
    final PackageParser.PackageLite pkg;
    final long sizeBytes;
    try {
        //轻量级解析apk包
        pkg = PackageParser.parsePackageLite(packageFile, 0);
        sizeBytes = PackageHelper.calculateInstalledSize(pkg, isForwardLocked, abiOverride);
    } catch (PackageParserException | IOException e) {
        Slog.w(TAG, "Failed to parse package at " + packagePath + ": " + e);

        if (!packageFile.exists()) {
            ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_URI;
        } else {
            ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_APK;
        }

        return ret;
    }

    final int recommendedInstallLocation;
    final long token = Binder.clearCallingIdentity();
    try {
        recommendedInstallLocation = PackageHelper.resolveInstallLocation(context,
                pkg.packageName, pkg.installLocation, sizeBytes, flags);
    } finally {
        Binder.restoreCallingIdentity(token);
    }
    // 将解析的包信息赋予PackageInfoLite对象并返还
    ret.packageName = pkg.packageName;
    ret.splitNames = pkg.splitNames;
    ret.versionCode = pkg.versionCode;
    ret.baseRevisionCode = pkg.baseRevisionCode;
    ret.splitRevisionCodes = pkg.splitRevisionCodes;
    ret.installLocation = pkg.installLocation;
    ret.verifiers = pkg.verifiers;
    ret.recommendedInstallLocation = recommendedInstallLocation;
    ret.multiArch = pkg.multiArch;

    return ret;
}
```

接下来看下parsePackageLite的代码。

> \frameworks\base\core\java\android\content\pm\PackageParser.java

```java
public static PackageLite parsePackageLite(File packageFile, int flags)
        throws PackageParserException {
    if (packageFile.isDirectory()) {
        //解析集群式apk包文件
        return parseClusterPackageLite(packageFile, flags);
    } else {
        //解析单一apk包文件
        return parseMonolithicPackageLite(packageFile, flags);
    }
}
```

翻译该方法的注释如下：

> 仅分析给定位置的包的轻量级详细信息。自动检测包是单片样式（单个APK文件）还是群集样式（APK目录）。
>
> 这将对群集样式的包执行健全性检查，例如要求相同的包名称和版本代码、单个基本APK和唯一的拆分名称。

 之所以要轻量级解析是因为解析APK是一个复杂耗时的操作，这里的逻辑并不需要APK所有的信息。 

所谓的群集apk包，简单来说就是可以安装一个目录多个具有相同的包名称和版本的apk。简单来说就是 Split APK机制。Split APK是Google为解决65536上限，以及APK安装包越来越大等问题，在Android L中引入的机制。 
Split APK可以将一个庞大的APK，按屏幕密度，ABI等形式拆分成多个独立的APK，在应用程序更新时，不必下载整个APK，只需单独下载某个模块即可安装更新。 
Split APK将原来一个APK中多个模块共享同一份资源的模型分离成多个APK使用各自的资源，并且可以继承Base APK中的资源，多个APK有相同的data，cache目录，多个dex文件，相同的进程，在Settings.apk中只显示一个APK，并且使用相同的包名。 

##### (1) 轻量级解析Cluster APK包#parseClusterPackageLite

```java
static PackageLite parseClusterPackageLite(File packageDir, int flags)
        throws PackageParserException {
    // 获取目录下的所有文件
    final File[] files = packageDir.listFiles();
    if (ArrayUtils.isEmpty(files)) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                "No packages found in split");
    }

    String packageName = null;
    int versionCode = 0;
    // 开始验证包名和版本号是否一致
    final ArrayMap<String, ApkLite> apks = new ArrayMap<>();
    // 遍历所有文件，逐个解析单个apk文件
    for (File file : files) {
        if (isApkFile(file)) {
            //解析单个APK文件成ApkLite对象
            final ApkLite lite = parseApkLite(file, flags);
            // 遍历的时候只有第一个文件的时候packageName为null
            if (packageName == null) {
                packageName = lite.packageName;
                versionCode = lite.versionCode;
            } else {
                // 对比当前的lite对象和上一个lite对象的包名是否一致
                if (!packageName.equals(lite.packageName)) {
                    throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                            "Inconsistent package " + lite.packageName + " in " + file
                                + "; expected " + packageName);
                }
                // 对比当前的lite对象和上一个lite对象的版本号是否一致
                if (versionCode != lite.versionCode) {
                    throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                            "Inconsistent version " + lite.versionCode + " in " + file
                            + "; expected " + versionCode);
                }
            }

            // Assert that each split is defined only once
            // 保证不重复添加
            if (apks.put(lite.splitName, lite) != null) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                        "Split name " + lite.splitName
                        + " defined more than once; most recent was " + file);
            }
        }
    }
    // 获取base APK
    final ApkLite baseApk = apks.remove(null);
    if (baseApk == null) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                "Missing base APK in " + packageDir);
    }

    // Always apply deterministic ordering based on splitName
    final int size = apks.size();

    String[] splitNames = null;
    boolean[] isFeatureSplits = null;
    String[] usesSplitNames = null;
    String[] configForSplits = null;
    String[] splitCodePaths = null;
    int[] splitRevisionCodes = null;
    String[] splitClassLoaderNames = null;
    if (size > 0) {
        // 初始化 splitNames
        splitNames = new String[size];
        isFeatureSplits = new boolean[size];
        usesSplitNames = new String[size];
        configForSplits = new String[size];
        splitCodePaths = new String[size];
        splitRevisionCodes = new int[size];

        splitNames = apks.keySet().toArray(splitNames);
        //  splitNames排序
        Arrays.sort(splitNames, sSplitNameComparator);
        // 初始化splitCodePaths和splitRevisionCodes
        for (int i = 0; i < size; i++) {
            final ApkLite apk = apks.get(splitNames[i]);
            usesSplitNames[i] = apk.usesSplitName;
            isFeatureSplits[i] = apk.isFeatureSplit;
            configForSplits[i] = apk.configForSplit;
            splitCodePaths[i] = apk.codePath;
            splitRevisionCodes[i] = apk.revisionCode;
        }
    }

    final String codePath = packageDir.getAbsolutePath();
    //new一个PackageLite对象
    return new PackageLite(codePath, baseApk, splitNames, isFeatureSplits, usesSplitNames,
            configForSplits, splitCodePaths, splitRevisionCodes);
}
```

该方法做了以下几件事：

1. 校验参数，主要是校验包名和版本号是否一致
2. 给baseApk、splitNames、splitCodePaths、splitRevisionCodes、codePath完成初始化
3. 根据第二步给出的变量，new一个PackageLite对象

**轻量级解析单个APK包#parseApkLite**

```java
public static ApkLite parseApkLite(File apkFile, int flags)
        throws PackageParserException {
    final String apkPath = apkFile.getAbsolutePath();

    AssetManager assets = null;
    XmlResourceParser parser = null;
    try {
        assets = newConfiguredAssetManager();
        int cookie = assets.addAssetPath(apkPath);
        if (cookie == 0) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                    "Failed to parse " + apkPath);
        }

        final DisplayMetrics metrics = new DisplayMetrics();
        metrics.setToDefaults();

        parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

        final Signature[] signatures;
        final Certificate[][] certificates;
        if ((flags & PARSE_COLLECT_CERTIFICATES) != 0) {
            final Package tempPkg = new Package((String) null);
            try {
                collectCertificates(tempPkg, apkFile, flags);
            } finally {
            }
            signatures = tempPkg.mSignatures;
            certificates = tempPkg.mCertificates;
        } else {
            signatures = null;
            certificates = null;
        }

        final AttributeSet attrs = parser;
        return parseApkLite(apkPath, parser, attrs, signatures, certificates);

    } catch (XmlPullParserException | IOException | RuntimeException e) {
        Slog.w(TAG, "Failed to parse " + apkPath, e);
        throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                "Failed to parse " + apkPath, e);
    } finally {
        IoUtils.closeQuietly(parser);
        IoUtils.closeQuietly(assets);
    }
}
```

该方法主要做了以下工作：

1. 创建AssetManager对象assets并打开apk的AndroidManifest.xml文件
2. 根据步骤1的assets创建XmlResourceParser对象
3. 解析apk签名文件
4. 调用parseApkLite轻量级解析apk文件并返回一个轻量级的ApkLite对象。

接着分析parseApkLite。

```java
private static ApkLite parseApkLite(String codePath, XmlPullParser parser, AttributeSet attrs,Signature[] signatures, Certificate[][] certificates)
    throws IOException, XmlPullParserException, PackageParserException {
    //创建Pair数据结构
    final Pair<String, String> packageSplit = parsePackageSplitNames(parser, attrs);

    int installLocation = PARSE_DEFAULT_INSTALL_LOCATION;
    int versionCode = 0;
    int revisionCode = 0;
    boolean coreApp = false;
    boolean debuggable = false;
    boolean multiArch = false;
    boolean use32bitAbi = false;
    boolean extractNativeLibs = true;
    boolean isolatedSplits = false;
    boolean isFeatureSplit = false;
    String configForSplit = null;
    String usesSplitName = null;
    //解析apk文件的AndroidManifest.xml属性
    for (int i = 0; i < attrs.getAttributeCount(); i++) {
        final String attr = attrs.getAttributeName(i);
        if (attr.equals("installLocation")) {
            installLocation = attrs.getAttributeIntValue(i,
                    PARSE_DEFAULT_INSTALL_LOCATION);
        } else if (attr.equals("versionCode")) {
            versionCode = attrs.getAttributeIntValue(i, 0);
        } else if (attr.equals("revisionCode")) {
            revisionCode = attrs.getAttributeIntValue(i, 0);
        } else if (attr.equals("coreApp")) {
            coreApp = attrs.getAttributeBooleanValue(i, false);
        } else if (attr.equals("isolatedSplits")) {
            isolatedSplits = attrs.getAttributeBooleanValue(i, false);
        } else if (attr.equals("configForSplit")) {
            configForSplit = attrs.getAttributeValue(i);
        } else if (attr.equals("isFeatureSplit")) {
            isFeatureSplit = attrs.getAttributeBooleanValue(i, false);
        }
    }

    // Only search the tree when the tag is directly below <manifest>
    int type;
    final int searchDepth = parser.getDepth() + 1;

    final List<VerifierInfo> verifiers = new ArrayList<VerifierInfo>();
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
            && (type != XmlPullParser.END_TAG || parser.getDepth() >= searchDepth)) {
        if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
            continue;
        }

        if (parser.getDepth() != searchDepth) {
            continue;
        }

        if (TAG_PACKAGE_VERIFIER.equals(parser.getName())) {
            final VerifierInfo verifier = parseVerifier(attrs);
            if (verifier != null) {
                verifiers.add(verifier);
            }
        } else if (TAG_APPLICATION.equals(parser.getName())) {//"application"标签
            for (int i = 0; i < attrs.getAttributeCount(); ++i) {
                final String attr = attrs.getAttributeName(i);
                if ("debuggable".equals(attr)) {
                    debuggable = attrs.getAttributeBooleanValue(i, false);
                }
                if ("multiArch".equals(attr)) {
                    multiArch = attrs.getAttributeBooleanValue(i, false);
                }
                if ("use32bitAbi".equals(attr)) {
                    use32bitAbi = attrs.getAttributeBooleanValue(i, false);
                }
                if ("extractNativeLibs".equals(attr)) {
                    extractNativeLibs = attrs.getAttributeBooleanValue(i, true);
                }
            }
        } else if (TAG_USES_SPLIT.equals(parser.getName())) {
            if (usesSplitName != null) {
                Slog.w(TAG, "Only one <uses-split> permitted. Ignoring others.");
                continue;
            }

            usesSplitName = attrs.getAttributeValue(ANDROID_RESOURCES, "name");
            if (usesSplitName == null) {
                throw new PackageParserException(
                        PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                            "<uses-split> tag requires 'android:name' attribute");
            }
        }
    }

    //根据解析的APK文件信息创建一个轻量级的apk对象
    return new ApkLite(codePath, packageSplit.first, packageSplit.second, isFeatureSplit,
            configForSplit, usesSplitName, versionCode, revisionCode, installLocation,
            verifiers, signatures, certificates, coreApp, debuggable, multiArch, use32bitAbi,
            extractNativeLibs, isolatedSplits);
}
```

##### (2) 轻量级解析Monolithic类型的APK包#parseMonolithicPackageLite

所谓的Monolithic类型的APK包是相对Cluster APK包来说的，即我们平时使用的APK安装包，只包含一个APK。

```java
private static PackageLite parseMonolithicPackageLite(File packageFile, int flags)
        throws PackageParserException {
    final ApkLite baseApk = parseApkLite(packageFile, flags);
    final String packagePath = packageFile.getAbsolutePath();
    return new PackageLite(packagePath, baseApk, null, null, null, null, null, null);
}
```

调用parseApkLite轻量级解析apk包，得到一个ApkLite对象，然后使用ApkLite对象构建一个轻量级的包对象PackageLite。

#### 3.5.2 APK安装位置#installLocationPolicy

回到handleStartCopy方法，接着分析另外一个重要的方法installLocationPolicy。

```java
private int installLocationPolicy(PackageInfoLite pkgLite) {
    String packageName = pkgLite.packageName;
    int installLocation = pkgLite.installLocation;
    boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
        synchronized (mPackages) {
            //从mPackages查询是否包含需要安装的apk包名，如果有，说明已经安装了该apk
            PackageParser.Package installedPkg = mPackages.get(packageName);
            PackageParser.Package dataOwnerPkg = installedPkg;
            if (dataOwnerPkg  == null) {
                //mPackages没有该apk的包名，则从mSettings.mPackages查询，如果有，说明之前安装过，但已卸载
                PackageSetting ps = mSettings.mPackages.get(packageName);
                if (ps != null) {
                    dataOwnerPkg = ps.pkg;
                }
            }
            //如果安装了apk包或之前安装了但已卸载，则新安装的apk包可以访问其之前留在设备上的数据。
            //作为一种安全措施，只有在这不是版本降级或前置包标记为可调试且显式请求降级时，才允许这样做。
            if (dataOwnerPkg != null) {
                final boolean downgradeRequested =
                        (installFlags & PackageManager.INSTALL_ALLOW_DOWNGRADE) != 0;
                final boolean packageDebuggable =
                            (dataOwnerPkg.applicationInfo.flags
                                    & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
                final boolean downgradePermitted =
                        (downgradeRequested) && ((Build.IS_DEBUGGABLE) || (packageDebuggable));
                if (!downgradePermitted) {
                    try {
                        checkDowngrade(dataOwnerPkg, pkgLite);
                    } catch (PackageManagerException e) {
                        Slog.w(TAG, "Downgrade detected: " + e.getMessage());
                        return PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE;
                    }
                }
            }
            //检查系统apk是否想安装在SD上
            if (installedPkg != null) {
                if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                    // Check for updated system application.
                    if ((installedPkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                        if (onSd) {
                            Slog.w(TAG, "Cannot install update to system app on sdcard");
                            //系统apk不能安装在sd卡上
                            return PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION;
                        }
                        return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                    } else {
                        // apk安装包在sd卡上，系统推荐安装在sd卡上
                        if (onSd) {
                            return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
                        }
                        //apk安装包预先指定了只安装在内部存储上
                        if (installLocation == PackageInfo.INSTALL_LOCATION_INTERNAL_ONLY) {
                            return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                        }
                        //apk安装包预先指定了首选安装在sd卡上，这种情况让系统决定
                        else if (installLocation == PackageInfo.INSTALL_LOCATION_PREFER_EXTERNAL) {
                            // App explictly prefers external. Let policy decide
                        } else {
                            // Prefer previous location
                            //首选以前的安装位置
                            if (isExternal(installedPkg)) {
                                return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
                            }
                            return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                        }
                    }
                } else {
                    // Invalid install. Return error code
                    //无效的安装
                    return PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS;
                }
            }
    }
    // All the special cases have been taken care of.
    // Return result based on recommended install location.
    if (onSd) {
        return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
    }
    return pkgLite.recommendedInstallLocation;
}
```

该方法是根据策略来决定apk安装的位置，流程图：

![install-location](images/pms/pms/install-location.png)

#### 3.5.3 创建InstallArgs类createInstallArgs

```java
private InstallArgs createInstallArgs(InstallParams params) {
    if (params.move != null) {
        return new MoveInstallArgs(params);
    } else if (installOnExternalAsec(params.installFlags) || params.isForwardLocked()) {
        return new AsecInstallArgs(params);
    } else {
        return new FileInstallArgs(params);
    }
}
```

InstallArgs类是一个抽象类，子类有MoveInstallArgs、AsecInstallArgs和FileInstallArgs。功能如下：

- FileInstallArgs：用于处理安装到非ASEC的存储空间的APK，也就是内部存储空间(Data分区)
- AsecInstallArgs：用于处理安装到ASEC中（mnt/asec）即SD卡中的APK
- MoveInstallArgs用于处理已安装APK的移动的逻辑

我们这里的例子是FileInstallArgs，因此分析FileInstallArgs。

#### 3.5.4 拷贝APK#copyApk

回到2.5小节的handleStartCopy，最后会调用copyApk进行apk的拷贝和做进一步的安装工作。copyApk调用doCopyApk做具体的操作。

```java
private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
    //staged表示apk是否已经缓存，在8.1版本为true，，因为在packageInstaller时已经通过Session拷贝APK
    if (origin.staged) {
        codeFile = origin.file;
        resourceFile = origin.file;
        return PackageManager.INSTALL_SUCCEEDED;
    }

    try {
        // 非Instatnt APP
        final boolean isEphemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
        //创建临时存储目录,临时路径一般为/data/app/xxxxxxxxxx.tmp
        final File tempDir =
                mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);
        codeFile = tempDir;
        resourceFile = tempDir;
    } catch (IOException e) {
        Slog.w(TAG, "Failed to create copy file: " + e);
        return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
    }
    //复制文件的回调
    final IParcelFileDescriptorFactory target = new IParcelFileDescriptorFactory.Stub() {
        @Override
        public ParcelFileDescriptor open(String name, int mode) throws RemoteException {
            if (!FileUtils.isValidExtFilename(name)) {
                throw new IllegalArgumentException("Invalid filename: " + name);
            }
            try {
                // 当接口被回调时，需要创建并打开接口，同时赋予相应权限
                final File file = new File(codeFile, name);
                final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                        O_RDWR | O_CREAT, 0644);
                Os.chmod(file.getAbsolutePath(), 0644);
                return new ParcelFileDescriptor(fd);
            } catch (ErrnoException e) {
                throw new RemoteException("Failed to open: " + e.getMessage());
            }
        }
    };
    // 调用DefaultContainerService的copyPackage方法进行package复制操作，同时将回调接口传入
    int ret = PackageManager.INSTALL_SUCCEEDED;
    ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);
    if (ret != PackageManager.INSTALL_SUCCEEDED) {
        Slog.e(TAG, "Failed to copy package");
        return ret;
    }
    // 复制APK里面的native库文件
    final File libraryRoot = new File(codeFile, LIB_DIR_NAME);
    NativeLibraryHelper.Handle handle = null;
    try {
        handle = NativeLibraryHelper.Handle.create(codeFile);
        ret = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libraryRoot,
                abiOverride);
    } catch (IOException e) {
        Slog.e(TAG, "Copying native libraries failed", e);
        ret = PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
    } finally {
        IoUtils.closeQuietly(handle);
    }

    return ret;
}
```

FileInstallArgs的copyApk会调用doCopyApk做具体的拷贝工作。首先判断apk是否已经缓存，在Android8.1版本，在PackageInstaller调用PMS前已经利用Session进行了apk的拷贝工作，因此在doCopyApk就不需要再次拷贝，直接返回PackageManager.INSTALL_SUCCEEDED即可。 接下来看拷贝APK操作之后的操作，即startCopy方法中另外一个重要的方法handleReturnCode。

### 3.6 处理apk安装#handleReturnCode

2.4小节startCopy方法调用handleStartCopy拷贝完apk后，最后会调用handleReturnCode进行apk的安装。handleReturnCode没什么操作，直接调用processPendingInstall进行安装。

```java
private void processPendingInstall(final InstallArgs args, final int currentStatus) {
    // Queue up an async operation since the package installation may take a little while.
    mHandler.post(new Runnable() {
        public void run() {
            // 清除任务
            mHandler.removeCallbacks(this);
             // Result object to be returned
            // 安装结果信息对象
            PackageInstalledInfo res = new PackageInstalledInfo();
            res.setReturnCode(currentStatus);
            res.uid = -1;
            res.pkg = null;
            res.removedInfo = null;
            // 如果之前copyApk正常执行完成，PackageManager.INSTALL_SUCCEEDED成立
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                //apk安装前操作，删除安装残留文件
                args.doPreInstall(res.returnCode);
                synchronized (mInstallLock) {
                    //执行安装操作
                    installPackageTracedLI(args, res);//【3.6.1】
                }
                //安装完成以后的操作
                args.doPostInstall(res.returnCode, res.uid);
            }

            // 安装成功，且需要备份的情况
            final boolean update = res.removedInfo != null
                    && res.removedInfo.removedPackage != null;
            final int flags = (res.pkg == null) ? 0 : res.pkg.applicationInfo.flags;
            boolean doRestore = !update
                    && ((flags & ApplicationInfo.FLAG_ALLOW_BACKUP) != 0);

            int token;
            if (mNextInstallToken < 0) mNextInstallToken = 1;
            token = mNextInstallToken++;

            PostInstallData data = new PostInstallData(args, res);
            mRunningInstalls.put(token, data);
            if (DEBUG_INSTALL) Log.v(TAG, "+ starting restore round-trip " + token);

            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED && doRestore) {
                // 调用backup接口进行恢复
                IBackupManager bm = IBackupManager.Stub.asInterface(
                        ServiceManager.getService(Context.BACKUP_SERVICE));
                if (bm != null) {
                    try {
                        if (bm.isBackupServiceActive(UserHandle.USER_SYSTEM)) {
                            bm.restoreAtInstall(res.pkg.applicationInfo.packageName, token);
                        } else {
                            doRestore = false;
                        }
                    } catch (RemoteException e) {
                        // can't happen; the backup manager is local
                    } catch (Exception e) {
                        Slog.e(TAG, "Exception trying to enqueue restore", e);
                        doRestore = false;
                    }
                } else {
                    Slog.e(TAG, "Backup Manager not found!");
                    doRestore = false;
                }
            }
            // 安装阶段结束
            if (!doRestore) {
                // 安装完成后发送"POST_INSTALL"广播
                Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);// 【3.6.2】
                mHandler.sendMessage(msg);
            }
        }
    });
}
```

processPendingInstall方法，最终会调用到 InstallArgs 的 doPreInstall完成安装前的清理工作，调用 installPackageTracedLI实现真正的安装，以及调用 InstallArgs 的 doPostInstall完成收尾的清理工作，无论安装成功与否，最后都会发送一个类型为 POST_INSTALL 的消息执行安装完成的收尾工作。

#### 3.6.1 安装APK#installPackageTracedLI

installPackageTracedLI直接调用installPackageLI执行安装工作。

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    //installFlags属性表明apk安装的位置
    final int installFlags = args.installFlags;
    final String installerPackageName = args.installerPackageName;
    final String volumeUuid = args.volumeUuid;
    final File tmpPackageFile = new File(args.getCodePath());
    final boolean forwardLocked = ((installFlags & PackageManager.INSTALL_FORWARD_LOCK) != 0);
    // 是否安装到外部存储
    final boolean onExternal = (((installFlags & PackageManager.INSTALL_EXTERNAL) != 0)
            || (args.volumeUuid != null));
    final boolean instantApp = ((installFlags & PackageManager.INSTALL_INSTANT_APP) != 0);
    final boolean fullApp = ((installFlags & PackageManager.INSTALL_FULL_APP) != 0);
    final boolean forceSdk = ((installFlags & PackageManager.INSTALL_FORCE_SDK) != 0);
    final boolean virtualPreload =
            ((installFlags & PackageManager.INSTALL_VIRTUAL_PRELOAD) != 0);
    // 新安装还是更新安装的标志位
    boolean replace = false;
    int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
    ...

    // Result object to be returned
    // 设置默认安装返回值为INSTALL_SUCCEEDED
    res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
    // 安装包包名
    res.installerPackageName = installerPackageName;
    
    ...
    final PackageParser.Package pkg;
    try {
        // 解析APK，主要是解析AndroidManifest.xml文件，将结果记录在PackageParser.Package中
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);//【3.6.1-(1)】
    } catch (PackageParserException e) {
        res.setError("Failed parse during installPackageLI", e);
    return;
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }

    // Instant apps must have target SDK >= O and have targetSanboxVersion >= 2
    // 检查系统是否符合安装Instant APP的要求
    if (instantApp && pkg.applicationInfo.targetSdkVersion <= Build.VERSION_CODES.N_MR1) {
        Slog.w(TAG, "Instant app package " + pkg.packageName + " does not target O");
        res.setError(INSTALL_FAILED_SANDBOX_VERSION_DOWNGRADE,
                "Instant app package must target O");
        return;
    }
    if (instantApp && pkg.applicationInfo.targetSandboxVersion != 2) {
        Slog.w(TAG, "Instant app package " + pkg.packageName
                + " does not target targetSandboxVersion 2");
        res.setError(INSTALL_FAILED_SANDBOX_VERSION_DOWNGRADE,
                "Instant app package must use targetSanboxVersion 2");
        return;
    }

    // 安装静态共享库
    if (pkg.applicationInfo.isStaticSharedLibrary()) {
        // Static shared libraries have synthetic package names
        renameStaticSharedLibraryPackage(pkg);

        // No static shared libs on external storage
        if (onExternal) {
            Slog.i(TAG, "Static shared libs can only be installed on internal storage.");
            res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                    "Packages declaring static-shared libs cannot be updated");
            return;
        }
    }

    // If we are installing a clustered package add results for the children
    // 安装一簇安装包的时候，需要记录child package的安装情况
    if (pkg.childPackages != null) {
        synchronized (mPackages) {
            ...
        }
    }
    // If package doesn't declare API override, mark that we have an install
    // time CPU ABI override.
    // CPU ABI处理
    if (TextUtils.isEmpty(pkg.cpuAbiOverride)) {
        pkg.cpuAbiOverride = args.abiOverride;
    }

    String pkgName = res.name = pkg.packageName;
    ...
    try {
        // either use what we've been given or parse directly from the APK
        // 已经解析到APK的安装签名证书的话，不需要重新从apk包解析
        if (args.certificates != null) {
            try {
                // 从apk证书中提取信息到package对象中
                PackageParser.populateCertificates(pkg, args.certificates);
            } catch (PackageParserException e) {
                // there was something wrong with the certificates we were given;
                // try to pull them from the APK
                PackageParser.collectCertificates(pkg, parseFlags);
            }
        } else {
            // 还没有解析到APK的安装签名证书的话，需要重新从apk包解析
            PackageParser.collectCertificates(pkg, parseFlags);
        }
    } catch (PackageParserException e) {
        res.setError("Failed collect during installPackageLI", e);
        return;
    }
    pp = null;
    String oldCodePath = null;
    boolean systemApp = false;
    synchronized (mPackages) {
        // 检查是否已经安装过该安装包
        if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
            String oldName = mSettings.getRenamedPackageLPr(pkgName);
            // 该安装包是来自一个原始包的，而且设备已经从原始包升级而来
            // 这里需要继续使用原始包的包名
            if (pkg.mOriginalPackages != null
                    && pkg.mOriginalPackages.contains(oldName)
                    && mPackages.containsKey(oldName)) {
                pkg.setPackageName(oldName);
                pkgName = pkg.packageName;
                // 设置为替换
                replace = true;
                if (DEBUG_INSTALL) Slog.d(TAG, "Replacing existing renamed package: oldName=" + oldName + " pkgName=" + pkgName);
            } else if (mPackages.containsKey(pkgName)) {
                replace = true;
            ...
            // 如果需要替换
            if (replace) {
                PackageParser.Package oldPackage = mPackages.get(pkgName);
                final int oldTargetSdk = oldPackage.applicationInfo.targetSdkVersion;
                final int newTargetSdk = pkg.applicationInfo.targetSdkVersion;
                if (oldTargetSdk > Build.VERSION_CODES.LOLLIPOP_MR1
                        && newTargetSdk <= Build.VERSION_CODES.LOLLIPOP_MR1) {
                    return;
                }
                // 防止应用降低SandBox的版本
                final int oldTargetSandbox = oldPackage.applicationInfo.targetSandboxVersion;
                final int newTargetSandbox = pkg.applicationInfo.targetSandboxVersion;
                if (oldTargetSandbox == 2 && newTargetSandbox != 2) {
                    return;
                }
                // 防止直接安装子程序
                if (oldPackage.parentPackage != null) {
                    return;
                }
             }
        }
        // 查看Setting中是否存在要安装的APK的信息，如果有就获取安装包对应的信息
        PackageSetting ps = mSettings.mPackages.get(pkgName);
        if (ps != null) {
            // 可以存在多个静态共享库，在安装时需要检查其签名是否正确
            PackageSetting signatureCheckPs = ps;
            // 如果 ps 不为null，同样说明，已经存在一个具有相同安装包包名的程序，被安装，所以还是处理覆盖安装的问题。
            // 这里主要验证包名的签名，不一致的话，是不能覆盖安装的，另外版本号也不能比安装的低，否则不能替换安装
            if (pkg.applicationInfo.isStaticSharedLibrary()) {
                SharedLibraryEntry libraryEntry = getLatestSharedLibraVersionLPr(pkg);
                if (libraryEntry != null) {
                    signatureCheckPs = mSettings.getPackageLPr(libraryEntry.apk);
                }
            }
            //检查密钥集合是否一致
            if (shouldCheckUpgradeKeySetLP(signatureCheckPs, scanFlags)) {
                if (!checkUpgradeKeySetLP(signatureCheckPs, pkg)) {
                    return;
                }
            } else {
                try {
                    verifySignaturesLP(signatureCheckPs, pkg);
                } catch (PackageManagerException e) {
                    return;
                }
            }
             // 判断安装的应用是否存在同名的应用，如果存在，判断应用是否带有系统应用的标志
            oldCodePath = mSettings.mPackages.get(pkgName).codePathString;
            if (ps.pkg != null && ps.pkg.applicationInfo != null) {
                systemApp = (ps.pkg.applicationInfo.flags &
                        ApplicationInfo.FLAG_SYSTEM) != 0;
            }
            res.origUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
        }
        // 检查APK中定义的所有的权限是否已经被其他应用定义了，如果重定义的是系统应用定义的权限，
        // 那么忽略本apk定义的这个权限。如果重定义的是非系统引用的权限，那么本次安装就以失败返回。
        int N = pkg.permissions.size();
        for (int i = N-1; i >= 0; i--) {
            PackageParser.Permission perm = pkg.permissions.get(i);
            BasePermission bp = mSettings.mPermissions.get(perm.info.name);

            // 仅允许systemApp定义ephemeral权限
            if ((perm.info.protectionLevel & PermissionInfo.PROTECTION_FLAG_INSTANT) != 0
                    && !systemApp) {
                Slog.w(TAG, "Non-System package " + pkg.packageName
                        + " attempting to delcare ephemeral permission "
                        + perm.info.name + "; Removing ephemeral.");
                perm.info.protectionLevel &= ~PermissionInfo.PROTECTION_FLAG_INSTANT;
            }
            // 检查新安装的应用是否定义了已经定义过的权限
            if (bp != null) {
                final boolean sigsOk;
                if (bp.sourcePackage.equals(pkg.packageName)
                        && (bp.packageSetting instanceof PackageSetting)
                        && (shouldCheckUpgradeKeySetLP((PackageSetting) bp.packageSetting,
                                    scanFlags))) {
                    sigsOk = checkUpgradeKeySetLP((PackageSetting) bp.packageSetting, pkg);
                } else {
                    sigsOk = compareSignatures(bp.packageSetting.signatures.mSignatures,
                            pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
                }
                if (!sigsOk) {
                    if (!bp.sourcePackage.equals("android")) {
                        res.origPermission = perm.info.name;
                        res.origPackage = bp.sourcePackage;
                        return;
                    } else {
                        Slog.w(TAG, "Package " + pkg.packageName
                                + " attempting to redeclare system permission "
                                + perm.info.name + "; ignoring new declaration");
                        pkg.permissions.remove(i);
                    }
                } else if (!PLATFORM_PACKAGE_NAME.equals(pkg.packageName)) {
                    // 防止将protection级别的权限改为dangerous级别的权限
                    if ((perm.info.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE)
                            == PermissionInfo.PROTECTION_DANGEROUS) {
                        if (bp != null && !bp.isRuntime()) {
                            perm.info.protectionLevel = bp.protectionLevel;
                        }
                    }
                }
            }
        }
   }

    if (systemApp) {
        // 系统APP不能升级安装到SD卡中
        if (onExternal) {
            // Abort update; system app can't be replaced with app on sdcard
            res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                    "Cannot install updates to system apps on sdcard");
            return;
        // 系统APP不能升级替换为Instant APP
        } else if (instantApp) {
            // Abort update; system app can't be replaced with an instant app
            res.setError(INSTALL_FAILED_INSTANT_APP_INVALID,
                    "Cannot update a system app with an instant app");
            return;
        }
    }
    // 如果是迁移应用
    if (args.move != null) {
        // We did an in-place move, so dex is ready to roll
        scanFlags |= SCAN_NO_DEX;
        scanFlags |= SCAN_MOVE;

        synchronized (mPackages) {
            final PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps == null) {
                res.setError(INSTALL_FAILED_INTERNAL_ERROR,
                        "Missing settings for moved package " + pkgName);
            }
            pkg.applicationInfo.primaryCpuAbi = ps.primaryCpuAbiString;
            pkg.applicationInfo.secondaryCpuAbi = ps.secondaryCpuAbiString;
        }

    } else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
        scanFlags |= SCAN_NO_DEX;

        try {
            String abiOverride = (TextUtils.isEmpty(pkg.cpuAbiOverride) ?
                args.abiOverride : pkg.cpuAbiOverride);
            final boolean extractNativeLibs = !pkg.isLibrary();
            derivePackageAbi(pkg, new File(pkg.codePath), abiOverride,
                    extractNativeLibs, mAppLib32InstallDir);
        } catch (PackageManagerException pme) {
            Slog.e(TAG, "Error deriving application ABI", pme);
            res.setError(INSTALL_FAILED_INTERNAL_ERROR, "Error deriving application ABI");
            return;
        }
        synchronized (mPackages) {
            try {
                updateSharedLibrariesLPr(pkg, null);
            } catch (PackageManagerException e) {
                Slog.e(TAG, "updateAllSharedLibrariesLPw failed: " + e.getMessage());
            }
        }
    }
    // 重命名应用
    // 重命名应用会从原来的临时目录/data/app/xxxxxxxxxx.tmp修改为/data/app/包名-xxx
    //xxx是采用Base64生成的，如/data/app/包名-EaKhmZABPuuiWzijNru_rA==
    if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
        res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
        return;
    }
    try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
            "installPackageLI")) {
    if (replace) {
        if (pkg.applicationInfo.isStaticSharedLibrary()) {
           PackageParser.Package existingPkg = mPackages.get(pkg.packageName);
           if (existingPkg != null && existingPkg.mVersionCode != pkg.mVersionCode) {
                return;
            }
        }
        //如果之前已经安装apk，则替换
        //【3.6.1-(2)】
        replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                installerPackageName, res, args.installReason);
    } else {
        // 首次安装
        //【3.6.1-(3)】
        installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                args.user, installerPackageName, volumeUuid, res, args.installReason);
    }
            
    // 检查是否要进行dex优化
    // 优化的条件是1、没有添加前向索 2、不是安装在外置存储 3、不是Instant APP或者稍后会进行DEX优化
    final boolean performDexopt = (res.returnCode == PackageManager.INSTALL_SUCCEEDED)
            && !forwardLocked
            && !pkg.applicationInfo.isExternalAsec()
            && (!instantApp || Global.getInt(mContext.getContentResolver(),
            Global.INSTANT_APP_DEXOPT_ENABLED, 0) != 0);

    if (performDexopt) {
        DexoptOptions dexoptOptions = new DexoptOptions(pkg.packageName,
                REASON_INSTALL,
                DexoptOptions.DEXOPT_BOOT_COMPLETE);
        //对指定包内的代码和库执行dexopt优化代码
        mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles,
                null /* instructionSets */,
                getOrCreateCompilerPackageStats(pkg),
                mDexManager.getPackageUseInfoOrDefault(pkg.packageName),
                dexoptOptions);
    }
    BackgroundDexOptService.notifyPackageChanged(pkg.packageName);
    synchronized (mPackages) {
        // 下面更新应用程序所属的用户
        final PackageSetting ps = mSettings.mPackages.get(pkgName);
        if (ps != null) {
            res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            ps.setUpdateAvailable(false /*updateAvailable*/);
        }

        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = pkg.childPackages.get(i);
            PackageInstalledInfo childRes = res.addedChildPackages.get(childPkg.packageName);
            PackageSetting childPs = mSettings.getPackageLPr(childPkg.packageName);
            if (childPs != null) {
                childRes.newUsers = childPs.queryInstalledUsers(
                        sUserManager.getUserIds(), true);
            }
        }

        if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
            updateSequenceNumberLP(ps, res.newUsers);
            updateInstantAppInstallerLocked(pkgName);
        }
    }
}
```

该方法比较长，简单梳理下分为以下几部分：

1. 初始化变量
2. 解析APK
3. 判断是新安装还是升级安装，无论是新安装还是升级安装都是需要获取签名信息。
4. 检查权限
5. 重命名应用
6. 根据不同的安装标志，来进行操作，分为三种情况 
   - 移动操作
   - 如果之前已经安装apk，则替换
   - 首次安装
7. 检查是否要进行dex优化
8. 安装收尾，更新应用程序所属的用户

##### (1) 解析APK包#parsePackage

```java
public Package parsePackage(File packageFile, int flags, boolean useCaches)
        throws PackageParserException {
    Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
    if (parsed != null) {
        return parsed;
    }

    long parseTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
    if (packageFile.isDirectory()) {
        // 解析
        parsed = parseClusterPackage(packageFile, flags);
     } else {
        parsed = parseMonolithicPackage(packageFile, flags);
     }

    long cacheTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
    cacheResult(packageFile, flags, parsed);
    return parsed;
}
```

2.5.1小节也有解析包的操作：parseClusterPackageLite和parseMonolithicPackageLite，和这里的parseClusterPackage和parseMonolithicPackage的区别就是前者是轻量级解析，后者是完全解析，接下来看下完全解析的流程。

**(1-1) 解析Cluster APK包#parseClusterPackage**

 一个Cluster apk有一个base apk和其他一些split apk构成，其中这些split apk用一些数字来分割。这些split apk必须都是有效的安装，同时必须满足下面的几个条件：

- 所有的apk必须具有完全相同的软件包名称，版本代码和签名证书
- 所有的apk必须具有唯一的拆分名称
- 所有安装必须包含一个单一的apk



```java
private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
    //轻量级解析Cluster APK包，见3.5.1-(1)的分析
    final PackageLite lite = parseClusterPackageLite(packageDir, 0);
    if (mOnlyCoreApps && !lite.coreApp) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                "Not a coreApp: " + packageDir);
    }

    // Build the split dependency tree.
    // 创建一个数据结构存储split apks的依赖关系
    SparseArray<int[]> splitDependencies = null;
    final SplitAssetLoader assetLoader;
    // 把AssetManager 载入到每个"拆分APK"中
    if (lite.isolatedSplits && !ArrayUtils.isEmpty(lite.splitNames)) {
        try {
            splitDependencies = SplitAssetDependencyLoader.createDependenciesFromPackage(lite);
            assetLoader = new SplitAssetDependencyLoader(lite, splitDependencies, flags);
        } catch (SplitAssetDependencyLoader.IllegalDependencyException e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST, e.getMessage());
        }
    } else {
        assetLoader = new DefaultSplitAssetLoader(lite, flags);
    }

    try {
        // 获取AssetManager对象
        final AssetManager assets = assetLoader.getBaseAssetManager();
        final File baseApk = new File(lite.baseCodePath);
        // 开始解析"base" APK文件，并获得对应的Package 对象
        final Package pkg = parseBaseApk(baseApk, assets, flags);
        if (pkg == null) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                    "Failed to parse base APK: " + baseApk);
        }
        // 开始解析"拆分APK"
        if (!ArrayUtils.isEmpty(lite.splitNames)) {
            final int num = lite.splitNames.length;
            pkg.splitNames = lite.splitNames;
            pkg.splitCodePaths = lite.splitCodePaths;
            pkg.splitRevisionCodes = lite.splitRevisionCodes;
            pkg.splitFlags = new int[num];
            pkg.splitPrivateFlags = new int[num];
            pkg.applicationInfo.splitNames = pkg.splitNames;
            pkg.applicationInfo.splitDependencies = splitDependencies;
            pkg.applicationInfo.splitClassLoaderNames = new String[num];

            for (int i = 0; i < num; i++) {
                final AssetManager splitAssets = assetLoader.getSplitAssetManager(i);
                // 解析单个"拆分APK"
                parseSplitApk(pkg, i, splitAssets, flags);
            }
        }
        // 设置相应属性
        pkg.setCodePath(packageDir.getAbsolutePath());
        pkg.setUse32bitAbi(lite.use32bitAbi);
        return pkg;
    } finally {
        IoUtils.closeQuietly(assetLoader);
    }
}
```

解析cluster类型的apk时会分别调用parseBaseApk和parseSplitApk分别解析其中的base apk和split apk。由于Cluster类型的apk在国内使用不了(需要google play支持)，因此再次略过cluster类型的apk的分析。接下来重点分析普通的单个apk安装包的安装流程。

**(1-2)解析Monolithic类型的APK包#parseMonolithicPackage**

```java
public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
    //创建AssetManager对象，后续用来打开AndroidManifest.xml
    final AssetManager assets = newConfiguredAssetManager();
    //轻量级解析apk，获取该apk文件的简单的信息
    final PackageLite lite = parseMonolithicPackageLite(apkFile, flags);
    if (mOnlyCoreApps) {
        if (!lite.coreApp) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                    "Not a coreApp: " + apkFile);
        }
    }

    try {
        //核心方法，解析base apk，单个"整体式"的apk只有一个apk，即base apk
        final Package pkg = parseBaseApk(apkFile, assets, flags);
        pkg.setCodePath(apkFile.getAbsolutePath());
        pkg.setUse32bitAbi(lite.use32bitAbi);
        return pkg;
    } finally {
        IoUtils.closeQuietly(assets);
    }
}
```

解析单个"整体式"的apk时，先创建一个AssetManager对象，后续用来打开AndroidManifest.xml，然后通过轻量级解析apk安装包，最终调用parseBaseApk进行彻底的解析apk安装包。

**彻底解析APK包#parseBaseApk**

```java
private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
        throws PackageParserException {
    // 获取apk安装包的绝对路径
    final String apkPath = apkFile.getAbsolutePath();

    String volumeUuid = null;
    // 解析volumeUuid
    if (apkPath.startsWith(MNT_EXPAND)) {
        final int end = apkPath.indexOf('/', MNT_EXPAND.length());
        volumeUuid = apkPath.substring(MNT_EXPAND.length(), end);
    }

    mParseError = PackageManager.INSTALL_SUCCEEDED;
    mArchiveSourcePath = apkFile.getAbsolutePath();

    final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

    Resources res = null;
    XmlResourceParser parser = null;
    try {
        res = new Resources(assets, mMetrics, null);
        //打开AndroidManifest.xml文件
        parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

        final String[] outError = new String[1];
        //解析apk
        final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);
        if (pkg == null) {
            throw new PackageParserException(mParseError,
                    apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
        }

        pkg.setVolumeUuid(volumeUuid);
        pkg.setApplicationVolumeUuid(volumeUuid);
        pkg.setBaseCodePath(apkPath);
        pkg.setSignatures(null);

        return pkg;

    } catch (PackageParserException e) {
        throw e;
    } catch (Exception e) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                "Failed to read manifest from " + apkPath, e);
    } finally {
        IoUtils.closeQuietly(parser);
    }
}
```

简单总结下，该方法主要解析volumeUuid，如果apk路径的前置为"/mnt/expand/"，则获取从前缀之后的uuid，从而可以根据这个路径获取文件的路径。然后创建XmlResourceParser对象用于读取AndroidManifest.xml文件。最后调用另一个parseBaseApk方法进行真正的解析apk工作。

```java
private Package parseBaseApk(String apkPath, Resources res, XmlResourceParser parser, int flags,
        String[] outError) throws XmlPullParserException, IOException {
    // 因为是解析"base" apk，所以应该出现"拆包"APK的信息，如果出现，则返回
    final String splitName;
    final String pkgName;

    try {
        //调用parsePackageSplitNames获取packageSplit
        Pair<String, String> packageSplit = parsePackageSplitNames(parser, parser);
        pkgName = packageSplit.first;
        splitName = packageSplit.second;

        if (!TextUtils.isEmpty(splitName)) {
            outError[0] = "Expected base APK, but found split " + splitName;
            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
            return null;
        }
    } catch (PackageParserException e) {
        mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
        return null;
    }

    if (mCallback != null) {
        String[] overlayPaths = mCallback.getOverlayPaths(pkgName, apkPath);
        if (overlayPaths != null && overlayPaths.length > 0) {
            for (String overlayPath : overlayPaths) {
                res.getAssets().addOverlayPath(overlayPath);
            }
        }
    }
    // 用包名构造一个Package
    final Package pkg = new Package(pkgName);
    // 获取资源数组
    TypedArray sa = res.obtainAttributes(parser,
            com.android.internal.R.styleable.AndroidManifest);
    // 初始化pkg的属性mVersionCode、baseRevisionCode和mVersionName
    pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
            com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
    pkg.baseRevisionCode = sa.getInteger(
            com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
    pkg.mVersionName = sa.getNonConfigurationString(
            com.android.internal.R.styleable.AndroidManifest_versionName, 0);
    if (pkg.mVersionName != null) {
        pkg.mVersionName = pkg.mVersionName.intern();
    }
    //获取是否是核心app，普通app不是
    pkg.coreApp = parser.getAttributeBooleanValue(null, "coreApp", false);

    sa.recycle();

    return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
}
```

```java
private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
        XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
        IOException {
    mParseInstrumentationArgs = null;

    int type;
    boolean foundApp = false;
    
    TypedArray sa = res.obtainAttributes(parser,
            com.android.internal.R.styleable.AndroidManifest);
    // 获取sharedUID
    String str = sa.getNonConfigurationString(
            com.android.internal.R.styleable.AndroidManifest_sharedUserId, 0);
    if (str != null && str.length() > 0) {
        //如果是isntant app，则报错，因为isntant app不需要安装
        if ((flags & PARSE_IS_EPHEMERAL) != 0) {
            outError[0] = "sharedUserId not allowed in ephemeral application";
            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID;
            return null;
        }
        String nameError = validateName(str, true, false);
        // 如果不是系统包，即framework-res.apk(它的包名为"android")，则报错
        if (nameError != null && !"android".equals(pkg.packageName)) {
            outError[0] = "<manifest> specifies bad sharedUserId name \""
                + str + "\": " + nameError;
            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID;
            return null;
        }
        pkg.mSharedUserId = str.intern();
        pkg.mSharedUserLabel = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifest_sharedUserLabel, 0);
    }
    // 获取 安装路径的设置
    pkg.installLocation = sa.getInteger(
            com.android.internal.R.styleable.AndroidManifest_installLocation,
            PARSE_DEFAULT_INSTALL_LOCATION);
    pkg.applicationInfo.installLocation = pkg.installLocation;

    final int targetSandboxVersion = sa.getInteger(
            com.android.internal.R.styleable.AndroidManifest_targetSandboxVersion,
            PARSE_DEFAULT_TARGET_SANDBOX);
    pkg.applicationInfo.targetSandboxVersion = targetSandboxVersion;

    /* Set the global "forward lock" flag */
    if ((flags & PARSE_FORWARD_LOCK) != 0) {
        pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_FORWARD_LOCK;
    }
    // 是否要安装在SD卡上
    /* Set the global "on SD card" flag */
    if ((flags & PARSE_EXTERNAL_STORAGE) != 0) {
        pkg.applicationInfo.flags |= ApplicationInfo.FLAG_EXTERNAL_STORAGE;
    }

    if (sa.getBoolean(com.android.internal.R.styleable.AndroidManifest_isolatedSplits, false)) {
        pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_ISOLATED_SPLIT_LOADING;
    }

    // Resource boolean are -1, so 1 means we don't know the value.
    int supportsSmallScreens = 1;
    int supportsNormalScreens = 1;
    int supportsLargeScreens = 1;
    int supportsXLargeScreens = 1;
    int resizeable = 1;
    int anyDensity = 1;

    int outerDepth = parser.getDepth();
    // 开始解析AndroidManifest.xml文件
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
            && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
        if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
            continue;
        }

        String tagName = parser.getName();

        //跳过不支持的标签
        if (acceptedTags != null && !acceptedTags.contains(tagName)) {
            XmlUtils.skipCurrentTag(parser);
            continue;
        }
        // 解析<application> 标签
        if (tagName.equals(TAG_APPLICATION)) {
            ...
            if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                return null;
            }
        // 解析<overlay> 标签
        } else if (tagName.equals(TAG_OVERLAY)) {
            ...
        // 解析<key-sets> 标签
        } else if (tagName.equals(TAG_KEY_SETS)) {
            if (!parseKeySets(pkg, res, parser, outError)) {
                return null;
            }
        // 解析<permission-group> 标签
        } else if (tagName.equals(TAG_PERMISSION_GROUP)) {
            if (!parsePermissionGroup(pkg, flags, res, parser, outError)) {
                return null;
            }
        // 解析<permission> 标签
        } else if (tagName.equals(TAG_PERMISSION)) {
            if (!parsePermission(pkg, res, parser, outError)) {
                return null;
            }
        // 解析<permission-tree> 标签
        } else if (tagName.equals(TAG_PERMISSION_TREE)) {
            if (!parsePermissionTree(pkg, res, parser, outError)) {
                return null;
            }
        // 解析<uses-permission> 标签
        } else if (tagName.equals(TAG_USES_PERMISSION)) {
            if (!parseUsesPermission(pkg, res, parser)) {
                return null;
            }
        // 解析<uses-permission-sdk-m> 标签 或者 <uses-permission-sdk-23> 标签
        } else if (tagName.equals(TAG_USES_PERMISSION_SDK_M)
                || tagName.equals(TAG_USES_PERMISSION_SDK_23)) {
            if (!parseUsesPermission(pkg, res, parser)) {
                return null;
            }
        // 解析<uses-configuration>  标签
        } else if (tagName.equals(TAG_USES_CONFIGURATION)) {
        ...
        // 解析<uses-feature>  标签
        } else if (tagName.equals(TAG_USES_FEATURE)) {
            FeatureInfo fi = parseUsesFeature(res, parser);
            pkg.reqFeatures = ArrayUtils.add(pkg.reqFeatures, fi);
        ...
         // 解析<feature-group>  标签
        } else if (tagName.equals(TAG_FEATURE_GROUP)) {
        ...
        // 解析<uses-sdk>  标签
        } else if (tagName.equals(TAG_USES_SDK)) {
        ...  
        // 解析<supports-screens>  标签
        } else if (tagName.equals(TAG_SUPPORT_SCREENS)) {
        ...
        // 解析<protected-broadcast>  标签
        } else if (tagName.equals(TAG_PROTECTED_BROADCAST)) {
        ...
        // 解析<instrumentation>  标签
        } else if (tagName.equals(TAG_INSTRUMENTATION)) {
        ...
        // 解析<original-package>  标签
        } else if (tagName.equals(TAG_ORIGINAL_PACKAGE)) {
        ...
        // 解析<adopt-permissions>  标签
        } else if (tagName.equals(TAG_ADOPT_PERMISSIONS)) {
        ...
         // 解析<uses-gl-texture>  标签
        } else if (tagName.equals(TAG_USES_GL_TEXTURE)) {
        ...
        // 解析<compatible-screens>  标签
        } else if (tagName.equals(TAG_COMPATIBLE_SCREENS)) {
        ...
        // 解析<supports-input>  标签
        } else if (tagName.equals(TAG_SUPPORTS_INPUT)) {//
        ...
        // 解析<eat-comment>  标签
        } else if (tagName.equals(TAG_EAT_COMMENT)) {
        ...
        // 解析<package>  标签
        } else if (tagName.equals(TAG_PACKAGE)) {
        ...
        // 解析<restrict-update>  标签
        } else if (tagName.equals(TAG_RESTRICT_UPDATE)) {
        ...
        // 解析<uses-split>  标签
        } else if (RIGID_PARSER) {
        ...  
    }

    if (!foundApp && pkg.instrumentation.size() == 0) {
        outError[0] = "<manifest> does not contain an <application> or <instrumentation>";
        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_EMPTY;
    }

    // 权限相关操作
    final int NP = PackageParser.NEW_PERMISSIONS.length;
    StringBuilder implicitPerms = null;
    for (int ip=0; ip<NP; ip++) {
        final PackageParser.NewPermissionInfo npi
                = PackageParser.NEW_PERMISSIONS[ip];
        if (pkg.applicationInfo.targetSdkVersion >= npi.sdkVersion) {
            break;
        }
        if (!pkg.requestedPermissions.contains(npi.name)) {
            if (implicitPerms == null) {
                implicitPerms = new StringBuilder(128);
                implicitPerms.append(pkg.packageName);
                implicitPerms.append(": compat added ");
            } else {
                implicitPerms.append(' ');
            }
            implicitPerms.append(npi.name);
            pkg.requestedPermissions.add(npi.name);
        }
    }
    if (implicitPerms != null) {
        Slog.i(TAG, implicitPerms.toString());
    }

    final int NS = PackageParser.SPLIT_PERMISSIONS.length;
    for (int is=0; is<NS; is++) {
        final PackageParser.SplitPermissionInfo spi
                = PackageParser.SPLIT_PERMISSIONS[is];
        if (pkg.applicationInfo.targetSdkVersion >= spi.targetSdk
                || !pkg.requestedPermissions.contains(spi.rootPerm)) {
            continue;
        }
        for (int in=0; in<spi.newPerms.length; in++) {
            final String perm = spi.newPerms[in];
            if (!pkg.requestedPermissions.contains(perm)) {
                pkg.requestedPermissions.add(perm);
            }
        }
    }
    ...
    return pkg;
}
```

 这个方法非常长，主要用来解析APK的AndroidManifest中的各个 标签，比如application、permission等等，其中四大组件的标签在application标签下，解析application标签的方法为parseBaseApplication。 

**parseBaseApplication**

```java
private boolean parseBaseApplication(Package owner, Resources res,
        XmlResourceParser parser, int flags, String[] outError)
    throws XmlPullParserException, IOException {
    // 获取ApplicationInfo对象ai
    final ApplicationInfo ai = owner.applicationInfo;
    // 获取Application的名字
    final String pkgName = owner.applicationInfo.packageName;
    // 从资源里面获取AndroidManifest的数组
    TypedArray sa = res.obtainAttributes(parser,
            com.android.internal.R.styleable.AndroidManifestApplication);

    //解析apk的名字、icon等信息
    if (!parsePackageItemInfo(owner, ai, outError,
            "<application>", sa, false /*nameRequired*/,
            com.android.internal.R.styleable.AndroidManifestApplication_name,
            com.android.internal.R.styleable.AndroidManifestApplication_label,
            com.android.internal.R.styleable.AndroidManifestApplication_icon,
            com.android.internal.R.styleable.AndroidManifestApplication_roundIcon,
            com.android.internal.R.styleable.AndroidManifestApplication_logo,
            com.android.internal.R.styleable.AndroidManifestApplication_banner)) {
        sa.recycle();
        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
        return false;
    }
    // 如果有设置过Application，则设置ApplicationInfo的类名
    if (ai.name != null) {
        ai.className = ai.name;
    }
    //获取android:manageSpaceActivity属性
    String manageSpaceActivity = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestApplication_manageSpaceActivity,
            Configuration.NATIVE_CONFIG_VERSION);
    if (manageSpaceActivity != null) {
        ai.manageSpaceActivityName = buildClassName(pkgName, manageSpaceActivity,
                outError);
    }
    //获取android:allowBackup属性
    boolean allowBackup = sa.getBoolean(
            com.android.internal.R.styleable.AndroidManifestApplication_allowBackup, true);
    ...
    // 获取<Application> 里面的"theme"属性
    ai.theme = sa.getResourceId(
            com.android.internal.R.styleable.AndroidManifestApplication_theme, 0);
    ...
    // 解析四大组件
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
            && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
        if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
            continue;
        }

        String tagName = parser.getName();
        // 解析activity
        if (tagName.equals("activity")) {
            Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs, false,
                    owner.baseHardwareAccelerated);
            if (a == null) {
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return false;
            }
            owner.activities.add(a);
        // 解析receiver
        } else if (tagName.equals("receiver")) {
            Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs,
                    true, false);
            if (a == null) {
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return false;
                }
            owner.receivers.add(a);
        // 解析service
        } else if (tagName.equals("service")) {
            Service s = parseService(owner, res, parser, flags, outError, cachedArgs);
            if (s == null) {
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return false;
            }
            owner.services.add(s);
        // 解析provider
        } else if (tagName.equals("provider")) {
            Provider p = parseProvider(owner, res, parser, flags, outError, cachedArgs);
            if (p == null) {
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return false;
            }
            owner.providers.add(p);
        } 
        ...
    }
    setMaxAspectRatio(owner);

    PackageBackwardCompatibility.modifySharedLibraries(owner);

    if (hasDomainURLs(owner)) {
        owner.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
    } else {
        owner.applicationInfo.privateFlags &= ~ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
    }

    return true;
}
```

其实这个方法就是解析了application节点下的所有信息，比如activity、service、receiver、provider等信息，同时将解析后的每一个属性生成相应的对象，添加到传入的package里面，这些信息最后都会在PMS中用到。

##### (2) 覆盖安装 APK包#replacePackageLIF

上一小节解析完apk包后，接着分析安装apk的流程 。如果是已安装的apk再次安装，则走replacePackageLIF流程。

```java
private void replacePackageLIF(PackageParser.Package pkg, final int policyFlags, int scanFlags,
        UserHandle user, String installerPackageName, PackageInstalledInfo res,
        int installReason) {
    //是否是instant app
    final boolean isInstantApp = (scanFlags & SCAN_AS_INSTANT_APP) != 0;
    //代表一个apk包对象
    final PackageParser.Package oldPackage;
    final PackageSetting ps;
    final String pkgName = pkg.packageName;
    final int[] allUsers;
    final int[] installedUsers;

    synchronized(mPackages) {
        //旧的apk包
        oldPackage = mPackages.get(pkgName);
        // 不允许从预发行版的SDK升级到目标版本的SDK
        final boolean oldTargetsPreRelease = oldPackage.applicationInfo.targetSdkVersion
                == android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
        final boolean newTargetsPreRelease = pkg.applicationInfo.targetSdkVersion
                == android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
        if (oldTargetsPreRelease
                && !newTargetsPreRelease
                && ((policyFlags & PackageParser.PARSE_FORCE_SDK) == 0)) {
            Slog.w(TAG, "Can't install package targeting released sdk");
            res.setReturnCode(PackageManager.INSTALL_FAILED_UPDATE_INCOMPATIBLE);
            return;
        }
        //获取旧的apk的PackageSetting对象
        ps = mSettings.mPackages.get(pkgName);

        // 验证签名是否有效
        if (shouldCheckUpgradeKeySetLP(ps, scanFlags)) {
            if (!checkUpgradeKeySetLP(ps, pkg)) {
                res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                        "New package not signed by keys specified by upgrade-keysets: "
                                + pkgName);
                return;
            }
        } else {
            // 默认为原始签名匹配
            if (compareSignatures(oldPackage.mSignatures, pkg.mSignatures)
                    != PackageManager.SIGNATURE_MATCH) {
                res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                        "New package has a different signature: " + pkgName);
                return;
            }
        }

        // 除非升级哈希匹配，否则不允许系统升级
        if (oldPackage.restrictUpdateHash != null && oldPackage.isSystemApp()) {
            byte[] digestBytes = null;
            try {
                final MessageDigest digest = MessageDigest.getInstance("SHA-512");
                updateDigest(digest, new File(pkg.baseCodePath));
                if (!ArrayUtils.isEmpty(pkg.splitCodePaths)) {
                    for (String path : pkg.splitCodePaths) {
                        updateDigest(digest, new File(path));
                    }
                }
                digestBytes = digest.digest();
            } catch (NoSuchAlgorithmException | IOException e) {
                res.setError(INSTALL_FAILED_INVALID_APK,
                        "Could not compute hash: " + pkgName);
                return;
            }
            if (!Arrays.equals(oldPackage.restrictUpdateHash, digestBytes)) {
                res.setError(INSTALL_FAILED_INVALID_APK,
                        "New package fails restrict-update check: " + pkgName);
                return;
            }
            // 保留升级限制
            pkg.restrictUpdateHash = oldPackage.restrictUpdateHash;
        }

        // 检查共享用户id是否更改
        String invalidPackageName =
                getParentOrChildPackageChangedSharedUser(oldPackage, pkg);
        if (invalidPackageName != null) {
            res.setError(INSTALL_FAILED_SHARED_USER_INCOMPATIBLE,
                    "Package " + invalidPackageName + " tried to change user "
                            + oldPackage.mSharedUserId);
            return;
        }

        // 如果回滚，请记住每个用户/配置文件的安装状态
        allUsers = sUserManager.getUserIds();
        installedUsers = ps.queryInstalledUsers(allUsers, true);

        // instant升级规则
        if (isInstantApp) {
            if (user == null || user.getIdentifier() == UserHandle.USER_ALL) {
                for (int currentUser : allUsers) {
                    if (!ps.getInstantApp(currentUser)) {
                        // can't downgrade from full to instant
                        Slog.w(TAG, "Can't replace full app with instant app: " + pkgName
                                + " for user: " + currentUser);
                           
                    res.setReturnCode(PackageManager.INSTALL_FAILED_INSTANT_APP_INVALID);
                        return;
                    }
                }
            } else if (!ps.getInstantApp(user.getIdentifier())) {
                // can't downgrade from full to instant
                Slog.w(TAG, "Can't replace full app with instant app: " + pkgName
                        + " for user: " + user.getIdentifier());
                res.setReturnCode(PackageManager.INSTALL_FAILED_INSTANT_APP_INVALID);
                return;
            }
        }
    }

    // 更新删除的内容
    ...
    //如果是系统app
    boolean sysPkg = (isSystemApp(oldPackage));
    if (sysPkg) {
        // 根据需要设置系统/特权flags
        final boolean privileged =
                (oldPackage.applicationInfo.privateFlags
                        & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
        final int systemPolicyFlags = policyFlags
                | PackageParser.PARSE_IS_SYSTEM
                | (privileged ? PackageParser.PARSE_IS_PRIVILEGED : 0);
        // 替换系统app
        replaceSystemPackageLIF(oldPackage, pkg, systemPolicyFlags, scanFlags,
                user, allUsers, installerPackageName, res, installReason);
    } else {
         // 替换非系统app
        replaceNonSystemPackageLIF(oldPackage, pkg, policyFlags, scanFlags,
                user, allUsers, installerPackageName, res, installReason);
    }
}
```

该方法最后根据是否是系统apk，分别调用replaceSystemPackageLIF和replaceNonSystemPackageLIF。

**覆盖安装系统APK#replaceSystemPackageLIF**

```java
private void replaceSystemPackageLIF(PackageParser.Package deletedPackage,
        PackageParser.Package pkg, final int policyFlags, int scanFlags, UserHandle user,
        int[] allUsers, String installerPackageName, PackageInstalledInfo res,
        int installReason) {

    final boolean disabledSystem;

    // 删除旧的系统package信息
    removePackageLI(deletedPackage, true);

    synchronized (mPackages) {
        //是否需要禁用旧包
        disabledSystem = disableSystemPackageLPw(deletedPackage, pkg);
    }
    //如果不需要禁用，需要确保删除就的.apk文件
    if (!disabledSystem) {
        res.removedInfo.args = createInstallArgsForExisting(0,
                deletedPackage.applicationInfo.getCodePath(),
                deletedPackage.applicationInfo.getResourcePath(),
                getAppDexInstructionSets(deletedPackage.applicationInfo));
    } else {
        //禁用则将变量设为空
        res.removedInfo.args = null;
    }

    // 已成功禁用旧包。现在继续重新安装
    //清除旧包的app数据
    clearAppDataLIF(pkg, UserHandle.USER_ALL, StorageManager.FLAG_STORAGE_DE
            | StorageManager.FLAG_STORAGE_CE | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
    clearAppProfilesLIF(deletedPackage, UserHandle.USER_ALL);

    res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
    pkg.setApplicationInfoFlags(ApplicationInfo.FLAG_UPDATED_SYSTEM_APP,
            ApplicationInfo.FLAG_UPDATED_SYSTEM_APP);

    PackageParser.Package newPackage = null;
    try {
        // 扫描新的安装包
        newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags, 0, user);

        ...
        // 将新的安装包信息更新到mSettings中，mSettings持有系统上所有安装的apk信息
         updateSettingsLI(newPackage, installerPackageName, allUsers, res, user,
                 installReason);
        // 安装后创建相关的数据目录
         prepareAppDataAfterInstallLIF(newPackage);
         mDexManager.notifyPackageUpdated(newPackage.packageName,
                     newPackage.baseCodePath, newPackage.splitCodePaths);
        }
    } catch (PackageManagerException e) {
        res.setReturnCode(INSTALL_FAILED_INTERNAL_ERROR);
        res.setError("Package couldn't be installed in " + pkg.codePath, e);
    }
    // 安装失败
    if (res.returnCode != PackageManager.INSTALL_SUCCEEDED) {
        // 还原旧信息删除新包信息
        if (newPackage != null) {
            removeInstalledPackageLI(newPackage, true);
        }
        // 还原旧包信息
        try {
            scanPackageTracedLI(deletedPackage, policyFlags, SCAN_UPDATE_SIGNATURE, 0, user);
        } catch (PackageManagerException e) {
            Slog.e(TAG, "Failed to restore original package: " + e.getMessage());
        }

        synchronized (mPackages) {
            if (disabledSystem) {
                enableSystemPackageLPw(deletedPackage);
            }

            // Ensure the installer package name up to date
            setInstallerPackageNameLPw(deletedPackage, installerPackageName);

            // Update permissions for restored package
            updatePermissionsLPw(deletedPackage, UPDATE_PERMISSIONS_ALL);

            mSettings.writeLPr();
        }
    }
}
```

覆盖安装系统apk主要步骤如下：

1. 删除旧的系统package信息
2. 清除旧包的app数据
3. 扫描新的安装包
4. 将新的安装包信息更新到mSettings中，mSettings持有系统上所有安装的apk信息
5. 如果安装成功则创建相关的数据目录；如果安装失败，还原旧信息删除新包信息

##### (3) 新安装 APK包#installNewPackageLIF

​    回到3.6.1中继续分析如果是首次安装的apk流程。

```java
private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
        int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
        PackageInstalledInfo res, int installReason) {

    // Remember this for later, in case we need to rollback this install
    String pkgName = pkg.packageName;

    synchronized(mPackages) {
        // 如果要安装的apk包已安装，但想安装的apk已修改了包名，系统会禁止这种情况进行apk安装
        // 需要将旧的apk卸载才能安装新的apk
        final String renamedPackage = mSettings.getRenamedPackageLPr(pkgName);
        if (renamedPackage != null) {
            res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                    + " without first uninstalling package running as "
                    + renamedPackage);
            return;
        }
        //如果要安装的apk已安装，需要先卸载旧的apk才能安装新的apk
        if (mPackages.containsKey(pkgName)) {
            // Don't allow installation over an existing package with the same name.
            res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                    + " without first uninstalling.");
            return;
        }
    }

    try {
        // 扫描apk安装包构建一个PackageParser.Package对象
        PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                System.currentTimeMillis(), user);

        // 更新安装包的信息到数据结构Settings中，更新权限信息与安装完成信息
        updateSettingsLI(newPackage, installerPackageName, null, res, user, installReason);
        // 如果安装成功，则创建相关的目录等准备工作
        if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
            prepareAppDataAfterInstallLIF(newPackage);

        } else {
            // Remove package from internal structures, but keep around any
            // data that might have already existed
            // 安装失败后将安装包信息从内部的数据结构(即缓存)中移除
            deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                    PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
        }
    } catch (PackageManagerException e) {
        res.setError("Package couldn't be installed in " + pkg.codePath, e);
    } 
}
```

安装APK包的流程概括如下：

1. 如果要安装的apk已安装，需要先卸载旧的apk才能安装新的apk
2. 扫描apk安装包构建一个PackageParser.Package对象
3. 更新安装包的信息到数据结构Settings中，更新权限信息与安装完成信息
4. 如果安装成功，则创建相关的目录等准备工作，如创建数据目录/data/data/packageName
5. 安装失败后将安装包信息从内部的数据结构(即缓存)中移除

**scanPackageTracedLI**

scanPackageTracedLI没做实际性的工作，调用scanPackageLI进行扫描。

```java
private PackageParser.Package scanPackageLI(PackageParser.Package pkg, final int policyFlags,
        int scanFlags, long currentTime, @Nullable UserHandle user)
                throws PackageManagerException {
    boolean success = false;
    try {
        // 解析包目录生成package实例
        final PackageParser.Package res = scanPackageDirtyLI(pkg, policyFlags, scanFlags,
                currentTime, user);
        success = true;
        return res;
    } finally {
        if (!success && (scanFlags & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
            // DELETE_DATA_ON_FAILURES is only used by frozen paths
            destroyAppDataLIF(pkg, UserHandle.USER_ALL,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
            destroyAppProfilesLIF(pkg, UserHandle.USER_ALL);
        }
    }
}
```

scanPackageLI调用scanPackageDirtyLI生成package实例。

```java
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
        final int policyFlags, final int scanFlags, long currentTime, @Nullable UserHandle user)
                throws PackageManagerException {
        // 检查根据给定的策略分析的包是否有效
        assertPackageIsValid(pkg, policyFlags, scanFlags);
        // 初始化包和资源目录
        final File scanFile = new File(pkg.codePath);
        final File destCodeFile = new File(pkg.applicationInfo.getCodePath());
        final File destResourceFile = new File(pkg.applicationInfo.getResourcePath());

        SharedUserSetting suid = null;
        PackageSetting pkgSetting = null;

        PackageSetting nonMutatedPs = null;

        String primaryCpuAbiFromSettings = null;
        String secondaryCpuAbiFromSettings = null;

        final PackageParser.Package oldPkg;

        synchronized (mPackages) {
            if (pkg.mSharedUserId != null) {
                
        // 如果这个包原来是使用原始的名字，后面变更为新的名字，所以我们只需要更新到新的名字
        PackageSetting origPackage = null;
        String realName = null;
        if (pkg.mOriginalPackages != null) {
            final String renamed = mSettings.getRenamedPackageLPr(pkg.mRealPackage);
            if (pkg.mOriginalPackages.contains(renamed)) {
                realName = pkg.mRealPackage;
                if (!pkg.packageName.equals(renamed)) {
                    pkg.setPackageName(renamed);
                }
            } else {
                for (int i=pkg.mOriginalPackages.size()-1; i>=0; i--) {
                    if ((origPackage = mSettings.getPackageLPr(
                        pkg.mOriginalPackages.get(i))) != null) {
                        // 判断pkg的原始包的某一个是否出现在mSettings里面，如果出现过，则说明之前有过安装
                        // 包名非空验证
                        if (!verifyPackageUpdateLPr(origPackage, pkg)) {
                            origPackage = null;
                            continue;
                        } else if (origPackage.sharedUser != null) {
                            // 确保uid一致，如果uid不一致，则跳过
                            if (!origPackage.sharedUser.name.equals(pkg.mSharedUserId)) {
                                origPackage = null;
                                continue;
                            }
                        } else {
                        }
                        break;
                    }
                }
                }
        }
        // 首次启动或升级的情况
        if ((scanFlags & SCAN_FIRST_BOOT_OR_UPGRADE) == 0) {
            PackageSetting foundPs = mSettings.getPackageLPr(pkg.packageName);
            if (foundPs != null) {
                primaryCpuAbiFromSettings = foundPs.primaryCpuAbiString;
                secondaryCpuAbiFromSettings = foundPs.secondaryCpuAbiString;
            }
        }
        // 获取PackageSetting实例，如果存在，说明之前安装过
        pkgSetting = mSettings.getPackageLPr(pkg.packageName);
        if (pkgSetting != null && pkgSetting.sharedUser != suid) {
            pkgSetting = null;
        }
        // 检查此包是否安装过
        final PackageSetting oldPkgSetting =
                pkgSetting == null ? null : new PackageSetting(pkgSetting);
        final PackageSetting disabledPkgSetting =
                mSettings.getDisabledSystemPkgLPr(pkg.packageName);

        if (oldPkgSetting == null) {
            oldPkg = null;
        } else {
            oldPkg = oldPkgSetting.pkg;
        }

        String[] usesStaticLibraries = null;
        if (pkg.usesStaticLibraries != null) {
            usesStaticLibraries = new String[pkg.usesStaticLibraries.size()];
                pkg.usesStaticLibraries.toArray(usesStaticLibraries);
        }

        // 第一次安装此包的情况
        if (pkgSetting == null) {
            final String parentPackageName = (pkg.parentPackage != null)
                    ? pkg.parentPackage.packageName : null;
            final boolean instantApp = (scanFlags & SCAN_AS_INSTANT_APP) != 0;
            final boolean virtualPreload = (scanFlags & SCAN_AS_VIRTUAL_PRELOAD) != 0;
            // 根据包名等信息创建PackageSetting实例
            pkgSetting = Settings.createNewSetting(pkg.packageName, origPackage,
                    disabledPkgSetting, realName, suid, destCodeFile, destResourceFile,
                    pkg.applicationInfo.nativeLibraryRootDir, pkg.applicationInfo.primaryCpuAbi,
                    pkg.applicationInfo.secondaryCpuAbi, pkg.mVersionCode,
                    pkg.applicationInfo.flags, pkg.applicationInfo.privateFlags, user,
                    true /*allowInstall*/, instantApp, virtualPreload,
                    parentPackageName, pkg.getChildPackageNames(),
                    UserManagerService.getInstance(), usesStaticLibraries,
                    pkg.usesStaticLibrariesVersions);
            // 升级的情况
            if (origPackage != null) {
                mSettings.addRenamedPackageLPw(pkg.packageName, origPackage.name);
            }
            // 向系统注册用户ID，可能会分配新的用户ID
            mSettings.addUserToSettingLPw(pkgSetting);
        } else {
            // 升级的情况，更新PackageSetting实例
            Settings.updatePackageSetting(pkgSetting, disabledPkgSetting, suid, destCodeFile,
                    pkg.applicationInfo.nativeLibraryDir, pkg.applicationInfo.primaryCpuAbi,
                    pkg.applicationInfo.secondaryCpuAbi, pkg.applicationInfo.flags,
                    pkg.applicationInfo.privateFlags, pkg.getChildPackageNames(),
                    UserManagerService.getInstance(), usesStaticLibraries,
                    pkg.usesStaticLibrariesVersions);
        }
        mSettings.writeUserRestrictionsLPw(pkgSetting, oldPkgSetting);
        if (pkgSetting.origPackage != null) {
            pkg.setPackageName(origPackage.name);
            if ((scanFlags & SCAN_CHECK_ONLY) == 0) {
                mTransferedPackages.add(origPackage.name);
            }
            pkgSetting.origPackage = null;
        }
        // 开机扫描非系统目录
        if ((scanFlags & SCAN_BOOTING) == 0
                && (policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
            //检查所有共享库并映射到其具体的文件路径
            updateSharedLibrariesLPr(pkg, null);
        }
        // 如果找到用于seinfo标签的mac_permissions.xml
        // 根据policy文件，找到Package对应的seinfo，然后存入Package的applicationInfo中
        if (mFoundPolicyFile) {
            SELinuxMMAC.assignSeInfoValue(pkg);
        }
        // 处理Package的签名信息，还包括更新和验证
        pkg.applicationInfo.uid = pkgSetting.appId;
        // 将pkgSetting保存到pkg.mExtras中
        pkg.mExtras = pkgSetting;

        PackageSetting signatureCheckPs = pkgSetting;
        if (pkg.applicationInfo.isStaticSharedLibrary()) {
            SharedLibraryEntry libraryEntry = getLatestSharedLibraVersionLPr(pkg);
            if (libraryEntry != null) {
                signatureCheckPs = mSettings.getPackageLPr(libraryEntry.apk);
            }
        }
        //进行密钥检查，是否一致
        if (shouldCheckUpgradeKeySetLP(signatureCheckPs, scanFlags)) {
            if (checkUpgradeKeySetLP(signatureCheckPs, pkg)) {
                // 检查签名是否正确
                pkgSetting.signatures.mSignatures = pkg.mSignatures;
            } else {
                // 签名错误
                if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                    throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                            "Package " + pkg.packageName + " upgrade keys do not match the "
                            + "previously installed version");
                } else {
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                }
            }
        } else {
            // 如果检查升级签名错误
            try {
                // 重新验证签名
                verifySignaturesLP(signatureCheckPs, pkg);
                pkgSetting.signatures.mSignatures = pkg.mSignatures;
            } catch (PackageManagerException e) {
                // 验证签名错误
                if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                    throw e;
                }
                pkgSetting.signatures.mSignatures = pkg.mSignatures;
                if (signatureCheckPs.sharedUser != null) {
                    if (compareSignatures(signatureCheckPs.sharedUser.signatures.mSignatures,
                            pkg.mSignatures) != PackageManager.SIGNATURE_MATCH) {
                        throw new PackageManagerException(
                                INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES,
                                "Signature mismatch for shared user: "
                                        + pkgSetting.sharedUser);
                    }
                }
            }
        }

    // 非新安装情况
    if ((scanFlags & SCAN_NEW_INSTALL) == 0) {
        // 第一次开机或者升级的情况，处理so库策略
        if ((scanFlags & SCAN_FIRST_BOOT_OR_UPGRADE) != 0) {
            final boolean extractNativeLibs = !pkg.isLibrary();
            // 在/data/data/packageName/lib下建立和CPU类型对应的目录，例如ARM平台 arm，MIP平台 mips/
            derivePackageAbi(pkg, scanFile, cpuAbiOverride, extractNativeLibs,
                    mAppLib32InstallDir);
            // 如果是系统APP，系统APP的native库统一放到/system/lib下
            // 所以系统不会提取系统APP目录apk包中native库
            if (isSystemApp(pkg) && !pkg.isUpdatedSystemApp() &&
                    pkg.applicationInfo.primaryCpuAbi == null) {
                setBundledAppAbisAndRoots(pkg, pkgSetting);
                setNativeLibraryPaths(pkg, mAppLib32InstallDir);
            }
        } else {
            pkg.applicationInfo.primaryCpuAbi = primaryCpuAbiFromSettings;
            pkg.applicationInfo.secondaryCpuAbi = secondaryCpuAbiFromSettings;

            setNativeLibraryPaths(pkg, mAppLib32InstallDir);
        }
    } else {
        // apk迁移情况
        if ((scanFlags & SCAN_MOVE) != 0) {
            pkg.applicationInfo.primaryCpuAbi = pkgSetting.primaryCpuAbiString;
            pkg.applicationInfo.secondaryCpuAbi = pkgSetting.secondaryCpuAbiString;
        }
        // 设置native 库的路径
        setNativeLibraryPaths(pkg, mAppLib32InstallDir);
    }

    if (mPlatformPackage == pkg) {
        pkg.applicationInfo.primaryCpuAbi = VMRuntime.getRuntime().is64Bit() ?
                Build.SUPPORTED_64_BIT_ABIS[0] : Build.SUPPORTED_32_BIT_ABIS[0];
    } 

    pkgSetting.primaryCpuAbiString = pkg.applicationInfo.primaryCpuAbi;
    pkgSetting.secondaryCpuAbiString = pkg.applicationInfo.secondaryCpuAbi;
    pkgSetting.cpuAbiOverrideString = cpuAbiOverride;

    pkg.cpuAbiOverride = cpuAbiOverride;

    pkgSetting.legacyNativeLibraryPathString = pkg.applicationInfo.nativeLibraryRootDir;

    if ((scanFlags & SCAN_BOOTING) == 0 && pkgSetting.sharedUser != null) {
        // 调整共享用户的abi
        adjustCpuAbisForSharedUserLPw(pkgSetting.sharedUser.packages, pkg);
    }

    if (isSystemApp(pkg)) {
        pkgSetting.isOrphaned = true;
    }
    if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
        if (nonMutatedPs != null) {
            synchronized (mPackages) {
                mSettings.mPackages.put(nonMutatedPs.name, nonMutatedPs);
            }
        }
    } else {
        final int userId = user == null ? 0 : user.getIdentifier();
        // Modify state for the given package setting
       commitPackageSettings(pkg, pkgSetting, user, scanFlags,
                (policyFlags & PackageParser.PARSE_CHATTY) != 0 /*chatty*/);
        if (pkgSetting.getInstantApp(userId)) {
            mInstantAppRegistry.addInstantAppLPw(userId, pkgSetting.appId);
        }
    }

    if (oldPkg != null) {
        final ArrayList<String> allPackageNames = new ArrayList<>(mPackages.keySet());

        AsyncTask.execute(new Runnable() {
            public void run() {
                revokeRuntimePermissionsIfGroupChanged(pkg, oldPkg, allPackageNames);
            }
        });
    }
    return pkg;
}
```

该方法首先调用assertPackageIsValid检查该包是否有效，以下情况视为无效的安装包：

- 代码和资源路径设置不正确
- 特殊的“android”包已经安装，再次安装则失败
- 包名称必须唯一；不允许重复，重复的包名安装失败
- 只有特权应用和更新的特权应用才能添加子包
- 如果我们只安装已经存在的包，因为是已经存在apk，所以可以通过PackageSetting获取它的路径，如果路径不一致，则抛异常
- 内容提供提供器和已安装的apk的内容提供器重名
- 静态库的相关策略

scanPackageDirtyLI的最后会调用commitPackageSettings将扫描的包添加到系统中，此方法完成后，包将可用于查询、解析等。

```java
private void commitPackageSettings(PackageParser.Package pkg, PackageSetting pkgSetting,
        UserHandle user, int scanFlags, boolean chatty) throws PackageManagerException {
    final String pkgName = pkg.packageName;
    //mCustomResolverComponentName是从系统资源中读出的，可以配置
    if (mCustomResolverComponentName != null &&
            mCustomResolverComponentName.getPackageName().equals(pkg.packageName)) {
        // 这里的用途和下面判断packageName是否为"android"有联系
        // 因为调用setUpCustomResolverActivity(pkg)后mResolverReplaced为true。
        setUpCustomResolverActivity(pkg);
    }

    // 针对包名为"android" 的APK进行处理，即framework-res.apk
    // 这个APK里面有两个常用的Acitvity：ChooserActivity(当多个Activity符合某个Intent的时候，系统会弹出的Activity)
    // ShutdownActivity(长按电源键关机时，弹出的Activity)
    if (pkg.packageName.equals("android")) {
        synchronized (mPackages) {
            ...
        }
    }

    ArrayList<PackageParser.Package> clientLibPkgs = null;
    synchronized (mPackages) {
       boolean hasStaticSharedLibs = false;
        // 任何应用程序都可以添加新的静态共享库
        if (pkg.staticSharedLibName != null) {
            // 静态共享库不允许重命名
            if (addSharedLibraryLPw(null, pkg.packageName, pkg.staticSharedLibName,
                    pkg.staticSharedLibVersion, SharedLibraryInfo.TYPE_STATIC,
                    pkg.manifestPackageName, pkg.mVersionCode)) {
                hasStaticSharedLibs = true;
            } else {
                Slog.w(TAG, "Package " + pkg.packageName + " library "
                            + pkg.staticSharedLibName + " already exists; skipping");
            }
        }
        ...
    }
    synchronized (mPackages) {
        //将新的PackageSetting实例添加到mSettings
        mSettings.insertPackageSettingLPw(pkgSetting, pkg);
        // 将pkg保存到PackageManagerService的成员变量mPackages中，key为包名
        mPackages.put(pkg.applicationInfo.packageName, pkg);
        // 确保我们不会意外删除其数据
        //清理空间,删除已经卸载的但还占用存储空间的软件
        final Iterator<PackageCleanItem> iter = mSettings.mPackagesToBeCleaned.iterator();
        while (iter.hasNext()) {
            PackageCleanItem item = iter.next();
            if (pkgName.equals(item.packageName)) {
                iter.remove();
            }
        }
        // 将包的密钥集添加到KeySetManagerService
        KeySetManagerService ksms = mSettings.mKeySetManagerService;
        // 添加安装包的到全局的KeySetManagerService里面
        ksms.addScannedPackageLPw(pkg);
        // 在此之前，四大组件的信息都是Package对象的私有变量，通过下面的代码，将他们注册到PackageManagerService里面，
        // 这样PackageManagerService就有了所有的组件信息

        // 注册pkg里面的provider到PackageManagerService上的mProvider上
        ...
        // 注册该Package中的service到PackageManagerService的mServices上
        ...
        // 注册pkg里面的receiver到PackageManagerService上的receivers上
        ...
        // 注册pkg里面的activity到PackageManagerService上的activities上
        ...
        // 注册pkg里面的Permission到PackageManagerService上的mPermissionGroups上
        ...
        // 注册pkg里面的Permission到PackageManagerService上的mPermission上
        ...
        //  注册pkg里面的instrumentation到PackageManagerService的mInstrumentation中
        // Instrumentation用来跟踪本应用内的application及activity的生命周期
        ...
        // 如果有包内广播
        ...
    }
}
```

scanPackageDirtyLI方法主要的流程如下：

1. 检查待扫描的apk包是否有效
2. 检查是否需要重命名
3. 检测所有共享库：并且映射到真实的路径
4. 如果是升级更新安装，则检查升级更新包的签名，如果是新安装，则验证签名
5. 设置Native Library的路径,即so文件目录
6. 如果该包已经存在了，需要杀死该进程
7. 将一个安装包的内容从pkg里面映射到PackageManagerService里面，包括四大组件等信息

#### 3.6.2 安装后的工作#POST_INSTALL

回到3.6小节，processPendingInstall调用installPackageTracedLI执行完安装操作后，最后发送POST_INSTALL广播。

```java
case POST_INSTALL: {
    PostInstallData data = mRunningInstalls.get(msg.arg1);
    final boolean didRestore = (msg.arg2 != 0);
    // 因为已经安装成功了，所以在 正在安装列表中删除了这个选项
    mRunningInstalls.delete(msg.arg1);
    if (data != null) {
        InstallArgs args = data.args;
        PackageInstalledInfo parentRes = data.res;
        final boolean grantPermissions = (args.installFlags
                & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0;
        final boolean killApp = (args.installFlags
                & PackageManager.INSTALL_DONT_KILL_APP) == 0;
        final boolean virtualPreload = ((args.installFlags
                & PackageManager.INSTALL_VIRTUAL_PRELOAD) != 0);
        final String[] grantedPermissions = args.installGrantPermissions;
        
        handlePackagePostInstall(parentRes, grantPermissions, killApp,
                virtualPreload, grantedPermissions, didRestore,
                args.installerPackageName, args.observer);
        final int childCount = (parentRes.addedChildPackages != null)
                ? parentRes.addedChildPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageInstalledInfo childRes = parentRes.addedChildPackages.valueAt(i);
            handlePackagePostInstall(childRes, grantPermissions, killApp,
                virtualPreload, grantedPermissions, false /*didRestore*/,
                args.installerPackageName, args.observer);
        }
} 
break;
```

1. 先将安装信息从安装列列表中移除，这个也是前面在processPendingInstall中添加的
2. 安装成功后，获取运行时权限
3. 获取权限后，发送ACTION_PACKAGE_ADDED广播，通知Laucher
4. 如果是升级更新则在发送两条广播 
   - ACTION_PACKAGE_REPLACED：一个新版本的应用安装到设备上，替换换之前已经存在的版本
   - ACTION_MY_PACKAGE_REPLACED：应用的新版本替换旧版本被安装，只发给被更新的应用自己
5. 如果安装包中设置了PRIVATE_FLAG_FORWARD_LOCK或者被要求安装在SD卡上，则调用sendResourcesChangedBroadcast方法来发送一个资源更改的广播
6. 强制调用gc，出发JVM进行垃圾回收操作
7. 删除旧的安装信息
8. 回调onPackageInstalled方法。告诉PackageInstaller安装结果。

## 4 总结

简单来说，第三方安装apk的流程如下：

1. 复制apk安装包到/data/app目录下，解压并扫描安装包
2. 向资源管理器注入apk资源，解析AndroidManifest文件，并在/data/data目录下创建对应的应用数据目录
3. 针对dalvik/art环境优化dex文件，保存到dalvik-cache目录
4. 将AndroidManifest文件解析出的组件、权限注册到PackageManagerService
5. 完成安装后发送广播通知Laucher和PackageInstaller

详细的流程图如下：

![third-app-install-apk](images/pms/pms/third-app-install-apk.png)

