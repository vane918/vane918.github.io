---
layout: post
title:  "Android O系统启动流程--init篇(第二阶段)"
date:   2019-11-06 00:00:01
catalog:  true
tags:
    - android
    - 系统启动
    - init
---



>  system/core/init
>
>  - init.cpp
>  - property_service.cpp
>  - signal_handler.cpp
>  - service.cpp
>  - action.cpp

## 1 概述

 在前一篇博客中， 分析了init进程第一阶段(内核态)的流程。 在本篇博客中，我们来看看init进程第二阶段(用户态)的工作。 

## 2 主线代码

[->init.cpp]

init启动的第二阶段。

```cpp
int main(int argc, char** argv) {
    ...
// At this point we're in the second stage of init.
    // 同样屏蔽标准输入输出及定义Kernel logger
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    // 最后调用syscall，设置安全相关的值
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.
    // 这里的功能类似于“锁”
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
    //初始化属性域
    property_init();
    //初始化完属性域后，以下均完成一些属性的设定
    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);
    // 清除掉之前使用过的环境变量
    // Clean up our environment.
    unsetenv("INIT_SECOND_STAGE");
    unsetenv("INIT_STARTED_AT");
    unsetenv("INIT_SELINUX_TOOK");
    unsetenv("INIT_AVB_VERSION");
    // 再次完成selinux相关的工作
    // Now set up SELinux for second stage.
    selinux_initialize(false);
    selinux_restore_context();
    //init进程调用epoll_create1创建epoll句柄
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }
    //init进程调用signal_handler_init装载子进程信号处理器
    signal_handler_init();
    //设置默认系统属性及启动配置属性的服务端
    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
    set_usb_controller();
```

## 3 创建进程会话密钥

```c++
// Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);
```

这里使用到的是内核提供给用户空间使用的 密钥保留服务 (key retention service)，它的主要意图是在 Linux 内核中缓存身份验证数据。远程文件系统和其他内核服务可以使用这个服务来管理密码学、身份验证标记、跨域用户映射和其他安全问题。它还使 Linux 内核能够快速访问所需的密钥，并可以用来将密钥操作（比如添加、更新和删除）委托给用户空间。详细介绍请参考：[Linux 密钥保留服务入门](https://www.ibm.com/developerworks/cn/linux/l-key-retention.html?S_TACT=105AGX52&S_CMP=techcsdn)

## 4 **初始化属性域**  

[->property_service.cpp]

在Android平台中，为了让运行中的所有进程共享系统运行时所需要的各种设置值， 系统开辟了属性存储区域，并提供了访问该区域的API。如下面代码所示，最终调用_system_property_area_init函数初始化属性域。

```c++
void property_init() {
    if (__system_property_area_init()) {
        LOG(ERROR) << "Failed to initialize property area";
        exit(1);
    }
}
```

## 5  **完成selinux相关的工作** 

```c++
// Now set up SELinux for second stage.
    selinux_initialize(false);
    selinux_restore_context();
```

```cpp
static void selinux_initialize(bool in_kernel_domain) {
    ...
    if (in_kernel_domain) {
        //第一阶段的工作
        ...
    } else {
        //注册处理器
        selinux_init_all_handles();
    }
}
```

 前面 init 内核态执行初始化了 selinux，在用户态执行主要是调用 selinux_init_all_handles 函数初始化 file_contexts 和 property_contexts 文件 handle。 selinux_restore_context()的作用主要是恢复指定文件的安全上下文，因为这些文件是在 SELinux 安全机制初始化前创建，所以需要重新恢复下安全性  。

```cpp
// The files and directories that were created before initial sepolicy load or
// files on ramdisk need to have their security context restored to the proper
// value. This must happen before /dev is populated by ueventd.
static void selinux_restore_context() {
    LOG(INFO) << "Running restorecon...";
    selinux_android_restorecon("/dev", 0);
    selinux_android_restorecon("/dev/kmsg", 0);
    selinux_android_restorecon("/dev/socket", 0);
    selinux_android_restorecon("/dev/random", 0);
    selinux_android_restorecon("/dev/urandom", 0);
    selinux_android_restorecon("/dev/__properties__", 0);

    selinux_android_restorecon("/plat_file_contexts", 0);
    selinux_android_restorecon("/nonplat_file_contexts", 0);
    selinux_android_restorecon("/plat_property_contexts", 0);
    selinux_android_restorecon("/nonplat_property_contexts", 0);
    selinux_android_restorecon("/plat_seapp_contexts", 0);
    selinux_android_restorecon("/nonplat_seapp_contexts", 0);
    selinux_android_restorecon("/plat_service_contexts", 0);
    selinux_android_restorecon("/nonplat_service_contexts", 0);
    selinux_android_restorecon("/plat_hwservice_contexts", 0);
    selinux_android_restorecon("/nonplat_hwservice_contexts", 0);
    selinux_android_restorecon("/sepolicy", 0);
    selinux_android_restorecon("/vndservice_contexts", 0);

    selinux_android_restorecon("/dev/block", SELINUX_ANDROID_RESTORECON_RECURSE);
    selinux_android_restorecon("/dev/device-mapper", 0);

    selinux_android_restorecon("/sbin/mke2fs_static", 0);
    selinux_android_restorecon("/sbin/e2fsdroid_static", 0);

    selinux_android_restorecon("/sbin/mkfs.f2fs", 0);
    selinux_android_restorecon("/sbin/sload.f2fs", 0);
}
```

## 6  **创建epoll句柄** 

 接下来如下面代码所示，init进程调用epoll_create1创建epoll句柄。 

```c++
epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }
```

在linux的网络编程中，很长的时间都在使用select来做事件触发。 在linux新的内核中，有了一种替换它的机制，就是epoll。 
相比于select，epoll最大的好处在于它不会随着监听fd数目的增长而降低效率。 因为在内核中的select实现中，它是采用轮询来处理的，轮询的fd数目越多，自然耗时越多。epoll机制一般使用epoll_create(int size)函数创建epoll句柄， size用来告诉内核这个句柄可监听的fd的数目。 注意这个参数不同于select()中的第一个参数，在select中需给出最大监听数加1的值。此外，当创建好epoll句柄后，它就会占用一个fd值， 在linux下如果查看/proc/进程id/fd/，能够看到创建出的fd，因此在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。 
上述代码使用的epoll_create1(EPOLL_CLOEXEC)来创建epoll句柄， 该标志位表示生成的epoll fd具有“执行后关闭”特性。

## 7  **装载子进程信号处理器** 

init是一个守护进程，为了防止init的子进程成为僵尸进程(zombie process)， 需要init在子进程在结束时获取子进程的结束码，通过结束码将程序表中的子进程移除， 防止成为僵尸进程的子进程占用程序表的空间（程序表的空间达到上限时，系统就不能再启动新的进程了，会引起严重的系统问题）。
在linux当中，父进程是通过捕捉SIGCHLD信号来得知子进程运行结束的情况， 此处init进程调用`signal_handler_init`的目的就是捕获子进程结束的信号。

[->signal_handler.cpp]

```cpp
void signal_handler_init() {
    // Create a signalling mechanism for SIGCHLD.
    int s[2];
    if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -1) {
        PLOG(ERROR) << "socketpair failed";
        exit(1);
    }

    signal_write_fd = s[0];
    signal_read_fd = s[1];

    // Write to signal_write_fd if we catch SIGCHLD.
    struct sigaction act;
    memset(&act, 0, sizeof(act));
    //信号处理器对应的执行函数为SIGCHLD_handler.被存在sigaction结构体中，负责处理SIGCHLD消息
    act.sa_handler = SIGCHLD_handler;
    act.sa_flags = SA_NOCLDSTOP;
    //调用信号安装函数sigaction，将监听的信号及对应的信号处理器注册到内核中
    sigaction(SIGCHLD, &act, 0);
    //用于终止出现问题的子进程
    ServiceManager::GetInstance().ReapAnyOutstandingChildren();
    //注册信号处理函数handle_signal
    register_epoll_handler(signal_read_fd, handle_signal);
}
```

在深入分析代码前，我们需要了解一些基本概念： 
Linux进程通过互相发送消息来实现进程间的通信，这些消息被称为“信号”。 
每个进程在处理其它进程发送的信号时都要注册处理者，处理者被称为信号处理器。
注意到sigaction结构体的sa_flags为SA_NOCLDSTOP。 
由于系统默认在子进程暂停时也会发送信号SIGCHLD，init需要忽略子进程在暂停时发出的SIGCHLD信号， 因此将act.sa_flags 置为SA_NOCLDSTOP，该标志位表示仅当进程终止时才接受SIGCHLD信号。
`signal_handler_init`需要关注的内容还是比较多的，我们分步骤来看看。

### 7.1 **SIGCHLD_handler**  

```c++
static void SIGCHLD_handler(int) {
    if (TEMP_FAILURE_RETRY(write(signal_write_fd, "1", 1)) == -1) {
        PLOG(ERROR) << "write(signal_write_fd) failed";
    }
}
```

从上面代码我们知道，init进程是所有进程的父进程，当其子进程终止产生SIGCHLD信号时， `SIGCHLD_handler`将对`signal_write_fd`执行写操作。 
由于`socketpair`的绑定关系，这将触发信号对应的`signal_read_fd`收到数据。

### 7.2 register_epoll_handler

```c++
void register_epoll_handler(int fd, void (*fn)()) {
    epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = reinterpret_cast<void*>(fn);
     //epoll_fd增加一个监听对象fd,fd上有数据到来时，调用fn处理
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        PLOG(ERROR) << "epoll_ctl failed";
    }
}
```

 当epoll句柄监听到`signal_read_fd`中有数据可读时，将调用`handle_signal`进行处理。 至此，结合上文我们知道： 
当init进程调用`signal_handler_init`后，一旦收到子进程终止带来的SIGCHLD消息后， 将利用信号处理者`SIGCHLD_handler`向`signal_write_fd`写入信息。 由于绑定的关系，epoll句柄将监听到`signal_read_fd`收到消息， 于是将调用`handle_signal`进行处理。
整个过程如下图所示：

![load-sub-thread](/images/startup/load-sub-thread.PNG)

(1) init 进程接收到 SIGCHLD 信号
    通过 sigaction 函数将信号处理过程转移到 sigaction 结构体
(2) sigaction 成员变量 sa_flags 另外指定所关心的具体信号为 SA_NOCLDSTOP，也就是子进程终止信号
    成员变量 sa_handler 表明当子进程终止时，通过 SIGCHLD_handler 函数处理。
(3) SIGCHLD_handler 信号处理函数中通过 s[0] 文件描述符写了一个"1"
    由于 socketpair 的特性，s[1] 能接读到该字符串。
(4) 通过 register_epoll_handler 将 s[1] 注册到 epoll 内核事件表中
    handle_signal 是 s[1] 有数据到来时的处理函数 
(5) handle_signal： ReapAnyOutstandingChildren 

### 7.3  **handle_signal** 

```c++
static void handle_signal() {
    // Clear outstanding requests.
    char buf[32];
    read(signal_read_fd, buf, sizeof(buf));

    ServiceManager::GetInstance().ReapAnyOutstandingChildren();
}
```

[->service.cpp]

```c++
void ServiceManager::ReapAnyOutstandingChildren() {
    while (ReapOneProcess()) {
    }
}
```

```c++
bool ServiceManager::ReapOneProcess() {
    siginfo_t siginfo = {};
    // This returns a zombie pid or informs us that there are no zombies left to be reaped.
    // It does NOT reap the pid; that is done below.
    //用waitpid函数获取状态发生变化的子进程pid
    //waitpid的标记为WNOHANG，即非阻塞，返回为正值就说明有进程挂掉了
    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT)) != 0) {
        PLOG(ERROR) << "waitid failed";
        return false;
    }

    auto pid = siginfo.si_pid;
    if (pid == 0) return false;

    // At this point we know we have a zombie pid, so we use this scopeguard to reap the pid
    // whenever the function returns from this point forward.
    // We do NOT want to reap the zombie earlier as in Service::Reap(), we kill(-pid, ...) and we
    // want the pid to remain valid throughout that (and potentially future) usages.
    auto reaper = make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });

    if (PropertyChildReap(pid)) {
        return true;
    }
    //利用FindServiceByPid函数，找到pid对应的服务。
    //FindServiceByPid主要通过轮询解析init.rc生成的service_list，找到pid与参数一致的srvc。
    Service* svc = FindServiceByPid(pid);
    //输出服务结束的原因
    std::string name;
    std::string wait_string;
    if (svc) {
        name = StringPrintf("Service '%s' (pid %d)", svc->name().c_str(), pid);
        if (svc->flags() & SVC_EXEC) {
            wait_string = StringPrintf(" waiting took %f seconds",
                                       exec_waiter_->duration().count() / 1000.0f);
        }
    } else {
        name = StringPrintf("Untracked pid %d", pid);
    }

    auto status = siginfo.si_status;
    if (WIFEXITED(status)) {
        LOG(INFO) << name << " exited with status " << WEXITSTATUS(status) << wait_string;
    } else if (WIFSIGNALED(status)) {
        LOG(INFO) << name << " killed by signal " << WTERMSIG(status) << wait_string;
    }
    //没有找到，说明已经结束了
    if (!svc) {
        return true;
    }

    svc->Reap();
    //根据svc的类型，决定后续的处理方式
    if (svc->flags() & SVC_EXEC) {
        //可执行服务则重置对应的waiter
        exec_waiter_.reset();
    }
    if (svc->flags() & SVC_TEMPORARY) {
        //移除临时服务
        RemoveService(*svc);
    }

    return true;
}
```

### 7.4  **Reap** 

```cpp
void Service::Reap() {
    //清理未携带SVC_ONESHOT 或 携带了SVC_RESTART标志的srvc的进程组
    if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
        KillProcessGroup(SIGKILL);
    }

    // Remove any descriptor resources we may have created.
    //清除srvc中创建出的任意描述符
    std::for_each(descriptors_.begin(), descriptors_.end(),
                  std::bind(&DescriptorInfo::Clean, std::placeholders::_1));
    //清理工作完毕后，后面决定是否重启机器或重启服务，TEMP服务不用参与这种判断
    if (flags_ & SVC_TEMPORARY) {
        return;
    }

    pid_ = 0;
    flags_ &= (~SVC_RUNNING);

    // Oneshot processes go into the disabled state on exit,
    // except when manually restarted.
    //对于携带了SVC_ONESHOT并且未携带SVC_RESTART的srvc，将这类服务的标志置为SVC_DISABLED，不再自启动
    if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART)) {
        flags_ |= SVC_DISABLED;
    }

    // Disabled and reset processes do not get restarted automatically.
    if (flags_ & (SVC_DISABLED | SVC_RESET))  {
        NotifyStateChange("stopped");
        return;
    }

    // If we crash > 4 times in 4 minutes, reboot into recovery.
    //未携带SVC_RESTART的关键服务，在规定的间隔内，crash字数过多时，会导致整机重启；
    boot_clock::time_point now = boot_clock::now();
    if ((flags_ & SVC_CRITICAL) && !(flags_ & SVC_RESTART)) {
        if (now < time_crashed_ + 4min) {
            if (++crash_count_ > 4) {
                LOG(ERROR) << "critical process '" << name_ << "' exited 4 times in 4 minutes";
                panic();
            }
        } else {
            time_crashed_ = now;
            crash_count_ = 1;
        }
    }
    //将待重启srvc的标志位置为SVC_RESTARTING（init进程将根据该标志位，重启服务）
    flags_ &= (~SVC_RESTART);
    flags_ |= SVC_RESTARTING;

    // Execute all onrestart commands for this service.
    //重启在init.rc文件中带有onrestart选项的服务
    onrestart_.ExecuteAllCommands();

    NotifyStateChange("restarting");
    return;
}
```

不难看出，Reap函数的主要作用就是清除问题进程相关的资源， 然后根据进程对应的类型，决定是否重启机器或重启进程。 

### 7.5  **ExecuteAllCommands**  

[->action.cpp]

```c++
void Action::ExecuteCommand(const Command& command) const {
    android::base::Timer t;
    //进程重启时，将执行对应的函数
    int result = command.InvokeFunc();
    //打印log
    auto duration = t.duration();
    // Any action longer than 50ms will be warned to user as slow operation
    if (duration > 50ms || android::base::GetMinimumLogSeverity() <= android::base::DEBUG) {
        std::string trigger_name = BuildTriggersString();
        std::string cmd_str = command.BuildCommandString();

        LOG(INFO) << "Command '" << cmd_str << "' action=" << trigger_name << " (" << filename_
                  << ":" << command.line() << ") returned " << result << " took "
                  << duration.count() << "ms.";
    }
}
```

整个signal_handler_init的内容比较多，在此总结一下： 
`signal_handler_init`的本质就是监听子进程死亡的信息，然后进行对应的清理工作，并根据死亡进程的类型， 
决定是否需要重启进程或机器。上述过程其实最终可以简化为下图：

![signal_handler_init](/images/startup/signal_handler_init.PNG)

## 8  **设置默认系统属性及启动配置属性的服务端** 

 重新将视角拉回到init的main函数，看看接下来的工作： 

```c++
property_load_boot_defaults();
export_oem_lock_status();
start_property_service();
set_usb_controller();
```

 这部分工作最终都会更改一些系统属性。 

### 8.1 property_load_boot_defaults

[->property_service.cpp]

```c++
void property_load_boot_defaults() {
    //从各种路径读取默认配置
    if (!load_properties_from_file("/system/etc/prop.default", NULL)) {
        // Try recovery path
        if (!load_properties_from_file("/prop.default", NULL)) {
            // Try legacy path
            load_properties_from_file("/default.prop", NULL);
        }
    }
    load_properties_from_file("/odm/default.prop", NULL);
    load_properties_from_file("/vendor/default.prop", NULL);
    //设置"persist.sys.usb.config"相关的配置
    update_sys_usb_config();
}
```

如代码所示，property_load_boot_defaults实际上就是调用load_properties_from_file解析配置文件，然后根据解析的结果，设置系统属性。  

### 8.2 start_property_service

```c++
void start_property_service() {
    property_set("ro.property_service.version", "2");
    //创建了一个非阻塞socket
    property_set_fd = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                   false, 0666, 0, 0, nullptr, sehandle);
    if (property_set_fd == -1) {
        PLOG(ERROR) << "start_property_service socket creation failed";
        exit(1);
    }
    //调用listen函数监听property_set_fd， 于是该socket变成一个server
    listen(property_set_fd, 8);
    //监听server socket上是否有数据到来
    register_epoll_handler(property_set_fd, handle_property_set_fd);
}
```

init进程在共享内存区域中，创建并初始化属性域。其它进程可以访问属性域中的值，但更改属性值仅能在init进程中进行。 
这就是init进程调用start_property_service的原因。其它进程修改属性值时，要预先向init进程提交值变更申请， 然后init进程处理该申请，并修改属性值。在访问和修改属性时，init进程都可以进行权限控制。
总结一下，其它进程修改系统属性时，其它的进程像init进程发送请求后，由init进程检查权限后，修改共享内存区。  大致的流程如下图所示：

![system-property_set](/images/startup/system-property_set.PNG)

## 9 总结

init启动的第二阶段主要工作总结如下：

1. 初始化属性域， 让运行中的所有进程共享系统运行时所需要的各种设置值 
2. 完成selinux相关的工作：注册一些处理器 
3. 创建epoll句柄
4. 装在子进程信号处理器， 防止init的子进程成为僵尸进程 
5. 设置默认系统属性及启动配置属性的服务端