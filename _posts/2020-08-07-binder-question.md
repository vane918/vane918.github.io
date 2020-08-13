---
layout: post
title:  "Binder十万个为什么"
date:   2020-08-07 00:00:00
catalog:  true
tags:
    - 系统
    - Binder
    - Android

---



>  以提问的方式探讨 Android 中 Binder 机制。

## 1. 概述

大家都知道Binder是一种进程间通信机制， 在 Linux 中，Binder 被看做一个字符设备，Binder 驱动会为每个打开 Binder的进程在内核里分配一块地址空间，Client 向 Server 传递数据，其实就是将数据写道内核空间中 Server 的地址里面，然后通知Server去取数据。原理其实很简单，但是 Google 为了更加合理的使用 Binder，自己进行了很多层次的封装与优化，大大增加了 Binder 的架构复杂度。本文以提问的方式带大家理解 Binder 机制。

## 2. Binder常见问题

1. ServiceManager 如何管理  Servers 

   每个server进程在注册的时候，首先往本地进程的内核空间的 Binders 红黑树插入 Binder 实体服务的 bind_node 节点，然后会在 ServiceManager 进程的内核空间为其添加引用 ref 结构体， ref会保存相应的信息，如名字、ptr地址等。

2. Client如何找到 Server，并且向其发送请求

   Client 在 getService 的时候，ServiceManager 会找到 Server 的 node 节点，并在 Client 中创建 Server 的 bind_ref 引用，Client 可以在自己进程的内核空间中找到该引用，最终获取 Server 的 bind_node 节点，直接访问 Server，传输数据并唤醒。 

3. Client 端，Server 实体的引用 bind_ref 存在哪里 

   Binder 驱动会在内核空间为打开 Binder 设备的进程（包括 Client 及 Server 端）创建 bind_proc 结构体，bind_proc包含4棵红黑树：threads、bind_refs、bind_nodes、bind_desc 这四棵树分别记录该进程的线程树、Binder 引用树、本地 Binder 实体，等信息，方便本地查找 。

4. 如何唤醒目标进程或者线程

   每个 Binder 进程或者线程在内核中都设置了自己的等待队列，Client 将目标进程或者线程告诉 Binder 驱动，驱动负责唤醒挂起在等待队列上的线程或者进程。 

5. Server 如何找到返回目标进程或者线程

   Client 在请求的时候，会在 bind_trasaction 的 from 中添加请求端信息 

6. Binder 节点与 ref 节点的添加时机是什么

   驱动中存在一个 TYPE_BINDER 与 TYPR_HANDLE 的转换，Binder 节点是 Binder Server 进程（一般是 Native 进程）在向 Servicemanager 注册时候添加的，而 ref 是 Client 在 getService 的时候添加的，并且是由 ServiceManager 添加的。 

7. Binder 如何实现只拷贝一次

   传统的 IPC 通信方式 一次数据传递需要经历：内存缓存区 --> 内核缓存区 --> 内存缓存区，需要 2 次数据拷贝 。 Binder IPC 机制 Binder IPC 正是基于内存映射（mmap）来实现的 ，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。Binder 驱动使用 mmap() 并不是为了在物理介质和用户空间之间建立映射，而是用来在内核空间创建数据接收的缓存空间。 一次完整的 Binder IPC 通信过程通常是这样： 

   1. Binder 驱动在内核空间创建一个数据接收缓存区；
   2. 在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系； 
   3. 发送方进程通过系统调用 copy*from*user() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

   ![IPC](/images/binder/IPC.png)

   ![binder-IPC](/images/binder/binder-IPC.png)

8. Binder Server都会在ServiceManager中注册吗

   Java 层的 Binder 实体就不会去 ServiceManager，尤其是 bindService 这样一种，其实是 ActivityManagerService 充当了 ServiceManager 的角色

9. IPCThreadState::joinThreadPool 的真正意义是什么

   可以理解加入该进程内核的线程池，进行循环，多个线程开启，其实一个就可以，怕处理不过来，可以开启多个线程处理起来，其实跟线程池类似。 

10. 为何 ServiceManager 启动的时候没有采用 joinThreadPool，而是自己通过for循环来实现自己 Loop

    因为 Binder 环境还没准备好，所以自己控制，所以也没有 talkWithDriver 那套逻辑，不用 onTransact 实现。在binder_transaction 中，会对目标是 ServiceManager 的请求进行特殊处理。 

11. ServiceManager 是由谁启动的

    init 进程在init.rc中启动。

12. ServiceManager 如何成为系统 Server 的大管家

    servicemanager  主要做了以下几件事情：

    1.  调用函数 binder_open 打开设备文件 /dev/binder 并使用 mmap 将其映射到调用者进程的地址空间中；
    2.  调用 binder_become_context_manager 将自己注册为 Binder 进程间通信机制的上下文管理者；
    3.  调用函数 binder_loop 开启循环，监听 Client 进程的通信要求 

13. Client 进程与 Server 进程的一次进程间通信过程

    1. Client 进程将进程间通信数据封装成一个 Parcel 对象，以便可以将进程间通信数据传递给 Binder 驱动程序。
    2. Client 进程向 Binder 驱动程序发送一个 BC_TRANSACTION 命令协议。Binder 驱动程序根据协议内容找到目标Server 进程之后，就会向 Client 进程发送一个 BR_TRANSACTION_COMPLETE返回协议，表示它的进程间通信请求已经被接受。Client 进程接收到 Binfer 进程接收到 Binder 驱动程序发送给它的返回协议，并对它进程处理之后，就会再次进入到 Binder 驱动程序中去等待目标 Server 进程返回进程间通信结果。
    3. Binder 驱动程序在向 Client 进程发送 BR_TRANSACTION_COMPLETE返回协议的同时，也会向目标 Server 进程发送一个 BR_TRANSACTION 返回协议，请求目标 Server 进程处理该进程间通信请求。
    4. Server 进程接收到 Binder 驱动程序发来的 BR_TRANSACTION 返回协议，并且对它进程处理之后，就会向 Binder 驱动程序发送一个 BC_REPLY命令协议。Binder 驱动程序根据协议内容找到目标 Client 进程之后，就会向 Server 进程发送一个 BR_TRANSACTION_COMPLETE 返回协议，表示它返回的进程间通信结果已经收到了。Server 进程接收到 Binder 驱动程序发给它的 BR_TRANSACTION_COMPLETE 返回协议，并对它进行处理后，一次进程间通信过程就结束了。接着它会再次进入到 Binder 驱动程序中去等待下一次进程间通信请求。
    5. Binder 驱动程序向 Server 进程发送 BR_TRANSACTION_COMPLETE 返回协议的同时，也会向目标 Client 进程发送一个 BR_REPLY 返回协议，表示 Server 进程已经处理完成它的进程间通信请求了，并且将进程间通信结果返回给它。

14. Binder 驱动程序如何找到目标的 Binder 实体，并唤醒进程或线程

    Binder 实体服务有两种：通过 addService注册到 ServiceManager 中的服务，如AMS、PMS 等；还有一种是通过 bindService 拉起的一些服务，一般是app 层面的服务。ServiceManager 是一个特殊的服务，是管理其他服务的“管家”，其 Handle 句柄是固定的，值为0，所以 ServiceManager 服务不需要查询，可以直接使用。

    binder_proc：用来描述一个正在使用 Binder 进程间通信机制的进程。当一个进程调用函数 open 来打开设备文件 /dev/binder 时，Binder 驱动程序会为它创建一个 binder_proc 结构体。

    在 binder_proc 内部有四棵红黑树，threads，nodes，refs_by_desc，refs_by_node：

    - threads：以线程 ID 作为关键字来组织一个进程的 Binder 线程池
    - nodes：Binder实体在内核中对应的数据结构
    - refs_by_desc：Binder 引用对象；以 Binder 引用对象的成员变量 desc 作为关键字
    - refs_by_node：；Binder 引用对象；以 Binder 引用对象的成员变量 node 作为关键字

     假设存在一堆Client与Service，Client如何才能访问Service呢？ 

    步骤如下：

    1. Service 通过 addService 函数将 Binder 实体注册到 ServiceManager 中；ServiceManager 将 Service 相关信息存储在自己进程的 Service 列表中去，同时在 ServiceManager 进程的 binder_ref 红黑树中为 Service 添加 binder_ref 节点，这样 ServiceManager 就能获取 Service的 Binder 实体信息。

    2. Client 如果想使用 Service，就需要通过 getService 函数向 ServiceManager 请求该服务；ServiceManager 会在注册 Service 列表中查找该服务，如果找到就将该服务返回给 Client，在这个过程中，ServiceManager 会在 Client 进程的 binder_ref 红黑树中添加 binder_ref节点.(可见进程中的 binder_ref 红黑树节点都不是本进程自己创建的，要么是 Service 进程将 binder_ref 插入到 ServiceManager 中，要么就是 ServiceManager 进程将 binder_ref 插入到 Client 中。)

    3. 通过第二步 Client 通过 Handle 句柄获取 binder_ref，进而就可以访问 Service服务了。

       ![IPC](/images/binder/client-find-service.PNG)

    getService 之后，便可以获取 binder_ref 引用，进而获取到 binder_proc 与 binder_node 信息，之后 Client 便可有目的的将 binder_transaction 事务插入到 binder_proc 的待处理列表，并且，如果进程正在睡眠，就唤起进程，其实这里到底是唤起进程还是线程也有讲究，对于 Client 向 Service 发送请求的状况，一般都是唤醒 binder_proc 上睡眠的线程。

15. binder_proc 为何会有两棵binder_ref红黑树

    binder_proc 中存在两棵 binder_ref 红黑树，其实两棵红黑树中的节点是复用的，只是查询方式不同，一个通过handle 句柄，一个通过 node 节点查找。个人理解：refs_by_node 红黑树主要是为了 Binder 驱动程序往用户空间写数据所使用的，而 refs_by_desc 是用户空间向 Binder 驱动程序写数据使用的，只是方向问题。比如在服务addService 的时候，Binder 驱动会在在 ServiceManager 进程的 binder_proc 中查找 binder_ref 结构体，如果没有就会新建 binder_ref 结构体，再比如在 Client 端 getService 的时候，Binder 驱动会在 Client 进程中通过 binder_get_ref_for_node 为 Client 创建 binder_ref 结构体，并分配句柄，同时插入到 refs_by_desc 红黑树中，可见refs_by_node 红黑树，主要是给Binder 驱动往用户空间写数据使用的。相对的 refs_by_desc 主要是为了用户空间往Binder 驱动写数据使用的，当用户空间已经获得 Binder 驱动为其创建的 binder_ref 引用句柄后，就可以通过binder_get_ref 从 refs_by_desc 找到响应 binder_ref，进而找到目标 binder_node。可见有两棵红黑树主要是区分使用对象及数据流动方向。

16. Binder 传输数据的大小限制

    在 Activity 之间传输 BitMap 的时候，如果 Bitmap 过大，就会引起问题，比如崩溃等，这其实就跟 Binder 传输数据大小的限制有关系，在拷贝映射过程中，mmap 函数会为 Binder 数据传递映射一块连续的虚拟地址，这块虚拟内存空间其实是有大小限制的，不同的进程可能还不一样：

    - 普通的由Zygote孵化而来的用户进程，所映射的 Binder 内存大小是不到1M的，准确说是**1M-8K**。这个限制定义在ProcessState类中，如果传输数据超过这个大小，系统就会报错，因为 Binder 本身就是为了进程间频繁而灵活的通信所设计的，并不是为了拷贝大数据而使用的。
    - 而在内核中，其实也有个限制，是4M，不过由于APP中已经限制了不到1M，这里的限制似乎也没多大用途。
    - ServiceManager 进程，它为自己申请的 Binder 内核空间是128K，这个同 ServiceManager 的用途是分不开的，ServcieManager 主要面向系统 Service，只是简单的提供一些 addServcie，getService 的功能，不涉及多大的数据传输，因此不需要申请多大的内存。

17. Binder 内存限制是1m-8k, 为什么一次调用最大传输数据只有大约508k

    Binder 的线程池数量默认是15个，由15个线程共享这 1MB-8KB 的内存空间，所以实际传输大小并没有那么大 

18. 系统服务与 bindService 等启动的服务的区别

    服务可分为系统服务与普通服务，系统服务一般是在系统启动的时候，由 SystemServer 进程创建并注册到ServiceManager 中的。而普通服务一般是通过 ActivityManagerService 启动的服务，或者说通过四大组件中的Service 组件启动的服务。这两种服务在实现跟使用上是有不同的，主要从以下几个方面：

    - 服务的启动方式

      系统服务一般都是 SystemServer 进程负责启动，比如AMS，WMS，PKMS，电源管理等，这些服务本身其实实现了 Binder 接口，作为 Binder 实体注册到 ServiceManager 中，被 ServiceManager 管理，而 SystemServer 进程里面会启动一些 Binder 线程，主要用于监听 Client 的请求，并分发给响应的服务实体类，可以看出，这些系统服务是位于 SystemServer 进程中（有例外，比如 Media 服务）。在来看一下 bindService 类型的服务，这类服务一般是通过 Activity 的 startService 或者其他 context 的 startService 启动的，这里的 Service 组件只是个封装，主要的是里面 Binder 服务实体类，这个启动过程不是 ServcieManager 管理的，而是通过AMS进行管理的，同 Activity 管理类似。

    - 服务的注册与管理

      系统服务一般都是通过 ServiceManager 的 addService 进行注册的，这些服务一般都是需要拥有特定的权限才能注册到 ServiceManager，而 bindService 启动的服务可以算是注册到 ActivityManagerService，只不过AMS管理服务的方式同 ServiceManager 不一样，而是采用了 Activity 的管理模型。

    - 服务的请求使用方式

      使用系统服务一般都是通过 ServiceManager 的 getService 得到服务的句柄，这个过程其实就是去ServiceManager 中查询注册系统服务。而 bindService 启动的服务，主要是去AMS中去查找相应的 Service 组件，最终会将 Service 内部 Binder 的句柄传给 Client。

19. Binder 线程、Binder 主线程、Client 请求线程的概念与区别

    Binder 线程是执行 Binder 服务的载体，只对于服务端才有意义，对请求端来说，是不需要考虑 Binder 线程的，但Android 系统的处理机制其实大部分是互为 C/S 的。比如 APP 与 AMS 进行交互的时候，都互为对方的 C 与 S，先看Binder线程的概念。Binder 线程就是执行 Binder 实体业务的线程，一个普通线程如何才能成为 Binder 线程呢？很简单，只要开启一个监听 Binder 字符设备的 Loop 线程即可，在 Android 中有很多种方法，不过归根到底都是监听Binder，换成代码就是通过 ioctl 来进行监听。 拿 ServerManager 进程来说，其主线程就是 Binder 线程，其做法是通过 binder_loop 实现不死线程： 

    ```c
    void binder_loop(struct binder_state *bs, binder_handler func)
    {
        ...
        for (;;) {
            //关键点1
            res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
            //关键点2
            res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
            ...
        }
    }
    ```

    上面的关键代码1就是阻塞监听客户端请求，2 就是处理请求，并且这是一个死循环，不退出。再来看 SystemServer进程中的线程，在 Android4.3（6.0以后的代码就不一样了）中 SystemSever 主线程便是 Binder 线程，同时一个Binder 主线程，Binder 线程与 Binder 主线程的区别是：线程是否可以终止 Loop，不过目前启动的 Binder 线程都是无法退出的，其实可以全部看做是 Binder 主线程，其实现原理是，在 SystemServer 主线程执行到最后的时候，Loop 监听 Binder 设备，变身死循环线程，关键代码如下：

    ```c++
    extern "C" status_t system_init()
    {
        ...
        ALOGI("System server: entering thread pool.\n");
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
        ALOGI("System server: exiting thread pool.\n");
        return NO_ERROR;
    }
    ```

    `ProcessState::self()->startThreadPool()`是新建一个 Binder 主线程，而`IPCThreadState::self()->joinThreadPool()`是将当前线程变成 Binder 主线程。其实 startThreadPool 最终也会调用 joinThreadPool，看下其关键函数：

    ```c++
    void IPCThreadState::joinThreadPool(bool isMain)
    {
        ...
        status_t result;
        do {
            int32_t cmd;
            // 关键点1 
            result = talkWithDriver();
            if (result >= NO_ERROR) {
                // 关键点2 
                result = executeCommand(cmd);
            }
            // 非主线程的可以退出
            if(result == TIMED_OUT && !isMain) {
                break;
            }
            // 死循环，不完结，调用了这个，就好比是开启了Binder监听循环，
        } while (result != -ECONNREFUSED && result != -EBADF);
     }
    
    status_t IPCThreadState::talkWithDriver(bool doReceive)
    {  
        do {
            ...关键点3 
            if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
       }   
    ```

    关键点1 talkWithDriver，其实质还是去调用 ioctl 去不断的监听 Binder 字符设备，获取到 Client 传输的数据后，再通过 executeCommand 去执行相应的请求，joinThreadPool 是普通线程化身 Binder 线程最常见的方式。

    最后来看一下普通 Client 的 binder 请求线程，比如我们 APP 的主线程，在 startActivity 请求 AMS 的时候，APP 的主线程成其实就是 Binder 请求线程，在进行 Binder 通信的过程中，Client 的 Binder 请求线程会一直阻塞，直到Service 处理完毕返回处理结果。

20. Binder 请求的同步与异步

    很多人都会说，Binder 是对 Client 端同步，而对 Service 端异步，其实并不完全正确，在单次 Binder 数据传递的过程中，其实都是同步的。只不过，Client 在请求 Server 端服务的过程中，是需要返回结果的，即使是你看不到返回数据，其实还是会有个成功与失败的处理结果返回给 Client，这就是所说的 Client 端是同步的。至于说服务端是异步的，可以这么理解：在服务端在被唤醒后，就去处理请求，处理结束后，服务端就将结果返回给正在等待的 Client 线程，将结果写入到 Client 的内核空间后，服务端就会直接返回了，不会再等待 Client 端的确认，这就是所说的服务端是异步的。

21. APP 进程天生支持 Binder 通信的原理是什么

    Android APP 进程都是由 Zygote 进程孵化出来的。常见场景：点击桌面 icon 启动 APP，或者 startActivity 启动一个新进程里面的 Activity，最终都会由 AMS 去调用 Process.start() 方法去向 Zygote 进程发送请求，让 Zygote 去 fork 一个新进程，Zygote 收到请求后会调用 Zygote.forkAndSpecialize() 来 fork 出新进程,之后会通过RuntimeInit.nativeZygoteInit 来初始化 Andriod APP 运行需要的一些环境，而 binder 线程就是在这个时候新建启动的

    关键就是onZygoteInit：

    ```c++
    virtual void onZygoteInit()
        {
            sp proc = ProcessState::self();
            //启动新binder线程loop
            proc->startThreadPool();
        }
    ```

    ProcessState::self() 函数会调用 open() 打开 /dev/binder 设备，这个时候 Client 就能通过 Binder 进行远程通信；其次，proc->startThreadPool() 负责新建一个 binder 线程，监听 Binder 设备，这样进程就具备了作为 Binder 服务端的资格。每个 APP 的进程都会通过 onZygoteInit 打开 Binder，既能作为 Client，也能作为 Server，这就是 Android 进程天然支持 Binder 通信的原因。

22. APP 有多少 Binder 线程，是固定的吗

    我们知道 Android APP 进程在 Zygote fork 之初就为它新建了一个 Binder 主线程，使得 APP 端也可以作为 Binder 的服务端，这个时候 Binder 线程的数量就只有一个，假设我们的 APP 自身实现了很多的 Binder 服务，一个线程够用的吗？这里不妨想想一下 SystemServer 进程，SystemServer 拥有很多系统服务，一个线程应该是不够用的，对于Android4.3 的源码，其实一开始为该服务开启了两个 Binder 线程。还有个分析 Binder 常用的服务，media 服务，也是在一开始的时候开启了两个线程。

    可以看出Android APP上层应用的进程一般是开启一个 Binder 线程，而对于 SystemServer 或者 media 服务等使用频率高，服务复杂的进程，一般都是开启两个或者更多。

    来看第二个问题，Binder线程的数目是固定的吗？答案是否定的，Binder 驱动会根据目标进程中是否存在足够多的Binder 线程来告诉进程是不是要新建 Binder 线程。

23. 同一个线程的请求必定是顺序执行，即使是异步请求

    一般而言，Client 同步阻塞请求 Service，直到 Service 提供完服务后才返回，不过，也有特殊的，比如请求用ONE_WAY 方式，这种场景一般主要是用来通知，至于通知被谁消费，是否被消费压根不会关心。拿 ContentService服务为例子，它是一个全局的通知中心，负责转发通知，而且，一般是群发，由于在转发的时候，ContentService 被看做 Client，如果这个时候采用普通的同步阻塞势必会造成通知的延时发送送，所以这里的 Client 采用了oneway，异步。

    这种机制可能也会影响 Service 的性能，比如同一个线程中的 Client 请求的服务是一个耗时操作的时候，通过 oneway 的方式发送请求的话，如果之前的请求还没被执行完，则 Service 不会启动新的线程去响应，该请求线程的所有操作都会被放到同一个Binder线程中依次执行，这样其实没有利用Binder机制的动态线程池，如果是多个线程中的 Client 并发请求，则还是会动态增加 Binder 线程的，大概这个是为了保证同一个线程中的 Binder 请求要依次执行吧，这种表现好像是反过来了，Client 异步，而 Service 阻塞了，也就是说虽然解决了 Client 请求不被阻塞的问题，但是请求的处理并未被加速。

24. Binder协议中BC与BR的区别

    BC与BR主要是标记数据及 Transaction 流向，其中BC是从用户空间流向内核，而BR是从内核流线用户空间，比如Client 向 Server 发送请求的时候，用的是 BC_TRANSACTION，当数据被写入到目标进程后，target_proc 所在的进程被唤醒，在内核空间中，会将BC转换为BR，并将数据与操作传递该用户空间。

25. Binder 在传输数据的时候是如何层层封装的--不同层次使用的数据结构（命令的封装）

    ![binder-data](/images/binder/binder-data.jpg) 

26. ServiceManager addService的限制

    1. ServiceManager 其实主要的面向对象是系统服务，大部分系统服务都是由 SystemServer 进程总添加到ServiceManager 中去的，在通过 ServiceManager 添加服务的时候，是有些权限校验的，源码如下：

    ```cpp
    int svc_can_register(unsigned uid, uint16_t *name)
     {
        unsigned n;
        // 谁有权限add_service 0进程，或者 AID_SYSTEM进程
        if ((uid == 0) || (uid == AID_SYSTEM))
            return 1;
         for (n = 0; n < sizeof(allowed) / sizeof(allowed[0]); n++)
            if ((uid == allowed[n].uid) && str16eq(name, allowed[n].name))
                return 1;
        return 0;
    }
    ```

    可以看到 (uid == 0) 或者 (uid == AID_SYSTEM)的进程都是可以添加服务的，uid=0，代表root用户，而uid=AID_SYSTEM，代表系统用户    。或者是一些特殊的配置进程。SystemServer 进程在被 Zygote 创建的时候，就被分配了 UID 是 AID_SYSTEM（1000）

    ```java
    private static boolean startSystemServer()
            throws MethodAndArgsCaller, RuntimeException {
    /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,3001,3002,3003,3006,3007",
            "--capabilities=130104352,130104352",
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
    ```

    Android 每个 APP 的 UID，都是不同的，用了Linux的UID那一套，但是没完全沿用，这里不探讨，总之，普通的进程是没有权限注册到 ServiceManager 中的，那么 APP 平时通过 bindService 启动的服务怎么注册于查询的呢？接管这个任务的就是 SystemServer 的 ActivityManagerService。

27. bindService 启动 Service 与 Binder 服务实体的流程

    1. Activity 调用bindService：通过 Binder 通知 ActivityManagerService，要启动哪个 Service
    2. ActivityManagerService 创建 ServiceRecord，并利用 ApplicationThreadProxy 回调，通知 APP 新建并启动Service启动起来
    3. ActivityManagerService 把 Service 启动起来后，继续通过 ApplicationThreadProxy，通知 APP，bindService，其实就是让 Service 返回一个 Binder 对象给 ActivityManagerService，以便 AMS 传递给 Client
    4. ActivityManagerService 把从 Service 处得到这个 Binder 对象传给 Activity，这里是通过 IServiceConnection binder 实现。
    5. Activity 被唤醒后通过 Binder Stub 的 asInterface 函数将 Binder 转换为代理 Proxy，完成业务代理的转换，之后就能利用 Proxy 进行通信了。

28. Binder 通信过程

    1. 首先，一个进程使用 BINDER*SET*CONTEXT_MGR 命令通过 Binder 驱动将自己注册成为 ServiceManager；
    2. Server 通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager，ServiceManger 将其填入查找表。
    3. Client 通过名字，在 Binder 驱动的帮助下从 ServiceManager 中获取到对 Binder 实体的引用，通过这个引用就能实现和 Server 进程的通信。

29. Binder 通信中的代理模式

     A 进程想要 B 进程中某个对象（object）是如何实现的呢？毕竟它们分属不同的进程，A 进程 没法直接使用 B 进程中的 object。 

    前面我们介绍过跨进程通信的过程都有 Binder 驱动的参与，因此在数据流经 Binder 驱动的时候驱动会对数据做一层转换。当 A 进程想要获取 B 进程中的 object 时，驱动并不会真的把 object 返回给 A，而是返回了一个跟 object 看起来一模一样的代理对象 objectProxy，这个 objectProxy 具有和 object 一摸一样的方法，但是这些方法并没有 B 进程中 object 对象那些方法的能力，这些方法只需要把把请求参数交给驱动即可。对于 A 进程来说和直接调用 object 中的方法是一样的。

    当 Binder 驱动接收到 A 进程的消息后，发现这是个 objectProxy 就去查询自己维护的表单，一查发现这是 B 进程 object 的代理对象。于是就会去通知 B 进程调用 object 的方法，并要求 B 进程把返回结果发给自己。当驱动拿到 B 进程的返回结果后就会转发给 A 进程，一次通信就完成了