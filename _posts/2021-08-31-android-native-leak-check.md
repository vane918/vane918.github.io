---
layout: post
title:  "Android Native内存泄漏案例"
date:   2021-08-31 00:00:00
catalog:  true
tags:
    - native内存泄漏
    - 优化
    - Android
---



项目中遇到一个内存泄漏的情形：usb camera预览时出现了内存泄漏，但内存泄漏很小，测试一晚上泄漏20M内存左右。因此借此机会学习下当前市面上用于Android检测内存泄漏的工具。

以下内容是搬运字节旗下的西瓜团队的文章：

西瓜视频稳定性治理体系建设二：Raphael 原理及实践

[西瓜视频稳定性治理体系建设二：Raphael 原理及实践](https://mp.weixin.qq.com/s/RF3m9_v5bYTYbwY-d1RloQ)

# **背景**

Android 平台上的内存问题一直是性能优化和稳定性治理的焦点和痛点，Java 堆内存因为有比较成熟的工具和方法论，加上 hprof 快照作为补充，定位和治理都很方便。而 native 内存问题一直缺乏稳定、高效的工具，仅有的 malloc debug [6]不仅性能和稳定性难以满足需要，还存在 Android 版本兼容的问题。

# **现状**

事实上，native 内存泄漏治理一直不乏优秀的工具，已知的可用于调查 native 内存泄漏问题的工具主要有：LeakTracer、MTrace、MemWatch、Valgrind-memcheck、TCMalloc、LeakSanitizer 等。但由于 Android 平台的特殊性，这些工具要么不兼容，要么接入成本过高，很难在 Android 平台上落地。这些工具的原理基本都是：先代理内存分配/释放相关的函数（如：malloc/calloc/realloc/memalign/free），再通过 unwind 回溯调用堆栈，最后借助缓存管理过滤出未释放的内存分配记录。因此，这些工具的主要差异也就体现在代理实现、栈回溯和缓存管理三个方面。根据这些工具代理实现的差异，大致可以分为 hook 和 LD_PRELOAD 两大类，典型的如 malloc debug [5] 和 LeakTracer。

![malloc-leaktracer.png](/images/native-leak/native-leak.png)

## **malloc debug**

malloc debug 是 Android 系统自带的内存调试工具（官方 Native 内存调试 有相关介绍 **）** ，虽然没有额外的接入代码，但开启方式和核心功能等都受 Android 版本限制。

地址：[https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md](https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md)

![malloc.png](/images/native-leak/native-leak.png)

我们在线下尝试使用 malloc debug 监控西瓜视频 App（配置 wrap.sh）时发现，正常启动时间小于 1s 的机型（Pixel 2 & Android 10），其冷启动时间被拉长到了 11s+。而且在正常使用过程中滑动时的卡顿感非常明显，页面切换时耗时难以接受，监控过程中应用的使用体验极差。不仅如此，西瓜视频在 malloc debug 监控过程中还会遇到必现的栈回溯 crash（堆栈如下，《libunwind llvm 编年史》[8] 有相关分析）。

网友的使用分享：

[malloc debug 内存泄露案例分析_wozhouwang的博客-CSDN博客](https://blog.csdn.net/wozhouwang/article/details/103620184)

## **LeakTracer**

LeakTracer 是另一个比较知名的内存泄漏监控工具。

github地址：[https://github.com/fredericgermain/LeakTracer](https://github.com/fredericgermain/LeakTracer)。

其原理是：通过 LD_PRELOAD 机制抢先加载一个定义了 malloc/calloc/realloc/memalign/free 等同名函数的代理库，这样就全局代理了应用层内存的分配和释放，通过 unwind 回溯调用栈并过滤出疑似的内存泄漏信息。Android 平台上的 LD_PRELOAD 是被严格限制的，因为其没有独立的 unwind 实现，依赖系统的 unwind 能力，也会遇到 malloc debug 遇到的栈帧兼容问题；如果把 LeakTracer 集成到目标 so 里通过 override 方式实现代理，只能拦截到本 so 里显式的内存分配/释放，无法拦截到其他 so 和跨 so 调用的内存分配/释放。通过 native 插桩的方式也是如此，只能监控局部单纯的内存泄漏，无法全局监控内存使用。

网友使用分享：

[Android上使用LeakTracer检测native内存泄露_taohongtaohuyiwei的博客-CSDN博客](https://blog.csdn.net/taohongtaohuyiwei/article/details/104694747)

# **综合评估**

## **功能**

相对于 malloc debug 和 LeakTracer，Raphael 不仅支持 malloc/calloc/realloc/memalign/free，也支持监控 mmap/mmap64/munmap 等，使监控范围扩展到了线程、webview、Flutter、显存等，基本完全覆盖了 Android 平台上的 native 内存使用场景

## **性能**

Android 平台上的 native 内存泄漏检测通常都是在程序运行过程中进行的，栈回溯和缓存管理会消耗部分 CPU 和内存，带来一定的性能损失。Raphael 可配置的监控能力有很大的伸缩性，性能影响可以限制在可接受范围内，以下数据基于西瓜视频 App 32 位模式评测（中高端机型和 64 位下的性能更高）：

- CPU：32 位模式 & ≥1024 的监控阈值下，在低端机上 CPU 消耗< 3%
- 内存：32 位模式下默认会有约 16M 的虚拟内存消耗
- 帧率：32 位模式 & ≥1024 的监控阈值下，低端机上帧率没有明显变化

## **稳定性**

已开源的版本是基于开源 inline hook 实现的，在部分 Android 6 机型上存在卡死问题，除此之外暂未发现其他稳定性问题。此外，字节跳动这边早期的治理实践集中在线下，并基于 Raphael 建设完善了线下的防治体系，更为稳定的版本可以满足线上的监控需求，我们会在后续迭代开源。

## **治理实践**

Raphael 在字节跳动内部使用非常广泛，是字节跳动 native 协会指定的 native 内存泄漏检测工具。在治理实践中，Raphael 覆盖了几乎所有的 native 内存使用场景，辅助解决了大量的 native 内存泄漏和内存使用不合理的问题。接下来通过四个典型的案例简单介绍下 Raphael 的监控能力和基于 Raphael 的数据分析方法（应用自身的，Java 层的，webview 的，系统层的）

# 案例 使用Raphael 定位内存泄漏

综合以上分析和接入体验，我们不难发现，这些内存泄漏监控工具在 Android 平台上实际接入时基本都存在以下三个比较典型的问题：

- **流程繁琐**：需要配置 wrap.sh/root permission/setprop 等，受 Android 版本限制
- **兼容问题**：unwind 库存在严重的兼容性问题，libunwind_llvm 无法正确回溯 GNU 编译的栈帧
- **性能问题**：官方的 malloc debug 性能数据是损失 10 倍以上，实测西瓜开启后在中高端机上不可用

定位native的内存泄漏，最开始是使用android原生支持的**malloc debug，**但是正如上文所说的，我在基于Android10的rk3288的开发板上，正常情况下打开APP时1s内，但是打开**malloc debug**检测后，直接启动activity超时，因此放弃了**malloc debug。**后面又使用了**LeakTracer，**但也正如上文说的：只能拦截到本 so 里显式的内存分配/释放，无法拦截到其他 so 和跨 so 调用的内存分配/释放。通过 native 插桩的方式也是如此，只能监控局部单纯的内存泄漏，无法全局监控内存使用。

最后使用了Raphael来定位native的内存泄漏问题。

详细的说明请参考：

[https://github.com/bytedance/memory-leak-detector/blob/master/README_cn.md](https://github.com/bytedance/memory-leak-detector/blob/master/README_cn.md)

```
// 监控整个进程
Raphael.start(
    Raphael.MAP64_MODE|Raphael.ALLOC_MODE|0x0F0000|1024,
    "/storage/emulated/0/raphael", // need sdcard permission
    null
);
```

需要注意的是，上面启动检测的参数：1024的意思是只检测超过或等于1024bytes的泄漏，可以根据自己的项目修改该值。如我的项目是将其修改为4.

复现内存泄漏的情形后，使用adb发送广播来抓取目前的内存分配堆栈：

adb shell am broadcast -a com.bytedance.raphael.ACTION_PRINT -f 0x01000000

发送该广播后，在上面开启内存泄漏检测的参数，即上面的/storage/emulated/0/raphael目录下，会生成一份raphael文件，包括maps和report。

然后再隔一段时间发送抓取堆栈的广播：

adb shell am broadcast -a com.bytedance.raphael.ACTION_PRINT -f 0x01000000。

通过对比两次抓取的堆栈文件：report，就可以知道哪里发生内存泄漏了。

比如对比我的项目两次抓取的report：

![native-leak.png](/images/native-leak/native-leak.png)

根据前面的地址，然后通过addr2line，就可以找到对应发生内存泄漏的代码了。如我的案例：

![addr2line.png](/images/native-leak/addr2line.png)

对应的代码：

![create.png](/images/native-leak/create.png)

由于我的项目预览时两路流，每一路流都有一个线程，两路流轮流调用getThreadEvent函数，因此每次m_tid都和当前的线程不等，导致每次取流时都会调用上图红框的函数进行创建hEvent，从而导致内存泄漏。

**相关资料**

> 1. **Raphael 开源地址：**
>
>    https://github.com/bytedance/memory-leak-detector
>
> 2. **xHook 链接：**
>
>    https://github.com/iqiyi/xHook
>
> 3. **xDL 链接：**
>
>    https://github.com/hexhacking/xDL
>
> 4. **Android-Inline-Hook 链接：**
>
>    https://github.com/ele7enxxh/Android-Inline-Hook
>
> 5. **And64InlineHook 链接：**
>
>    https://github.com/Rprop/And64InlineHook
>
> 6. **malloc debug 链接：**https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md
>
> 7. **LeakTracer 链接：**
>
>    http://www.andreasen.org/LeakTracer/
>
> 8. [ **Android Camera内存问题剖析**](http://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247486499&idx=1&sn=1f38a8dd301d6fe1d0b62f7e027113de&chksm=e9d0c7c1dea74ed70621fb46b1f081626177610d98e1fbb4c4867099eb43edc051f61f1a2371&scene=21#wechat_redirect)
>
> 9.  **libunwind llvm 编年史：**
>
>   https://zhuanlan.zhihu.com/p/33937283
>
> 10. **ART 视角 | 如何让 GC 同步回收 native 内存：**
>
> ​    https://juejin.cn/post/6894153239907237902