---
layout: post
title:  "Android O Binder系列-ServerManager的启动"
date:   2019-12-18 00:00:00
catalog:  true
tags:
    - android
    - ServerManager启动
    - Binder
---



> frameworks/native/cmds/servicemanager/servicemanager.rc
>
> frameworks/native/cmds/servicemanager/service_manager.c
>
> frameworks/native/cmds/servicemanager/binder.c



## 1 概述

ServiceManager是Binder IPC通信过程中的守护进程，本身也是一个Binder服务，通过ioctrl直接和Binder驱动来通信。ServiceManager本身工作相对简单，其功能：查询和注册服务。

## 2 启动过程

 ServiceManager是由init进程通过解析init.rc文件而创建的，最终调用到servicemanager.rc文件，其所对应的可执行程序/system/bin/servicemanager，所对应的源文件是service_manager.c，进程名为/system/bin/servicemanager。下面讲解源码前先给出servermanager启动过程的时序图。

![server_manager启动流程](/images/binder/server_manager启动流程.png)

[->servicemanager.rc]

```logcat
class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

 启动Service Manager的入口函数是service_manager.c中的main()方法，下面详细分析。

### 2.1 main 

[->service_manager.c]

```c
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }
    //打开binder驱动，申请128k字节大小的内存空间【见小节2.2】
    bs = binder_open(driver, 128*1024);
    if (!bs) {
#ifdef VENDORSERVICEMANAGER
        ALOGW("failed to open binder driver %s\n", driver);
        while (true) {
            sleep(UINT_MAX);
        }
#else
        ALOGE("failed to open binder driver %s\n", driver);
#endif
        return -1;
    }
    //成为上下文管理者【见小节2.3】
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

#ifdef VENDORSERVICEMANAGER
    sehandle = selinux_android_vendor_service_context_handle();
#else
    sehandle = selinux_android_service_context_handle();
#endif
    selinux_status_open(true);

    if (sehandle == NULL) {
        ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
        abort();//无法获取sehandle
    }

    if (getcon(&service_manager_context) != 0) {
        ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
        abort();//无法获取service_manager上下文
    }

    //进入无限循环，处理client端发来的请求 【见小节2.4】
    binder_loop(bs, svcmgr_handler);

    return 0;
}
```

### 2.2 binder_open

[->binder.c]

```c
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }
    //通过系统调用陷入内核，打开Binder设备驱动
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open %s (%s)\n",
                driver, strerror(errno));
        goto fail_open;// 无法打开binder设备
    }
     //通过系统调用，ioctl获取binder版本信息
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        //内核空间与用户空间的binder不是同一版本
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
    //通过系统调用，mmap内存映射
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;// binder设备内存无法映射
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

### 2.3 binder_become_context_manager

[->binder.c]

```c
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```

成为上下文的管理者，整个系统中只有一个这样的管理者。 通过ioctl()方法经过系统调用，对应于Binder驱动层kernel/drivers/staging/android/binder.c的binder_ioctl()方法。

### 2.4 binder_loop

由于Service Manager需要在系统运行期间为Service组件和Client组件提供服务， 因此， 它就需要通过一个无限循环来等待和处理Service组件和Client组件的进程间通信请求， 这是通过调用函数binder_loop 来实现的。

[->binder.c]

```c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    //将BC_ENTER_LOOPER命令发送给binder驱动，让Service Manager进入循环
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        //进入循环，不断地binder读写过程
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        // 解析binder信息 【见小节2.5】
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```

 一个线程要通过协议BC_ENTER_LOOPER或者BC_REGISTER_LOOPER将自己注册为Binder线程，以便Binder驱动程序可以将进程间通信请求分发给它处理。 由于ServiceManager进程的主线程是主动成为一个Binder线程的， 因此，它就使用BC_ENTER_LOOPER协议将自己注册到Binder驱动程序中。
binder_loop开启一个循环，不断使用IO控制命令BINDER_WRITE_READ来检查Binder 驱动程序是否有新的进程间通信请求需要它来处理。 如果有， 就将它们交给函数binder_parse来处理；否则， 当前线程就会在Binder驱动程序中睡眠等待， 直到有新的进程间通信请求到来为止。 下面看下binder_parse是如何解析binder信息的。

### 2.5 binder_parse

[->binder.c]

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
#if TRACE
        fprintf(stderr,"%s:\n", cmd_name(cmd));
#endif
        switch(cmd) {
        case BR_NOOP:
            break;//无操作，退出循环
        case BR_TRANSACTION_COMPLETE:
            break;
        case BR_INCREFS:
        case BR_ACQUIRE:
        case BR_RELEASE:
        case BR_DECREFS:
#if TRACE
            fprintf(stderr,"  %p, %p\n", (void *)ptr, (void *)(ptr + sizeof(void *)));
#endif
            ptr += sizeof(struct binder_ptr_cookie);
            break;
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;
                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);//从txn解析出binder_io信息
                //func是调用binder_loop函数传进来的svcmgr_handler【见小节2.6】
                res = func(bs, txn, &msg, &reply);
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
        case BR_REPLY: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: reply too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (bio) {
                bio_init_from_txn(bio, txn);
                bio = 0;
            } else {
                /* todo FREE BUFFER */
            }
            ptr += sizeof(*txn);
            r = 0;
            break;
        }
        // binder死亡消息【见小节2.7】
        case BR_DEAD_BINDER: {
            struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
            ptr += sizeof(binder_uintptr_t);
            death->func(bs, death->ptr); 
            break;
        }
        case BR_FAILED_REPLY:
            r = -1;
            break;
        case BR_DEAD_REPLY:
            r = -1;
            break;
        default:
            ALOGE("parse: OOPS %d\n", cmd);
            return -1;
        }
    }

    return r;
}
```

 解析binder信息，此处参数ptr指向BC_ENTER_LOOPER，func指向svcmgr_handler。故有请求到来，则调用svcmgr_handler。 

### 2.6 svcmgr_handler

[->service_manager.c]

```c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    ...
    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        //根据名称查找相应服务 【见小节2.6.1】
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        //注册指定服务 【见小节2.6.2】
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;

    case SVC_MGR_LIST_SERVICES: {
        uint32_t n = bio_get_uint32(msg);

        if (!svc_can_list(txn->sender_pid, txn->sender_euid)) {
            ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                    txn->sender_euid);
            return -1;
        }
        si = svclist;
        while ((n-- > 0) && si)
            si = si->next;
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

 svcmgr_handler的功能：查询服务、注册服务，以及列举所有服务 。

#### 2.6.1 do_find_service

[-> service_manager.c] 

```c
uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    //查询相应的服务，每一个服务用svcinfo结构体来表示
    struct svcinfo *si = find_svc(s, len);
    if (!si || !si->handle) {
        return 0;
    }

    if (!si->allow_isolated) {
        // If this service doesn't allow access from isolated processes,
        // then check the uid to see if it is isolated.
        uid_t appid = uid % AID_USER;
        //检查该服务是否允许孤立于进程而单独存在
        if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
            return 0;
        }
    }
    //服务是否满足查询条件
    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }

    return si->handle;
}
```

 do_find_service查询到目标服务后，返回该服务所对应的handle 。

#### 2.6.2 do_add_service

[-> service_manager.c] 

```c
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    //ALOGI("add_service('%s',%x,%s) uid=%d\n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);

    if (!handle || (len == 0) || (len > 127))
        return -1;
    //权限检查
    if (!svc_can_register(s, len, spid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }
    //服务检索
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);//服务已注册时，释放相应的服务
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        //内存不足，无法分配足够内存
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        //内存拷贝服务信息
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;// svclist保存所有已注册的服务
        svclist = si;
    }
    //以BC_ACQUIRE命令，handle为目标的信息，通过ioctl发送给binder驱动
    binder_acquire(bs, handle);
    //以BC_REQUEST_DEATH_NOTIFICATION命令的信息，通过ioctl发送给binder驱动，
    //主要用于清理内存等收尾工作
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

注册服务的分以下3部分工作：

- svc_can_register：检查权限，检查selinux权限是否满足；
- find_svc：服务检索，根据服务名来查询匹配的服务；
- svcinfo_death：释放服务，当查询到已存在同名的服务，则先清理该服务信息，再将当前的服务加入到服务列表svclist；

### 2.7 处理binder dead

当binder所在进程死亡后,便会发出死亡通知的回调，最终由内核空间回到binder_parse函数的BR_DEAD_BINDER分支中处理。

```c
case BR_DEAD_BINDER: {
    struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
    ptr += sizeof(binder_uintptr_t);
    death->func(bs, death->ptr);
    break;
}
```

  由2.6.2小节do_add_service的si->death.func = (void*) svcinfo_death;可知，此处 death->func是执行svcinfo_death()方法。

#### 2.7.1 svcinfo_death

 [-> service_manager.c] 

```c
void svcinfo_death(struct binder_state *bs, void *ptr)
{
    struct svcinfo *si = (struct svcinfo* ) ptr;

    ALOGI("service '%s' died\n", str8(si->name, si->len));
    if (si->handle) {
        binder_release(bs, si->handle);
        si->handle = 0;
    }
}
```

主要是调用binder_release向binder驱动写入BC_RELEASE命令。

## 3 总结

Service Manager的启动过程由三个步骤组成：

1. 调用函数binder_open打开设备文件/dev/binder, 以及将它映射到本进程的地址空间；

2. 调用函数binder_become_context_manager将自己注册为 Binder进程间通信机制的上下文管理者；

3. 调用函数binder_loop来循环等待和处理Client进程的通信请求。

   ServiceManager最核心的两个功能为查询和注册服务

- 注册服务：记录服务名和handle信息，保存到svclist列表
- 查询服务：根据服务名查询相应的的handle信息