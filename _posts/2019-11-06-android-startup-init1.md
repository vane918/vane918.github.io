---
layout: post
title:  "Android O系统启动流程--init篇(第一阶段)"
date:   2019-11-06 00:00:00
catalog:  true
tags:
    - android
    - 系统启动
    - init
---



> system/core/init
>  - init.cpp
>  - init_first_state.cpp



## 1 概述

 当linux内核启动之后， 用户空间运行的第一个进程是init， 进程号固定为1。 init启动流程分三个阶段进行讲解。

## 2 init入口

[->init.cpp]

init的启动流程在init.cpp的main函数，这里先分析init启动的第一阶段。

```cpp
int main(int argc, char** argv) {
    //根据参数，判断是否需要启动ueventd，由init解析.rc文件start ueventd来启动
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    //根据参数，判断是否需要启动watchdogd
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
    //若紧急重启，则安装对应的消息处理器
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
    //增加环境变量
    add_environment("PATH", _PATH_DEFPATH);

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);
    //初始化的第一阶段
    if (is_first_stage) {
        boot_clock::time_point start_time = boot_clock::now();//用于记录启动时间
        //清除屏蔽字(file mode creation mask)，保证新建的目录的访问权限不受屏蔽字影响。
        // Clear the umask.
        umask(0);
        //创建文件系统目录并挂载相关的文件系统
        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
        gid_t groups[] = { AID_READPROC };
        setgroups(arraysize(groups), groups);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        //提前创建了kmsg设备节点文件，用于输出log信息
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));
        //屏蔽标准的输入输出并设置Kernel logger
        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        InitKernelLogging(argv);

        LOG(INFO) << "init first stage started!";
        //挂载一些分区设备
        if (!DoFirstStageMount()) {
            LOG(ERROR) << "Failed to mount required partitions early ...";
            panic();
        }
        //初始化安全框架：Android Verified Boot
        //AVB主要用于防止系统文件本身被篡改，包含了防止系统回滚的功能，以免有人试图回滚系统并利用以前的漏洞
        SetInitAvbVersionInRecovery();
        //初始化SELinux，加载SELinux规则
        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true);

        // We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        // 按selinux policy要求，重新设置init文件属性
        if (selinux_android_restorecon("/init", 0) == -1) {
            PLOG(ERROR) << "restorecon failed";
            //失败的话会reboot
            security_failure();
        }
        //设置变量INIT_SECOND_STAGE为true，为第二阶段做准备
        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        //记录初始化时的时间
        setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);
        //再次调用init的main函数，启动用户态的init进程
        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        //内核态的进程不应该退出，若退出则会重启
        security_failure();
    }
```

## 3  **挂载分区设备**

```c++
if (!DoFirstStageMount()) {
    LOG(ERROR) << "Failed to mount required partitions early ...";
    panic();
}
```

 我们跟进`DoFirstStageMount`看看： 

```c++
// Mounts partitions specified by fstab in device tree.
bool DoFirstStageMount() {
    // Skips first stage mount if we're in recovery mode.
    if (IsRecoveryMode()) {
        LOG(INFO) << "First stage mount skipped (recovery mode)";
        return true;
    }

    // Firstly checks if device tree fstab entries are compatible.
    if (!is_android_dt_value_expected("fstab/compatible", "android,fstab")) {
        LOG(INFO) << "First stage mount skipped (missing/incompatible fstab in device tree)";
        return true;
    }

    std::unique_ptr<FirstStageMount> handle = FirstStageMount::Create();
    if (!handle) {
        LOG(ERROR) << "Failed to create FirstStageMount";
        return false;
    }
    //满足上述条件时，就会调用FirstStageMount的DoFirstStageMount函数
    //主要是初始化特定设备并挂载，如system、vendor等分区
    return handle->DoFirstStageMount();
}
```

Android 8.1 系统将 system、vendor 分区的挂载功能移植到 kernel device-tree 中进行。在 kernel 的 dts 文件中，需要包含如下的 firmware 分区挂载节点，在 DoFirstStageMount 函数执行过程中会检查、读取 device-tree 中记录的分区挂载信息。

```c++
firmware {
    android {
        compatible = "android,firmware";
        fstab {
            compatible = "android,fstab";
            system {
                compatible = "android,system";
                dev = "/dev/block/by-name/system";
                type = "ext4";
                mnt_flags = "ro,barrier=1,inode_readahead_blks=8";
                fsmgr_flags = "wait";                                                                                       
            };  
            vendor {
                compatible = "android,vendor";
                dev = "/dev/block/by-name/vendor";
                type = "ext4";
                mnt_flags = "ro,barrier=1,inode_readahead_blks=8";
                fsmgr_flags = "wait";
            };  
        };  
    };  
};
```

FirstStageMount 的构造函数中通过 fs_mgr_read_fstab_dt 函数读取 /proc/device-tree/firmware/android/fstab 目录下的分布挂载信息，最后统计成 fstab_rec 类型的 vector 数组， MountPartitions() 函数遍历 fstab_rec 数组，找到 mount_source 和 mount_target，使用 mount 函数将 system、vendor或者 oem 分区挂载上。

成功挂载的 Log 打印如下：

```
I/init    (    0): [libfs_mgr]__mount(source=/dev/block/dm-0,target=/system,type=ext4)=0: Success
I/init    (    0): [libfs_mgr]__mount(source=/dev/block/dm-1,target=/vendor,type=ext4)=0: Success
```

上面的log是开启了dm-verity功能的打印。

## 4  启动 SELinux 安全策略 

 在SELinux这种访问控制体系的限制下，进程只能访问那些在他的任务中所需要文件。 

```c++
SetInitAvbVersionInRecovery();
// Set up SELinux, loading the SELinux policy.
selinux_initialize(true);
```

 看下`selinux_initialize`相关的工作： 

```c++
static void selinux_initialize(bool in_kernel_domain) {
    Timer t;

    selinux_callback cb;
    //用于打印log的回调函数
    cb.func_log = selinux_klog_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    //用于检查权限的回调函数
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    //first_stage运行在内核态
    if (in_kernel_domain) {
        LOG(INFO) << "Loading SELinux policy";
        //用于加载sepolicy文件，失败会重启设置
        if (!selinux_load_policy()) {
            panic();
        }
        //内核中读取的信息
        bool kernel_enforcing = (security_getenforce() == 1);
        bool is_enforcing = selinux_is_enforcing();
        if (kernel_enforcing != is_enforcing) {
            //用于设置selinux的工作模式。selinux有两种工作模式：
            //1、”permissive”，所有的操作都被允许（即没有MAC），但是如果违反权限的话，会记录日志
            //2、”enforcing”，所有操作都会进行权限检查。在一般的终端中，应该工作于enforing模式
            if (security_setenforce(is_enforcing)) {
                PLOG(ERROR) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");
                //将重启进入recovery mode
                security_failure();
            }
        }

        std::string err;
        if (!WriteFile("/sys/fs/selinux/checkreqprot", "0", &err)) {
            LOG(ERROR) << err;
            security_failure();
        }

        // init's first stage can't set properties, so pass the time to the second stage.
        setenv("INIT_SELINUX_TOOK", std::to_string(t.duration().count()).c_str(), 1);
    } else {
        //在second stage调用时，初始化所有的handle
        selinux_init_all_handles();
    }
}
```

`selinux_initialize`主要是做了一些初始化SELinux并加载SELinux policy的工作。

## 5  **第一阶段收尾工作**  

```c++
// We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        // 按selinux policy要求，重新设置init文件属性
        if (selinux_android_restorecon("/init", 0) == -1) {
            PLOG(ERROR) << "restorecon failed";
            //失败的话会reboot
            security_failure();
        }
        //设置变量INIT_SECOND_STAGE为true，为第二阶段做准备
        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        //记录初始化时的时间
        setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);
        //再次调用init的main函数，启动用户态的init进程
        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        //内核态的进程不应该退出，若退出则会重启
        security_failure();
    }
```

第一阶段结束时，会复位一些信息，并设置一些环境变量， 最后启动用户态的用户态的init进程，进入init启动的第二阶段。 

## 6 总结

init 进程第一阶段做的主要工作是如下：

1. 挂载分区 dev、system、vendor 
2. 创建设备节点及设备节点热插拔事件监听处理（ueventd） 
3. 创建一些关键目录、初始化日志输出系统 
4. 启用 SELinux 安全策略