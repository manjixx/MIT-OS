# Lec 03. OS Organization and System Calls

- [Lec 03. OS Organization and System Calls](#lec-03-os-organization-and-system-calls)
  - [3.1 课程回顾](#31-课程回顾)
  - [3.2 OS的隔离性](#32-os的隔离性)
    - [3.2.1 为什么需要隔离性](#321-为什么需要隔离性)
    - [3.2.2 Strawman Design /No OS：没有操作系统会怎样](#322-strawman-design-no-os没有操作系统会怎样)
      - [缺点一：无法实现multiplexing](#缺点一无法实现multiplexing)
      - [缺点二：内存无法隔离](#缺点二内存无法隔离)
    - [3.2.3 总结](#323-总结)
    - [3.2.4 举例](#324-举例)
      - [fork -- CPU抽象](#fork----cpu抽象)
      - [exec -- 内存抽象](#exec----内存抽象)
      - [file -- 磁盘抽象](#file----磁盘抽象)
      - [总结](#总结)
  - [3.3 OS的防御性](#33-os的防御性)
  - [3.4 硬件对强隔离的支持](#34-硬件对强隔离的支持)
    - [3.4.1 用户态与内核态（user/kernel mode）](#341-用户态与内核态userkernel-mode)
      - [处理器的两种操作模式](#处理器的两种操作模式)
      - [普通/特殊权限指令](#普通特殊权限指令)
      - [RISC-V privilege架构的文档](#risc-v-privilege架构的文档)
    - [3.4.2 虚拟内存](#342-虚拟内存)
  - [3.5 User/Kernel mode 切换](#35-userkernel-mode-切换)
    - [3.5.1 **ECALL**](#351-ecall)
    - [3.5.2 举例](#352-举例)
      - [fork系统调用](#fork系统调用)
      - [write系统调用](#write系统调用)
  - [3.6 宏内核 vs 微内核 （Monolithic Kernel vs Micro Kernel）](#36-宏内核-vs-微内核-monolithic-kernel-vs-micro-kernel)
    - [3.6.1 内核 - TCB（Trusted Computing Base)](#361-内核---tcbtrusted-computing-base)
    - [3.6.2 宏内核](#362-宏内核)
      - [什么是宏内核](#什么是宏内核)
      - [宏内核的优缺点](#宏内核的优缺点)
    - [3.6.3 微内核](#363-微内核)
      - [什么是微内核](#什么是微内核)
      - [微内核的优缺点](#微内核的优缺点)
    - [3.6.4 总结](#364-总结)
  - [3.7 编译运行Kernel](#37-编译运行kernel)
    - [3.7.1 xv6代码结构](#371-xv6代码结构)
    - [3.7.2 内核编译](#372-内核编译)
      - [编译过程](#编译过程)
      - [查看 `kernel.asm`文件](#查看-kernelasm文件)
      - [运行 XV6（不使用 gdb）](#运行-xv6不使用-gdb)
  - [3.8 QEMU底层原理](#38-qemu底层原理)
    - [3.8.1 QEMU的本质：硬件级仿真](#381-qemu的本质硬件级仿真)
    - [3.8.2 RISC-V的结构图(硬件架构)](#382-risc-v的结构图硬件架构)
    - [3.8.3 QEMU工作原理：指令级仿真](#383-qemu工作原理指令级仿真)
    - [3.8.4 学生问答](#384-学生问答)
  - [3.9 XV6 启动过程详解](#39-xv6-启动过程详解)
    - [3.9.1 概述](#391-概述)
    - [3.9.2 调试环境搭建](#392-调试环境搭建)
      - [3.9.2.1 QEMU启动（GDB服务端）](#3921-qemu启动gdb服务端)
      - [3.9.2.2 GDB客户端连接](#3922-gdb客户端连接)
    - [3.9.3 内核入口分析](#393-内核入口分析)
      - [3.9.3.1 初始指令分析](#3931-初始指令分析)
      - [3.9.2 执行环境特征](#392-执行环境特征)
    - [3.9.3 内核初始化流程](#393-内核初始化流程)
      - [3.9.3.1 进入main函数](#3931-进入main函数)
      - [3.9.3.2 consoleinit 验证](#3932-consoleinit-验证)
      - [3.9.3.3 初始化序列（main 函数）](#3933-初始化序列main-函数)
    - [3.9.4 用户空间启动](#394-用户空间启动)
      - [3.9.4.1 首个用户进程创建（`userinit`）](#3941-首个用户进程创建userinit)
      - [3.9.4.2 initcode 执行流程](#3942-initcode-执行流程)
      - [3.9.4.3 首个系统调用捕获（syscall）](#3943-首个系统调用捕获syscall)
      - [3.9.4.3 init 程序详解](#3943-init-程序详解)
      - [3.9.4.5 shell 启动验证](#3945-shell-启动验证)
    - [3.9.5 启动时序总结](#395-启动时序总结)

## 3.1 课程回顾

> **本节课内容**

- **Isolation(隔离性)**：设计操作系统组织结构的**驱动力**。
- **Kernel 和 User mode**：隔离操作系统内核和用户应用程序。
- **System calls**：应用程序转换到内核执行的基本方法，这样用户态应用程序才能使用内核服务。
- 以上内容如何以一种简单的方式在 XV6 中实现。

> **课程内容回顾**

**操作系统结构**

- 首先，会有类似于`shell，echo，find`或者任何你实现的工具程序，这些**程序**运行在操作系统之上。
- 操作系统又抽象了一些**硬件资源**，例如磁盘，CPU。
- 通常来说操作系统和应用程序之前的接口被称为**系统调用接口**（System call interface），我们这门课程看到的接口都是Unix风格的接口。基于这些Unix接口，你们在lab1中，完成了不同的应用程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MIn4HtztTgrF0S7cUjs%2Fimage.png?alt=media&token=1182ec32-78aa-4e00-aba8-82c90c050982)

lab1 主要目的：理解图中的应用程序到操作系统内核之间的接口，即**理解系统调用**。

## 3.2 OS的隔离性

- 隔离性（`isolation`）
- 隔离性的重要性

### 3.2.1 为什么需要隔离性

- **应用程序间需要强隔离：** 隔离性的核心思想相对来说比较简单。在用户空间有多个应用程序，例如`Shell，echo，find`。如果通过`Shell`运行`Prime`的代码（lab1中的一个部分）时，假设`Prime`代码出现了问题，此时不应该会影响到其他应用程序。如果`Shell`出现问题时，杀掉了其他的进程，这将会非常糟糕。因此，**在不同的应用程序之间需要有强隔离性**。
- **应用程序与操作系统间需要强隔离：** 操作系统某种程度上为所有的应用程序服务。当应用程序出现问题时，我们希望操作系统不会因此而崩溃。比如，当我们向操作系统传递了一些奇怪的参数，此时希望操作系统能够很好的处理异常情况。因此，**应用程序和操作系统之间需要有强隔离性**。

### 3.2.2 Strawman Design /No OS：没有操作系统会怎样

假设不使用传统意义上的操作系统，而是将其简化为一些库文件。

比如在 Python 中通过 import os 加载所需功能。此时，我们有一个 Shell 和一些应用程序（如 echo），它们直接运行并使用这些库函数与硬件交互。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KILhA85NwqYCibtS%2Fimage.png?alt=media&token=192f826a-63e9-4b27-a3f5-6f09b9dd63cb)

#### 缺点一：无法实现multiplexing
  
**如果没有操作系统，应用程序会直接与硬件交互**。应用程序可以直接访问CPU的多个核、磁盘与内存。应用程序和硬件资源之间没有一个额外的抽象层，如下图所示。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KoKptVF_4Wk7FvsK%2Fimage.png?alt=media&token=f9fb466f-444f-4b0c-8a9f-d2792034dc34)

实际上，从隔离性的角度来看，这并不是一个很好的设计。从本例可以看到这种设计是如何破坏隔离性。使用操作系统的一个目的是为了同时运行多个应用程序，因此CPU会周期性的从一个应用程序切换到另一个应用程序。

比如假设系统只有一个 CPU 核心，Shell 正在运行。此时若想切换执行其他程序，就必须依赖 Shell 主动释放 CPU。**程序本身要具备“自觉”，周期性地让出 CPU——称为协同调度（Cooperative Scheduling）。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2M2HKdZx0nGp_0ynT%2Fimage.png?alt=media&token=57595e20-1fe3-4cec-95d4-8a9340ec81a5)

但这种方式缺乏强制性和隔离性。一旦 `Shell` 出现死循环，它将永久占用 CPU，其他程序无法运行，甚至无法启动第三方工具终止`Shell`。

系统无法真正实现多任务并发运行，更谈不上**真正的multiplexing（CPU在多进程同分时复用）**。

#### 缺点二：内存无法隔离

在没有操作系统的情况下，所有应用程序直接使用物理内存，没有边界隔离。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7V45GuLznMv101NEG%2Fimage.png?alt=media&token=44f17f7f-7aca-420c-9210-0ea4ba218393)

比如 `Shell` 和 `echo` 同时运行，它们的数据和代码混杂在内存中。**如果 `echo` 无意间写入了属于 `Shell` 的地址（比如地址 1000），就会破坏 `Shell` 的数据或指令**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7W_gYEqzDbGZ65RA1%2Fimage.png?alt=media&token=5f133e02-7be1-4b42-85ac-f1e12f30e398)

这类问题不仅破坏程序功能，还极难调试和定位。只有通过操作系统实现虚拟内存和地址隔离，才能从根本上避免此类错误。

### 3.2.3 总结

操作系统的存在，核心是为了解决两个关键问题：

- 多路复用（`Multiplexing`）：让多个程序能公平、高效地共享 CPU、内存等资源；

- 隔离性（`Isolation`）：防止应用程序相互干扰，提高系统安全性与稳定性。

将操作系统简化为应用库的思路，在某些嵌入式或实时系统中可能成立，因为这些系统内的程序彼此信任、不强调隔离。但在大多数通用计算环境中，这种方式无法满足可靠性和并发性的基本需求。

### 3.2.4 举例

Unix接口通过抽象硬件资源，从而提供了强隔离性。接口经过精心设计以实现资源的强隔离，即`multiplexing`和物理内存的隔离。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRLmqk_Z0lli966feS%2Fimage.png?alt=media&token=c867e904-807f-45b9-8faa-5673a8ac988a)

| 资源   | 抽象机制        | 系统调用         | 实现方式                |
| ---- | ----------- | ------------ | ------------------- |
| CPU  | 进程（Process） | `fork`       | 调度器 + 分时复用          |
| 内存   | 虚拟地址空间      | `exec`       | 内存镜像加载 + 虚拟内存管理     |
| 文件系统 | 文件描述符       | `open/read`  | inode 表 + 缓存 + 权限控制 |
| 设备   | 文件接口        | `read/write` | 设备驱动程序抽象为“文件”       |

#### fork -- CPU抽象
  
通过`fork`可以创建进程。进程本身不是CPU，但是它们抽象了CPU，使应用程序能够运行计算任务。应用程序无法直接与CPU交互，只能与进程交互。**操作系统内核负责在多个进程之间切换 CPU，因此，操作系统并非直接将 CPU 暴露给应用程序，而是提供“进程”这一抽象，使其得以复用一个或多个 CPU。**

>**Q**：这里说进程抽象了CPU，是不是指一个进程使用了 CPU 的一部分，另一个进程使用另一部分？CPU 和进程的关系是什么？
>**Frans教授**：我这里真实的意思是，我们在实验中使用的RISC-V处理器实际上是有4个核。所以你可以同时运行4个进程，一个进程占用一个核。但是假设你有8个应用程序，操作系统会分时复用这些CPU核，比如说对于一个进程运行100毫秒，之后内核会停止运行并将那个进程从CPU中卸载，再加载另一个应用程序并再运行100毫秒。通过这种方式使得每一个应用程序都不会连续运行超过100毫秒。这里只是一些基本概念，我们在接下来的几节课中会具体的看这里是如何实现的。
>**Q**：所以多个进程不能同时使用同一个 CPU 核，对吧？
>**Frans教授**：是的，分时复用意味着同一时间一个核只运行一个进程，之后再切换。

#### exec -- 内存抽象

**我们可以认为exec抽象了内存**。调用 exec 时，我们传入一个文件名，**该文件对应一个应用程序的内存镜像，包含程序指令和全局数据。** 应用程序虽然可以扩展内存，但不能直接访问物理内存，例如不能直接访问物理地址 1000–2000。

这是因为操作系统提供了隔离与控制，**应用程序与物理内存之间通过操作系统进行管理**`。exec` 体现了应用程序无法直接操作物理内存这一抽象。

#### file -- 磁盘抽象

**file 抽象了磁盘**。应用程序无法直接操作物理磁盘，**在 Unix 中，所有对存储的访问都通过文件接口完成**。通过 file，可以读写、命名文件，而具体文件与磁盘块的映射则由操作系统负责。这样确保了磁盘资源的隔离，可以实现不同用户之间和同一个用户的不同进程之间的文件强隔离。

#### 总结

上述系统调用都是你们在之前 lab 中使用过的。可以看到，它们经过精心设计，统一抽象了底层硬件，使得操作系统能够在多个应用程序之间共享资源的同时，保证强隔离性。

> **学生提问**：更复杂的内核是否会尽量将进程调度到同一个 CPU 核，以减少 Cache Miss？
> **Frans教授**：是的。有一种东西叫做Cache affinity。现在的操作系统的确非常复杂，并且会尽量避免Cache miss和类似的事情来提升性能。我们在这门课程后面介绍高性能网络的时候会介绍更多相关的内容。
> **学生提问**：在 xv6 中，哪部分代码体现了进程对 CPU 的复用？
> **Frans教授**：有挺多文件与这个相关，但是proc.c应该是最相关的一个。两三周之后的课程中会有一个话题介绍这个内容。我们会看大量的细节，并展示操作系统的multiplexing是如何发生的。这节课只是一个概览，为后续内容打下基础。

***

## 3.3 OS的防御性

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRW8U6v4rtiUyoZziy%2Fimage.png?alt=media&token=8b5d9da1-01e8-4367-ba5b-cd763efb20f3)

**操作系统的防御性**：当操作系统上运行多个应用程序时，必须具备防御能力（Defensive）。

- **防御恶意或错误的应用程序输入**。操作系统需要能够抵御来自应用程序的攻击或错误。内核开发时，必须确保系统稳定，避免因应用程序传入错误参数而导致崩溃。若内核崩溃，所有应用程序都会失去服务，这是不可接受的。

- **确保应用程序隔离不可突破**。应用程序可能是恶意的，攻击者可能试图打破隔离，控制内核。一旦内核被控制，攻击者就掌握了所有硬件资源。因此，内核必须设计得足够防御，避免被应用程序攻破。

实际中，这些防御措施很难做到完美。Linux 内核偶尔会出现漏洞，使得应用程序能够突破隔离控制内核。**因此，内核安全需要持续关注和改进。开发内核时，防御性思维尤为重要，因为应用程序可能本身就是恶意的。** 为此，操作系统与应用程序之间必须有一道坚固的“墙”，保证内核能够执行任何需要的策略来维护安全。

## 3.4 硬件对强隔离的支持

**实现强隔离通常依赖硬件支持**。本节简要介绍关键硬件机制，后续课程会详细展开。

**硬件支持主要包括两部分：**

- 用户态（`user mode`）和内核态（`kernel mode`）：在RISC-V中，`kernel mode`被称为`Supervisor mode`，本质相同。
- 页表（`page table`）或虚拟内存（`Virtual Memory`）

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRW8U6v4rtiUyoZziy%2Fimage.png?alt=media&token=8b5d9da1-01e8-4367-ba5b-cd763efb20f3)

所有的支持多应用程序操作系统的处理器，具体的实现或许会有细微的差别，但必须具备这两项硬件功能。我们使用的RISC-V处理器即包含这些特性。

### 3.4.1 用户态与内核态（user/kernel mode）

首先，我们来看一下`user/kernel mode`，这里会以尽可能全局的视角来介绍，有很多重要的细节在这节课中都不会涉及。

#### 处理器的两种操作模式

为了支持`user/kernel mode`，**处理器会有两种操作模式**

- `User mode`：CPU 只能执行普通权限指令（`unprivileged instructions`）。

- `Kernel mode`：CPU 可执行特殊权限指令（`privileged instructions`）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJReB5Yo_RJjpIo3RnA%2Fimage.png?alt=media&token=d7abdd2b-ab8e-4281-9725-337e95f54982)

#### 普通/特殊权限指令

**普通权限的指令：** 例如寄存器加法（ADD）、减法（SUB）、跳转（JRC）和分支（BRANCH）等，这些指令所有应用程序均可执行。

**特殊权限指令：** 主要用于直接操作硬件和设置保护机制，如设置页表寄存器、控制中断等，仅允许内核执行。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRhsBp1tiieCyd1xXD%2Fimage.png?alt=media&token=4d834d3f-8458-4b1e-bdff-cf9f147a5441)

**举例**：若应用程序在`user mode`下尝试执行特殊权限指令，处理器会拒绝执行该指令。通常来说，这时会将控制权限从`user mode`切换到`kernel mode`，当操作系统拿到控制权之后，由操作系统决定如何处理,或许会杀掉进程，因为应用程序执行了不该执行的指令。

#### RISC-V privilege架构的文档

下图展示了`RISC-V`的特殊权限指令文档。未来一个月内，你们将频繁接触这些特殊权限指令。这里我们先对这些指令有一些初步的认识：**应用程序不应执行这些指令，内核专属。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRjawN3LdVFfbgHTYB%2Fimage.png?alt=media&token=08f5a718-9279-4df6-a68a-215e17d90123)

> **学生提问**：谁负责检测当前模式并允许或拒绝指令执行？怎么判断当前是user还是kernel mode？
> **Frans教授**：处理器内部有一个标志位，用于区分`user mode（1）`和`kernel mode（0）`。当特殊权限指令在`user mode`执行时，会被拒绝，就像运算中除以0一样出错。
> **同一个学生继续问**：如何更改这个标志位？是特殊权限指令还是普通指令？
> **Frans教授**：你认为是什么指令更新了那个bit位？是特殊权限指令还是普通权限指令？（等了一会，那个学生没有回答）。很明显，**设置那个bit位的指令必须是特殊权限指令**，应用程序不能直接修改，否则会绕过保护机制。这保证了模式切换的安全性。

许多同学都已经知道了，**实际上RISC-V还有第三种模式称为`machine mode`**。在大多数场景下，会忽略这种模式，本课程也不太会介绍这种模式。 所以实际上我们有三级权限（`user/kernel/machine`），而不是两级(`user/kernel`)。

> **学生提问**：考虑到安全性，所有的用户代码都会通过内核访问硬件，但是有没有可能一个计算机的用户可以随意的操纵内核？
> **Frans教授**：不会，只要设计严谨，这种情况就能避免。某些程序或用户（如root）可能拥有额外权限以执行安全相关操作，但普通用户不能随意操作内核。
> **同一个学生提问**：那BIOS呢？BIOS会在操作系统之前运行还是之后？
> **Frans教授**：BIOS是一段固化在计算机中的代码，会在操作系统启动前运行。它必须是可信且无恶意的，因为它负责启动操作系统。

> **学生提问**：之前提到，设置处理器中`kernel mode`的 bit 位的指令是一条特殊权限指令，既然设置处理器模式位的指令是特殊权限指令，用户程序怎么能让内核执行这些指令？用户程序无法直接修改模式位。
> **Frans教授**：你说得对，这是设计使然。实际上，**用户程序不能直接执行特殊权限指令，而是通过系统调用来切换到kernel mode。当用户执行系统调用（如`ECALL`指令）时，会触发一个软中断（`software interrupt`），处理器转向操作系统预设的中断向量表，执行对应的内核中断处理程序**。这样，系统完成了从user mode到kernel mode的安全切换，并代替用户执行需要特殊权限的指令。

### 3.4.2 虚拟内存

接**下来介绍支持强隔离性的另一项硬件特性——虚拟内存**。几乎所有现代CPU都支持虚拟内存，下一节课会更深入讲解，这里先做简要介绍。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSIF-TGSf1vxlD7etq%2Fimage.png?alt=media&token=aa213fe5-d78d-4520-ba08-24f8936cdfdf)

- 处理器内含页表（`page table`），**用于将虚拟地址映射到物理地址。**

- **每个进程拥有独立的页表**，确保它只能访问映射到自身页表的物理内存。**操作系统负责设置页表，避免不同进程间物理内存重叠**。这样，一个进程无法访问或伪造访问其他进程的物理内存，实现了内存的强隔离。

页表定义了进程的内存视图，**每个进程都拥有从${0}$到${2^n}$的虚拟地址空间**，如下图所示：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSJsZ87csgExhbDSw1%2Fimage.png?alt=media&token=bd6c9aa4-56ce-4188-afea-416b17060c1b)

比如，`ls`程序和`echo`程序都从虚拟地址`0`开始，但操作系统会将它们映射到不同的物理地址，互相无法访问对方内存。

同理，内核（如XV6）也拥有独立的地址空间，并且与应用程序完全独立。

## 3.5 User/Kernel mode 切换

`User mode` 与 `Kernel mode` 是用户空间和内核空间之间的边界。用户程序运行在 `user mode`，操作系统代码运行在 `kernel mode`。

上一部分中我们**用矩形描述了程序的内存空间**，但是**没有描述如何从一个矩形将控制权转移到另一个矩形的**，而很明显这种转换是需要的。(但没有说明**程序如何将控制权从用户态切换到内核态**。而这种切换对于调用系统服务是必要的。)

例如：

- `ls`程序运行的时候，会调用`read/write`系统调用；

- `Shell`程序会调用`fork`或者`exec`系统调用

因此，必须有一种机制**使得用户程序将控制权以一种协同的方式转移到内核（Entering Kernel）**，安全地进入内核态，请求操作系统提供服务。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSMyew-n45ZU01CtUw%2Fimage.png?alt=media&token=0a1dab14-28d2-4c5e-a044-0cc6902140b3)

### 3.5.1 **ECALL**

在 RISC-V 架构中，使用专门的 `ECALL` 指令实现从用户态切换到内核态。当一个用户程序想要将程序执行的控制权转移到内核，它只需要执行`ECALL`指令，并传入一个数字。

- 用户程序执行 `ECALL`，并通过寄存器传入一个数字参数；这个数字参数代表了希望调用的 系统调用编号（`syscall number`）；

- `ECALL` 会触发一个 陷入（`trap`），跳转到操作系统中事先定义好的一个入口地址；XV6 中有一个**唯一的系统调用接入点**，每一次**应用程序执行ECALL指令**，应用程序都会**通过这个接入点进入到内核中。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXAkJxD8pTZOm1Tay_%2Fimage.png?alt=media&token=cf3e26a2-3c26-43b8-aee0-6d5787f8dcf5)

### 3.5.2 举例

#### fork系统调用

当 `Shell` 或其他用户程序需要创建一个新进程时，会使用 `fork` 系统调用。但这个调用并不是直接跳转到内核中的 `fork()` 函数。

其具体执行流程如下：

1. **用户空间发起调用`fork()`:**，应用程序不直接调用内核函数，而是通过标准库封装函数（如 `fork()/write()`）发起。封装函数会执行 `ECALL` 指令，触发软中断，并传入系统调用号（唯一标识操作类型）。
2. **硬件特权切换：** ECALL 触发后：CPU 从用户模式（`User Mode`）切换至内核模式（`Kernel Mode`）。处理器跳转至内核预设的统一入口点（如 `syscall` 函数，位于 `syscall.c`）。
3. 内核分发与执行：解析传入的系统调用号；路由至对应的内核处理函数（如 `sys_fork/sys_write`）；执行实际操作（如创建进程/写入数据）。

如下图所示，用户空间和内核空间之间由一条明确的界限分隔，用户程序不能直接调用内核函数，只能通过 ECALL 指令请求内核服务：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXO2n90L0ziqU8mTcg%2Fimage.png?alt=media&token=754f49c1-58a2-42d5-9427-094fc95ab613)

#### write系统调用

`write` 系统调用的过程与 `fork` 类似。用户程序不能直接访问内核中的写操作函数，而是通过一套标准流程请求内核完成写操作：

1. **用户空间发起调用`write（）`:**，应用程序不直接调用内核函数，而是通过标准库封装函数（如 `fork()/write()`）发起。封装函数会执行 `ECALL` 指令，并传入系统调用号（唯一标识操作类型）。
2. **硬件特权切换：** ECALL 触发后：CPU 从用户模式（`User Mode`）切换至内核模式（`Kernel Mode`）。处理器跳转至内核预设的统一入口点（如 `syscall` 函数，位于 `syscall.c`）。
3. 内核分发与执行：解析传入的系统调用号；路由至对应的内核处理函数（如 sys_fork/sys_write）；执行实际操作（如创建进程/写入数据）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXQ4BUMhscu13BPT1v%2Fimage.png?alt=media&token=92170af8-075f-4d3a-be59-d50852bba34c)

>**学生提问**：操作系统在什么时候检查是否允许执行fork或者write？现在看起来应用程序只需要执行ECALL再加上系统调用对应的数字就能完成调用，但是内核在什么时候决定这个应用程序是否有权限执行特定的系统调用？
>**Frans教授**：是个好问题。实际上，内核会在每一个系统调用的具体实现中进行权限检查。例如，在 write 系统调用中，内核会验证传入的地址是否属于调用该系统调用的用户进程。如果用户程序传入了一个非法地址（比如指向其他进程或内核空间），内核就可以拒绝这个请求。换句话说，**权限检查是在内核中实际执行系统调用逻辑的地方进行的，而不是在 ECALL 的入口处就一刀切判断。**

>**学生提问**：当应用程序表现的恶意或者就是在一个死循环中，内核是如何夺回控制权限的？
>**Frans教授**：内核会通过硬件定时器周期性的打断正在运行的用户程序，即所谓的定时中断。定时器到期之后，会触发一个中断，切换到内核态，之后内核就有了控制能力并可以重新调度CPU到另一个进程中。我们接下来会看一些更加详细的细节。

>**学生提问**：这其实是一个顶层设计的问题，是什么驱动了操作系统的设计人员使用编程语言C？
>**Frans教授**：啊，这是个好问题。C提供了很多对于硬件的控制能力，比如说当你需要去编程一个定时器芯片时，这更容易通过C来完成，因为你可以得到更多对于硬件资源的底层控制能力。所以，如果你要做大量的底层开发，C会是一个非常方便的编程语言，尤其是需要与硬件交互的时候。当然，不是说你不能用其他的编程语言，但是这是C成功的一个历史原因。
>**学生提问**：为什么C比C++流行的多？仅仅是因为历史原因吗？有没有其他的原因导致大部分的操作系统并没有采用C++？
>**Frans教授**：我认为有一些操作系统是用C++写的，这完全是可能的。但是大部分你知道的操作系统并不是用C++写的，这里的主要原因是Linus不喜欢C++，所以Linux主要是C语言实现。

## 3.6 宏内核 vs 微内核 （Monolithic Kernel vs Micro Kernel）

### 3.6.1 内核 - TCB（Trusted Computing Base)

通过上述学习，应用程序可以通过系统调用（使用 `ECALL` 指令）将控制权交给操作系统内核。接管控制权后，内核负责具体功能的实现，并对传入参数进行检查，以防止因恶意或错误的参数导致系统失控。

因此，**内核常被视为一个可信计算空间（Trusted Computing Base，TCB）**，在安全领域的术语中，这一称呼尤为常见。其应满足如下诉求：

- **内核首先要是正确且没有Bug：** 这是被称为TCB的基础。一旦内核存在漏洞，攻击者就可能利用这些缺陷，将其转化为安全漏洞，从而破坏操作系统的隔离性，甚至取得对内核的控制权。因此，确保内核的稳定性与安全性至关重要。

- **内核必须始终以防御性的视角看待用户进程（内核必须要将用户应用程序或者进程当做是恶意的）**：换言之，操作系统的设计者在开发内核时，必须以“用户程序可能是恶意的”为前提，采用“默认不信任”的安全思维来实现内核的功能。这种防御性编程理念在实际操作中非常困难，尤其是当操作系统功能逐渐丰富、代码规模迅速扩大时，很多逻辑变得不再直观。几乎所有广泛使用的现代操作系统都曾经或仍然存在安全漏洞——这些问题即使被修复，过段时间又可能出现新的漏洞。这并不令人意外，因为内核承担着大量复杂、底层且极易出错的职责：它需要直接操纵硬件资源、维护内存和进程隔离、执行严格的权限检查等任务。即便一处轻微的疏忽，也可能引发严重的 Bug，并被黑客利用。因此，构建一个完全无漏洞的内核，几乎是一项不可能完成的任务。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJXWRY7_m6ArPVBNgvJ%2F-MJZck3H4EAuhFVSva0U%2Fimage.png?alt=media&token=2a170d6b-b942-4ac5-b943-85f4f2bc7f1d)

### 3.6.2 宏内核

**一个常见的问题是：哪些程序应该运行在内核态（kernel mode）？**

显然，所有**与系统安全、资源访问密切相关的敏感代码都必须运行在内核态**，因为这些代码属于可信计算基（Trusted Computing Base, TCB）。

在系统架构设计中，我们通常会在用户态（`user mode`）和内核态（`kernel mode`）之间划定一个清晰的边界：上层是应用程序，运行在用户态；下层是操作系统内核代码，运行在内核态。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbThlJrd7ZaArRiPDt%2Fimage.png?alt=media&token=1eee503b-5a1b-46b3-9d0b-0dfb2e740603)

#### 什么是宏内核

一种实现方案是**将整个操作系统作为一个统一的程序运行在内核态中（kernel mode）**。这是**多数传统 Unix 操作系统**采用的设计方式。例如，在教学操作系统 XV6 中，所有操作系统服务均运行于内核模式。这种设计被称为 **宏内核架构（Monolithic Kernel Design）**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbVJDYYOlA_ZnJBg_i%2Fimage.png?alt=media&token=11b51e28-9327-450e-a077-9cb3488a2015)

#### 宏内核的优缺点

- **缺点（安全性问题）**
从安全性角度看，宏内核的一个主要问题是：所有操作系统组件都运行在内核态中，意味着一个组件中的 Bug 可能影响整个系统的稳定性与安全性。据统计，平均每 3000 行代码就可能包含几个 Bug，操作系统代码动辄上百万行，因此在宏内核中运行大量代码，无疑增加了产生严重漏洞的风险。

- **优点（性能与集成）**
宏内核也有显著的性能优势。一个操作系统包含众多子系统，如文件系统、虚拟内存、进程管理等，**在宏内核中，这些模块作为一个整体协同运行，模块间无需跨边界通信，调用开销低，协作效率高**。这种高度集成的设计有助于系统达到良好的运行性能。例如 Linux 内核就是一个典型的宏内核，其性能表现非常优异。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbYtEBzpGpaCAFdeZk%2Fimage.png?alt=media&token=2e005add-cd05-4201-9fab-5149d38765ab)

### 3.6.3 微内核

#### 什么是微内核

与宏内核相对，**微内核设计（`Microkernel Design`）** 的核心思想是：尽可能减少在内核态中运行的代码量。在这种架构中，虽然仍然保留了一个操作系统内核，**但该内核只包含最基本、最关键的功能模块**。

**典型的微内核只在 `kernel mode` 中提供以下几类功能：**

- 进程间通信机制（`IPC，Inter-Process Communication`），通常通过**消息传递（message passing）**实现；

- 最小限度的虚拟内存支持（如页表管理）；

- 基本的 CPU 时间分配和调度功能。

- 除此之外，其他大部分传统意义上的操作系统功能，比如文件系统、设备驱动、网络协议栈、甚至虚拟内存系统的高级部分，都被移出内核，以用户态程序的形式运行。

也就是说，**微内核系统中仍然存在用户态与内核态的边界，但我们将许多原本运行在内核中的子系统，转换为了运行在用户空间的独立服务**。比如文件系统，在微内核中可能只是一个普通的用户程序，与 `shell、echo` 命令处于同一层次。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbb2rd6KN3AuCoErQ-%2Fimage.png?alt=media&token=a93a6b20-563f-42d6-8284-daaaa9a0d254)

#### 微内核的优缺点

**优点：更高的安全性与可靠性**：由于在内核中运行的代码量大大减少，意味着潜在的漏洞数量也更少。代码越少，出错的可能性越低，这有助于提升系统的稳定性与安全性。微内核架构也更适合进行模块化开发与形式化验证。

**缺点：系统调用的效率降低**：微内核虽然简洁，但带来了额外的通信与性能开销。以 `Shell` 调用 `exec` 与文件系统交互为例，整个过程如下：

- Shell 将执行请求发送给微内核中的 IPC 模块；

- 微内核识别这是发送给文件系统的消息，并转发到文件系统服务；

- 文件系统完成处理后，回传结果消息至 IPC 模块；

- IPC 模块再将结果转发回 Shell。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbcquRoZotyofyd2sh%2Fimage.png?alt=media&token=71f32a22-7654-4116-9dee-a40537c01a13)

上述程中，涉及两次用户态与内核态的切换。相比之下，在宏内核中，仅需一次系统调用即可完成同样的任务。

**因此，微内核的主要挑战在于性能瓶颈，表现为：**

- 多次的用户态/内核态切换所带来的 CPU 上下文切换开销；

- 不同子系统（如文件系统与虚拟内存管理）运行在隔离的进程中，很难实现内存资源的高效共享（如 page cache），降低整体性能。

### 3.6.4 总结

本节介绍了**宏内核（Monolithic Kernel）**与**微内核（Microkernel）**之间的基本区别。需要注意的是，这里的讨论相对简化笼统，现实中的操作系统设计往往更复杂，许多系统采用的是介于两者之间的**混合型结构（Hybrid Kernel）**。

从实际情况来看：

- **历史原因与兼容性因素**导致大多数桌面操作系统（如 Linux、Windows）都采用了宏内核结构。
- **对性能要求极高的系统**，例如数据中心的服务器操作系统，通常也选择宏内核，因为宏内核允许紧密集成、降低上下文切换，从而带来更高的运行效率。Linux 的优秀性能是其在此类场景中广泛使用的主要原因。
- 相比之下，**许多嵌入式系统**（如 Minix、QNX、Cell OS）更倾向于采用微内核设计。微内核的模块化、可验证性与安全性更符合嵌入式系统对高可靠性和安全隔离的需求。

如果你尝试**从零开始开发一个操作系统**，采用微内核架构往往是一个合理的选择，它简洁、可控，更容易逐步构建和验证。但如果已经拥有了一个成熟的宏内核系统（如 Linux），将其**重构为微内核结构的代价非常高**，涉及大量系统模块的拆分与重写，这种工程成本与风险使得实际动机较低。更现实的做法是继续在宏内核的基础上**演进与扩展功能**。

**值得注意的是：XV6 是一个典型的宏内核设计**，和早期的大多数 Unix 系统一样。在本课程的后续内容中，我们还将深入探讨微内核设计的相关机制与案例，帮助你理解不同操作系统架构在设计哲学与实现上的权衡。

以下是关于 **宏内核（Monolithic Kernel）** 与 **微内核（Microkernel）** 的结构化对比表，总结了它们在架构设计、性能、安全性、可扩展性等方面的核心区别：

| 维度            | 宏内核（Monolithic Kernel）           | 微内核（Microkernel）             |
| ------------- | -------------------------------- | ---------------------------- |
| **架构特点**      | 所有操作系统服务（如文件系统、驱动、内存管理等）都运行在内核态中 | 仅保留最小核心功能于内核（如调度、IPC、基本内存管理） |
| **模块位置**      | 文件系统、驱动、网络协议栈等都在内核中              | 大部分模块如文件系统、驱动等运行在用户态         |
| **用户态/内核态切换** | 少：调用系统服务通常只需一次切换                 | 多：依赖消息传递，通常需要两次或更多切换         |
| **性能**        | 性能优越，紧耦合设计减少上下文切换开销              | 性能相对较低，频繁的消息传递带来额外开销         |
| **安全性**       | 内核庞大，Bug 面广，一个模块失误可能危及整个系统       | 内核代码少，模块隔离更强，理论上更安全          |
| **稳定性**       | 崩溃风险集中，某一模块出错可能导致整个系统崩溃          | 模块彼此独立，一个模块出错通常不会影响其他模块      |
| **开发难度**      | 实现较直接，模块间共享容易                    | 设计复杂，需要良好的消息通信机制             |
| **可维护性**      | 模块集成紧密，维护时需小心谨慎                  | 各模块独立，易于测试与调试                |
| **典型系统**      | Linux、Windows NT、Unix、XV6        | Minix、QNX、L4、GNU Hurd        |
| **适用场景**      | 桌面操作系统、服务器、高性能计算场景               | 嵌入式系统、安全性要求高的系统              |
| **可验证性**      | 难以形式验证，代码量大、模块耦合多                | 容易进行形式化验证，代码体积小              |
| **扩展性**       | 较差，添加新模块需修改内核                    | 好，新模块可作为用户空间服务添加             |

***

## 3.7 编译运行Kernel

### 3.7.1 xv6代码结构

首先，我们来看一下代码结构，**XV6 的代码主要有三个部分组成：**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJpn93PbVXmyk3cWz9q%2F-MJq7-1lrR0BNbiAaazr%2Fimage.png?alt=media&token=15dcbc6d-ed46-4bd1-bb4b-f71609e413c1)

**1. `kernel` 目录，内核代码**

`ls kernel`后，可以发现该目录**包含了 xv6 操作系统的所有内核源文件**。由于 xv6 采用的是宏内核设计（Monolithic Kernel），这里所有的文件（如进程调度、文件系统、系统调用处理等）会被编译成一个叫做`kernel`的二进制文件，然后这个二进制文件会被**运行在内核态`kernle mode`中。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJdjbpZW0Es0hM8rNrS%2Fimage.png?alt=media&token=d28e7220-ad13-4f76-9b5e-4d414dda7e3c)

**2. `user` 目录 —— 用户态程序**

该目录下的文件是运行在 **用户态（user mode）** 的程序。包括常见的用户工具（如 sh、ls、echo 等）和一些测试程序。这也是为什么一个目录称为`kernel`，另一个目录称为`user`的原因。
  
**3. `mkfs` 目录 —— 文件系统构建工具**

该目录中的代码负责生成一个 **空白文件系统镜像（filesystem image）**。该镜像将在 xv6 启动时被挂载，并作为根文件系统使用。

### 3.7.2 内核编译

#### 编译过程

理解内核的编译过程非常重要。虽然你可能已经执行过内核编译，但深入理解其步骤很有价值。**XV6内核的编译过程遵循标准流程，并由Makefile（位于XV6项目根目录）驱动**：

**1. 预处理与编译 (C -> Assembly)：**

- Makefile 选取一个内核C源文件（例如 proc.c）。

- 调用 gcc 编译器，将其编译成针对 RISC-V 架构的汇编语言文件（例如 proc.s）。

**2. 汇编 (Assembly -> Object Code)：**

- 调用汇编器（如 as），将汇编文件 proc.s 翻译成机器码，生成目标文件 proc.o。这个文件是二进制格式（通常是 ELF 格式），包含特定部分的代码和数据，但尚未进行最终链接。

**3. 链接 (Linking Object Files)：**

- 链接器（如 ld）被调用。它的任务是收集编译阶段生成的所有 .o 文件（例如 proc.o, pipe.o 等）。

- 链接器解析这些文件之间的符号引用（如函数调用、变量访问），将它们合并、重定位，并组合生成最终的可执行内核文件（通常命名为 kernel 或类似名称）。这个文件就是将在 QEMU 中运行的操作系统内核映像。

**4. 辅助文件生成 (kernel.asm)：**

- 为了方便调试，Makefile 通常还会生成一个名为 kernel.asm 的文件。这个文件包含了内核完整、反汇编后的汇编代码，并标注了对应的内存地址。这个文件极其有用，当你在调试器（如 gdb）中遇到问题时，可以通过查阅 kernel.asm 精确定位是哪个具体的机器指令导致了错误。

**图示流程概览：**

- `proc.c` -> (gcc) -> `proc.s` -> (Assembler) -> `proc.o` -> (Linker + other .o files) -> `kernel + kernel.asm`

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgXiV2KBGQeuPgX4Bj%2Fimage.png?alt=media&token=80f80f91-adbc-48c7-9767-e6db633cb141)
  
- Makefile会为所有内核文件做相同的操作，其他文件（如 `pipe.c`）遵循相同的流程：`pipe.c` -> `pipe.s` -> `pipe.o`

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgWjAGE9ASuK5Hbw2k%2Fimage.png?alt=media&token=cac68445-2b05-4a57-8435-6d495baed104)

#### 查看 `kernel.asm`文件

接下来查看`kernel.asm`文件，`kernel.asm` 文件提供了内核在汇编层面的完整视图(用汇编指令描述的内核)：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgYi8MwG-QEqdvZ63Q%2Fimage.png?alt=media&token=d82387cf-bc09-47a2-a9d0-7ff0eb0cd2b2)

- 内核的入口指令通常位于地址 `0x80000000`（这是 RISC-V 上许多内核约定的起始地址）。

- 第一条指令是 `auipc（Add Upper Immediate to PC）`，这是一条常见的用于构建地址的 RISC-V 指令。

- 第二列（例如 `0x0000a117, 0x83010113, 0x6505`）代表什么？
  - “这是汇编指令的16进制表现形式对吗？”
  - 解答： 完全正确！这些数值就是 `auipc` 等指令的机器码二进制编码（以十六进制形式显示）。每条汇编指令在 CPU 层面都对应一个特定的二进制数值。`kernel.asm` 显示这些编码对于深入理解程序在硬件层面的执行非常有帮助，尤其是在使用 gdb 进行底层调试时，查看这些机器码有助于精确追踪执行流。

这里你们可能已经注意到了，第一个指令位于地址0x80000000，对应的是一个RISC-V指令：auipc指令。

#### 运行 XV6（不使用 gdb）

当在命令行执行 `make`（或 `make qemu`）时：

- `make` 程序读取 `Makefile` 中的指令。
- 按照上述流程编译所有内核源文件。
- 最终调用 QEMU 模拟器来启动编译好的 XV6 内核（这里本质上是通过C语言来模拟仿真RISC-V处理器）。命令类似于：

```bash
qemu-system-riscv64 -kernel kernel/kernel -m 128M -smp 3 -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgd7lBxdCSukMsxDGi%2Fimage.png?alt=media&token=bfcf8e8d-34d8-44a6-b84b-e378bde5bc09)

关键启动参数解析：

- `kernel kernel/kernel`: 指定要运行的内核可执行文件路径。

- `m`: ISC-V虚拟机将会使用的内存数量。

- `smp`: 虚拟机可以使用的CPU核数。

- `drive`: 虚拟机使用的磁盘驱动

## 3.8 QEMU底层原理

### 3.8.1 QEMU的本质：硬件级仿真

`QEMU` 不是简单的应用程序，而是一个完整的硬件仿真平台。当使用 `QEMU` 运行内核时，应将其视为真实的物理主板系统,该仿真主板包含：

- `RISC-V`处理器芯片
- 物理内存模块
- 外设接口（网络/USB/PCI-E等）
- 电源开关与时钟电路

XV6操作系统正是在这个虚拟硬件平台上运行，管理与真实硬件完全一致的资源。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJghmTlvFOKAnfGeveB%2Fimage.png?alt=media&token=2e8f081f-1f48-43e8-8486-0352732fd28e)

### 3.8.2 RISC-V的结构图(硬件架构)

RISC-V硬件架构结构图如下，包含以下核心组件，QEMU精确模拟了这些元素：

- 4个核：U54 Core 1-4
- L2 cache：Banked L2
- 连接DRAM的连接器：DDR Controller
- 各种连接外部设备的方式，比如说UART0，一端连接了键盘，另一端连接了terminal。
- 以及连接了时钟的接口：Clock Generation

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgjgTH1N3waADySy12%2Fimage.png?alt=media&token=a66a77d5-f019-4ddb-921e-1a74fd055567)

我们后面会讨论更多的细节，但是这里**基本上就是RISC-V处理器的所有组件**，你通过它与实际的硬件交互。

注：QEMU的仿真实现与真实硬件（如SiFive开发板）非常相似，是硬件功能在软件层面的重建。

### 3.8.3 QEMU工作原理：指令级仿真

直观来看`QEMU`本质是开源C程序，其核心为通过动态二进制翻译实现处理器仿真。主循环流程如下：

- 读取4字节或者8字节的RISC-V指令。
- 解析RISC-V指令，并找出对应的操作码（op code）。之前阅读`kernel.asm`的时候，看过一些操作码的二进制版本。通过解析，或许可以知道这是一个`ADD`指令，或者是一个`SUB`指令。
- 之后，在软件中执行相应的指令。

```c
while (simulation_running) {
    // 1. 取指令
    instruction = fetch_4bytes(PC); 
    
    // 2. 解码指令
    opcode = decode(instruction);
    
    // 3. 执行语义
    execute_opcode(opcode);
    
    // 4. 更新PC
    PC += 4; 
}
```

**关键实现细节：**

- 寄存器状态维护:`QEMU`在内存中维护完整的`RISC-V`寄存器文件

-当`QEMU`在执行一条指令，比如`(ADD a0, 7, 1)`，这里会将常量`7`和`1`相加，并将结果存储在`a0`寄存器中，所以在这个例子中，寄存器`X0`会是`7`。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJliet0hZBL75tlaCZM%2F-MJljj-HF66Hcx_5N-6M%2Fimage.png?alt=media&token=cb09f40c-ed00-494f-a5ce-1d503f48eb80)

**除了仿真所有的普通权限指令之外，QEMU还会仿真所有的特殊权限指令，这就是QEMU的工作原理。**

### 3.8.4 学生问答

> **学生提问：** 我想知道，QEMU 有没有什么欺骗硬件的实现，比如说overlapping instruction？（QEMU是否使用硬件加速技巧（如指令重叠执行）？）
>**Frans教授：** 并没有，真正的CPU运行在QEMU的下层。当你运行QEMU时，很有可能你是运行在一个x86处理器上，这个x86处理器本身会做各种处理，比如顺序解析指令。所以QEMU对你来说就是个C语言程序。

> **学生提问：** 那多线程呢？程序能真正跑在4个核上吗？还是只能跑在一个核上？如果能跑在多个核上，那么QEMU是不是有多线程？
> **Frans教授：** 我们在Athena上使用的QEMU还有你们下载的QEMU，它们会使用多线程。**QEMU在内部通过多线程实现并行处理**。所以，当QEMU在仿真4个CPU核的时候，它是并行的模拟这4个核。我们在后面有个实验会演示这里是如何工作的。所以，（当QEMU仿真多个CPU核时）这里真的是在不同的CPU核上并行运算。

**总结：QEMU的定位-QEMU是硬件行为的重建者而非简单模拟器**：

- 指令集层：精确还原RISC-V指令语义
- 硬件层：完整仿真内存/中断/外设
- 系统层：提供与真实主板一致的运行环境

## 3.9 XV6 启动过程详解

这里简单的介绍了一下XV6是如何从0开始直到第一个Shell程序运行起来。并且我们也看了一下第一个系统调用是在什么时候发生的。本节并不会涉及系统调用背后的具体机制，这个在后面会介绍。

### 3.9.1 概述

XV6启动过程是从硬件初始化到用户空间的全流程，主要分为三个阶段：

- **硬件初始化**：从`_entry`入口到`main`函数执行前
- **内核初始化**：`main`函数中的系统组件初始化
- **用户空间启动**：从`initcode`到`Shell`启动

### 3.9.2 调试环境搭建

#### 3.9.2.1 QEMU启动（GDB服务端）

启动`QEMU`，本质上来说`QEMU`内部有一个`gdb server`，启动之后，`QEMU`会等待`gdb`客户端连接。

```bash
make CPUS=1 qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501
```

#### 3.9.2.2 GDB客户端连接

在另一个终端**启动一个gdb客户端**,注意当xv6-riscv 目录下有 `.gdbinit` 配置 有的情况下 `riscv64-unknown-elf-gdb` 会自动加载，如果没有`.gdbinit`则需要你手动 `source .gdbinit` 当打印 `0x0000000000001000 in ?? ()` 代表可以调试。

```bash
riscv64-unknown-elf-gdb

GNU gdb (GDB) 10.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=arm-apple-darwin21.4.0 --target=riscv64-unknown-elf".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
warning: File "/Users/iiixv/Documents/xv6-riscv/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
 add-auto-load-safe-path /Users/iiixv/Documents/xv6-riscv/.gdbinit
line to your configuration file "/Users/iiixv/.gdbinit".
To completely disable this security protection add
 set auto-load safe-path /
line to your configuration file "/Users/iiixv/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
 info "(gdb)Auto-loading safe path"
(gdb) source .gdbinit
The target architecture is set to "riscv:rv64".
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000001000 in ?? ()

```

### 3.9.3 内核入口分析

#### 3.9.3.1 初始指令分析

在连接上之后，在**程序的入口处设置一个端点**，这是`QEMU`会跳转到的第一个指令。

设置完断点之后，运行程序，可以发现代码并没有停在`0x8000000`（见3.7 `kernel.asm`中，`0x80000000`是程序的起始位置），而是停在了`0x8000000a`。

```bash
(gdb) b _entry
Breakpoint 1 at 0x8000000a
```

第一条指令： **如果查看`kernel`的汇编文件**，可以看到，在地址`0x8000000a`读取了控制系统寄存器`（Control System Register）mhartid`，并将结果加载到了`a1`寄存器。所以`QEMU`会模拟执行这条指令，之后执行下一条指令。地址`0x80000000`是一个被QEMU认可的地址。也就是说如果你想使用QEMU，那么第一个指令地址必须是它。所以，我们会让内核加载器从那个位置开始加载内核。

```bash
// csrr：特权指令（Control Status Register Read）
8000000a: f14025f3           csrr a1,mhartid
```

- **如果查看`kernel.ld`**，该文件定义了**内核是如何被加载的**，从这里也可以看到，内核使用的起始地址就是`QEMU`指定的`0x80000000`这个地址。这就是我们操作系统最初运行的步骤。

```ld
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;

  .text : {
    *(.text .text.*)
    . = ALIGN(0x1000);
    _trampoline = .;
    *(trampsec)
    . = ALIGN(0x1000);
    ASSERT(. - _trampoline == 0x1000, "error: trampoline larger than one page");
    PROVIDE(etext = .);
  }

  .rodata : {
    . = ALIGN(16);
    *(.srodata .srodata.*) /* do not need to distinguish this from .rodata */
    . = ALIGN(16);
    *(.rodata .rodata.*)
  }

  .data : {
    . = ALIGN(16);
    *(.sdata .sdata.*) /* do not need to distinguish this from .data */
    . = ALIGN(16);
    *(.data .data.*)
  }

  .bss : {
    . = ALIGN(16);
    *(.sbss .sbss.*) /* do not need to distinguish this from .bss */
    . = ALIGN(16);
    *(.bss .bss.*)
  }

  PROVIDE(end = .);
}

```

- 回到`gdb`， 继续执行。可以看到`gdb`显示了指令`csrr`（特权指令，`Control Status Register Read`）的二进制编码`f3 25 40 f1`，即`csrr`是一个4字节的指令。而`addi`是一个2字节的指令。

```bash
(gdb) b _entry
Breakpoint 1 at 0x8000000a
(gdb) c
Continuing.

Breakpoint 1, 0x000000008000000a in _entry ()
=> 0x000000008000000a <_entry+10>: f3 25 40 f1 csrr a1,mhartid
(gdb) si
0x000000008000000e in _entry ()
=> 0x000000008000000e <_entry+14>: 85 05 addi a1,a1,1
```

#### 3.9.2 执行环境特征

继续运行`si`,可以看出，`XV6`从`entry.s`开始启动，执行环境特征如下：

- **此时没有内存分页，没有隔离性**
- 运行在`M-mode（machine mode）`
- `XV6`会尽可能快的跳转到`kernel mode`或者说是`supervisor mode`。

### 3.9.3 内核初始化流程

#### 3.9.3.1 进入main函数

在`main`函数设置一个断点，注意：`main`函数已经运行在`supervisor mode`了。

继续运行程序，代码会在断点，也就是`main`函数的第一条指令停住。

```bash
# gdb断点显示
(gdb) b main
Breakpoint 2 at 0x80000e7a: file kernel/main.c, line 13.
(gdb) c
Continuing.

Breakpoint 2, main () at kernel/main.c:13
13   if(cpuid() == 0){
```

```c
/** 
  main函数源码 
*/
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "defs.h"

volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){ 
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```

注1: 运行在`gdb`的`layout split`模式，该模式下可以看出`gdb`要执行的下一条指令，断点的具体位置。

```bash
(gdb) layout split
```

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoCnirrECVo-lvg7dS%2Fimage.png?alt=media&token=a20802f5-715b-41b5-a942-2d571ea78584)

注2：目前只在一个`CPU`上运行`QEMU`（`make CPUS=1 qemu-gdb`），这使得gdb调试更加简单。同时，因为现在只指定了一个`CPU`核，`QEMU`只会仿真一个核，我们可以单步执行程序（因为在单核或者单线程场景下，单个断点就可以停止整个程序的运行）。

#### 3.9.3.2 consoleinit 验证

**在gdb中输入`n`**，可以跳到下一条指令。这里调用了一个名为`consoleinit`的函数，其功能为设置好`console`。`console`设置好后，便可以向`console`打印输出（代码16、17行）。

```bash

┌─kernel/main.c────────────────────────────────────────────────────────────────┐
│   14              consoleinit();                                             │
│   15              printfinit();                                              │
│   16              printf("\n");                                              │
│   17              printf("xv6 kernel is booting\n");                         │
│  >18              printf("\n");                                              │
│   19              kinit();         // physical page allocator                │
┌──────────────────────────────────────────────────────────────────────────────┐
│   0x80000ed0 <main+94>    auipc   ra,0xfffff                                 │
│   0x80000ed4 <main+98>    jalr    1402(ra)                                   │
│   0x80000ed8 <main+102>   auipc   ra,0x0                                     │
│   0x80000edc <main+106>   jalr    -1910(ra)                                  │
│   0x80000ee0 <main+110>   auipc   a0,0x7                                     │
│   0x80000ee4 <main+114>   addi    a0,a0,488                                  │
│   0x80000ee8 <main+118>   auipc   ra,0xfffff                                 │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: main                                 L18   PC: 0x80000f00 
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) 
```

执行完16、17行之后，我们可以在QEMU看到相应的输出，既 `xv6 kernel is booting`

```bash
iiixv@IIIXVdeAir xv6-riscv % make CPUS=1 qemu-gdb
sed "s/:1234/:25501/" < .gdbinit.tmpl-riscv > .gdbinit
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501
QEMU: Terminated
(base) iiixv@IIIXVdeAir xv6-riscv % make CPUS=1 qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501

xv6 kernel is booting

```

#### 3.9.3.3 初始化序列（main 函数）

- 除了`console`之外，还有许多代码来做初始化。关键初始化顺序：
  - 内存管理（kinit/kvminit）
  - 异常处理（trapinit）
  - 中断系统（plicinit）
  - 文件系统（binit/iinit）
  - 用户进程（userinit）

```c
    kinit();         // physical page allocator 设置好页表分配器（page allocator）
    kvminit();       // create kernel page table 设置好虚拟内存，这是下节课的内容
    kvminithart();   // turn on paging 打开页表，也是下节课的内容
    procinit();      // process table 设置好初始进程或者说设置好进程表单
    /**设置好user/kernel mode转换代码 */
    trapinit();      // trap vectors 
    trapinithart();  // install kernel trap vector
    /*设置好中断控制器PLIC（Platform Level Interrupt Controller） */
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache 分配buffer cache
    iinit();         // inode table 初始化inode缓存
    fileinit();      // file table 初始化文件系统
    virtio_disk_init(); // emulated hard disk 初始化磁盘
    userinit();      // first user process
```

> **学生提问**：这里的初始化函数的调用顺序重要吗？
> **Frans教授**：重要，哈哈。一些函数必须在另一些函数之后运行，某几个函数的顺序可能不重要，但是对它们又需要在其他的一些函数之后运行。

### 3.9.4 用户空间启动

#### 3.9.4.1 首个用户进程创建（`userinit`）

持续进行下一步`n`，通过`gdb`的`s`指令，跳到`userinit`内部。
如果不小心进错函数，则通过`finish`退出当前函数。

```bash
# gdb视图
┌─kernel/proc.c────────────────────────────────────────────────────────────────┐
│   228           struct proc *p;                                              │
│   229                                                                        │
│  >230           p = allocproc();                                             │
│   231           initproc = p;                                                │
│   232                                                                        │
│   233           // allocate one user page and copy init's instructions       │
┌──────────────────────────────────────────────────────────────────────────────┐
│   0x80001c72 <userinit+4>         sd      s0,16(sp)                          │
│   0x80001c74 <userinit+6>         sd      s1,8(sp)                           │
│   0x80001c76 <userinit+8>         addi    s0,sp,32                           │
│  >0x80001c78 <userinit+10>        auipc   ra,0x0                             │
│   0x80001c7c <userinit+14>        jalr    -216(ra)                           │
│   0x80001c80 <userinit+18>        mv      s1,a0                              │
│   0x80001c82 <userinit+20>        auipc   a5,0x7                             │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: userinit                             L230  PC: 0x80001c78 
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) s
userinit () at kernel/proc.c:230
(gdb) 
```

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoLEM84J5GQs0ERl8K%2Fimage.png?alt=media&token=859a93b3-5e2a-4747-8f91-8e087eaaace4)

上图是`userinit`函数，右边是源码，左边是gdb视图。

`userinit`有点像是胶水代码`Glue code`（胶水代码不实现具体的功能，只是为了适配不同的部分而存在），它利用了XV6的特性，并启动了第一个进程。总是需要有一个用户进程在运行，这样才能实现与操作系统的交互，所以这里需要**一个小程序来初始化第一个用户进程**，这个小程序定义在`initcode`中。

#### 3.9.4.2 initcode 执行流程

**综合来看，`initcode`完成了通过`exec`调用`init`程序的工作。**

**`initcode` 功能**：是二进制形式的程序，它会链接或者在内核中直接静态定义。这段代码对应了下面的汇编程序 `user/initCode.s`：

- 首先将`init`中的地址加载到`a0（la a0, init）`，
- `argv`中的地址加载到`a1（la a1, argv）`
- `exec`系统调用对应的数字加载到`a7（li a7, SYS_exec）`
- 最后调用`ECALL`。
- 即该段汇编程序执行了3条指令，之后在第4条指令将控制权交给了操作系统。

```c
// a user program that calls exec("/init")
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x35, 0x02,
  0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00,
  0x93, 0x08, 0x20, 0x00, 0x73, 0x00, 0x00, 0x00,
  0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e, 0x69,
  0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00
};
```

```s
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init  # 加载init路径地址
        la a1, argv  # 加载参数地址
        li a7, SYS_exec
        ecall          # 触发系统调用

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0

```

#### 3.9.4.3 首个系统调用捕获（syscall）

在`syscall`中设置一个断点，并让程序运行起来。

```bash
(gdb) b syscall
Breakpoint 3 at 0x80002abc: file kernel/syscall.c, line 134.
```

`userinit`会创建初始进程，返回到用户空间，执行刚刚介绍的3条指令，再回到内核空间。

通过在`gdb`中执行`c`，让程序运行起来，现在进入到了`syscall`函数。

```bash
┌─kernel/syscall.c─────────────────────────────────────────────────────────────┐
│   132         void                                                           │
│   133         syscall(void)                                                  │
│B+>134         {                                                              │
│   135           int num;                                                     │
│   136           struct proc *p = myproc();                                   │
│   137                                                                        │
┌──────────────────────────────────────────────────────────────────────────────┐
│B+>0x80002abc <syscall>    addi    sp,sp,-32                                  │
│   0x80002abe <syscall+2>  sd      ra,24(sp)                                  │
│   0x80002ac0 <syscall+4>  sd      s0,16(sp)                                  │
│   0x80002ac2 <syscall+6>  sd      s1,8(sp)                                   │
│   0x80002ac4 <syscall+8>  sd      s2,0(sp)                                   │
│   0x80002ac6 <syscall+10> addi    s0,sp,32                                   │
│   0x80002ac8 <syscall+12> auipc   ra,0xfffff                                 │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: syscall                              L134  PC: 0x80002abc 
```

**系统调用处理**

- 查看`syscall`的代码，`kenel/syscall.c`

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

- 执行完`num = p->trapframe->a7`后，在`gdb`中打印`num`，输出为7。

```bash
┌─kernel/syscall.c─────────────────────────────────────────────────────────────┐
│   136           struct proc *p = myproc();                                   │
│   137                                                                        │
│   138           num = p->trapframe->a7;                                      │
│  >139           if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {      │
│   140             p->trapframe->a0 = syscalls[num]();                        │
│   141           } else {                                                     │
┌──────────────────────────────────────────────────────────────────────────────┐
│   0x80002ad2 <syscall+22> ld      s2,88(a0)                                  │
│   0x80002ad6 <syscall+26> ld      a5,168(s2)                                 │
│   0x80002ada <syscall+30> sext.w  a3,a5                                      │
│  >0x80002ade <syscall+34> addiw   a5,a5,-1                                   │
│   0x80002ae0 <syscall+36> li      a4,20                                      │
│   0x80002ae2 <syscall+38> bltu    a4,a5,0x80002b00 <syscall+68>              │
│   0x80002ae6 <syscall+42> slli    a4,a3,0x3                                  │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: syscall                              L139  PC: 0x80002ade 
Breakpoint 3, syscall () at kernel/syscall.c:134
(gdb) n
(gdb) n
(gdb) n
(gdb) p num
$2 = 7

```

- 查看`kenel/syscall.h`，可以看到`7`对应的是`exec`系统调用。所以，这里本质上是告诉内核，某个用户应用程序执行了`ECALL`指令，并且要调用`exec`系统调用。

```h
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
#define SYS_kill    6
#define SYS_exec    7
#define SYS_fstat   8
#define SYS_chdir   9
#define SYS_dup    10
#define SYS_getpid 11
#define SYS_sbrk   12
#define SYS_sleep  13
#define SYS_uptime 14
#define SYS_open   15
#define SYS_write  16
#define SYS_mknod  17
#define SYS_unlink 18
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
```

- `p->trapframe->a0 = syscall[num]()` 表示**实际执行系统调用**。这里可以看出，`num`用来索引一个数组，这个数组是一个**函数指针数组**，可以预期的是`syscall[7]`对应了`exec`的入口函数。
  
- 进入`sys_exec()`函数中，位于`kernel/sysfile.c`。

```bash
┌─kernel/sysfile.c─────────────────────────────────────────────────────────────┐
│   420           uint64 uargv, uarg;                                          │
│   421                                                                        │
│  >422           if(argstr(0, path, MAXPATH) < 0 || argaddr(1, &uargv) < 0){  │
│   423             return -1;                                                 │
│   424           }                                                            │
│   425           memset(argv, 0, sizeof(argv));                               │
┌──────────────────────────────────────────────────────────────────────────────┐
│   0x8000586c <sys_exec+12>        sd      s4,416(sp)                         │
│   0x8000586e <sys_exec+14>        sd      s5,408(sp)                         │
│   0x80005870 <sys_exec+16>        addi    s0,sp,464                          │
│  >0x80005872 <sys_exec+18>        li      a2,128                             │
│   0x80005876 <sys_exec+22>        addi    a1,s0,-192                         │
│   0x8000587a <sys_exec+26>        li      a0,0                               │
│   0x8000587c <sys_exec+28>        auipc   ra,0xffffd                         │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: sys_exec                             L422  PC: 0x80005872 
(gdb) n
(gdb) n
(gdb) n
(gdb) p num
$2 = 7
(gdb) n
(gdb) s
sys_exec () at kernel/sysfile.c:422
```

`sys_exec`中的第一件事情是从用户空间读取参数，它会读取`path`，也就是要执行程序的文件名。这里首先会为参数分配空间，然后从用户空间将参数拷贝到内核空间。

之后我们打印`path`，可以看到传入的就是`init`程序。

```c
/* sys_exec */
uint64
sys_exec(void)
{
  char path[MAXPATH], *argv[MAXARG];
  int i;
  uint64 uargv, uarg;

  if(argstr(0, path, MAXPATH) < 0 || argaddr(1, &uargv) < 0){
    return -1;
  }
  memset(argv, 0, sizeof(argv));
  for(i=0;; i++){
    if(i >= NELEM(argv)){
      goto bad;
    }
    if(fetchaddr(uargv+sizeof(uint64)*i, (uint64*)&uarg) < 0){
      goto bad;
    }
    if(uarg == 0){
      argv[i] = 0;
      break;
    }
    argv[i] = kalloc();
    if(argv[i] == 0)
      goto bad;
    if(fetchstr(uarg, argv[i], PGSIZE) < 0)
      goto bad;
  }

  int ret = exec(path, argv);

  for(i = 0; i < NELEM(argv) && argv[i] != 0; i++)
    kfree(argv[i]);

  return ret;

 bad:
  for(i = 0; i < NELEM(argv) && argv[i] != 0; i++)
    kfree(argv[i]);
  return -1;
}

```

**所以，综合来看，`initcode`完成了通过`exec`调用`init`程序的工作。**

#### 3.9.4.3 init 程序详解

让我们来看看`user/init.c`程序，`init`会为用户空间设置好一些东西，比如配置好`console`，调用`fork`，并在`fork`出的子进程中执行`shell`。最终的效果就是`Shell`运行起来了。

```c
// init: The initial user-level program

#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/spinlock.h"
#include "kernel/sleeplock.h"
#include "kernel/fs.h"
#include "kernel/file.h"
#include "user/user.h"
#include "kernel/fcntl.h"

char *argv[] = { "sh", 0 };

int
main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf("init: starting sh\n");
    pid = fork();
    if(pid < 0){
      printf("init: fork failed\n");
      exit(1);
    }
    if(pid == 0){
      exec("sh", argv);
      printf("init: exec sh failed\n");
      exit(1);
    }

    for(;;){
      // this call to wait() returns if the shell exits,
      // or if a parentless process exits.
      wpid = wait((int *) 0);
      if(wpid == pid){
        // the shell exited; restart it.
        break;
      } else if(wpid < 0){
        printf("init: wait returned an error\n");
        exit(1);
      } else {
        // it was a parentless process; do nothing.
      }
    }
  }
}

```

#### 3.9.4.5 shell 启动验证

如果我再次运行代码，我还会陷入到`syscall`中的断点，并且同样也是调用`exec`系统调用，只是这次是通过`exec`运行`Shell`。当`Shell`运行起来之后，我们可以从`QEMU`看到`Shell`。

```bash
(base) iiixv@IIIXVdeAir xv6-riscv % make CPUS=1 qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501

xv6 kernel is booting

init: starting sh
```

```c
   132         void                                                           │
│   133         syscall(void)                                                  │
│B+>134         {                                                              │
│   135           int num;                                                     │
│   136           struct proc *p = myproc();                                   │
│   137                                                                        │
┌──────────────────────────────────────────────────────────────────────────────┐
│B+>0x80002abc <syscall>    addi    sp,sp,-32                                  │
│   0x80002abe <syscall+2>  sd      ra,24(sp)                                  │
│   0x80002ac0 <syscall+4>  sd      s0,16(sp)                                  │
│   0x80002ac2 <syscall+6>  sd      s1,8(sp)                                   │
│   0x80002ac4 <syscall+8>  sd      s2,0(sp)                                   │
│   0x80002ac6 <syscall+10> addi    s0,sp,32                                   │
│   0x80002ac8 <syscall+12> auipc   ra,0xfffff                                 │
└──────────────────────────────────────────────────────────────────────────────┘
remote Thread 1.1 In: syscall                              L134  PC: 0x80002abc 
Breakpoint 3, syscall () at kernel/syscall.c:134
(gdb) c
Continuing.

Breakpoint 3, syscall () at kernel/syscall.c:134
(gdb) del 3
(gdb) c
Continuing.
```

> **学生提问**：我们会处理网络吗，比如说网络相关的实验？
>**Frans教授**：是的，最后一个lab中你们会实现一个网络驱动。你们会写代码与硬件交互，操纵连接在RISC-V主板上网卡的驱动，以及寄存器，再向以太网发送一些网络报文。

### 3.9.5 启动时序总结

```mermaid
sequenceDiagram
    participant H as Hardware
    participant K as Kernel
    participant U as Userspace
    
    H->>K: 复位向量跳转 0x80000000
    activate Kß
    K->>K: _entry (M-mode)
    K->>K: 设置栈指针
    K->>K: 跳转至start()
    K->>K: main()初始化 (S-mode)
    deactivate K
    
    K->>U: userinit创建init进程
    activate U
    U->>U: initcode执行
    U->>K: ecall触发SYS_exec
    deactivate U
    
    activate K
    K->>K: sys_exec()处理
    K->>U: 加载/init程序
    deactivate K
    
    activate U
    U->>U: init初始化控制台
    U->>U: fork()+exec("sh")
    U->>U: Shell交互界面
    deactivate U
```

**核心阶段特征**：
| **阶段**         | **权限模式**   | **关键转折点**               |
|------------------|----------------|------------------------------|
| **硬件初始化**   | Machine Mode   | _entry入口指令执行           |
| **内核初始化**   | Supervisor Mode| main函数组件初始化           |
| **用户空间启动** | User Mode      | 首个ecall系统调用触发        |

**设计本质**：
1. **权限降级链条**：M-mode → S-mode → U-mode
2. **空间转换**：裸机环境 → 内核空间 → 用户空间
3. **控制权转移**：直接硬件控制 → 系统调用接口
4. **进程零创建**：首个用户进程由内核直接构造（无fork）