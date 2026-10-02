---
bibliography:
- honors/honors.bib
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^honors-topics|荣誉专题]]
  - [[#^the-linux-kernel|Linux 内核]]
    - [[#^what-kinds-of-kernels-are-there|内核有哪些种类？]]
    - [[#^system-calls-demystified|系统调用解密]]
  - [[#^containerization|容器化]]
    - [[#^what-is-a-container|什么是容器？]]


# 荣誉专题 ^honors-topics

**如果说我看得更远，那是因为我站在了巨人的肩膀上\[sic\]。** — **艾萨克·牛顿爵士**

本章收录了部分荣誉课程讲座（CS 296-41）的内容。这些专题面向希望更深入钻研 CS 341 各主题的学生。

## Linux 内核 ^the-linux-kernel

在 CS 341 整门课程中，你逐渐熟悉了系统调用——它是用户空间与内核交互的接口。这个内核究竟是如何工作的？内核到底是什么？在本节中，我们会更细致地探究这些问题，并为你在本课程中遇到的各种“黑箱”稍作解释。本章主要聚焦于 Linux 内核，因此除非另有说明，请假定所有示例都针对 Linux 内核。

### 内核有哪些种类？ ^what-kinds-of-kernels-are-there

就目前而言，你们大多数人或许已经熟悉了 Linux 内核，至少熟悉如何通过系统调用与它交互。你们中的一些人可能还研究过 Windows 内核，本章不会过多讨论；也可能了解过 `Darwin`，即 macOS 的类 UNIX 内核（BSD 的一个衍生版本）。那些挖掘得更深一些的同学，或许还接触过 `GNU HURD` 或 `Zircon` 这样的项目。

内核通常可以归为两类之一：宏内核（monolithic kernel）或微内核（micro-kernel）。宏内核本质上就是内核连同其所有相关服务合并成的单个程序。而微内核的设计目标则是保留一个*主要*组件，由它提供内核所需的最小功能。这通常意味着调度、内存管理与分页，以及 IPC——这些都是实现其他更高层功能所必需的功能。而更高层的功能（例如网络协议栈、文件系统和设备驱动）则作为独立的程序实现，并通过某种形式的 IPC（通常是 RPC）与内核交互。由于这种设计，微内核在传统上比宏内核要慢，原因是 IPC 的开销。

接下来我们的讨论将聚焦于宏内核，除非另有说明，**特别地**是 Linux 内核。

### 系统调用解密 ^system-calls-demystified

系统调用使用一条可由运行在用户空间的程序执行的指令，该指令会*陷入*内核（例如 x86-64 上的 `syscall` 指令，而不是 POSIX 信号）以完成调用。这包括把数据写入磁盘、与硬件直接交互，以及与获取或放弃权限相关的操作（例如成为 root 用户并获得全部能力）等动作。

为了满足用户的请求，内核会依赖 `kernel calls`。内核调用本质上是内核的“公开”函数——由其他开发者实现、供内核其他部分使用的函数。下面是一段内核调用的 man page 片段：

    Name

    kmalloc — allocate memory
    Synopsis
    void * kmalloc (	size_t size,
     	gfp_t flags);

    Arguments

    size_t size

        how many bytes of memory are required.
    gfp_t flags

        the type of memory to allocate.

    Description

    kmalloc is the normal method of allocating memory for objects smaller than page size in the kernel.

    The flags argument may be one of:

    GFP_USER - Allocate memory on behalf of user. May sleep.

    GFP_KERNEL - Allocate normal kernel ram. May sleep.

    GFP_ATOMIC - Allocation will not sleep. May use emergency pools. For example, use this inside interrupt handlers.

你会注意到有些标志被标记为可能导致睡眠。这告诉我们在某些特殊场景下能否使用这些标志——比如中断上下文，那里的速度至关重要，而可能阻塞或等待另一个进程的操作也许永远无法完成。

## 容器化 ^containerization

我们生活在一个规模空前庞大的时代，数十亿台设备接入互联网，因此我们需要能够帮助我们开发并维护可向上扩展的软件的技术。此外，随着软件复杂度上升，设计安全的软件也愈发困难，于是我们在开发应用时会受到新的约束。仿佛这还不够，像包管理器这类旨在简化软件分发与开发的努力，往往会带来自己的麻烦，导致损坏的软件包、无法解决的依赖关系，以及诸如此类如今已司空见惯的环境噩梦。虽然乍看之下这些问题彼此无关，但所有这些乃至更多问题，都可以通过向问题丢出 `containerization` 来解决。

### 什么是容器？ ^what-is-a-container

容器几乎就像虚拟机。从某种意义上说，容器之于虚拟机，就如同线程之于进程。容器是一种轻量级环境，它与宿主机共享资源和内核，同时把自己与宿主机上的其他容器或进程隔离开来。你可能在使用 `Docker` 这类技术时接触过容器，它大概是目前最广为人知的容器实现。
