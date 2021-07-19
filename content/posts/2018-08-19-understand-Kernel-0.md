---
title: 理解 Linux Kernel (0) - 概述
author: fangfeng
date: 2018-08-19
categories:
  - 技术
tags:
  - Linux
  - Kernel
---

## 概述

《理解Linux Kernel》系列最早开始于 2018 年中，当时我刚刚结束对 JVM (Java Virtual Machine, Java 虚拟机) [ClassFile 文件格式](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html)的学习。彼时，在*实现新编程语言*、*学习 Linux 内核实现*、*继续深入 JVM 源码*三个下一阶段的命题中，我选择了拥抱 Linux 源码。

时间过去了大半年，我已经简单地建立了对 Linux Kernel 的结构化认知。趁着现阶段有一些闲暇，整理过去的文章，以期删繁就简，并对错误的描述进行修正。

《理解 Linux Kernel》系列最早发布于[我的博客 (Utop's Blog)](https://www.ffutop.com/)。由于这些文章的完成完全出于个人兴趣，对每个内核子系统的学习往往浅尝辄止，但我依旧认为这个系列对初学者来说是大有裨益的。

本系列对内核的学习完全建立在 0.11、2.6.24 两个版本源码的基础上。从 0.11 版本入门，学习硬件启动阶段进行的主要操作、任务调度、文件系统等早期版本已初现雏形的子系统；从 2.6.24 版本进阶，学习网络、内存管理等子系统的实现。在此强烈推荐，直接阅读源码对理解的帮助最为深远。请随时备上两个版本的源码，以备深入理解。

<!--more-->

## 系列文章

=== 分割线 （以下待整理） ===

- [理解 Linux Kernel (0) - 概述](https://www.ffutop.com/2018-08-19-understand-Kernel-0/)
- [理解 Linux Kernel (1) - BIOS](https://www.ffutop.com/2018-08-19-understand-Kernel-1/)
- [理解 Linux Kernel (2) - 多任务切换](https://www.ffutop.com/2018-08-26-understand-Kernel-2/)
- [理解 Linux Kernel (3) - 操作系统启动](https://www.ffutop.com/2018-10-06-understand-Kernel-3/)
- [理解 Linux Kernel (4) - 任务调度](https://www.ffutop.com/2018-10-12-understand-Kernel-4/)
- [理解 Linux Kernel (5) - 文件系统](https://www.ffutop.com/2018-10-14-understand-Kernel-5/)
- [理解 Linux Kernel (6) - 文件操作（读、写）](https://www.ffutop.com/2018-11-11-understand-Kernel-6/)
- [理解 Linux Kernel (7) - 字符设备](https://www.ffutop.com/2018-12-28-understand-Kernel-7/)
- [理解 Linux Kernel (8) - 网络](https://www.ffutop.com/2019-01-15-understand-Kernel-8/)
- [理解 Linux Kernel (9) - I/O 多路复用](https://www.ffutop.com/2019-03-05-understand-Kernel-9/)
- [理解 Linux Kernel (10) - 执行的上下文](https://www.ffutop.com/2019-04-10-understand-Kernel-10/)

*以下是18年年中原文（稍作整理）*

## 开端

刚刚结束 JVM ClassFile 文件格式的学习，见识了 JVM 对 CPU 架构的模拟：

1. 有限而统一的指令集（不超过 256 个，可以用 1 字节维护）
2. 剔除寄存器概念而使用操作数栈作为替代
3. 使用局部变量表来模拟任务栈
4. 高度封装的成员变量和方法的寻址方式

Java 虚拟机的机制相当简洁。就寄存器而言，就已经极大的简化了 CPU 复杂的结构。更遑论不同 CPU 架构带来的多套指令集的问题。

其实我本来是想要继续学习 Hotspot 虚拟机的实现。但是，混杂的代码(C++, Java, 平台相关各种实现)给我带来了很大的阅读障碍。最根本的，我甚至找不到一个入口作为整个学习计划的起点。*当然，也与我根本没仔细去看有关，笑:)*。

受老大推荐，我开始看《程序员的自我修养——链接、装载与库》一书。此书对 ELF 格式的介绍，对静态链接与动态链接的深入剖析都给了我深刻的印象，向我展示了 C 语言更深入的一面。但是，macOS 给我带来了比较大的客观阻碍（即使使用 Docker 容器得到了一个可用的 Linux 环境，但与直接建立在硬件上的系统还是有所区别）。同时，书中的部分内容上下文不一致，给阅读带来了障碍。总之，此书中的内容不适合我我这种初学者逐一进行书中所述的全部实验。

多重因素最终使我决定学习 Linux Kernel，希望对操作系统的理解能够帮助我建立对各种用户层面应用的画像。

## 学习资料

1. Source Code of [linux-v0.11](https://www.oldlinux.org/Linux.old/kernel/0.1x/linux-0.11/) and [linux-2.6.24](https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/)
2. 《Linux 内核完全注释》
3. 《深入 Linux 内核架构》
4. Bochs 仿真器
5. [内核邮件组](https://vger.kernel.org/vger-lists.html)

## 软硬件描述

进行这方面的描述，是因为 Linux Kernel 将直接与底层硬件打交道(当然，这方面使用的 Bochs 仿真器)。
但是，准备诸如 .img (磁盘映像文件), 机器指令文件等都不得不借助于现有的平台。

- 操作系统: macOS Mojave 10.14.4 , Ubuntu 18.04 （云主机）
- 仿真器  : Bochs 2.6.9\_2 （macOS 与 Ubuntu 上相同）
- 更多    : 将直接在正文首次使用到时进行说明

*Read the Source Code. Have Fun.*

## 参考

\[1\]. 俞甲子. 程序员的自我修养[M]. 电子工业出版社, 2009.
\[2\]. 赵炯. Linux内核完全注释 内核版本0.11 V3.0[EB/OL]. https://www.oldlinux.org/, 2007

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
