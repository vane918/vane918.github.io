---
layout: post
title:  "深入理解Android相机体系结构(非原创)"
date:   2020-11-20 00:00:00
catalog:  true
tags:
    - camera
    - Android
---



## 1. 概述

  Android系统自2007年底被Google推出问世以来，已经走过13个春夏秋冬，历经多次的大大小小的迭代重构、架构调整，虽然时代年轮依旧滚滚，虽然每年技术依然在不断地推陈出新，但是到目前为止，依然可以窥见其接口与实现相分离的核心设计理念，所以其架构设计的优越性可见一斑，另外，随着智能手机的快速普及，面对这一庞大终端市场，作为系统中最重要的几个组件之一的相机系统也必定会作为主要战场在手机市场中与其它厂商展开竞争。近几年，谷歌针对相机框架体系进行了多次迭代优化，就而今的相机框架而言，整体架构设计十分优秀，作为一个相机系统开发者，个人感觉很有必要针对整个框架进行一次完整的梳理总结，所以便促使我写下该系列文章，以整个框架的梳理为出发点，深入介绍下Android 相机框架。

## 2.文章目录及链接

| 文章                 | 链接                                                         |
| -------------------- | ------------------------------------------------------------ |
| 相机简史             | [https://blog.csdn.net/u012596975/article/details/107136261](https://blog.csdn.net/u012596975/article/details/107136261) |
| 安卓相机架构概览     | [https://blog.csdn.net/u012596975/article/details/107136568](https://blog.csdn.net/u012596975/article/details/107136568) |
| 应用层               | [https://blog.csdn.net/u012596975/article/details/107137110](https://blog.csdn.net/u012596975/article/details/107137110) |
| 服务层               | [https://blog.csdn.net/u012596975/article/details/107137156](https://blog.csdn.net/u012596975/article/details/107137156) |
| 硬件抽象层           | [https://blog.csdn.net/u012596975/article/details/107137523](https://blog.csdn.net/u012596975/article/details/107137523) |
| 硬件抽象层实现       | [https://blog.csdn.net/u012596975/article/details/107138576](https://blog.csdn.net/u012596975/article/details/107138576) |
| 驱动层               | V4L2框架: [https://blog.csdn.net/u012596975/article/details/107137555](https://blog.csdn.net/u012596975/article/details/107137555)<br />高通KMD: [https://blog.csdn.net/u012596975/article/details/107138655](https://blog.csdn.net/u012596975/article/details/107138655  ) |
| 硬件层               | [https://blog.csdn.net/u012596975/article/details/107137883](https://blog.csdn.net/u012596975/article/details/107137883) |
| 安卓相机架构总结     | [https://blog.csdn.net/u012596975/article/details/107138177](https://blog.csdn.net/u012596975/article/details/107138177) |
| 手机相机的未来与发展 | [https://blog.csdn.net/u012596975/article/details/107138203](https://blog.csdn.net/u012596975/article/details/107138203) |