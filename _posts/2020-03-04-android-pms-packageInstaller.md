---
1layout: post
title:  "PackageInstaller安装APK流程"
date:   2020-03-04 00:00:00
catalog:  true
tags:
    - Android
    - PMS
---



> /packages/app/PackageInstaller
>
> /frameworks/base/core/java/android/content/pm

## 1. APK安装概述

 Android应用最终是打包成.apk格式(其实就是一个压缩包)，然后安装至手机并运行的。其中APK是Android Package的缩写。 安装APK的方式有以下几种：

| 安装方式                  | 触发时机                        | 面向的用户类型  |
| :------------------------ | :------------------------------ | :-------------- |
| 系统自带和厂商预装        | 系统首次启动                    | System Designer |
| adb命令安装               | 使用adb push命令                | Programmer      |
| adb命令安装               | 使用adb (shell pm) install命令  | Programmer      |
| 点击应用包安装            | 点击后安装                      | User            |
| 第三方应用安装(应用宝等） | 调用PackageInstaller.apk安装apk | User            |

本文从用户点击apk文件开始分析，即调用PackageInstaller安装apk，最终调用PMS的相关接口进行最终的apk安装工作。本文基于Android8.1的源码进行分析。

## 2 安装APK的入口

其他APK想要安装APK，只需要通过调用PackageInstaller来安装即可，如市面上的应用宝等第三方下载商店。手动点击APK文件进行安装是文件管理器应用调用PackageInstaller进行安装的。 PackagInstaller是安卓上默认的应用程序，用它来安装普通文件。PackageInstaller提供了用户界面来管理应用或者包文件。 其他APK可以通过下面的代码进行调用PackageInstaller进行安装APK:

Android 7.0以前的版本，可以通过下面的方式来调用PackageInstaller进行程序的安装：

```java
Intent intent = new Intent(Intent.ACTION_VIEW);  
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);  
intent.setDataAndType(Uri.parse("file://xxxx.apk"),"application/vnd.android.package-archive");  
context.startActivity(intent);  
```

Android 7.0及以上的版本可以使用如下代码调用PackageInstaller进行APK的安装：

```java
Intent intent = new Intent(Intent.ACTION_VIEW);  
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);  
if (Build.VERSION.SDK_INT >= 24) {  
    //参数1上下文；参数2 Provider主机地址 authorities 和配置文件中保持一致 ；参数3共享的文件  
    Uri apkUri = FileProvider.getUriForFile(paramContext, "com.xxx.xxx.xxx", file);  
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);  
    intent.setDataAndType(apkUri, "application/vnd.android.package-archive");  
} else {  
    intent.setDataAndType(Uri.fromFile(file), "application/vnd.android.package-archive");  
}
```

这两段代码的共同之处就是：1）设置Intent的action为ACTION_VIEW；2）设置Intent TYPE为`application/vnd.android.package-archive`； 3）隐式启动activity

接下来我们查看PackageInstaller的AndroidManifest.xml看是否有Activity能够对得上上面的条件。

```xml
<activity android:name=".InstallStart"
                android:exported="true"
                android:excludeFromRecents="true">
            <intent-filter android:priority="1">
                <action android:name="android.intent.action.VIEW" />
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:scheme="content" />
                <data android:mimeType="application/vnd.android.package-archive" />
            </intent-filter>
```

可以看到InstallStart匹配上面的action和mimeType，也就是说，上面的代码会隐式启动InstallStart这个activity，接下来以这种方式的调用安装APK分析PackageInstaller安装APK的流程。首先进入InstallStart分析。

## 3 APK安装前的准备工作

### 3.1 InstallStart类

```java
public class InstallStart extends Activity {  
@Override  
    protected void onCreate(@Nullable Bundle savedInstanceState) {  
        mIPackageManager = AppGlobals.getPackageManager();//获取PMS  
        Intent intent = getIntent();  
        String callingPackage = getCallingPackage();//获取APK安装的发起者包名  
  
        // If the activity was started via a PackageInstaller session, we retrieve the calling  
        // package from that session  
        // 获取sessionID，在前面启动Intent前并没有设置sessionID这里跳过  
        int sessionId = intent.getIntExtra(PackageInstaller.EXTRA_SESSION_ID, -1);  
        if (callingPackage == null && sessionId != -1) {  
            PackageInstaller packageInstaller = getPackageManager().getPackageInstaller();  
            PackageInstaller.SessionInfo sessionInfo = packageInstaller.getSessionInfo(sessionId);  
            callingPackage = (sessionInfo != null) ? sessionInfo.getInstallerPackageName() : null;  
        }  
        // 获取调用PackageInstaller的应用的信息  
        final ApplicationInfo sourceInfo = getSourceInfo(callingPackage);  
        final int originatingUid = getOriginatingUid(sourceInfo);  
        boolean isTrustedSource = false;  
        // 判断调用PackageInstaller的应用是否包含PRIVATE_FLAG_PRIVILEGED，普通app没有该flag【3.2】  
        if (sourceInfo != null  
                && (sourceInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0) {  
            isTrustedSource = intent.getBooleanExtra(Intent.EXTRA_NOT_UNKNOWN_SOURCE, false);  
        }  
          
        if (!isTrustedSource && originatingUid != PackageInstaller.SessionParams.UID_UNKNOWN) {  
            final int targetSdkVersion = getMaxTargetSdkVersionForUid(originatingUid);  
            if (targetSdkVersion < 0) {  
                Log.w(LOG_TAG, "Cannot get target sdk version for uid " + originatingUid);  
                // Invalid originating uid supplied. Abort install.  
                mAbortInstall = true;  
            // Android O及以上版本需要申明REQUEST_INSTALL_PACKAGES权限  
            } else if (targetSdkVersion >= Build.VERSION_CODES.O && !declaresAppOpPermission(  
                    originatingUid, Manifest.permission.REQUEST_INSTALL_PACKAGES)) {  
                Log.e(LOG_TAG, "Requesting uid " + originatingUid + " needs to declare permission "  
                        + Manifest.permission.REQUEST_INSTALL_PACKAGES);  
                mAbortInstall = true;  
            }  
        }  
        //没申明REQUEST_INSTALL_PACKAGES权限则退出安装流程  
        if (mAbortInstall) {  
            setResult(RESULT_CANCELED);  
            finish();  
            return;  
        }  
        ...  
        Intent nextActivity = new Intent(intent);  
        nextActivity.setFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);  
        //将发起安装请求的activity信息存起来，转发到下一个activity继续安装  
        nextActivity.putExtra(PackageInstallerActivity.EXTRA_CALLING_PACKAGE, callingPackage);  
        nextActivity.putExtra(PackageInstallerActivity.EXTRA_ORIGINAL_SOURCE_INFO, sourceInfo);  
        nextActivity.putExtra(Intent.EXTRA_ORIGINATING_UID, originatingUid);  
        // intent的action为ACTION_VIEW  
        if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {  
            nextActivity.setClass(this, PackageInstallerActivity.class);  
        } else {  
            //获取发起安装者的Uri  
            Uri packageUri = intent.getData();  
            //APK安装发起者通过FileProvider来处理URI的时候会将路径转换为“content://Uri”  
            if (packageUri != null && (packageUri.getScheme().equals(ContentResolver.SCHEME_FILE)  
                    || packageUri.getScheme().equals(ContentResolver.SCHEME_CONTENT))) {  
                //进入该分支，接下来是启动InstallStaging  
                // Copy file to prevent it from being changed underneath this process  
                nextActivity.setClass(this, InstallStaging.class);  
            } else if (packageUri != null && packageUri.getScheme().equals(  
                    PackageInstallerActivity.SCHEME_PACKAGE)) {  
                nextActivity.setClass(this, PackageInstallerActivity.class);  
            } else {  
                Intent result = new Intent();  
                result.putExtra(Intent.EXTRA_INSTALL_RESULT,  
                        PackageManager.INSTALL_FAILED_INVALID_URI);  
                setResult(RESULT_FIRST_USER, result);//告知调用者安装失败结果  
  
                nextActivity = null;  
            }  
        }  
        if (nextActivity != null) {  
            startActivity(nextActivity);  
        }  
        finish();  
    }  
```

 onCreate最后显示的设置跳转到InstallStaging Activity中，首先先简单介绍下APP的类型及PMS赋予给他们的权限。

### 3.2 APP类型及其权限

PMS将app分为三类：普通app、系统app(system app)和特权app(privileged app)。

#### 3.2.1 普通APP

所谓的普通app，就是第三方开发者开发的app，这种app不依赖android系统镜像，可以在任何的android设备上运行(前提是app的最低运行sdk不能低于android设备的SDK)。

#### 3.2.2 系统APP

系统app即只能运行在特定的android设备上，因为系统app需要系统签名，正常来说，不同android设备的系统签名是不一样的。准确来说，系统app， 就是具有ApplicationInfo.FLAG_SYSTEM的标记，该标记是在PMS初始化安装app的时候赋予的。那么PMS在什么情况下会赋予app ApplicationInfo.FLAG_SYSTEM标记呢。有以下几种情况

- 特定shareUID的app 

  分别有：android.uid.system、android.uid.phone、android.uid.log、android.uid.nfc、android.uid.bluetooth和android.uid.shell。shareUID是在app的AndroidManifest.xml中配置。

- 扫描安装特定目录的app 

  分别有： /vendor/overlay、 /system/framework 、 /system/priv-app 、 /system/app 、 /vendor/app 和/oem/app等

#### 3.2.3 特权APP

特权app在系统app的基础上还需要有 ApplicationInfo.PRIVATE_FLAG_PRIVILEGED标记 。有以下两种情况PMS赋予app ApplicationInfo.PRIVATE_FLAG_PRIVILEGED标记。

- 特定shareUID的app 

   特定shareUID的app有ApplicationInfo.FLAG_SYSTEM的同时都ApplicationInfo.PRIVATE_FLAG_PRIVILEGED 

- 扫描安装特定目录的app 

  分别有：  /system/framework、/system/priv-app和vendor/priv-app 

###  3.3 InstallStaging类

接着3.1的分析，InstallStart最终会跳转到InstallStaging中。
InstallStaging其实没做什么实质性的工作， 主要内容是通过StagingAsyncTask异步调用进行的。

```java
public class InstallStaging extends Activity {
...
    private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
        @Override
        protected Boolean doInBackground(Uri... params) {
            if (params == null || params.length <= 0) {
                return false;
            }
            Uri packageUri = params[0];
            // 打开读取Uri指定的文件
            try (InputStream in = getContentResolver().openInputStream(packageUri)) {
                // Despite the comments in ContentResolver#openInputStream the returned stream can
                // be null.
                if (in == null) {
                    return false;
                }
                // 将指定Uri的文件写入到mStagedFile中
                try (OutputStream out = new FileOutputStream(mStagedFile)) {
                    byte[] buffer = new byte[1024 * 1024];
                    int bytesRead;
                    while ((bytesRead = in.read(buffer)) >= 0) {
                        // Be nice and respond to a cancellation
                        // 响应cancel事件
                        if (isCancelled()) {
                            return false;
                        }
                        out.write(buffer, 0, bytesRead);
                    }
                }
            } catch (IOException | SecurityException e) {
                Log.w(LOG_TAG, "Error staging apk from content URI", e);
                return false;
            }
            return true;
        }

        @Override
        protected void onPostExecute(Boolean success) {
            if (success) {
                // Now start the installation again from a file
                // 这里通过file协议Uri来调用PackageInstallerActivity来进行安装
                Intent installIntent = new Intent(getIntent());
                installIntent.setClass(InstallStaging.this, PackageInstallerActivity.class);
                installIntent.setData(Uri.fromFile(mStagedFile));
                installIntent
                        .setFlags(installIntent.getFlags() & ~Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                installIntent.addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
                startActivityForResult(installIntent, 0);
            } else {
                showError();
            }
        }
    }
```

主要的工作是将content协议的Uri的内容读取出来然后写入到mStagedFile中，然后接着在onPostExcute中从mStagedFile产生file协议的Uri，然后使用给Uri来调用PackageInstallerActivity。由于Android N启用了StrideMode后，需要将content协议的Uri转到file协议的Uri，最后还是通过file协议的Uri来调用PackageInstaller。mStagedFile是调用TemporaryFileManager.getStagedFile在PackageInstaller apk目录创建一个前缀为package，后缀为.apk的临时文件如/data/user_de/0/com.android.packageinstaller/no_backup/package7914054629702149874.apk

### 3.4 PackageInstallerActivity类

```java
@Override
protected void onCreate(Bundle icicle) {
    super.onCreate(icicle);
    Log.d("vane","PackageInstallerActivity onCreate");

    if (icicle != null) {
        mAllowUnknownSources = icicle.getBoolean(ALLOW_UNKNOWN_SOURCES_KEY);
    }
    //第一步
    // 获取PackageManager，具体用来执行安装操作
    mPm = getPackageManager();
    // 获取IPackageManager，与PMS通信
    mIpm = AppGlobals.getPackageManager();
    // 获取OPSManager
    mAppOpsManager = (AppOpsManager) getSystemService(Context.APP_OPS_SERVICE);
    // 获取PackageInstaller，在该对象中包含了安装APK的基本信息
    mInstaller = mPm.getPackageInstaller();
    // 获取UserManager
    mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);
    // 第二步
    final Intent intent = getIntent();
    // 获取Intent中携带的数据，有安装apk发起者传入
    mCallingPackage = intent.getStringExtra(EXTRA_CALLING_PACKAGE);
    mSourceInfo = intent.getParcelableExtra(EXTRA_ORIGINAL_SOURCE_INFO);
    mOriginatingUid = intent.getIntExtra(Intent.EXTRA_ORIGINATING_UID,
            PackageInstaller.SessionParams.UID_UNKNOWN);
    mOriginatingPackage = (mOriginatingUid != PackageInstaller.SessionParams.UID_UNKNOWN)
            ? getPackageNameForUid(mOriginatingUid) : null;

    final Uri packageUri;

    if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
        final int sessionId = intent.getIntExtra(PackageInstaller.EXTRA_SESSION_ID, -1);
        final PackageInstaller.SessionInfo info = mInstaller.getSessionInfo(sessionId);
        if (info == null || !info.sealed || info.resolvedBaseCodePath == null) {
            Log.w(TAG, "Session " + mSessionId + " in funky state; ignoring");
            finish();
            return;
        }

        mSessionId = sessionId;
        packageUri = Uri.fromFile(new File(info.resolvedBaseCodePath));
        mOriginatingURI = null;
        mReferrerURI = null;
    } else {
         // 进入该分支
        mSessionId = -1;
        packageUri = intent.getData();
        mOriginatingURI = intent.getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
        mReferrerURI = intent.getParcelableExtra(Intent.EXTRA_REFERRER);
    }

    // if there's nothing to do, quietly slip into the ether
    if (packageUri == null) {
        Log.w(TAG, "Unspecified source");
        setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
        finish();
        return;
    }

    if (DeviceUtils.isWear(this)) {
        showDialogInner(DLG_NOT_SUPPORTED_ON_WEAR);
        return;
    }
    // 第三步
    // 对apk Uri进行解析
    boolean wasSetUp = processPackageUri(packageUri);//【3.4.1】
    if (!wasSetUp) {
        return;
    }
    // 第四步
    // 绑定UI界面
    bindUi(R.layout.install_confirm, false);
    // 第五步
    //检查是否允许安装程序包
    checkIfAllowedAndInitiateInstall();//【3.4.2】
}
        
```

上面的几个管理类功能如下：

|      管理类      |                         功能                         |
| :--------------: | :--------------------------------------------------: |
|  PackageManager  |      用于向APP提供部分功能，主要实现还是在PMS中      |
| IPackageManager  | AIDL接口，用于与PMS进行binder通信，相当于PMS的代理类 |
|  AppOpsManager   |                   用于动态权限检测                   |
|   UserManager    |             用于管理包的安装、卸载和升级             |
| PackageInstaller |                    用于管理多用户                    |

 代码很多，那我们来看重点：

1.   给mPm、mInstaller、mAppOpsManager、mUserManager进行初始化
2.  获取mSessionId、mSourceInfo、mOriginatingUid、mOriginatingPackage这四个重要参数 
3. 调用processPackageUri对安装包的uri进行解析，3.4.1小节详细讲解
4. 绑定相对应的UI界面
5. 检查是否允许安装程序包，3.4.2小节详细讲解

#### 3.4.1processPackageUri

```java
private boolean processPackageUri(final Uri packageUri) {
    mPackageURI = packageUri;
    final String scheme = packageUri.getScheme();
    switch (scheme) {
        //处理scheme为package的情况
        case SCHEME_PACKAGE: {
            try {
                //获取package对应的Android应用信息PackageInfo如果应用名称，权限列表等...
                mPkgInfo = mPm.getPackageInfo(packageUri.getSchemeSpecificPart(),
                        PackageManager.GET_PERMISSIONS
                                | PackageManager.MATCH_UNINSTALLED_PACKAGES);
            } catch (NameNotFoundException e) {
            }
            //如果无法获取PackageInfo，弹出一个错误的对话框，然后直接退出安装
            if (mPkgInfo == null) {
                Log.w(TAG, "Requested package " + packageUri.getScheme()
                        + " not available. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                return false;
            }
            //创建AppSnipet对象，该对象封装了待安装Android应用的标题和图标
            mAppSnippet = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                    mPm.getApplicationIcon(mPkgInfo.applicationInfo));
        } break;
        //处理scheme为file的情况
        case ContentResolver.SCHEME_FILE: {
            // 获取APK文件的实际路径
            File sourceFile = new File(packageUri.getPath());
            // 创建APK文件的分析器 parsed，同时分析安装包
            PackageParser.Package parsed = PackageUtil.getPackageInfo(this, sourceFile);
            // Check for parse errors
            //如果parsed == null，则说明解析出错，则弹出对话框，并退出安装
            if (parsed == null) {
                Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                return false;
            }
            //解析没出错，生成PackageInfo，这里面包含APK文件的相关信息
            mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
            // 设置apk的程序名称和图标，这是另一种创建AppSnippet的方式
            mAppSnippet = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
        } break;
        default: {
            throw new IllegalArgumentException("Unexpected URI scheme " + packageUri);
        }
    }
    return true;
}
```

 processPackageUri对安装包的uri进行解析，检查scheme是否支持，如果不支持则直接结束，如果支持scheme，这里面又分为两种情况 ：

- 处理scheme为package的情况
- 处理scheme为file的情况

无论是上面的哪种情况，都是要首先获取PackageInfo对象，如果scheme是package的情况下是直接调用PackageManager. getPackageInfo()方法获取的；如果scheme是file则是通过APK的实际路径即mPackageURI.getPath()来构造一个File，然后通过 PackageParser.generatePackageInfo()方法来获取的。然后创建AppSnippet对象，AppSnippet是PackageUtil的静态内部类，内部封装了icon和label。

#### 3.4.2 checkIfAllowedAndInitiateInstall

```java
private void checkIfAllowedAndInitiateInstall() {
    // Check for install apps user restriction first.
    // 支持多用户的设备上，首先检查当前用户安装应用的权限
    //正常情况下installAppsRestrictionSource为0，代表没限制
    final int installAppsRestrictionSource = mUserManager.getUserRestrictionSource(
            UserManager.DISALLOW_INSTALL_APPS, Process.myUserHandle());
    if ((installAppsRestrictionSource & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {
        showDialogInner(DLG_INSTALL_APPS_RESTRICTED_FOR_USER);
        return;
    } else if (installAppsRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
        startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
        finish();
        return;
    }
    // 允许从未知源安装应用或安装请求者不是来自未知源，则执行安装
    if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {
        initiateInstall();
    } else {
        // Check for unknown sources restriction
        // 查看是否未知源应用的安装进行了限制
        final int unknownSourcesRestrictionSource = mUserManager.getUserRestrictionSource(
                UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES, Process.myUserHandle());
        // 如果从未知源安装应用功能被系统或者用户限制
        if ((unknownSourcesRestrictionSource & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {
            showDialogInner(DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER);
        } 
        // 如果apk安装发起者还没申请到安装未知源应用的权限，则可以手动跳转Settings界面进行授权
        else if (unknownSourcesRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
            startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
            finish();
        } else {
            // 处理未知源应用安装
            handleUnknownSources();
        }
    }
}
```

```java
private boolean isInstallRequestFromUnknownSource(Intent intent) {
    if (mCallingPackage != null && intent.getBooleanExtra(
            Intent.EXTRA_NOT_UNKNOWN_SOURCE, false)) {
        if (mSourceInfo != null) {
            if ((mSourceInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED)
                    != 0) {
                // Privileged apps can bypass unknown sources check if they want.
                return false;
            }
        }
    }
    return true;
}
```

由上面的代码可以知道，如果安装apk的发起者是特权app，那么是不需要授权”安装未知来源的应用“的。如果发起安装apk请求的apk不是特权app，而且系统还没对其授权”安装未知来源的应用“，则会弹出权限申请界面，如下图所示：

![unknown-source-perm](/images/pms/packageInstaller/unknown-source-perm.png)

如果权限检测通过，则会调用initiateInstall准备安装apk。

```java
private void initiateInstall() {
    String pkgName = mPkgInfo.packageName;
    //查看应用是否已经安装过该应用，但是已经被重命名了
    String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
    if (oldName != null && oldName.length > 0 && oldName[0] != null) {
        pkgName = oldName[0];
        mPkgInfo.packageName = pkgName;
        mPkgInfo.applicationInfo.packageName = pkgName;
    }
    // 检查该应用是否已经安装，如果已经安装显示重安装提示信息
    try {
        // 获取设备上的残存数据，并且标记为“installed”,实际上已经被卸载的应用
        mAppInfo = mPm.getApplicationInfo(pkgName,
                PackageManager.MATCH_UNINSTALLED_PACKAGES);
        //如果应用是被卸载的，但是又是被标识成安装过的，则认为是新安装
        if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
            mAppInfo = null;
        }
    } catch (NameNotFoundException e) {
        mAppInfo = null;
    }
    // 显示确认安装界面
    startInstallConfirm();
}
```

 initiateInstall方法里面主要做了三件事：

1.  检查设备是否有一个现在不同名，但是曾经是相同的包名，即是否是同名安装，如果是则后续是替换安装 
2.  检查设备上是否已经安装了这个安装包，如果是，后面是替换安装 
3.  调用startInstallConfirm这个方法是安装的核心代码 

最后到了确认安装界面，如下图所示：

![ask-install](/images/pms/packageInstaller/ask-install.png)

### 3.5 小节总结

1. 根据Uri的scheme不同，跳转到不同的界面，Android 7.0及以上的版本通过FilePorvider来提供Uri的话，其scheme是content，这里会先跳转到InstallStart activity。其余的会跳转到PackageInstallerActivity
2. InstallStart完成的工作主要就是讲content协议的Uri转换为file协议的Uri，然后将该信息传递给PackageInstallerActivity
3. PackageInstallerActivity会对传入的Url进行解析apk包处理，生成这个apk的一些轻量级信息 
4. 接着PackageInstallerActivity会对应用的来源进行判断，如果Package的来源于未知源并且系统设置了允许安装未知源的APK就会执行安装，否则显示弹出相关安装未知源应用的提示框

## 4 APK安装的流程

### 4.1 startInstallConfirm

```java
private void startInstallConfirm() {
    // We might need to show permissions, load layout with permissions
    if (mAppInfo != null) {
        // 如果程序已经安装过了，那么就显示install_confirm_perm_update界面
        bindUi(R.layout.install_confirm_perm_update, true);
    } else {
        // 如果程序没有安装，则显示install_confirm_perm界面
        bindUi(R.layout.install_confirm_perm, true);
    }
    ...
    // If the app supports runtime permissions the new permissions will
    // be requested at runtime, hence we do not show them at install.
    // 如果是runtime permission的话在这个界面不会显示
    boolean supportsRuntimePermissions = mPkgInfo.applicationInfo.targetSdkVersion
            >= Build.VERSION_CODES.M;
    boolean permVisible = false;
    mScrollView = null;
    mOkCanInstall = false;
    int msg = 0;
    // 这里创建AppSecurityPermissions，能够将mPkgInfo中的Permission相关信息提出取来
    // 并且其内部有一个VIEW类PermissionItemView专门用于展示提出到的权限
    AppSecurityPermissions perms = new AppSecurityPermissions(this, mPkgInfo);
    final int N = perms.getPermissionCount(AppSecurityPermissions.WHICH_ALL);
    // 如果mAppInfo不为null，表示这是升级应用
    if (mAppInfo != null) {
        // 判断是否是system app
        msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                ? R.string.install_confirm_question_update_system
                : R.string.install_confirm_question_update;
        mScrollView = new CaffeinatedScrollView(this);
        mScrollView.setFillViewport(true);
        boolean newPermissionsFound = false;
        // 根据新添加Permission的情况以及系统版本情况显示不同的提示信息
        if (!supportsRuntimePermissions) {
            newPermissionsFound =
                    (perms.getPermissionCount(AppSecurityPermissions.WHICH_NEW) > 0);
            if (newPermissionsFound) {
                permVisible = true;
                mScrollView.addView(perms.getPermissionsView(
                        AppSecurityPermissions.WHICH_NEW));
            }
        }
        if (!supportsRuntimePermissions && !newPermissionsFound) {
            LayoutInflater inflater = (LayoutInflater)getSystemService(
                    Context.LAYOUT_INFLATER_SERVICE);
            TextView label = (TextView)inflater.inflate(R.layout.label, null);
            label.setText(R.string.no_new_perms);
            mScrollView.addView(label);
        }
        // 将升级后的应用所需要的权限显示出来
        adapter.addTab(tabHost.newTabSpec(TAB_ID_NEW).setIndicator(
                getText(R.string.newPerms)), mScrollView);
    }
    // 如果不支持运行时权限而且应用所需要的权限个数不为0
    if (!supportsRuntimePermissions && N > 0) {
        permVisible = true;
        LayoutInflater inflater = (LayoutInflater)getSystemService(
                Context.LAYOUT_INFLATER_SERVICE);
        View root = inflater.inflate(R.layout.permissions_list, null);
        if (mScrollView == null) {
            mScrollView = (CaffeinatedScrollView)root.findViewById(R.id.scrollview);
        }
        ((ViewGroup)root.findViewById(R.id.permission_list)).addView(
                    perms.getPermissionsView(AppSecurityPermissions.WHICH_ALL));
        // 显示该应用所需要的全部权限
        adapter.addTab(tabHost.newTabSpec(TAB_ID_ALL).setIndicator(
                getText(R.string.allPerms)), root);
    }
    if (!permVisible) {
        if (mAppInfo != null) {
            // This is an update to an application, but there are no
            // permissions at all.
            // 升级的应用，不需要更多额外的权限
            msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                    ? R.string.install_confirm_question_update_system_no_perms
                    : R.string.install_confirm_question_update_no_perms;
        } else {
            // This is a new application with no permissions.
            // 新安装的应用，不需要其他权限
            msg = R.string.install_confirm_question_no_perms;
        }

        // We do not need to show any permissions, load layout without permissions
        // 加载显示应用不需要权限的界面
        bindUi(R.layout.install_confirm, true);
        mScrollView = null;
    }
    if (msg != 0) {
        ((TextView)findViewById(R.id.install_confirm_question)).setText(msg);
    }
    if (mScrollView == null) {
        // There is nothing to scroll view, so the ok button is immediately
        // set to install.
        // 显示权限的scrollView中没有任何东西时，直接将安装界面的右下角的NEXT按钮修改为install
        mOk.setText(R.string.install);
        mOkCanInstall = true;
    } else {
        // 否则监听ScrollView事件
        mScrollView.setFullScrollAction(new Runnable() {
            @Override
            public void run() {
                mOk.setText(R.string.install);
                mOkCanInstall = true;
            }
        });
    }
}
```

上面的代码显示，如果不支持运行时权限检测功能，即设备的版本低于Android M的，并且检测到的权限不为0，那么就会在安装前显示在界面上。但市面上很多手机厂家无论手机是什么Android版本，都会显示安装的app锁需要的权限，其实也很合理，预先告知用户安装的app会获取什么权限，最终用户授权的是在app运行时再授予权限。

### 4.2 startInstall

onClick方法中点击INSTALL后，会调用startInstall方法进行安装apk。

```java
private void startInstall() {
    // Start subactivity to actually install the application
    // 跳转到其他Activity进行实际安装过程
    Intent newIntent = new Intent();
    // 携带PackageInfo信息
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
            mPkgInfo.applicationInfo);
    // 传递file协议的Uri
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallInstalling.class);
    String installerPackageName = getIntent().getStringExtra(
            Intent.EXTRA_INSTALLER_PACKAGE_NAME);
    ...
    startActivity(newIntent);
    finish();
}
```

 这里并没有立刻开始进行实际的安装动作，而是将相关信息添加到一个Intent中，然后跳转到InstallInstalling这个Activity进行安装，下面看下InstallInstalling具体的安装apk代码。

### 4.3 InstallInstalling类

#### 4.3.1 onCreate

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //显示正在安装的界面
        setContentView(R.layout.install_installing);
        ApplicationInfo appInfo = getIntent()
                .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mPackageURI = getIntent().getData();
        if ("package".equals(mPackageURI.getScheme())) {
           ...
        } else {
            // 打开Uri指定的文件
            final File sourceFile = new File(mPackageURI.getPath());
            PackageUtil.initSnippetForNewApp(this, PackageUtil.getAppSnippet(this, appInfo,
                    sourceFile), R.id.app_snippet);
            // 读取savedInstanceState的信息
            if (savedInstanceState != null) {
                mSessionId = savedInstanceState.getInt(SESSION_ID);
                mInstallId = savedInstanceState.getInt(INSTALL_ID);
                // 向InstallEventReceiver注册一个回调
                try {
                    InstallEventReceiver.addObserver(this, mInstallId,
                            this::launchFinishBasedOnResult);
                } catch (EventResultPersister.OutOfIdsException e) {
                }
            } else {
                //创建一个安装apk的seesion参数类，该类包含一些需要安装apk的信息
                PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                        PackageInstaller.SessionParams.MODE_FULL_INSTALL);
                params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
                params.originatingUri = getIntent()
                        .getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
                params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                        UID_UNKNOWN);
                File file = new File(mPackageURI.getPath());
                try {
                    // 调用parsePackageLite进行轻量级解析
                    PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);
                    ...

                try {
                    //注册一个回调，launchFinishBasedOnResult会接收到安装事件结果的回调
                    mInstallId = InstallEventReceiver
                            .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                    this::launchFinishBasedOnResult);//【5.1】
                } catch (EventResultPersister.OutOfIdsException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }

                try {
                    // 根据SessionParams生成安装该应用的sessionId，同时PackageInstallerService会保存该sessionId
                    mSessionId = getPackageManager().getPackageInstaller().createSession(params);
                } catch (IOException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }
            }
           // 监听cancel按钮事件
            mCancelButton = (Button) findViewById(R.id.cancel_button);
            mCancelButton.setOnClickListener(view -> {
                if (mInstallingTask != null) {
                    mInstallingTask.cancel(true);
                }

                if (mSessionId > 0) {
                    getPackageManager().getPackageInstaller().abandonSession(mSessionId);
                    mSessionId = 0;
                }

                setResult(RESULT_CANCELED);
                finish();
            });

            mSessionCallback = new InstallSessionCallback();
        }
    }
```

点击安装后显示正在安装的界面如下：

![installing-apk](/images/pms/packageInstaller/installing-apk.png)

Apk的安装是交由Session来进行的，而且任何一个应用都可以创建这样的Session。SessionParams是用来创建Session的参数，这个参数保存了apk的一些信息，诸如进度，图标。根据SessionParams生成安装该应用的sessionId，同时PackageInstallerService会保存该sessionId。接着我们分析onResume的代码。 

#### 4.3.2 onResume

```java
protected void onResume() {
    super.onResume();
    // This is the first onResume in a single life of the activity
    if (mInstallingTask == null) {
        // 获取PackageInstaller对象
        PackageInstaller installer = getPackageManager().getPackageInstaller();
        // sessionid获取SessionInfo信息
        PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);
        if (sessionInfo != null && !sessionInfo.isActive()) {
            // 创建安装的异步任务
            mInstallingTask = new InstallingAsyncTask();
            // 调用execute会触发AsyncTask的onPostExecute调用
            mInstallingTask.execute();
        } else {
            // we will receive a broadcast when the install is finished
            mCancelButton.setEnabled(false);
            setFinishOnTouchOutside(false);
        }
    }
}
```

 上述SessionInfo代表的是一个安装会话的详细信息，如果我们安装应用第一次到这里的话，那么这个sessionInfo是处于非活动状态的。接着看InstallingAsyncTask完成什么类型工作。 

#### 4.3.3 InstallingAsyncTask

```java
private final class InstallingAsyncTask extends AsyncTask<Void, Void,
        PackageInstaller.Session> {
    volatile boolean isDone;

    @Override
    protected PackageInstaller.Session doInBackground(Void... params) {
        PackageInstaller.Session session;
        try {
            // 根据sessionId创建session
            //这个Session一旦被创建，这个Session可以在多次重启的设备中多次打开，简单的就是说设备重启，未安装完成的apk的进度会被缓存起来
            session = getPackageManager().getPackageInstaller().openSession(mSessionId);
        } catch (IOException e) {
            return null;
        }
        // 设置安装进度条
        session.setStagingProgress(0);
        try {
            // 根据Uri打开文件
            File file = new File(mPackageURI.getPath());
            // 读取文件
            try (InputStream in = new FileInputStream(file)) {
                long sizeBytes = file.length();
                // 将文件中的内容写到PackageInstaller的session中
                try (OutputStream out = session
                        .openWrite("PackageInstaller", 0, sizeBytes)) {
                    byte[] buffer = new byte[1024 * 1024];
                    while (true) {
                        int numRead = in.read(buffer);
                        if (numRead == -1) {
                            session.fsync(out);
                            break;
                        }
                        if (isCancelled()) {
                            session.close();
                            break;
                        }
                        out.write(buffer, 0, numRead);
                        if (sizeBytes > 0) {
                            float fraction = ((float) numRead / (float) sizeBytes);
                            // 设置进度条进度
                            session.addProgress(fraction);
                        }
                    }
                }
            }
             return session;
        } catch (IOException | SecurityException e) {
            Log.e(LOG_TAG, "Could not write package", e);
            session.close();
            return null;
        } finally {
            synchronized (this) {
                isDone = true;
                notifyAll();
            }
        }
    }

    @Override
    protected void onPostExecute(PackageInstaller.Session session) {
        if (session != null) {
            // 创建一个intent
            Intent broadcastIntent = new Intent(BROADCAST_ACTION);
            broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
            broadcastIntent.setPackage(
                    getPackageManager().getPermissionControllerPackageName());
            broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);
            // 创建一个pendingIntent
            PendingIntent pendingIntent = PendingIntent.getBroadcast(
                    InstallInstalling.this,
                    mInstallId,
                    broadcastIntent,
                    PendingIntent.FLAG_UPDATE_CURRENT);
            // 调用session的commit方法将intent发送出去
            session.commit(pendingIntent.getIntentSender());
            mCancelButton.setEnabled(false);
            setFinishOnTouchOutside(false);
        } else {
            getPackageManager().getPackageInstaller().abandonSession(mSessionId);
            if (!isCancelled()) {
                launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
            }
        }
    }
}
```

这里我们简单说下doPackageStage方法整体，因为有两处session，有可能会让人产生误解 。

1.  通过PackageInstaller类的createSession，通过binder的方式在PackageInstallerService类中完成对其成员PackageInstallerSession的赋值，并返回sessionid给createSession方法
2. 通过PackageInstaller类的openSession，完成对PackageInstaller类中私有静态内部类Session的赋值，并通过其Session的构造函数，使得Session具有PackageInstallerSession的能力，即PackageInstaller通过其内部的Session来使用PackageInstallerSession的功能
3. 通过Session的openWrite，该方法将调用PackageInstallerSession的openWrite方法完成输入流的构建；即此刻，PackageInstallerSession已经拥有了管理目标文件的能力。最终将文件从/data/user_de/0/com.android.packageinstaller/no_backup/拷贝到/data/app/，如/data/user_de/0/com.android.packageinstaller/no_backup/package7914054629702149874.apk拷贝到/data/app/vmdl1272912134.tmp。
4. 通过Session的commit，调用PackageInstallerSession的commit方法，完成此session中的存储内容的提交，最终的提交结果将通过commit方法中的IntentSender进行回调

最后通过Session的commit进入framework层的PMS进行实际的安装工作，该部分的流程不在此展开讨论。

安装完成后，通过在4.3.1注册的安装事件监听器，可以知道安装的最终结果，代码如下：

```java
mInstallId = InstallEventReceiver
                            .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                    this::launchFinishBasedOnResult);
```

将结果回调给launchFinishBasedOnResult，接下来看下apk安装后的流程。

## 5 APK安装后的流程

### 5.1 launchFinishBasedOnResult

```java
private void launchFinishBasedOnResult(int statusCode, int legacyStatus, String statusMessage) {
    if (statusCode == PackageInstaller.STATUS_SUCCESS) {
        launchSuccess();//【5.2】
    } else {
        launchFailure(legacyStatus, statusMessage);//【5.3】
    }
}
```

如果安装成功，则调用launchSuccess，安装失败则调用launchFailure，先来分析launchSuccess的流程。

### 5.2 APK安装成功的流程

```java
private void launchSuccess() {
    Intent successIntent = new Intent(getIntent());
    successIntent.setClass(this, InstallSuccess.class);
    successIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);

    startActivity(successIntent);
    finish();
}
```

可知安装成功后跳转到InstallSuccess类。接着分析InstallSuccess的处理流程。

#### 5.2.1 InstallSuccess类

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //如果需要返回结果给请求apk安装的发起者
    if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
        // Return result if requested
        Intent result = new Intent();
        result.putExtra(Intent.EXTRA_INSTALL_RESULT, PackageManager.INSTALL_SUCCEEDED);
        setResult(Activity.RESULT_OK, result);
        finish();
    } else {
        Intent intent = getIntent();
        ApplicationInfo appInfo =
                intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        Uri packageURI = intent.getData();
        setContentView(R.layout.install_success);
        // Set header icon and title
        PackageUtil.AppSnippet as;
        PackageManager pm = getPackageManager();
        if ("package".equals(packageURI.getScheme())) {
            as = new PackageUtil.AppSnippet(pm.getApplicationLabel(appInfo),
                    pm.getApplicationIcon(appInfo));
        } else {
            File sourceFile = new File(packageURI.getPath());
            as = PackageUtil.getAppSnippet(this, appInfo, sourceFile);
        }
        PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);
        // Set up "done" button
        //安装成功后界面上的”done“按钮，点击则销毁该activity
        findViewById(R.id.done_button).setOnClickListener(view -> {
            if (appInfo.packageName != null) {
                Log.i(LOG_TAG, "Finished installing " + appInfo.packageName);
            }
            finish();
        });
        // Enable or disable "launch" button
        //安装成功后界面上的”launch“按钮，点击则启动安装的apk
        Intent launchIntent = getPackageManager().getLaunchIntentForPackage(
                appInfo.packageName);
        boolean enabled = false;
        if (launchIntent != null) {
            List<ResolveInfo> list = getPackageManager().queryIntentActivities(launchIntent, 0);
            if (list != null && list.size() > 0) {
                enabled = true;
            }
        }

        Button launchButton = (Button)findViewById(R.id.launch_button);
        if (enabled) {
            launchButton.setOnClickListener(view -> {
                try {
                    startActivity(launchIntent);
                } catch (ActivityNotFoundException | SecurityException e) {
                    Log.e(LOG_TAG, "Could not start activity", e);
                }
                finish();
            });
        } else {
            launchButton.setEnabled(false);
        }
    } 
}
```

安装成功后的逻辑很简单，显示apk安装成功的界面，界面有两个按钮：”done“和”launch“，分别对应完成后启动刚安装的apk。至此，packageInstaller安装apk成功的流程到此结束。

### 5.3 APK安装失败的流程

```java
private void launchFailure(int legacyStatus, String statusMessage) {
        Intent failureIntent = new Intent(getIntent());
        failureIntent.setClass(this, InstallFailed.class);
        failureIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
        failureIntent.putExtra(PackageInstaller.EXTRA_LEGACY_STATUS, legacyStatus);
        failureIntent.putExtra(PackageInstaller.EXTRA_STATUS_MESSAGE, statusMessage);

        startActivity(failureIntent);
        finish();
    }
```

调用InstallFailed来执行apk安装失败的后续处理。

#### 5.3.1 InstallFailed类

```java
public class InstallFailed extends Activity {
    private static final String LOG_TAG = InstallFailed.class.getSimpleName();
    private CharSequence mLabel;
    private int getExplanationFromErrorCode(int statusCode) {
        Log.d(LOG_TAG, "Installation status code: " + statusCode);

        switch (statusCode) {
            case PackageInstaller.STATUS_FAILURE_BLOCKED:
                return R.string.install_failed_blocked;
            case PackageInstaller.STATUS_FAILURE_CONFLICT:
                return R.string.install_failed_conflict;
            case PackageInstaller.STATUS_FAILURE_INCOMPATIBLE:
                return R.string.install_failed_incompatible;
            case PackageInstaller.STATUS_FAILURE_INVALID:
                return R.string.install_failed_invalid_apk;
            default:
                return R.string.install_failed;
        }
    }

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        int statusCode = getIntent().getIntExtra(PackageInstaller.EXTRA_STATUS,
                PackageInstaller.STATUS_FAILURE);

        if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
            int legacyStatus = getIntent().getIntExtra(PackageInstaller.EXTRA_LEGACY_STATUS,
                    PackageManager.INSTALL_FAILED_INTERNAL_ERROR);

            // Return result if requested
            Intent result = new Intent();
            result.putExtra(Intent.EXTRA_INSTALL_RESULT, legacyStatus);
            setResult(Activity.RESULT_FIRST_USER, result);
            finish();
        } else {
            Intent intent = getIntent();
            ApplicationInfo appInfo = intent
                    .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
            Uri packageURI = intent.getData();

            setContentView(R.layout.install_failed);

            // Set header icon and title
            PackageUtil.AppSnippet as;
            PackageManager pm = getPackageManager();

            if ("package".equals(packageURI.getScheme())) {
                as = new PackageUtil.AppSnippet(pm.getApplicationLabel(appInfo),
                        pm.getApplicationIcon(appInfo));
            } else {
                final File sourceFile = new File(packageURI.getPath());
                as = PackageUtil.getAppSnippet(this, appInfo, sourceFile);
            }

            // Store label for dialog
            mLabel = as.label;

            PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);

            // Show out of space dialog if needed
            if (statusCode == PackageInstaller.STATUS_FAILURE_STORAGE) {
                (new OutOfSpaceDialog()).show(getFragmentManager(), "outofspace");
            }

            // Get status messages
            ((TextView) findViewById(R.id.simple_status)).setText(
                    getExplanationFromErrorCode(statusCode));

            // Set up "done" button
            findViewById(R.id.done_button).setOnClickListener(view -> finish());
        }
    }

    /**
     * Dialog shown when we ran out of space during installation. This contains a link to the
     * "manage applications" settings page.
     */
    public static class OutOfSpaceDialog extends DialogFragment {
        private InstallFailed mActivity;

        @Override
        public void onAttach(Context context) {
            super.onAttach(context);

            mActivity = (InstallFailed) context;
        }

        @Override
        public Dialog onCreateDialog(Bundle savedInstanceState) {
            return new AlertDialog.Builder(mActivity)
                    .setTitle(R.string.out_of_space_dlg_title)
                    .setMessage(getString(R.string.out_of_space_dlg_text, mActivity.mLabel))
                    .setPositiveButton(R.string.manage_applications, (dialog, which) -> {
                        // launch manage applications
                        Intent intent = new Intent("android.intent.action.MANAGE_PACKAGE_STORAGE");
                        startActivity(intent);
                        mActivity.finish();
                    })
                    .setNegativeButton(R.string.cancel, (dialog, which) -> mActivity.finish())
                    .create();
        }

        @Override
        public void onCancel(DialogInterface dialog) {
            super.onCancel(dialog);

            mActivity.finish();
        }
}
```

apk安装失败的逻辑也很简单：将状态代码返回给调用方或者将失败用的户界面显示给用户。

## 6 总结

上面的流程如下面的流程图所示：

![preapare-apk](/images/pms/packageInstaller/preapare-apk.png)

![apk-installing](/images/pms/packageInstaller/apk-installing.png)