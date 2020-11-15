---
layout: post
title:  "MacBooK pro编译Android 9.0源码"
date:   2020-11-15 00:00:00
catalog:  true
tags:
    - Android
---



## 当前设备环境

操作系统：macOS Mojave 10.14.6

手机：谷歌手机Pixel

下载源码版本：android-9.0.0_r46

## 创建磁盘映像

由于mac系统文件系统是大小写不敏感的系统，所以需要在mac中创建一个支持大小写敏感的文件系统。

- 创建分区

  安卓9.0编译后比较大,笔者编译完成占用了140多G的空间,所以建议直接设置150G

  `sudo hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 150g ~/android.dmg`

- 挂载分区

  `sudo hdiutil attach ~/android.dmg.sparseimage -mountpoint /Volumes/android`

- 卸载分区

  `hdiutil detach /Volumes/android`

- 进入分区

  `cd /Volumes/android`

- 改变分区大小

  `hdiutil resize -size <new-size-you-want>g ~/android.dmg.sparseimage`

为了方便mount分区，在~/.bash_profile添加一个mountAndroid的函数：

`mountAndroid() { hdiutil attach /Users/xxx/android.dmg.sparseimage -mountpoint /Volumes/android/android9; }`

## 安装Repo工具

- 创建目录 ： `mkdir ～/bin`

- 写入到path中 ： `PATH=～/bin:$PATH`

- 下载Repo

  `curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`

  `chmod a+x ~/bin/repo`

- 将Repo中的下载地址改为清华大学的镜像源。
  编辑`~/bin/repo` 文件，修改REPO_URL为的源地址

  `export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'`

- 初始化repo

  `cd /Volumes/android/android9`

  `repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-9.0.0_r46 --depth=1 --platform=auto`

我的设备是google pixel，在https://source.android.google.cn/setup/start/build-numbers#source-code-tags-and-builds查看android P分支支持的机型，r46支持pixel，因此选择了r46版本。

`repo init`推荐使用`-b 分支标签`、`--depth=1`和`--platform=auto`这几个选项加快速度。

## 执行repo命令进行下载源码

编写脚本下载源码

sync.sh

```sh
#!/bin/bash
repo sync -j4 --current-branch
while [ $? = 1 ]; do
echo "================sync failed, re-sync again ====="
sleep 3
repo sync -j4 --current-branch
done
```

`repo sync`推荐使用`--current-branch`选项加快速度

下载代码非常漫长，期间有一些打印：

报错详情

```
error: RPC failed; curl 18 transfer closed with outstanding read data remaining 
fatal: The remote end hung up unexpectedly 
fatal: early EOF 
fatal: index-pack failed
```

根据的解决方案[https://blog.csdn.net/weixin_40355324/article/details/84845678](https://blog.csdn.net/weixin_40355324/article/details/84845678)，我这边添加了下面的指令：

`git  config --global https.postBuffer 2428800000`

注意：是https而不是http

经过将近10个小时后下载完毕。

清华镜像还有一种拉取代码的方法，直接下载压缩包： [https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/](https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/)

参考[https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)

下载完毕后，如果容量有限,此时可以删除掉.repo文件了.

## 谷歌手机设备驱动的下载

**Pixel binaries for Android 9.0.0 (PQ3A.190801.002)**

| Hardware Component                                          | Company  | Download                                                     | SHA-256 Checksum                                             |
| :---------------------------------------------------------- | :------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Vendor image                                                | Google   | [Link](https://dl.google.com/dl/android/aosp/google_devices-sailfish-pq3a.190801.002-b6b1636d.tgz) | 5b9429aa3eacc15351d2c15cf4d0a59cc200fb91645519aae79c081e47908aab |
| GPS, Audio, Camera, Gestures, Graphics, DRM, Video, Sensors | Qualcomm | [Link](https://dl.google.com/dl/android/aosp/qcom-sailfish-pq3a.190801.002-a69b0aa6.tgz) | 2d7ddecb494aa9b17705689d7dcaad8e1c1a51f659b67920c056a905649cc6bf |

在 [https://source.android.google.cn/setup/start/build-numbers#source-code-tags-and-builds](https://source.android.google.cn/setup/start/build-numbers#source-code-tags-and-builds) 查看代码版本所对应的硬件驱动

| PQ3A.190801.002 | android-9.0.0_r46 | Pie  | Pixel 3 XL、Pixel 3、Pixel 2 XL、Pixel 2、Pixel XL、Pixel | 2019-08-01 |
| --------------- | ----------------- | ---- | --------------------------------------------------------- | ---------- |
|                 |                   |      |                                                           |            |

然后你会得到两个文件tgz的压缩文件，把这两个文件放到下载好的源码根目录，用tar -zxvf + 文件名,解压出来，得出两个sh文件,然后执行两个文件。我的是extract-qcom-sargo.sh和extract-google_devices-sargo.sh，执行命令为

`./extract-google_devices-sargo.sh`

`./extract-qcom-sargo.sh`

其中在执行时，需要输入键盘ENTER和输入I ACCEPT

## 编译Android源码

由于我之前编译过android8.1的源码，所以执行下面的命令后一次性就编译通过了，但之前编译Android8.1的时候需要搭建一些环境,参考

[MacBooK pro编译Android 8.1源码](https://blog.csdn.net/u010407220/article/details/100898372)

[Mac OS10.12 编译Android源码8.1](https://blog.csdn.net/a740169405/article/details/81142263)

`source build/envsetup.sh`

`lunch`

lunch后会列出一个编译的设备表，pixel对应的是47. aosp_sailfish-userdebug

因此lunch后选择输入47，然后就执行下面的make指令，开始经过漫长的编译。奇怪的是，我编译android P才花费了2个多小时。

`make update-api -j && make -j8`

编译成功的log：

```
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 83900/84495] Target boot image from recovery: out/target/product/sailfish/boot.img
cp: out/target/product/sailfish/root/init.recovery.*.rc: No such file or directory
[ 99% 83913/84495] //art/dexlayout:libart-dexlayout link libart-dexlayout.so
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 83926/84495] //art/runtime:libart link libart.so [arm]
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 83970/84495] //art/dexlayout:libart-dexlayout link libart-dexlayout.so [arm]
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 83977/84495] //frameworks/base/libs/hwui:libhwui link libhwui.so
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 83979/84495] //frameworks/base/libs/hwui:libhwui link libhwui.so [arm]
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 84018/84495] //art/compiler:libart-compiler link libart-compiler.so
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 84467/84495] //art/compiler:libart-compiler link libart-compiler.so [arm]
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[ 99% 84480/84495] //art/dex2oat:dex2oat link dex2oat [arm]
Warning: request a ThreadPool with 1 threads, but LLVM_ENABLE_THREADS has been turned off
[100% 84495/84495] Install system fs image: out/target/product/sailfish/system.img

#### build completed successfully (02:38:57 (hh:mm:ss)) ####
```

编译成功后就在源码根目录生成了一个out目录，里面是编译的产物。代码加编译产物加起来117G。

## 刷入手机

编译系统完成后，在`out/target/product/sailfish/`目录下，能看到很多`.img`后缀的文件，此时，就可以愉快的刷机了。

1. 手机进入开发者选项，打开`OEM解锁`，打开`USB调试`

2. 执行`adb reboot bootloader`进入bootloader模式（需要安装Android SDK并配置环境变量）

3. 进入bootloader模式后，需要解锁bootloader，执行`fastboot flashing unlock`，手机会进入到解锁页面，用音量上下键选中unlock选项，使用电源键确认。

4. 解锁bootloader之后，再次执行`adb reboot bootloader`进入bootloader模式

5. 执行下面的指令：

   `export ANDROID_PRODUCT_OUT=/Volumes/android/android9/out/target/product/sailfish`

6. 执行`fastboot flashall -w`

```
xxxMacBook-Pro:android9 xxx$ fastboot flashall -w
ERROR: Couldn't create a device interface iterator: (e00002bd)
ERROR: Couldn't create a device interface iterator: (e00002bd)
ERROR: Couldn't create a device interface iterator: (e00002bd)
--------------------------------------------
Bootloader Version...: 8996-012001-1908071822
Baseband Version.....: 8996-130361-1905270421
Serial Number........: FA6CG0300776
--------------------------------------------
Checking product
OKAY [  0.050s]
target reported max download size of 536870912 bytes
mke2fs 1.43.3 (04-Sep-2016)
Creating filesystem with 6509568 4k blocks and 1630208 inodes
Filesystem UUID: 03fba336-a6bc-478b-9089-64f6ca8c5daa
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

Sending 'boot_a' (29897 KB)...
OKAY [  0.901s]
Writing 'boot_a'...
OKAY [  0.294s]
Erasing 'system_a'...
OKAY [  0.362s]
Sending sparse 'system_a' 1/2 (524284 KB)...
OKAY [ 14.457s]
Writing 'system_a' 1/2...
OKAY [  3.502s]
Sending sparse 'system_a' 2/2 (379180 KB)...
OKAY [ 10.150s]
Writing 'system_a' 2/2...
OKAY [  2.550s]
Erasing 'system_b'...
OKAY [  0.596s]
Sending 'system_b' (33348 KB)...
OKAY [  1.045s]
Writing 'system_b'...
OKAY [  0.316s]
Erasing 'vendor_a'...
OKAY [  0.139s]
Sending 'vendor_a' (266668 KB)...
OKAY [  7.407s]
Writing 'vendor_a'...
OKAY [  1.844s]
Setting current slot to 'a'...
OKAY [  0.066s]
Erasing 'userdata'...
OKAY [  2.096s]
Sending 'userdata' (4316 KB)...
OKAY [  0.198s]
Writing 'userdata'...
OKAY [  0.101s]
Rebooting...

Finished. Total time: 47.532s
```

我的pixel从Android 10成功烧录到了编译的Android 9.

注意：不要烧录[https://developers.google.com/android/images](https://developers.google.com/android/images)的镜像，烧录后会导致OEM锁重新锁上，如果是V版的话，就比较麻烦了，需要去某宝找专业人士解锁。