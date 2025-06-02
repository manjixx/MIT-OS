# Lec 03. OS Organization and System Calls

- [Lec 03. OS Organization and System Calls](#lec-03-os-organization-and-system-calls)
  - [3.1 课程回顾](#31-课程回顾)
  - [3.2 OS的隔离性](#32-os的隔离性)
  - [3.3 OS的防御性](#33-os的防御性)
  - [3.4 硬件对强隔离的支持](#34-硬件对强隔离的支持)
    - [3.4.1 user/kernal mode](#341-userkernal-mode)
    - [3.4.2 虚拟内存](#342-虚拟内存)
  - [3.5 User/Kernel mode 切换](#35-userkernel-mode-切换)
    - [3.5.1 **ECALL**](#351-ecall)
    - [3.5.2 举例](#352-举例)
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
- **System calls**：应用程序能够转换到内核执行的基本方法，这样用户态应用程序才能使用内核服务。
- 最后我们会看到所有的这些是如何以一种简单的方式在XV6中实现。

> **课程内容回顾**

- 首先，会有类似于`shell，echo，find`或者任何你实现的工具程序，这些**程序**运行在操作系统之上。
- 操作系统又抽象了一些**硬件资源**，例如磁盘，CPU。
- 通常来说操作系统和应用程序之前的接口被称为**系统调用接口**（System call interface），我们这门课程看到的接口都是Unix风格的接口。基于这些Unix接口，你们在lab1中，完成了不同的应用程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MIn4HtztTgrF0S7cUjs%2Fimage.png?alt=media&token=1182ec32-78aa-4e00-aba8-82c90c050982)

lab1 主要集中在理解上图中的应用程序到操作系统内核之间的接口。

## 3.2 OS的隔离性

- 隔离性（`isolation`）
- 隔离性的重要性

> **为什么需要隔离性**

- **应用程序间互不影响：** 这里的核心思想相对来说比较简单。我们在用户空间有多个应用程序，例如Shell，echo，find。但是，如果你通过Shell运行你们的Prime代码（lab1中的一个部分）时，假设你们的代码出现了问题，Shell不应该会影响到其他的应用程序。举个反例，如果Shell出现问题时，杀掉了其他的进程，这将会非常糟糕。所以你需要在不同的应用程序之间有强隔离性。
- **应用程序与操作系统间需要强隔离：** 类似的，操作系统某种程度上为所有的应用程序服务。当你的应用程序出现问题时，你会希望操作系统不会因此而崩溃。比如说你向操作系统传递了一些奇怪的参数，你会希望操作系统仍然能够很好的处理它们（能较好的处理异常情况）。所以，你也需要在应用程序和操作系统之间有强隔离性。

> **Strawman Design /No OS**

这里我们可以这样想，如果没有操作系统会怎样？我们可以用稻草人提案法（就是通过头脑风暴找出缺点）来考虑一下，如果没有操作系统，或者操作系统只是一些库文件，比如说你在使用Python，通过import os你就可以将整个操作系统加载到你的应用程序中。那么现在，我们有一个Shell，并且我们引用了代表操作系统的库。同时，我们有一些其他的应用程序例如，echo。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KILhA85NwqYCibtS%2Fimage.png?alt=media&token=192f826a-63e9-4b27-a3f5-6f09b9dd63cb)

- **缺点一：无法实现multiplexing**
  
通常来说，**如果没有操作系统，应用程序会直接与硬件交互**。比如，应用程序可以直接看到CPU的多个核，看到磁盘，内存。所以现在应用程序和硬件资源之间没有一个额外的抽象层，如下图所示。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KoKptVF_4Wk7FvsK%2Fimage.png?alt=media&token=f9fb466f-444f-4b0c-8a9f-d2792034dc34)

实际上，从隔离性的角度来看，这并不是一个很好的设计。这里你可以看到这种设计是如何破坏隔离性的。使用操作系统的一个目的是为了同时运行多个应用程序，所以时不时的，CPU会从一个应用程序切换到另一个应用程序。我们假设硬件资源里只有一个CPU核，并且我们现在在这个CPU核上运行Shell。但是时不时的，也需要让其他的应用程序也可以运行。**现在我们没有操作系统来帮我们完成切换，所以Shell就需要时不时的释放CPU资源。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2M2HKdZx0nGp_0ynT%2Fimage.png?alt=media&token=57595e20-1fe3-4cec-95d4-8a9340ec81a5)

为了不变成一个恶意程序，Shell在发现自己运行了一段时间之后，需要让别的程序也有机会能运行。这种机制有时候称为**协同调度（Cooperative Scheduling）**。但是这里的场景并没有很好的隔离性，比如说Shell中的某个函数有一个死循环，那么Shell永远也不会释放CPU，进而其他的应用程序也不能够运行，甚至都不能运行一个第三方的程序来停止或者杀死Shell程序。所以这种场景下，**我们基本上得不到真正的multiplexing（CPU在多进程同分时复用）**。而这个特性是非常有用的，不论应用程序在执行什么操作，multiplexing都会迫使应用程序时不时的释放CPU，这样其他的应用程序才能运行。

- **缺点二：无法内存隔离**

从内存的角度来说，如果应用程序直接运行在硬件资源之上，那么每个应用程序的文本，代码和数据都直接保存在物理内存中。物理内存中的一部分被Shell使用，另一部分被echo使用。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7V45GuLznMv101NEG%2Fimage.png?alt=media&token=44f17f7f-7aca-420c-9210-0ea4ba218393)

即使在这么简单的例子中，**因为两个应用程序的内存之间没有边界，如果echo程序将数据存储在属于Shell的一个内存地址中（下图中的1000），那么就echo就会覆盖Shell程序内存中的内容。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7W_gYEqzDbGZ65RA1%2Fimage.png?alt=media&token=5f133e02-7be1-4b42-85ac-f1e12f30e398)

这是非常不想看到的场景，因为echo现在渗透到了Shell中来，并且这类的问题是非常难定位的。所以这里也没有为我们提供好的隔离性。我们希望不同应用程序之间的内存是隔离的，这样一个应用程序就不会覆盖另一个应用程序的内存。

**总结**：使用操作系统的一个原因，甚至可以说是主要原因就是为了**实现multiplexing**和**内存隔离**。如果不使用操作系统，并且应用程序直接与硬件交互，就很难实现这两点。所以，将操作系统设计成一个库，并不是一种常见的设计。或许可以在一些实时操作系统中看到这样的设计，因为在这些实时操作系统中，应用程序之间彼此相互信任。但是在大部分的其他操作系统中，都会强制实现硬件资源的隔离。


> **举例**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRLmqk_Z0lli966feS%2Fimage.png?alt=media&token=c867e904-807f-45b9-8faa-5673a8ac988a)

Unix接口抽象了硬件资源从而使得提供强隔离成为可能。我们可以发现，接口被精心设计以实现资源的强隔离，也就是multiplexing和物理内存的隔离。

- fork:
  
  之前通过fork创建了进程。进程本身不是CPU，但是它们对应了CPU，它们使得你可以在CPU上运行计算任务。所以你懂的，应用程序不能直接与CPU交互，只能与进程交互。操作系统内核会完成不同进程在CPU上的切换。所以，操作系统不是直接将CPU提供给应用程序，而是向应用程序提供“进程”，进程抽象了CPU，这样操作系统才能在多个应用程序之间复用一个或者多个CPU。

  >**Q**：这里说进程抽象了CPU，是不是说一个进程使用了部分的CPU，另一个进程使用了CPU的另一部分？这里CPU和进程的关系是什么？
  >**Frans教授**：我这里真实的意思是，我们在实验中使用的RISC-V处理器实际上是有4个核。所以你可以同时运行4个进程，一个进程占用一个核。但是假设你有8个应用程序，操作系统会分时复用这些CPU核，比如说对于一个进程运行100毫秒，之后内核会停止运行并将那个进程从CPU中卸载，再加载另一个应用程序并再运行100毫秒。通过这种方式使得每一个应用程序都不会连续运行超过100毫秒。这里只是一些基本概念，我们在接下来的几节课中会具体的看这里是如何实现的。
  >**Q**：好的，但是多个进程不能在同一时间使用同一个CPU核，对吧？
  >**Frans教授**：是的，这里是分时复用。CPU运行一个进程一段时间，再运行另一个进程。

- `exec`

    **我们可以认为exec抽象了内存**。当我们在执行exec系统调用的时候，我们会传入一个文件名，而这个**文件名对应了一个应用程序的内存镜像**。内存镜像里面包括了程序对应的指令，全局的数据。应用程序可以逐渐扩展自己的内存，但是**应用程序并没有直接访问物理内存的权限**，例如应用程序不能直接访问物理内存的1000-2000这段地址。不能直接访问的原因是，操作系统会提供内存隔离并控制内存，操作系统会在应用程序和硬件资源之间提供一个中间层。exec是这样一种系统调用，它表明了应用程序不能直接访问物理内存。

- `file`

    **`files`基本上来说抽象了磁盘**。应用程序不会直接读写挂在计算机上的磁盘本身，并且在Unix中这也是不被允许的。**在Unix中，与存储系统交互的唯一方式就是通过files。** files提供了非常方便的磁盘抽象，你可以对文件命名，读写文件等等。之后，操作系统会决定如何将文件与磁盘中的块对应，确保一个磁盘块只出现在一个文件中，并且确保用户A不能操作用户B的文件。通过files的抽象，可以实现不同用户之间和同一个用户的不同进程之间的文件强隔离。

- 总结：
  
    这些都是你们在上一个lab中使用的Unix系统调用接口，或许你们已经看出来了，这些接口看起来像是经过精心的设计以抽象计算机资源，这样这些接口的实现，或者说操作系统本身可以在多个应用程序之间复用计算机硬件资源，同时还提供了强隔离性。

> **学生提问**：更复杂的内核会不会尝试将进程调度到同一个CPU核上来减少Cache Miss？
> **Frans教授**：是的。有一种东西叫做Cache affinity。现在的操作系统的确非常复杂，并且会尽量避免Cache miss和类似的事情来提升性能。我们在这门课程后面介绍高性能网络的时候会介绍更多相关的内容。
> **学生提问**：XV6的代码中，哪一部分可以看到操作系统为多个进程复用了CPU？
> **Frans教授**：有挺多文件与这个相关，但是proc.c应该是最相关的一个。两三周之后的课程中会有一个话题介绍这个内容。我们会看大量的细节，并展示操作系统的multiplexing是如何发生的。所以可以这么看待这节课，这节课的内容是对许多不同内容的初始介绍，因为我们总得从某个地方开始吧。

***

## 3.3 OS的防御性

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRW8U6v4rtiUyoZziy%2Fimage.png?alt=media&token=8b5d9da1-01e8-4367-ba5b-cd763efb20f3)

> **操作系统的防御性**

现在我们有一个操作系统，并且有一些应用程序正在运行。这里有一件事情需要考虑：操作系统应该具有防御性（Defensive）。

- **操作系统需要能够应对恶意的应用程序**。当你在做内核开发时，这是一种你需要熟悉的重要思想。操作系统需要确保所有的组件都能工作，所以它需要做好准备抵御来自应用程序的攻击。如果说应用程序无意或者恶意的向系统调用传入一些错误的参数就会导致操作系统崩溃，那就太糟糕了。在这种场景下，操作系统因为崩溃了会拒绝为其他所有的应用程序提供服务。所以操作系统需要以这样一种方式来完成：操作系统需要能够应对恶意的应用程序。

- **应用程序不能够打破对它的隔离**。应用程序非常有可能是恶意的，它或许是由攻击者写出来的，攻击者或许想要打破对应用程序的隔离，进而控制内核。一旦有了对于内核的控制能力，你可以做任何事情，因为内核控制了所有的硬件资源。

所以操作系统或者说内核需要具备防御性来避免类似的事情发生。实际中，要满足这些要求还有点棘手。在Linux中，时不时的有一些内核的bug使得应用程序可以打破它的隔离域并控制内核。这里需要持续的关注，并尽可能的提供最好的防御性。当你在开发内核时，防御性是你必须掌握的一个思想。实际中的应用程序或许就是恶意的，这意味着我们需要在应用程序和操作系统之间提供强隔离性。如果操作系统需要具备防御性，那么在应用程序和操作系统之间需要有一堵厚墙，并且操作系统可以在这堵墙上执行任何它想执行的策略。

## 3.4 硬件对强隔离的支持

通常来说，需要通过硬件来实现这的强隔离性。我们这节课会简单介绍一些硬件隔离的内容，但是在后续的课程我们会介绍的更加详细。

这里的硬件支持包括了两部分，

- **user/kernel mode**，kernel mode在RISC-V中被称为Supervisor mode但是其实是同一个东西；
- **page table**或者**虚拟内存**（Virtual Memory）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRW8U6v4rtiUyoZziy%2Fimage.png?alt=media&token=8b5d9da1-01e8-4367-ba5b-cd763efb20f3)

所以，所有的处理器，如果需要运行能够支持多个应用程序的操作系统，需要同时支持user/kernle mode和虚拟内存。具体的实现或许会有细微的差别，但是基本上来说所有的处理器需要能支持这些。我们在这门课中使用的RISC-V处理器就支持了这些功能。

### 3.4.1 user/kernal mode

首先，我们来看一下user/kernel mode，这里会以尽可能全局的视角来介绍，有很多重要的细节在这节课中都不会涉及。

为了支持user/kernel mode，**处理器会有两种操作模式**

- 第一种是user mode, 当运行在user mode时，CPU只能运行普通权限的指令（unprivileged instructions）。
- 第二种是kernel mode。当运行在kernel mode时，CPU可以运行特定权限的指令（privileged instructions）；

> **普通/特殊权限指令**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJReB5Yo_RJjpIo3RnA%2Fimage.png?alt=media&token=d7abdd2b-ab8e-4281-9725-337e95f54982)

- **普通权限的指令：** 都是一些你们熟悉的指令，例如将两个寄存器相加的指令ADD、将两个寄存器相减的指令SUB、跳转指令JRC、BRANCH指令等等。这些都是普通权限指令，所有的应用程序都允许执行这些指令。

- **特殊权限指令：**主要是一些**直接操纵硬件的指令和设置保护的指令**，例如设置page table寄存器、关闭时钟中断。在处理器上有各种各样的状态，操作系统会使用这些状态，但是只能通过特殊权限指令来变更这些状态。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRhsBp1tiieCyd1xXD%2Fimage.png?alt=media&token=4d834d3f-8458-4b1e-bdff-cf9f147a5441)

举个例子，当一个应用程序尝试执行一条特殊权限指令，因为不允许在user mode执行特殊权限指令，处理器会拒绝执行这条指令。通常来说，这时会将控制权限从user mode切换到kernel mode，当操作系统拿到控制权之后，或许会杀掉进程，因为应用程序执行了不该执行的指令。

> **RISC-V privilege架构的文档**
> 
下图是RISC-V privilege架构的文档，这个文档包括了所有的特殊权限指令。在接下来的一个月，你们都会与这些特殊权限指令打交道。我们下节课就会详细介绍其中一些指令。这里我们先对这些指令有一些初步的认识：应用程序不应该执行这些指令，这些指令只能被内核执行。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRjawN3LdVFfbgHTYB%2Fimage.png?alt=media&token=08f5a718-9279-4df6-a68a-215e17d90123)


> **学生提问**：如果kernel mode允许一些指令的执行，user mode不允许一些指令的执行，那么是谁在检查当前的mode并实际运行这些指令，并且怎么知道当前是不是kernel mode？是有什么标志位吗？
> **Frans教授**：是的，在处理器里面有一个flag。在处理器的一个bit，当它为1的时候是user mode，当它为0时是kernel mode。当处理器在解析指令时，如果指令是特殊权限指令，并且该bit被设置为1，处理器会拒绝执行这条指令，就像在运算时不能除以0一样。
> **同一个学生继续问**：所以，唯一的控制方式就是通过某种方式更新了那个bit？
> **Frans教授**：你认为是什么指令更新了那个bit位？是特殊权限指令还是普通权限指令？（等了一会，那个学生没有回答）。很明显，**设置那个bit位的指令必须是特殊权限指令**，因为应用程序不应该能够设置那个bit到kernel mode，否则的话应用程序就可以运行各种特殊权限指令了。所以那个bit是被保护的，这样回答了你的问题吗？

许多同学都已经知道了，**实际上RISC-V还有第三种模式称为machine mode**。在大多数场景下，我们会忽略这种模式，所以我也不太会介绍这种模式。 所以实际上我们有三级权限（user/kernel/machine），而不是两级(user/kernel)。

> **学生提问**：考虑到安全性，所有的用户代码都会通过内核访问硬件，但是有没有可能一个计算机的用户可以随意的操纵内核？
> **Frans教授**：并不会，至少小心的设计就不会发生这种事。或许一些程序会有额外的权限，操作系统也会认可这一点。但是这些额外的权限并不会给每一个用户，比如只有root用户有特定的权限来完成安全相关的操作。
> **同一个学生提问**：那BIOS呢？BIOS会在操作系统之前运行还是之后？
> **Frans教授**：BIOS是一段计算机自带的代码，**它会先启动，之后它会启动操作系统**，所以BIOS需要是一段可被信任的代码，它最好是正确的，且不是恶意的。


> **学生提问**：之前提到，设置处理器中kernel mode的bit位的指令是一条特殊权限指令，那么一个用户程序怎么才能让内核执行任何内核指令？因为现在切换到kernel mode的指令都是一条特殊权限指令了，对于用户程序来说也没法修改那个bit位。
> **Frans教授**：你说的对，这也是我们想要看到的结果。可以这么来看这个问题，首先这里不是完全按照你说的方式工作，在RISC-V中，如果你在用户空间（user space）尝试执行一条特殊权限指令（后面Frans那边的Zoom就断了，等他重新接入，他也没有再继续回答，所以后半段回答是我补充的）用户程序会通过系统调用来切换到kernel mode。当用户程序执行系统调用，会通过ECALL触发一个软中断（software interrupt），软中断会查询操作系统预先设定的中断向量表，并执行中断向量表中包含的中断处理程序。中断处理程序在内核中，这样就完成了user mode到kernel mode的切换，并执行用户程序想要执行的特殊权限指令。

### 3.4.2 虚拟内存

我们接下来看看硬件对于支持强隔离性的第二个特性，基本上所有的CPU都支持虚拟内存。我下节课会更加深入的讨论虚拟内存，这里先简单看一下。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSIF-TGSf1vxlD7etq%2Fimage.png?alt=media&token=aa213fe5-d78d-4520-ba08-24f8936cdfdf)

- 处理器包含了page table，而page table将**虚拟内存地址与物理内存地址**做了映射。

- **每一个进程都会有自己独立的page table**，这样的话，每一个进程只能访问出现在自己page table中的物理内存。**操作系统会设置page table，使得每一个进程都有不重合的物理内存，** 这样一个进程就不能访问其他进程的物理内存，因为其他进程的物理内存都不在它的page table中。一个进程甚至都不能随意编造一个内存地址，然后通过这个内存地址来访问其他进程的物理内存。这样就给了我们内存的强隔离性。

**page table定义了对于内存的视图**，而每一个用户进程都有自己对于内存的独立视图。这给了我们非常强的内存隔离性。如下图所示，每个矩形表示一个虚拟内存，**每个进程都有一个虚拟内存地址，从${0}$ 到 ${2^n}$。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSJsZ87csgExhbDSw1%2Fimage.png?alt=media&token=bd6c9aa4-56ce-4188-afea-416b17060c1b)

ls程序有了一个内存地址0，echo程序也有了一个内存地址0。操作系统会将两个程序的内存地址0映射到不同的物理内存地址，所以ls程序不能访问echo程序的内存，同样echo程序也不能访问ls程序的内存。

类似的，内核位于应用程序下方，假设是XV6，那么它也有自己的内存地址空间，并且与应用程序完全独立。

## 3.5 User/Kernel mode 切换

我们可以认为user/kernel mode是分隔用户空间和内核空间的边界，用户空间运行的程序运行在user mode，内核空间的程序运行在kernel mode。操作系统位于内核空间。

上一部分中我们**用矩形包括了一个程序的所有部**分，但是这里**没有描述如何从一个矩形将控制权转移到另一个矩形的**，而很明显这种转换是需要的。

例如当ls程序运行的时候，会调用read/write系统调用；Shell程序会调用fork或者exec系统调用，所以必须要有一种方式可以**使得用户的应用程序能够将控制权以一种协同工作的方式转移到内核（Entering Kernel），这样内核才能提供相应的服务**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSMyew-n45ZU01CtUw%2Fimage.png?alt=media&token=0a1dab14-28d2-4c5e-a044-0cc6902140b3)

### 3.5.1 **ECALL**

在RISC-V中，有一个专门的指令用来实现这个功能，叫做`ECALL`。当一个用户程序想要将程序执行的控制权转移到内核，它只需要执行`ECALL`指令，并传入一个数字。

**ECALL接收一个数字参数**。这里的**数字参数代表了应用程序想要调用的`System Call`**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXAkJxD8pTZOm1Tay_%2Fimage.png?alt=media&token=cf3e26a2-3c26-43b8-aee0-6d5787f8dcf5)

ECALL会跳转到内核中**一个特定的由内核控制的位置**。我们在这节课的最后可以看到在XV6中存在一个**唯一的系统调用接入点**，每一次**应用程序执行ECALL指令**，应用程序都会**通过这个接入点进入到内核中。**

### 3.5.2 举例

> **fork系统调用**

举个例子，不论是Shell还是其他的应用程序，当它在用户空间执行fork时，它并不是直接调用操作系统中对应的函数，而是调用ECALL指令，并将fork对应的数字作为参数传给ECALL。之后再通过ECALL跳转到内核。

下图中通过一根竖线来区分用户空间和内核空间，**左边是用户空间，右边是内核空间。**

- 在内核侧，有一个位于syscall.c的函数syscall，每一个从应用程序发起的系统调用都会调用到这个syscall函数，syscall函数会检查ECALL的参数，通过这个参数内核可以知道需要调用的是fork（3.9会有相应的代码跟踪介绍）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXO2n90L0ziqU8mTcg%2Fimage.png?alt=media&token=754f49c1-58a2-42d5-9427-094fc95ab613)

- 用户空间和内核空间的界限是一个硬性的界限，用户不能直接调用fork，用户的应用程序执行系统调用的唯一方法就是通过这里的ECALL指令。

> **write系统调用**

假设我现在要执行另一个系统调用write，相应的流程是类似的，write系统调用不能直接调用内核中的write代码，而是由封装好的系统调用函数执行ECALL指令。所以write函数实际上调用的是ECALL指令，指令的参数是代表了write系统调用的数字。之后控制权到了syscall函数，syscall会实际调用write系统调用。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXQ4BUMhscu13BPT1v%2Fimage.png?alt=media&token=92170af8-075f-4d3a-be59-d50852bba34c)


>**学生提问**：操作系统在什么时候检查是否允许执行fork或者write？现在看起来应用程序只需要执行ECALL再加上系统调用对应的数字就能完成调用，但是内核在什么时候决定这个应用程序是否有权限执行特定的系统调用？
>**Frans教授**：是个好问题。原则上来说，在内核侧实现fork的位置可以实现任何的检查，例如检查系统调用的参数，并决定应用程序是否被允许执行fork系统调用。在Unix中，任何应用程序都能调用fork，我们以write为例吧，write的实现需要检查传递给write的地址（需要写入数据的指针）属于用户应用程序，这样内核才不会被欺骗从别的不属于应用程序的位置写入数据。

>**学生提问**：当应用程序表现的恶意或者就是在一个死循环中，内核是如何夺回控制权限的？
>**Frans教授**：内核会通过硬件设置一个定时器，定时器到期之后会将控制权限从用户空间转移到内核空间，之后内核就有了控制能力并可以重新调度CPU到另一个进程中。我们接下来会看一些更加详细的细节。

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
8000000a:	f14025f3          	csrr	a1,mhartid
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
=> 0x000000008000000a <_entry+10>:	f3 25 40 f1	csrr	a1,mhartid
(gdb) si
0x000000008000000e in _entry ()
=> 0x000000008000000e <_entry+14>:	85 05	addi	a1,a1,1
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
13	  if(cpuid() == 0){
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

`num = p->trapframe->a7 `当代码执行完这一行之后，我们可以在gdb中打印num，可以看到是7。

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
