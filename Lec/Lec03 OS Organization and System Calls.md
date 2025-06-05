# Lec 03. OS Organization and System Calls

- [Lec 03. OS Organization and System Calls](#lec-03-os-organization-and-system-calls)
  - [3.1 课程回顾](#31-课程回顾)
  - [3.2 OS的隔离性](#32-os的隔离性)
    - [3.2.1 为什么需要隔离性](#321-为什么需要隔离性)
    - [3.2.2 Strawman Design /No OS：没有操作系统会怎样](#322-strawman-design-no-os没有操作系统会怎样)
      - [缺点一：无法实现multiplexing](#缺点一无法实现multiplexing)
      - [缺点二：无法内存隔离](#缺点二无法内存隔离)
    - [3.2.3 总结](#323-总结)
    - [3.2.4 举例](#324-举例)
      - [fork - CPU抽象](#fork---cpu抽象)
      - [exec -- 内存抽象](#exec----内存抽象)
      - [file -- 磁盘抽象](#file----磁盘抽象)
      - [总结](#总结)
  - [3.3 OS的防御性](#33-os的防御性)
    - [3.3.1 操作系统的防御性](#331-操作系统的防御性)
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
    - [3.6.3 微内核](#363-微内核)
  - [3.7 编译运行Kernel](#37-编译运行kernel)
    - [3.7.1 xv6代码结构](#371-xv6代码结构)
    - [3.7.2 内核编译](#372-内核编译)
  - [3.8 QEMU底层原理](#38-qemu底层原理)
  - [3.9 XV6 启动过程](#39-xv6-启动过程)

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

#### 缺点二：无法内存隔离

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

Unix接口通过抽象硬件资源，从而提供了强隔离性。可以发现，接口被精心设计以实现资源的强隔离，也就是`multiplexing`和物理内存的隔离。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRLmqk_Z0lli966feS%2Fimage.png?alt=media&token=c867e904-807f-45b9-8faa-5673a8ac988a)

| 资源   | 抽象机制        | 系统调用         | 实现方式                |
| ---- | ----------- | ------------ | ------------------- |
| CPU  | 进程（Process） | `fork`       | 调度器 + 分时复用          |
| 内存   | 虚拟地址空间      | `exec`       | 内存镜像加载 + 虚拟内存管理     |
| 文件系统 | 文件描述符       | `open/read`  | inode 表 + 缓存 + 权限控制 |
| 设备   | 文件接口        | `read/write` | 设备驱动程序抽象为“文件”       |

#### fork - CPU抽象
  
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

### 3.3.1 操作系统的防御性

当操作系统上运行多个应用程序时，必须具备防御能力（Defensive）。

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

首先，我们来看一下user/kernel mode，这里会以尽可能全局的视角来介绍，有很多重要的细节在这节课中都不会涉及。

#### 处理器的两种操作模式

为了支持user/kernel mode，**处理器会有两种操作模式**

- `User mode`：CPU 只能执行普通权限指令（`unprivileged instructions`）。

- `Kernel mode`：CPU 可执行特殊权限指令（`privileged instructions`）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJReB5Yo_RJjpIo3RnA%2Fimage.png?alt=media&token=d7abdd2b-ab8e-4281-9725-337e95f54982)

#### 普通/特殊权限指令

**普通权限的指令：** 例如寄存器加法（ADD）、减法（SUB）、跳转（JRC）和分支（BRANCH）等，这些指令所有应用程序均可执行。

**特殊权限指令：** 主要用于直接操作硬件和设置保护机制，如设置页表寄存器、控制中断等，仅允许内核执行。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRhsBp1tiieCyd1xXD%2Fimage.png?alt=media&token=4d834d3f-8458-4b1e-bdff-cf9f147a5441)

**举例**：若应用程序在`user mode`下尝试执行特殊权限指令，处理器会拒绝执行该指令。通常来说，这时会将控制权限从`user mode`切换到`kernel mode`，当操作系统拿到控制权之后，由操作系统决定如何处理,或许会杀掉进程，因为应用程序执行了不该执行的指令。

#### RISC-V privilege架构的文档
下图展示了RISC-V的特殊权限指令文档。未来一个月内，你们将频繁接触这些特殊权限指令。这里我们先对这些指令有一些初步的认识：**应用程序不应执行这些指令，内核专属。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRjawN3LdVFfbgHTYB%2Fimage.png?alt=media&token=08f5a718-9279-4df6-a68a-215e17d90123)

> **学生提问**：谁负责检测当前模式并允许或拒绝指令执行？怎么判断当前是user还是kernel mode？
> **Frans教授**：处理器内部有一个标志位，用于区分user mode（1）和kernel mode（0）。当特殊权限指令在user mode执行时，会被拒绝，就像运算中除以0一样出错。
> **同一个学生继续问**：如何更改这个标志位？是特殊权限指令还是普通指令？
> **Frans教授**：你认为是什么指令更新了那个bit位？是特殊权限指令还是普通权限指令？（等了一会，那个学生没有回答）。很明显，**设置那个bit位的指令必须是特殊权限指令**，应用程序不能直接修改，否则会绕过保护机制。这保证了模式切换的安全性。

许多同学都已经知道了，**实际上RISC-V还有第三种模式称为machine mode**。在大多数场景下，会忽略这种模式，本课程也不太会介绍这种模式。 所以实际上我们有三级权限（user/kernel/machine），而不是两级(user/kernel)。

> **学生提问**：考虑到安全性，所有的用户代码都会通过内核访问硬件，但是有没有可能一个计算机的用户可以随意的操纵内核？
> **Frans教授**：不会，只要设计严谨，这种情况就能避免。某些程序或用户（如root）可能拥有额外权限以执行安全相关操作，但普通用户不能随意操作内核。
> **同一个学生提问**：那BIOS呢？BIOS会在操作系统之前运行还是之后？
> **Frans教授**：BIOS是一段固化在计算机中的代码，会在操作系统启动前运行。它必须是可信且无恶意的，因为它负责启动操作系统。

> **学生提问**：之前提到，设置处理器中kernel mode的bit位的指令是一条特殊权限指令，既然设置处理器模式位的指令是特殊权限指令，用户程序怎么能让内核执行这些指令？用户程序无法直接修改模式位。
> **Frans教授**：你说得对，这是设计使然。实际上，用户程序不能直接执行特殊权限指令，而是通过系统调用来切换到kernel mode。当用户执行系统调用（如ECALL指令）时，会触发一个软中断（software interrupt），处理器转向操作系统预设的中断向量表，执行对应的内核中断处理程序。这样，系统完成了从user mode到kernel mode的安全切换，并代替用户执行需要特殊权限的指令。

### 3.4.2 虚拟内存

接下来介绍支持强隔离性的另一项硬件特性——虚拟内存。几乎所有现代CPU都支持虚拟内存，下一节课会更深入讲解，这里先做简要介绍。

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

- 用户程序执行 `ECALL`，并通过寄存器传入一个数字参数；这个数字参数代表了希望调用的 系统调用编号（syscall number）；

- `ECALL` 会触发一个 陷入（`trap`），跳转到操作系统中事先定义好的一个入口地址；XV6 中有一个**唯一的系统调用接入点**，每一次**应用程序执行ECALL指令**，应用程序都会**通过这个接入点进入到内核中。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXAkJxD8pTZOm1Tay_%2Fimage.png?alt=media&token=cf3e26a2-3c26-43b8-aee0-6d5787f8dcf5)

### 3.5.2 举例

#### fork系统调用

当 `Shell` 或其他用户程序需要创建一个新进程时，会使用 `fork` 系统调用。但这个调用并不是直接跳转到内核中的 `fork()` 函数。

其具体执行流程如下：

1. **用户空间发起调用`fork()`:**，应用程序不直接调用内核函数，而是通过标准库封装函数（如 `fork()/write()`）发起。封装函数会执行 `ECALL` 指令，触发软中断，并传入系统调用号（唯一标识操作类型）。
2. **硬件特权切换：** ECALL 触发后：CPU 从用户模式（`User Mode`）切换至内核模式（`Kernel Mode`）。处理器跳转至内核预设的统一入口点（如 `syscall` 函数，位于 `syscall.c`）。
3. 内核分发与执行：解析传入的系统调用号；路由至对应的内核处理函数（如 sys_fork/sys_write）；执行实际操作（如创建进程/写入数据）。

如下图所示，用户空间和内核空间之间由一条明确的界限分隔，用户程序不能直接调用内核函数，只能通过 ECALL 指令请求内核服务：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXO2n90L0ziqU8mTcg%2Fimage.png?alt=media&token=754f49c1-58a2-42d5-9427-094fc95ab613)

#### write系统调用

write 系统调用的过程与 fork 类似。用户程序不能直接访问内核中的写操作函数，而是通过一套标准流程请求内核完成写操作：

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

现在，我们有了一种方法，可以通过系统调用或者说ECALL指令，将控制权从应用程序转到操作系统中。之后内核负责实现具体的功能并检查参数以确保不会被一些坏的参数所欺骗。所以内核有时候也被称为可被信任的计算空间（Trusted Computing Base），在一些安全的术语中也被称为TCB。

基本上来说，**要被称为TCB，内核首先要是正确且没有Bug的**。假设内核中有Bug，攻击者可能会利用那个Bug，并将这个Bug转变成漏洞，这个漏洞使得攻击者可以打破操作系统的隔离性并接管内核。所以内核真的是需要越少的Bug越好（但是谁不是呢）。

**另一方面，内核必须要将用户应用程序或者进程当做是恶意的**。如我之前所说的，内核的设计人员在编写和实现内核代码时，必须要有安全的思想。这个目标很难实现，因为当你的操作系统变得足够大的时候，很多事情就不是那么直观了。你知道的，几乎每一个你用过的或者被广泛使用的操作系统，时不时的都有一个安全漏洞。就算被修复了，但是过了一段时间，又会出现一个新的漏洞。我们之后会介绍为什么很难让所有部分都正确工作，但是你要知道是内核需要做一些tricky的工作，需要操纵硬件，需要非常小心做检查，所以很容易就出现一些小的疏漏，进而触发一个Bug。这也是可以理解的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJXWRY7_m6ArPVBNgvJ%2F-MJZck3H4EAuhFVSva0U%2Fimage.png?alt=media&token=2a170d6b-b942-4ac5-b943-85f4f2bc7f1d)

### 3.6.2 宏内核

一个有趣的问题是，什么程序应该运行在kernel mode？敏感的代码肯定是运行在kernel mode，因为这是Trusted Computing Base。
对于这个问题的一个答案是，首先我们会有user/kernel边界，在上面是应用程序，在下面是运行在kernel mode的程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbThlJrd7ZaArRiPDt%2Fimage.png?alt=media&token=1eee503b-5a1b-46b3-9d0b-0dfb2e740603)

> **宏内核**

**其中一个选项是让整个操作系统代码都运行在kernel mode**。大多数的**Unix操作系统**实现都运行在kernel mode。比如，XV6中，所有的操作系统服务都在kernel mode中，这种形式被称为**Monolithic Kernel Design（宏内核）。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbVJDYYOlA_ZnJBg_i%2Fimage.png?alt=media&token=11b51e28-9327-450e-a077-9cb3488a2015)

> **宏内核的优缺点**

- 首先，如果考虑Bug的话，这种方式不太好。在一个宏内核中，任何一个操作系统的Bug都有可能成为漏洞。因为我们现在在内核中运行了一个巨大的操作系统，出现Bug的可能性更大了。你们可以去查一些统计信息，平均每3000行代码都会有几个Bug，所以如果有许多行代码运行在内核中，那么出现严重Bug的可能性也变得更大。所以从安全的角度来说，**在内核中有大量的代码是宏内核的缺点。**

- 另一方面，如果你去看一个操作系统，它包含了各种各样的组成部分，比如说文件系统，虚拟内存，进程管理，这些都是操作系统内实现了特定功能的子模块。**宏内核的优势在于**，因为这些子模块现在都位于同一个程序中，它们可以紧密的集成在一起，这样的集成提供很好的性能。例如Linux，它就有很不错的性能。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbYtEBzpGpaCAFdeZk%2Fimage.png?alt=media&token=2e005add-cd05-4201-9fab-5149d38765ab)

### 3.6.3 微内核

另一种设计主要关注点是减少内核中的代码，它被称为**Micro Kernel Design**。在这种模式下，希望在kernel mode中运行尽可能少的代码。所以这种设计下还是有内核，但是**内核只有非常少的几个模块**，例如，内核通常会有一些IPC的实现或者是Message passing；非常少的虚拟内存的支持，可能只支持了page table；以及分时复用CPU的一些支持。

**微内核的目的在于将大部分的操作系统运行在内核之外**。所以，我们还是会有user mode以及user/kernel mode的边界。但是我们现在会将原来在内核中的其他部分，作为普通的用户程序来运行。比如文件系统可能就是个常规的用户空间程序，就像echo，Shell一样，这些程序都运行在用户空间。可能还会有一些其他的用户应用程序，例如虚拟内存系统的一部分也会以一个普通的应用程序的形式运行在user mode。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbb2rd6KN3AuCoErQ-%2Fimage.png?alt=media&token=a93a6b20-563f-42d6-8284-daaaa9a0d254)

> **微内核的优缺点**

**优点**：某种程度上来说，这是一种好的设计。因为在**内核中的代码的数量较小**，更少的代码意味着更少的Bug。

**缺点**：假设我们需要让Shell能与文件系统交互，比如Shell调用了exec，必须有种方式可以接入到文件系统中。

- Shell会通过内核中的IPC系统发送一条消息，内核会查看这条消息并发现这是给文件系统的消息，之后内核会把消息发送给文件系统。
- 文件系统会完成它的工作之后会向IPC系统发送回一条消息说，这是你的exec系统调用的结果，之后IPC系统再将这条消息发送给Shell。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbcquRoZotyofyd2sh%2Fimage.png?alt=media&token=71f32a22-7654-4116-9dee-a40537c01a13)

所以，这里是典型的**通过消息来实现传统的系统调用**。现在，对于任何文件系统的交互，都需要分别完成2次**用户空间<->内核空间的跳转**。与宏内核对比，在宏内核中如果一个应用程序需要与文件系统交互，只需要完成1次用户空间<->内核空间的跳转，所以微内核的的跳转是宏内核的两倍。**通常微内核的挑战在于性能更差**，这里有两个方面需要考虑：

- 在user/kernel mode反复跳转带来的性能损耗。
- 在一个类似宏内核的紧耦合系统，各个组成部分，例如文件系统和虚拟内存系统，可以很容易的共享page cache。而在微内核中，每个部分之间都很好的隔离开了，这种共享更难实现。进而导致更难在微内核中得到更高的性能。
  
> **总结**

我们这里介绍的有关宏内核和微内核的区别都特别的笼统。在实际中，两种内核设计都会出现，**出于历史原因大部分的桌面操作系统是宏内核**

如果你运行需要大量内核计算的应用程序，例如在数据中心服务器上的操作系统，通常也是使用的宏内核，主要的原因是Linux提供了很好的性能。

但是很多嵌入式系统，例如Minix，Cell，这些都是**微内核设计**。这两种设计都很流行。

如果你从头开始写一个操作系统，你可能会从一个微内核设计开始。但是一旦你有了类似于Linux这样的宏内核设计，将它重写到一个微内核设计将会是巨大的工作。并且这样重构的动机也不足，因为人们总是想把时间花在实现新功能上，而不是重构他们的内核。

**XV6是一种宏内核设计，如大多数经典的Unix系统一样**。但是在这个学期的后半部分，我们会讨论更多有关微内核设计的内容。

## 3.7 编译运行Kernel

### 3.7.1 xv6代码结构

首先，我们来看一下代码结构，你们或许已经看过了。**代码主要有三个部分组成：**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJpn93PbVXmyk3cWz9q%2F-MJq7-1lrR0BNbiAaazr%2Fimage.png?alt=media&token=15dcbc6d-ed46-4bd1-bb4b-f71609e413c1)

- 第一个是 `kernel`。我们可以`ls kernel`的内容，里面包含了基本上**所有的内核文件**。因为XV6是一个宏内核结构，这里所有的文件会被编译成一个叫做`kernel`的二进制文件，然后这个二进制文件会被运行在`kernle mode`中。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJdjbpZW0Es0hM8rNrS%2Fimage.png?alt=media&token=d28e7220-ad13-4f76-9b5e-4d414dda7e3c)

- 第二个部分是`user`。这基本上是运行在`user mode`的程序。这也是为什么一个目录称为`kernel`，另一个目录称为`user`的原因。
  
- 第三部分叫做`mkfs`。它会创建一个**空的文件镜像**，我们会将这个镜像存在磁盘上，这样我们就可以直接使用一个空的文件系统。

### 3.7.2 内核编译

> **编译过程**

接下来，我想简单的介绍一下内核是如何编译的。你们可能已经编译过内核，但是还没有真正的理解编译过程，这个过程还是比较重要的。

- 首先，`Makefile`（XV6目录下的文件）会读取一个C文件，例如`proc.c`
- 调用`gcc`编译器，生成一个文件叫做`proc.s`，这是RISC-V 汇编语言文件
- 调用汇编解释器，生成`proc.o`，这是汇编语言的二进制格式。
- 之后，系统加载器（Loader）会收集所有的.o文件，将它们链接在一起，并生成内核文件。这里生成的内核文件就是我们将会在QEMU中运行的文件。同时，为了你们的方便，Makefile还会创建kernel.asm，这里包含了内核的完整汇编语言，你们可以通过查看它来定位究竟是哪个指令导致了Bug。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgXiV2KBGQeuPgX4Bj%2Fimage.png?alt=media&token=80f80f91-adbc-48c7-9767-e6db633cb141)
  
Makefile会为所有内核文件做相同的操作，比如说pipe.c，会按照同样的套路，先经过gcc编译成pipe.s，再通过汇编解释器生成pipe.o。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgWjAGE9ASuK5Hbw2k%2Fimage.png?alt=media&token=cac68445-2b05-4a57-8435-6d495baed104)

> **查看 kernel.asm文件**

比如，我接下来查看kernel.asm文件，我们可以看到用汇编指令描述的内核：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgYi8MwG-QEqdvZ63Q%2Fimage.png?alt=media&token=d82387cf-bc09-47a2-a9d0-7ff0eb0cd2b2)

这里你们可能已经注意到了，第一个指令位于地址0x80000000，对应的是一个RISC-V指令：auipc指令。

有人知道第二列，例如0x0000a117、0x83010113、0x6505，是什么意思吗？有人想来回答这个问题吗？
学生回答：这是汇编指令的16进制表现形式对吗？
是的，完全正确。所以这里0x0000a117就是auipc，这里是**二进制编码后的指令**。因为每个指令都有一个二进制编码，kernel的asm文件会显示这些二进制编码。当你在运行gdb时，如果你想知道具体在运行什么，你可以看具体的二进制编码是什么，有的时候这还挺方便的。

接下来，让我们**不带gdb运行XV6**（make会读取Makefile文件中的指令）。这里会编译文件，然后调用QEMU（qemu-system-riscv64指令）。这里本质上是通过C语言来模拟仿真RISC-V处理器。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgd7lBxdCSukMsxDGi%2Fimage.png?alt=media&token=bfcf8e8d-34d8-44a6-b84b-e378bde5bc09)

我们来看传给QEMU的几个参数：这样，XV6系统就在QEMU中启动了。

- `kernel`：这里传递的是内核文件（kernel目录下的kernel文件），这是将在QEMU中运行的程序文件。
- `m`：这里传递的是RISC-V虚拟机将会使用的内存数量
- `smp`：这里传递的是虚拟机可以使用的CPU核数
- `drive`：传递的是虚拟机使用的磁盘驱动，这里传入的是fs.img文件

## 3.8 QEMU底层原理

QEMU表现的就像一个真正的计算机一样。当使用**QEMU时，你不应该认为它是一个C程序，你应该把它想成是下图，一个真正的主板**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJghmTlvFOKAnfGeveB%2Fimage.png?alt=media&token=2e8f081f-1f48-43e8-8486-0352732fd28e)

当我们通过QEMU来运行你的内核时，你应该认为你的内核是运行在这样一个主板之上。主板有一个开关，一个RISC-V处理器，有支持外设的空间，比如说一个接口是连接网线的，一个是PCI-E插槽，主板上还有一些内存芯片，这是一个你可以在上面编程的物理硬件，而XV6操作系统管理这样一块主板，你在你的脑海中应该有这么一张图。

> **RISC-V的结构图**

对于RISC-V，有完整的文档介绍，比如说下图是一个RISC-V的结构图，图中包括:

- 4个核：U54 Core 1-4
- L2 cache：Banked L2
- 连接DRAM的连接器：DDR Controller
- 各种连接外部设备的方式，比如说UART0，一端连接了键盘，另一端连接了terminal。
- 以及连接了时钟的接口：Clock Generation

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgjgTH1N3waADySy12%2Fimage.png?alt=media&token=a66a77d5-f019-4ddb-921e-1a74fd055567)

我们后面会讨论更多的细节，但是这里**基本上就是RISC-V处理器的所有组件**，你通过它与实际的硬件交互。

实际上抛开一些细节，通过QEMU模拟的计算机系统或者说计算机主板，与这里由SiFive生产的计算机主板非常相似。本来想给你们展示一下这块主板的。当你们在运行QEMU时，你们需要知道，你们基本上跟在运行硬件是一样的，只是说同样的东西，QEMU在软件中实现了而已。

> **当我们说QEMU仿真了RISC-V处理器时，背后的含义是什么？**

直观来看，QEMU是一个大型的开源C程序，你可以下载或者git clone它。

但是在**内部，在QEMU的主循环中，只在做一件事情**：

- 读取4字节或者8字节的RISC-V指令。
- 解析RISC-V指令，并找出对应的操作码（op code）。
- 之后，在软件中执行相应的指令。

我们之前在看kernel.asm的时候，看过一些操作码的二进制版本。通过解析，或许可以知道这是一个ADD指令，或者是一个SUB指令。

这基本上就是QEMU的全部工作了，对于每个CPU核，QEMU都会运行这么一个循环。

为了完成这里的工作，**QEMU的主循环需要维护寄存器的状态**。所以QEMU会有以C语言声明的类似于X0，X1寄存器等等。当QEMU在执行一条指令，比如(ADD a0, 7, 1)，这里会将常量7和1相加，并将结果存储在a0寄存器中，所以在这个例子中，寄存器X0会是7。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJliet0hZBL75tlaCZM%2F-MJljj-HF66Hcx_5N-6M%2Fimage.png?alt=media&token=cb09f40c-ed00-494f-a5ce-1d503f48eb80)

之后QEMU会执行下一条指令，并持续不断的执行指令。

**除了仿真所有的普通权限指令之外，QEMU还会仿真所有的特殊权限指令，这就是QEMU的工作原理。**

> **学生提问：** 我想知道，QEMU有没有什么欺骗硬件的实现，比如说overlapping instruction？
>**Frans教授：** 并没有，真正的CPU运行在QEMU的下层。当你运行QEMU时，很有可能你是运行在一个x86处理器上，这个x86处理器本身会做各种处理，比如顺序解析指令。所以QEMU对你来说就是个C语言程序。

> **学生提问：** 那多线程呢？程序能真正跑在4个核上吗？还是只能跑在一个核上？如果能跑在多个核上，那么QEMU是不是有多线程？
> **Frans教授：** 我们在Athena上使用的QEMU还有你们下载的QEMU，它们会使用多线程。QEMU在内部通过多线程实现并行处理。所以，当QEMU在仿真4个CPU核的时候，它是并行的模拟这4个核。我们在后面有个实验会演示这里是如何工作的。所以，（当QEMU仿真多个CPU核时）这里真的是在不同的CPU核上并行运算。

## 3.9 XV6 启动过程

接下来，我会系统的介绍XV6，让你们对XV6的结构有个大概的了解。

- 首先，我会启动QEMU，并打开gdb。本质上来说QEMU内部有一个gdb server，当我们启动之后，QEMU会等待gdb客户端连接。

```bash
make CPUS=1 qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501
```

- 在另一个终端**启动一个gdb客户端**,注意当xv6-riscv 目录下有 `.gdbinit` 配置 有的情况下 `riscv64-unknown-elf-gdb` 会自动加载，如果没有`.gdbinit`则需要你手动 `source .gdbinit` 当打印 `0x0000000000001000 in ?? ()` 代表可以调试。

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

- 在连接上之后，在**程序的入口处设置一个端点**，因为我们知道这是QEMU会跳转到的第一个指令。设置完断点之后，我运行程序，可以发现代码并没有停在0x8000000（见3.7 kernel.asm中，0x80000000是程序的起始位置），而是停在了0x8000000a。

```bash
(gdb) b _entry
Breakpoint 1 at 0x8000000a
```

- **如果我们查看kernel的汇编文件**,我们可以看到，在地址0x8000000a读取了控制系统寄存器（Control System Register）mhartid，并将结果加载到了a1寄存器。所以QEMU会模拟执行这条指令，之后执行下一条指令。地址0x80000000是一个被QEMU认可的地址。也就是说如果你想使用QEMU，那么第一个指令地址必须是它。所以，我们会让内核加载器从那个位置开始加载内核。

```bash
8000000a: f14025f3           csrr a1,mhartid
```

- **如果我们查看`kernel.ld`**，这个文件定义了**内核是如何被加载的**，从这里也可以看到，内核使用的起始地址就是QEMU指定的`0x80000000`这个地址。这就是我们操作系统最初运行的步骤。

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

- 回到gdb，我们可以看到gdb也显示了指令的二进制编码`f3 25 40 f1`,可以看出`csrr`是一个4字节的指令，而`addi`是一个2字节的指令。

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

我们这里可以看到，XV6从entry.s开始启动，这个时候没有内存分页，没有隔离性，并且运行在M-mode（machine mode）。XV6会尽可能快的跳转到kernel mode或者说是supervisor mode。

- 我们在main函数设置一个断点，main函数已经运行在`supervisor mode`了。接下来我运行程序，代码会在断点，也就是main函数的第一条指令停住。

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
/**main函数源码 */
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

- 运行在`gdb`的`layout split`模式,从这个视图可以看出gdb要执行的下一条指令是什么，断点具体在什么位置。这里只在一个CPU上运行QEMU（`make CPUS=1 qemu-gdb`），这样会使得gdb调试更加简单。因为现在只指定了一个CPU核，QEMU只会仿真一个核，我可以单步执行程序（因为在单核或者单线程场景下，单个断点就可以停止整个程序的运行）。

```bash
(gdb) layout split
```

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoCnirrECVo-lvg7dS%2Fimage.png?alt=media&token=a20802f5-715b-41b5-a942-2d571ea78584)

- 通过**在gdb中输入`n`**，可以跳到下一条指令。这里调用了一个名为`consoleinit`的函数，它的工作与你想象的完全一样，也就是设置好`console`。一旦`console`设置好了，接下来可以向`console`打印输出（代码16、17行）。

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

除了`console`之外，还有许多代码来做初始化。

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

- 可以通过`gdb`的`s`指令，跳到`userinit`内部。

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

- `userinit`有点像是胶水代码`Glue code`（胶水代码不实现具体的功能，只是为了适配不同的部分而存在），它利用了XV6的特性，并启动了第一个进程。我们总是需要有一个用户进程在运行，这样才能实现与操作系统的交互，所以这里需要**一个小程序来初始化第一个用户进程**。这个小程序定义在`initcode`中。

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

这里直接是程序的二进制形式，它会链接或者在内核中直接静态定义。实际上，这段代码对应了下面的汇编程序 `user/initCode.s`。这个汇编程序中，

- 首先将init中的地址加载到`a0（la a0, init）`，`argv`中的地址加载到`a1（la a1, argv）`，exec系统调用对应的数字加载到`a7（li a7, SYS_exec）`
- 最后调用`ECALL`。
- 所以这里执行了3条指令，之后在第4条指令将控制权交给了操作系统。

```s
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

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

如果我在syscall中设置一个断点，并让程序运行起来。

```bash
(gdb) b syscall
Breakpoint 3 at 0x80002abc: file kernel/syscall.c, line 134.
```

- `userinit`会创建初始进程，返回到用户空间，执行刚刚介绍的3条指令，再回到内核空间。

这里是任何XV6用户会使用到的第一个系统调用。让我们来看一下会发生什么。

- 通过在`gdb`中执行`c`，让程序运行起来，我们现在进入到了`syscall`函数。

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

- 我们可以查看syscall的代码，`kenel/syscall.c`

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

`num = p->trapframe->a7`当代码执行完这一行之后，我们可以在gdb中打印num，可以看到是7。

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

如果我们查看`kenel/syscall.h`，可以看到`7`对应的是`exec`系统调用。所以，这里本质上是告诉内核，某个用户应用程序执行了`ECALL`指令，并且想要调用`exec`系统调用。

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

`p->trapframe->a0 = syscall[num]()` 这一行是**实际执行系统调用**。这里可以看出，num用来索引一个数组，这个数组是一个**函数指针数组**，可以预期的是`syscall[7]`对应了`exec`的入口函数。我们跳到这个函数中去，可以看到，我们现在在`kernel/sysfile.c`的`sys_exec`函数中。

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

`sys_exec`中的第一件事情是从用户空间读取参数，它会读取`path`，也就是要执行程序的文件名。这里首先会为参数分配空间，然后从用户空间将参数拷贝到内核空间。之后我们打印`path`，可以看到传入的就是`init`程序。

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

所以，综合来看，`initcode`完成了通过`exec`调用`init`程序。让我们来看看`user/init.c`程序，init会为用户空间设置好一些东西，比如配置好console，调用fork，并在fork出的子进程中执行shell。最终的效果就是Shell运行起来了。

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

如果我再次运行代码，我还会陷入到syscall中的断点，并且同样也是调用exec系统调用，只是这次是通过exec运行Shell。当Shell运行起来之后，我们可以从QEMU看到Shell。

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

这里简单的介绍了一下XV6是如何从0开始直到第一个Shell程序运行起来。并且我们也看了一下第一个系统调用是在什么时候发生的。我们并没有看系统调用背后的具体机制，这个在后面会介绍。

> **学生提问**：我们会处理网络吗，比如说网络相关的实验？
>**Frans教授**：是的，最后一个lab中你们会实现一个网络驱动。你们会写代码与硬件交互，操纵连接在RISC-V主板上网卡的驱动，以及寄存器，再向以太网发送一些网络报文。
