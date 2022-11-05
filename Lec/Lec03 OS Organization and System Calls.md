# Lec 03 OS Organization and System Calls

- Frans Kaashoek
- 课程回顾
- 操作系统的隔离性与防御性
- 硬件对强隔离的支持
- User/Kernel mode切换
- 宏内核 VS 微内核(Monolithic Kernel vs Micro Kernel)
- 编译运行kernel
- QEMU
- XV6启动过程

## 3.1 课程回顾

> **本节课内容**

- Isolation。隔离性是设计操作系统组织结构的驱动力。
- Kernel和User mode。这两种模式用来隔离操作系统内核和用户应用程序。
- System calls。系统调用是你的应用程序能够转换到内核执行的基本方法，这样你的用户态应用程序才能使用内核服务。
- 最后我们会看到所有的这些是如何以一种简单的方式在XV6中实现。

> **课程内容回顾**

- 首先，会有类似于Shell，echo，find或者任何你实现的工具程序，这些程序运行在操作系统之上。
- 操作系统又抽象了一些硬件资源，例如磁盘，CPU。
- 通常来说操作系统和应用程序之前的接口被称为系统调用接口（System call interface），我们这门课程看到的接口都是Unix风格的接口。基于这些Unix接口，你们在lab1中，完成了不同的应用程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MIn4HtztTgrF0S7cUjs%2Fimage.png?alt=media&token=1182ec32-78aa-4e00-aba8-82c90c050982)

lab1 主要集中在理解上图中的应用程序到操作系统内核之间的接口。

## 3.2 OS的隔离性

我们首先来简单的介绍一下隔离性（isolation），以及介绍为什么它很重要，为什么我们需要关心它？
这里的核心思想相对来说比较简单。我们在用户空间有多个应用程序，例如Shell，echo，find。但是，如果你通过Shell运行你们的Prime代码（lab1中的一个部分）时，假设你们的代码出现了问题，Shell不应该会影响到其他的应用程序。举个反例，如果Shell出现问题时，杀掉了其他的进程，这将会非常糟糕。所以你需要在不同的应用程序之间有强隔离性。
类似的，操作系统某种程度上为所有的应用程序服务。当你的应用程序出现问题时，你会希望操作系统不会因此而崩溃。比如说你向操作系统传递了一些奇怪的参数，你会希望操作系统仍然能够很好的处理它们（能较好的处理异常情况）。所以，你也需要在应用程序和操作系统之间有强隔离性。
这里我们可以这样想，如果没有操作系统会怎样？我们可以用稻草人提案法（就是通过头脑风暴找出缺点）来考虑一下，如果没有操作系统，或者操作系统只是一些库文件，比如说你在使用Python，通过import os你就可以将整个操作系统加载到你的应用程序中。那么现在，我们有一个Shell，并且我们引用了代表操作系统的库。同时，我们有一些其他的应用程序例如，echo。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KILhA85NwqYCibtS%2Fimage.png?alt=media&token=192f826a-63e9-4b27-a3f5-6f09b9dd63cb)

通常来说，如果没有操作系统，应用程序会直接与硬件交互。比如，应用程序可以直接看到CPU的多个核，看到磁盘，内存。所以现在应用程序和硬件资源之间没有一个额外的抽象层，如下图所示。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2KoKptVF_4Wk7FvsK%2Fimage.png?alt=media&token=f9fb466f-444f-4b0c-8a9f-d2792034dc34)

实际上，从隔离性的角度来看，这并不是一个很好的设计。这里你可以看到这种设计是如何破坏隔离性的。使用操作系统的一个目的是为了同时运行多个应用程序，所以时不时的，CPU会从一个应用程序切换到另一个应用程序。我们假设硬件资源里只有一个CPU核，并且我们现在在这个CPU核上运行Shell。但是时不时的，也需要让其他的应用程序也可以运行。现在我们没有操作系统来帮我们完成切换，所以Shell就需要时不时的释放CPU资源。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ2M2HKdZx0nGp_0ynT%2Fimage.png?alt=media&token=57595e20-1fe3-4cec-95d4-8a9340ec81a5)
为了不变成一个恶意程序，Shell在发现自己运行了一段时间之后，需要让别的程序也有机会能运行。这种机制有时候称为协同调度（Cooperative Scheduling）。但是这里的场景并没有很好的隔离性，比如说Shell中的某个函数有一个死循环，那么Shell永远也不会释放CPU，进而其他的应用程序也不能够运行，甚至都不能运行一个第三方的程序来停止或者杀死Shell程序。所以这种场景下，我们基本上得不到真正的multiplexing（CPU在多进程同分时复用）。而这个特性是非常有用的，不论应用程序在执行什么操作，multiplexing都会迫使应用程序时不时的释放CPU，这样其他的应用程序才能运行。
从内存的角度来说，如果应用程序直接运行在硬件资源之上，那么每个应用程序的文本，代码和数据都直接保存在物理内存中。物理内存中的一部分被Shell使用，另一部分被echo使用。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7V45GuLznMv101NEG%2Fimage.png?alt=media&token=44f17f7f-7aca-420c-9210-0ea4ba218393)

即使在这么简单的例子中，因为两个应用程序的内存之间没有边界，如果echo程序将数据存储在属于Shell的一个内存地址中（下图中的1000），那么就echo就会覆盖Shell程序内存中的内容。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MIh_lLv4sI790Kw_cTT%2F-MJ7W_gYEqzDbGZ65RA1%2Fimage.png?alt=media&token=5f133e02-7be1-4b42-85ac-f1e12f30e398)

这是非常不想看到的场景，因为echo现在渗透到了Shell中来，并且这类的问题是非常难定位的。所以这里也没有为我们提供好的隔离性。我们希望不同应用程序之间的内存是隔离的，这样一个应用程序就不会覆盖另一个应用程序的内存。
使用操作系统的一个原因，甚至可以说是主要原因就是为了实现multiplexing和内存隔离。如果你不使用操作系统，并且应用程序直接与硬件交互，就很难实现这两点。所以，将操作系统设计成一个库，并不是一种常见的设计。你或许可以在一些实时操作系统中看到这样的设计，因为在这些实时操作系统中，应用程序之间彼此相互信任。但是在大部分的其他操作系统中，都会强制实现硬件资源的隔离。
如果我们从隔离的角度来稍微看看Unix接口，那么我们可以发现，接口被精心设计以实现资源的强隔离，也就是multiplexing和物理内存的隔离。接口通过抽象硬件资源，从而使得提供强隔离性成为可能。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRJwuXtSpYh7KuPgo_%2Fimage.png?alt=media&token=75d00b0b-bee4-4970-8c03-d603a5d3edaa)

接下来我们举几个例子。
之前通过fork创建了进程。进程本身不是CPU，但是它们对应了CPU，它们使得你可以在CPU上运行计算任务。所以你懂的，应用程序不能直接与CPU交互，只能与进程交互。操作系统内核会完成不同进程在CPU上的切换。所以，操作系统不是直接将CPU提供给应用程序，而是向应用程序提供“进程”，进程抽象了CPU，这样操作系统才能在多个应用程序之间复用一个或者多个CPU。
学生提问：这里说进程抽象了CPU，是不是说一个进程使用了部分的CPU，另一个进程使用了CPU的另一部分？这里CPU和进程的关系是什么？
Frans教授：我这里真实的意思是，我们在实验中使用的RISC-V处理器实际上是有4个核。所以你可以同时运行4个进程，一个进程占用一个核。但是假设你有8个应用程序，操作系统会分时复用这些CPU核，比如说对于一个进程运行100毫秒，之后内核会停止运行并将那个进程从CPU中卸载，再加载另一个应用程序并再运行100毫秒。通过这种方式使得每一个应用程序都不会连续运行超过100毫秒。这里只是一些基本概念，我们在接下来的几节课中会具体的看这里是如何实现的。
学生提问：好的，但是多个进程不能在同一时间使用同一个CPU核，对吧？
Frans教授：是的，这里是分时复用。CPU运行一个进程一段时间，再运行另一个进程。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRK9FMkpjqgQaNqaIQ%2Fimage.png?alt=media&token=e450cc55-5720-4125-8cef-f61f6236eba5)

我们可以认为exec抽象了内存。当我们在执行exec系统调用的时候，我们会传入一个文件名，而这个文件名对应了一个应用程序的内存镜像。内存镜像里面包括了程序对应的指令，全局的数据。应用程序可以逐渐扩展自己的内存，但是应用程序并没有直接访问物理内存的权限，例如应用程序不能直接访问物理内存的1000-2000这段地址。不能直接访问的原因是，操作系统会提供内存隔离并控制内存，操作系统会在应用程序和硬件资源之间提供一个中间层。exec是这样一种系统调用，它表明了应用程序不能直接访问物理内存。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRKGoCUmAhvflDmhzD%2Fimage.png?alt=media&token=fe916ae2-da82-47ff-b64a-3acfcafdcd3b)

另一个例子是files，files基本上来说抽象了磁盘。应用程序不会直接读写挂在计算机上的磁盘本身，并且在Unix中这也是不被允许的。在Unix中，与存储系统交互的唯一方式就是通过files。Files提供了非常方便的磁盘抽象，你可以对文件命名，读写文件等等。之后，操作系统会决定如何将文件与磁盘中的块对应，确保一个磁盘块只出现在一个文件中，并且确保用户A不能操作用户B的文件。通过files的抽象，可以实现不同用户之间和同一个用户的不同进程之间的文件强隔离。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRLmqk_Z0lli966feS%2Fimage.png?alt=media&token=c867e904-807f-45b9-8faa-5673a8ac988a)

这些都是你们在上一个lab中使用的Unix系统调用接口，或许你们已经看出来了，这些接口看起来像是经过精心的设计以抽象计算机资源，这样这些接口的实现，或者说操作系统本身可以在多个应用程序之间复用计算机硬件资源，同时还提供了强隔离性。
这里有什么问题吗？
学生提问：更复杂的内核会不会尝试将进程调度到同一个CPU核上来减少Cache Miss？
Frans教授：是的。有一种东西叫做Cache affinity。现在的操作系统的确非常复杂，并且会尽量避免Cache miss和类似的事情来提升性能。我们在这门课程后面介绍高性能网络的时候会介绍更多相关的内容。
学生提问：XV6的代码中，哪一部分可以看到操作系统为多个进程复用了CPU？
Frans教授：有挺多文件与这个相关，但是proc.c应该是最相关的一个。两三周之后的课程中会有一个话题介绍这个内容。我们会看大量的细节，并展示操作系统的multiplexing是如何发生的。所以可以这么看待这节课，这节课的内容是对许多不同内容的初始介绍，因为我们总得从某个地方开始吧。

***

## 3.3 OS的防御性

现在我们有一个操作系统，并且有一些应用程序正在运行。这里有一件事情需要考虑：操作系统应该具有防御性（Defensive）。
当你在做内核开发时，这是一种你需要熟悉的重要思想。操作系统需要确保所有的组件都能工作，所以它需要做好准备抵御来自应用程序的攻击。如果说应用程序无意或者恶意的向系统调用传入一些错误的参数就会导致操作系统崩溃，那就太糟糕了。在这种场景下，操作系统因为崩溃了会拒绝为其他所有的应用程序提供服务。所以操作系统需要以这样一种方式来完成：操作系统需要能够应对恶意的应用程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRRWXZun4VAFPAuaEU%2Fimage.png?alt=media&token=382ae298-d0a4-4fc4-91de-d7e4b3ea5fec)

另一个需要考虑的是，应用程序不能够打破对它的隔离。应用程序非常有可能是恶意的，它或许是由攻击者写出来的，攻击者或许想要打破对应用程序的隔离，进而控制内核。一旦有了对于内核的控制能力，你可以做任何事情，因为内核控制了所有的硬件资源。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRSp-VdfBDsIS-lkI6%2Fimage.png?alt=media&token=b90c7775-99a2-41ff-a885-4f582f9e442d)

所以操作系统或者说内核需要具备防御性来避免类似的事情发生。实际中，要满足这些要求还有点棘手。在Linux中，时不时的有一些内核的bug使得应用程序可以打破它的隔离域并控制内核。这里需要持续的关注，并尽可能的提供最好的防御性。当你在开发内核时，防御性是你必须掌握的一个思想。实际中的应用程序或许就是恶意的，这意味着我们需要在应用程序和操作系统之间提供强隔离性。如果操作系统需要具备防御性，那么在应用程序和操作系统之间需要有一堵厚墙，并且操作系统可以在这堵墙上执行任何它想执行的策略。
通常来说，需要通过硬件来实现这的强隔离性。我们这节课会简单介绍一些硬件隔离的内容，但是在后续的课程我们会介绍的更加详细。这里的硬件支持包括了两部分，第一部分是user/kernel mode，kernel mode在RISC-V中被称为Supervisor mode但是其实是同一个东西；第二部分是page table或者虚拟内存（Virtual Memory）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRFZrJamt0HBB0n7Zh%2F-MJRW8U6v4rtiUyoZziy%2Fimage.png?alt=media&token=8b5d9da1-01e8-4367-ba5b-cd763efb20f3)

所以，所有的处理器，如果需要运行能够支持多个应用程序的操作系统，需要同时支持user/kernle mode和虚拟内存。具体的实现或许会有细微的差别，但是基本上来说所有的处理器需要能支持这些。我们在这门课中使用的RISC-V处理器就支持了这些功能。

## 3.4 硬件对强隔离的支持

硬件对于强隔离的支持包括了：user/kernle mode和虚拟内存。
首先，我们来看一下user/kernel mode，这里会以尽可能全局的视角来介绍，有很多重要的细节在这节课中都不会涉及。为了支持user/kernel mode，处理器会有两种操作模式，第一种是user mode，第二种是kernel mode。当运行在kernel mode时，CPU可以运行特定权限的指令（privileged instructions）；当运行在user mode时，CPU只能运行普通权限的指令（unprivileged instructions）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJReB5Yo_RJjpIo3RnA%2Fimage.png?alt=media&token=d7abdd2b-ab8e-4281-9725-337e95f54982)

普通权限的指令都是一些你们熟悉的指令，例如将两个寄存器相加的指令ADD、将两个寄存器相减的指令SUB、跳转指令JRC、BRANCH指令等等。这些都是普通权限指令，所有的应用程序都允许执行这些指令。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRftCebf2LWwoSnE7N%2Fimage.png?alt=media&token=985eda4b-b563-4619-b091-b9196a22a1d9)

特殊权限指令主要是一些直接操纵硬件的指令和设置保护的指令，例如设置page table寄存器、关闭时钟中断。在处理器上有各种各样的状态，操作系统会使用这些状态，但是只能通过特殊权限指令来变更这些状态。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRhsBp1tiieCyd1xXD%2Fimage.png?alt=media&token=4d834d3f-8458-4b1e-bdff-cf9f147a5441)

举个例子，当一个应用程序尝试执行一条特殊权限指令，因为不允许在user mode执行特殊权限指令，处理器会拒绝执行这条指令。通常来说，这时会将控制权限从user mode切换到kernel mode，当操作系统拿到控制权之后，或许会杀掉进程，因为应用程序执行了不该执行的指令。
下图是RISC-V privilege架构的文档，这个文档包括了所有的特殊权限指令。在接下来的一个月，你们都会与这些特殊权限指令打交道。我们下节课就会详细介绍其中一些指令。这里我们先对这些指令有一些初步的认识：应用程序不应该执行这些指令，这些指令只能被内核执行。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRXiMbb_-U8xBy-3_E%2F-MJRjawN3LdVFfbgHTYB%2Fimage.png?alt=media&token=08f5a718-9279-4df6-a68a-215e17d90123)

这里是硬件支持强隔离的一个方面。
学生提问：如果kernel mode允许一些指令的执行，user mode不允许一些指令的执行，那么是谁在检查当前的mode并实际运行这些指令，并且怎么知道当前是不是kernel mode？是有什么标志位吗？
Frans教授：是的，在处理器里面有一个flag。在处理器的一个bit，当它为1的时候是user mode，当它为0时是kernel mode。当处理器在解析指令时，如果指令是特殊权限指令，并且该bit被设置为1，处理器会拒绝执行这条指令，就像在运算时不能除以0一样。
同一个学生继续问：所以，唯一的控制方式就是通过某种方式更新了那个bit？
Frans教授：你认为是什么指令更新了那个bit位？是特殊权限指令还是普通权限指令？（等了一会，那个学生没有回答）。很明显，设置那个bit位的指令必须是特殊权限指令，因为应用程序不应该能够设置那个bit到kernel mode，否则的话应用程序就可以运行各种特殊权限指令了。所以那个bit是被保护的，这样回答了你的问题吗？
许多同学都已经知道了，实际上RISC-V还有第三种模式称为machine mode。在大多数场景下，我们会忽略这种模式，所以我也不太会介绍这种模式。 所以实际上我们有三级权限（user/kernel/machine），而不是两级(user/kernel)。
学生提问：考虑到安全性，所有的用户代码都会通过内核访问硬件，但是有没有可能一个计算机的用户可以随意的操纵内核？
Frans教授：并不会，至少小心的设计就不会发生这种事。或许一些程序会有额外的权限，操作系统也会认可这一点。但是这些额外的权限并不会给每一个用户，比如只有root用户有特定的权限来完成安全相关的操作。
同一个学生提问：那BIOS呢？BIOS会在操作系统之前运行还是之后？
Frans教授：BIOS是一段计算机自带的代码，它会先启动，之后它会启动操作系统，所以BIOS需要是一段可被信任的代码，它最好是正确的，且不是恶意的。
学生提问：之前提到，设置处理器中kernel mode的bit位的指令是一条特殊权限指令，那么一个用户程序怎么才能让内核执行任何内核指令？因为现在切换到kernel mode的指令都是一条特殊权限指令了，对于用户程序来说也没法修改那个bit位。
Frans教授：你说的对，这也是我们想要看到的结果。可以这么来看这个问题，首先这里不是完全按照你说的方式工作，在RISC-V中，如果你在用户空间（user space）尝试执行一条特殊权限指令（后面Frans那边的Zoom就断了，等他重新接入，他也没有再继续回答，所以后半段回答是我补充的）用户程序会通过系统调用来切换到kernel mode。当用户程序执行系统调用，会通过ECALL触发一个软中断（software interrupt），软中断会查询操作系统预先设定的中断向量表，并执行中断向量表中包含的中断处理程序。中断处理程序在内核中，这样就完成了user mode到kernel mode的切换，并执行用户程序想要执行的特殊权限指令。
我们接下来看看硬件对于支持强隔离性的第二个特性，基本上所有的CPU都支持虚拟内存。我下节课会更加深入的讨论虚拟内存，这里先简单看一下。基本上来说，处理器包含了page table，而page table将虚拟内存地址与物理内存地址做了对应。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSGWRxbsh0tfB8tNge%2Fimage.png?alt=media&token=0c365ec3-befb-40d4-a83c-2c928fb289eb)

每一个进程都会有自己独立的page table，这样的话，每一个进程只能访问出现在自己page table中的物理内存。操作系统会设置page table，使得每一个进程都有不重合的物理内存，这样一个进程就不能访问其他进程的物理内存，因为其他进程的物理内存都不在它的page table中。一个进程甚至都不能随意编造一个内存地址，然后通过这个内存地址来访问其他进程的物理内存。这样就给了我们内存的强隔离性。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSIF-TGSf1vxlD7etq%2Fimage.png?alt=media&token=aa213fe5-d78d-4520-ba08-24f8936cdfdf)

基本上来说，page table定义了对于内存的视图，而每一个用户进程都有自己对于内存的独立视图。这给了我们非常强的内存隔离性。
基于硬件的支持，我们可以重新画一下之前的一张图，我们先画一个矩形，ls程序位于这个矩形中；再画一个矩形，echo程序位于这个矩形中。每个矩形都有一个虚拟内存地址，从0开始到2的n次方。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSJsZ87csgExhbDSw1%2Fimage.png?alt=media&token=bd6c9aa4-56ce-4188-afea-416b17060c1b)

这样，ls程序有了一个内存地址0，echo程序也有了一个内存地址0。但是操作系统会将两个程序的内存地址0映射到不同的物理内存地址，所以ls程序不能访问echo程序的内存，同样echo程序也不能访问ls程序的内存。
类似的，内核位于应用程序下方，假设是XV6，那么它也有自己的内存地址空间，并且与应用程序完全独立。


## 3.5 User/Kernel mode 切换

我们可以认为user/kernel mode是分隔用户空间和内核空间的边界，用户空间运行的程序运行在user mode，内核空间的程序运行在kernel mode。操作系统位于内核空间。

![]( https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSLRk51PCuHwzbLCIs%2Fimage.png?alt=media&token=85acbd7d-0753-4917-ab71-fa3e2180103c)

你们应该将这张图记在你们的脑子中。但是基于我们已经介绍的内容，这张图有点太过严格了。因为我们用矩形包括了一个程序的所有部分，但是这里没有描述如何从一个矩形将控制权转移到另一个矩形的，而很明显这种转换是需要的，例如当ls程序运行的时候，会调用read/write系统调用；Shell程序会调用fork或者exec系统调用，所以必须要有一种方式可以使得用户的应用程序能够将控制权以一种协同工作的方式转移到内核，这样内核才能提供相应的服务。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJRnQQkLpnAR3xcpBOe%2F-MJSMyew-n45ZU01CtUw%2Fimage.png?alt=media&token=0a1dab14-28d2-4c5e-a044-0cc6902140b3)

所以，需要有一种方式能够让应用程序可以将控制权转移给内核（Entering Kernel）。
在RISC-V中，有一个专门的指令用来实现这个功能，叫做ECALL。ECALL接收一个数字参数，当一个用户程序想要将程序执行的控制权转移到内核，它只需要执行ECALL指令，并传入一个数字。这里的数字参数代表了应用程序想要调用的System Call。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXAkJxD8pTZOm1Tay_%2Fimage.png?alt=media&token=cf3e26a2-3c26-43b8-aee0-6d5787f8dcf5)

ECALL会跳转到内核中一个特定，由内核控制的位置。我们在这节课的最后可以看到在XV6中存在一个唯一的系统调用接入点，每一次应用程序执行ECALL指令，应用程序都会通过这个接入点进入到内核中。举个例子，不论是Shell还是其他的应用程序，当它在用户空间执行fork时，它并不是直接调用操作系统中对应的函数，而是调用ECALL指令，并将fork对应的数字作为参数传给ECALL。之后再通过ECALL跳转到内核。
下图中通过一根竖线来区分用户空间和内核空间，左边是用户空间，右边是内核空间。在内核侧，有一个位于syscall.c的函数syscall，每一个从应用程序发起的系统调用都会调用到这个syscall函数，syscall函数会检查ECALL的参数，通过这个参数内核可以知道需要调用的是fork（3.9会有相应的代码跟踪介绍）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXO2n90L0ziqU8mTcg%2Fimage.png?alt=media&token=754f49c1-58a2-42d5-9427-094fc95ab613)

这里需要澄清的是，用户空间和内核空间的界限是一个硬性的界限，用户不能直接调用fork，用户的应用程序执行系统调用的唯一方法就是通过这里的ECALL指令。
假设我现在要执行另一个系统调用write，相应的流程是类似的，write系统调用不能直接调用内核中的write代码，而是由封装好的系统调用函数执行ECALL指令。所以write函数实际上调用的是ECALL指令，指令的参数是代表了write系统调用的数字。之后控制权到了syscall函数，syscall会实际调用write系统调用。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJX963vbIPHKetjGrZN%2F-MJXQ4BUMhscu13BPT1v%2Fimage.png?alt=media&token=92170af8-075f-4d3a-be59-d50852bba34c)

学生提问：操作系统在什么时候检查是否允许执行fork或者write？现在看起来应用程序只需要执行ECALL再加上系统调用对应的数字就能完成调用，但是内核在什么时候决定这个应用程序是否有权限执行特定的系统调用？
Frans教授：是个好问题。原则上来说，在内核侧实现fork的位置可以实现任何的检查，例如检查系统调用的参数，并决定应用程序是否被允许执行fork系统调用。在Unix中，任何应用程序都能调用fork，我们以write为例吧，write的实现需要检查传递给write的地址（需要写入数据的指针）属于用户应用程序，这样内核才不会被欺骗从别的不属于应用程序的位置写入数据。
学生提问：当应用程序表现的恶意或者就是在一个死循环中，内核是如何夺回控制权限的？
Frans教授：内核会通过硬件设置一个定时器，定时器到期之后会将控制权限从用户空间转移到内核空间，之后内核就有了控制能力并可以重新调度CPU到另一个进程中。我们接下来会看一些更加详细的细节。
学生提问：这其实是一个顶层设计的问题，是什么驱动了操作系统的设计人员使用编程语言C？
Frans教授：啊，这是个好问题。C提供了很多对于硬件的控制能力，比如说当你需要去编程一个定时器芯片时，这更容易通过C来完成，因为你可以得到更多对于硬件资源的底层控制能力。所以，如果你要做大量的底层开发，C会是一个非常方便的编程语言，尤其是需要与硬件交互的时候。当然，不是说你不能用其他的编程语言，但是这是C成功的一个历史原因。
学生提问：为什么C比C++流行的多？仅仅是因为历史原因吗？有没有其他的原因导致大部分的操作系统并没有采用C++？
Frans教授：我认为有一些操作系统是用C++写的，这完全是可能的。但是大部分你知道的操作系统并不是用C++写的，这里的主要原因是Linus不喜欢C++，所以Linux主要是C语言实现。

## 3.6 宏内核 vs 微内核 （Monolithic Kernel vs Micro Kernel）

现在，我们有了一种方法，可以通过系统调用或者说ECALL指令，将控制权从应用程序转到操作系统中。之后内核负责实现具体的功能并检查参数以确保不会被一些坏的参数所欺骗。所以内核有时候也被称为可被信任的计算空间（Trusted Computing Base），在一些安全的术语中也被称为TCB。
基本上来说，要被称为TCB，内核首先要是正确且没有Bug的。假设内核中有Bug，攻击者可能会利用那个Bug，并将这个Bug转变成漏洞，这个漏洞使得攻击者可以打破操作系统的隔离性并接管内核。所以内核真的是需要越少的Bug越好（但是谁不是呢）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJXWRY7_m6ArPVBNgvJ%2F-MJZZnNHX8hH6zaP4OFv%2Fimage.png?alt=media&token=358a8815-9e0b-484c-adec-b7051b1622e9)

另一方面，内核必须要将用户应用程序或者进程当做是恶意的。如我之前所说的，内核的设计人员在编写和实现内核代码时，必须要有安全的思想。这个目标很难实现，因为当你的操作系统变得足够大的时候，很多事情就不是那么直观了。你知道的，几乎每一个你用过的或者被广泛使用的操作系统，时不时的都有一个安全漏洞。就算被修复了，但是过了一段时间，又会出现一个新的漏洞。我们之后会介绍为什么很难让所有部分都正确工作，但是你要知道是内核需要做一些tricky的工作，需要操纵硬件，需要非常小心做检查，所以很容易就出现一些小的疏漏，进而触发一个Bug。这也是可以理解的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJXWRY7_m6ArPVBNgvJ%2F-MJZck3H4EAuhFVSva0U%2Fimage.png?alt=media&token=2a170d6b-b942-4ac5-b943-85f4f2bc7f1d)

一个有趣的问题是，什么程序应该运行在kernel mode？敏感的代码肯定是运行在kernel mode，因为这是Trusted Computing Base。
对于这个问题的一个答案是，首先我们会有user/kernel边界，在上面是应用程序，在下面是运行在kernel mode的程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbThlJrd7ZaArRiPDt%2Fimage.png?alt=media&token=1eee503b-5a1b-46b3-9d0b-0dfb2e740603)

其中一个选项是让整个操作系统代码都运行在kernel mode。大多数的Unix操作系统实现都运行在kernel mode。比如，XV6中，所有的操作系统服务都在kernel mode中，这种形式被称为Monolithic Kernel Design（宏内核）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbVJDYYOlA_ZnJBg_i%2Fimage.png?alt=media&token=11b51e28-9327-450e-a077-9cb3488a2015)

这里有几件事情需要注意：
首先，如果考虑Bug的话，这种方式不太好。在一个宏内核中，任何一个操作系统的Bug都有可能成为漏洞。因为我们现在在内核中运行了一个巨大的操作系统，出现Bug的可能性更大了。你们可以去查一些统计信息，平均每3000行代码都会有几个Bug，所以如果有许多行代码运行在内核中，那么出现严重Bug的可能性也变得更大。所以从安全的角度来说，在内核中有大量的代码是宏内核的缺点。
另一方面，如果你去看一个操作系统，它包含了各种各样的组成部分，比如说文件系统，虚拟内存，进程管理，这些都是操作系统内实现了特定功能的子模块。宏内核的优势在于，因为这些子模块现在都位于同一个程序中，它们可以紧密的集成在一起，这样的集成提供很好的性能。例如Linux，它就有很不错的性能。
上面是对于内核的一种设计方式。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbYtEBzpGpaCAFdeZk%2Fimage.png?alt=media&token=2e005add-cd05-4201-9fab-5149d38765ab)

另一种设计主要关注点是减少内核中的代码，它被称为Micro Kernel Design（）。在这种模式下，希望在kernel mode中运行尽可能少的代码。所以这种设计下还是有内核，但是内核只有非常少的几个模块，例如，内核通常会有一些IPC的实现或者是Message passing；非常少的虚拟内存的支持，可能只支持了page table；以及分时复用CPU的一些支持。
微内核的目的在于将大部分的操作系统运行在内核之外。所以，我们还是会有user mode以及user/kernel mode的边界。但是我们现在会将原来在内核中的其他部分，作为普通的用户程序来运行。比如文件系统可能就是个常规的用户空间程序。这个文件系统我不小心画成了红色，其实我想画成黑色的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbaEhCH03MFZIN58d8%2Fimage.png?alt=media&token=4699cb28-4a59-4c64-aba5-6364df9db0c3)

现在，文件系统运行的就像一个普通的用户程序，就像echo，Shell一样，这些程序都运行在用户空间。可能还会有一些其他的用户应用程序，例如虚拟内存系统的一部分也会以一个普通的应用程序的形式运行在user mode。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbb2rd6KN3AuCoErQ-%2Fimage.png?alt=media&token=a93a6b20-563f-42d6-8284-daaaa9a0d254)

某种程度上来说，这是一种好的设计。因为在内核中的代码的数量较小，更少的代码意味着更少的Bug。
但是这种设计也有相应的问题。假设我们需要让Shell能与文件系统交互，比如Shell调用了exec，必须有种方式可以接入到文件系统中。通常来说，这里工作的方式是，Shell会通过内核中的IPC系统发送一条消息，内核会查看这条消息并发现这是给文件系统的消息，之后内核会把消息发送给文件系统。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbcOEEsVLZivNXiWaO%2Fimage.png?alt=media&token=e7b6a13d-1939-4623-a23a-6a0f10c4b439)

文件系统会完成它的工作之后会向IPC系统发送回一条消息说，这是你的exec系统调用的结果，之后IPC系统再将这条消息发送给Shell。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJbcquRoZotyofyd2sh%2Fimage.png?alt=media&token=71f32a22-7654-4116-9dee-a40537c01a13)

所以，这里是典型的通过消息来实现传统的系统调用。现在，对于任何文件系统的交互，都需要分别完成2次用户空间<->内核空间的跳转。与宏内核对比，在宏内核中如果一个应用程序需要与文件系统交互，只需要完成1次用户空间<->内核空间的跳转，所以微内核的的跳转是宏内核的两倍。通常微内核的挑战在于性能更差，这里有两个方面需要考虑：
在user/kernel mode反复跳转带来的性能损耗。
在一个类似宏内核的紧耦合系统，各个组成部分，例如文件系统和虚拟内存系统，可以很容易的共享page cache。而在微内核中，每个部分之间都很好的隔离开了，这种共享更难实现。进而导致更难在微内核中得到更高的性能。
我们这里介绍的有关宏内核和微内核的区别都特别的笼统。在实际中，两种内核设计都会出现，出于历史原因大部分的桌面操作系统是宏内核，如果你运行需要大量内核计算的应用程序，例如在数据中心服务器上的操作系统，通常也是使用的宏内核，主要的原因是Linux提供了很好的性能。但是很多嵌入式系统，例如Minix，Cell，这些都是微内核设计。这两种设计都很流行，如果你从头开始写一个操作系统，你可能会从一个微内核设计开始。但是一旦你有了类似于Linux这样的宏内核设计，将它重写到一个微内核设计将会是巨大的工作。并且这样重构的动机也不足，因为人们总是想把时间花在实现新功能上，而不是重构他们的内核。
所以这里是操作系统的两种主要设计。如你们所知的，XV6是一种宏内核设计，如大多数经典的Unix系统一样。但是在这个学期的后半部分，我们会讨论更多有关微内核设计的内容。
这里有什么问题吗？因为在课前的邮件提问中，这块问的还挺多的。（没人提问）

## 3.7 编译运行Kernel

接下来我会切换到代码介绍，来看一下XV6是如何工作的。
首先，我们来看一下代码结构，你们或许已经看过了。代码主要有三个部分组成：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJpn93PbVXmyk3cWz9q%2F-MJq7-1lrR0BNbiAaazr%2Fimage.png?alt=media&token=15dcbc6d-ed46-4bd1-bb4b-f71609e413c1)

第一个是kernel。我们可以ls kernel的内容，里面包含了基本上所有的内核文件。因为XV6是一个宏内核结构，这里所有的文件会被编译成一个叫做kernel的二进制文件，然后这个二进制文件会被运行在kernle mode中。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJdjbpZW0Es0hM8rNrS%2Fimage.png?alt=media&token=d28e7220-ad13-4f76-9b5e-4d414dda7e3c)

第二个部分是user。这基本上是运行在user mode的程序。这也是为什么一个目录称为kernel，另一个目录称为user的原因。
第三部分叫做mkfs。它会创建一个空的文件镜像，我们会将这个镜像存在磁盘上，这样我们就可以直接使用一个空的文件系统。
接下来，我想简单的介绍一下内核是如何编译的。你们可能已经编译过内核，但是还没有真正的理解编译过程，这个过程还是比较重要的。
首先，Makefile（XV6目录下的文件）会读取一个C文件，例如proc.c；之后调用gcc编译器，生成一个文件叫做proc.s，这是RISC-V 汇编语言文件；之后再走到汇编解释器，生成proc.o，这是汇编语言的二进制格式。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJbSdGiMLB2VO1kFUtK%2F-MJdman0zmPUk2wgyfGn%2Fimage.png?alt=media&token=4604dc5e-8f13-47f0-b543-0e063228c192)

Makefile会为所有内核文件做相同的操作，比如说pipe.c，会按照同样的套路，先经过gcc编译成pipe.s，再通过汇编解释器生成pipe.o。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgWjAGE9ASuK5Hbw2k%2Fimage.png?alt=media&token=cac68445-2b05-4a57-8435-6d495baed104)

之后，系统加载器（Loader）会收集所有的.o文件，将它们链接在一起，并生成内核文件。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgXiV2KBGQeuPgX4Bj%2Fimage.png?alt=media&token=80f80f91-adbc-48c7-9767-e6db633cb141)

这里生成的内核文件就是我们将会在QEMU中运行的文件。同时，为了你们的方便，Makefile还会创建kernel.asm，这里包含了内核的完整汇编语言，你们可以通过查看它来定位究竟是哪个指令导致了Bug。比如，我接下来查看kernel.asm文件，我们可以看到用汇编指令描述的内核：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJdmgC_aByY8_wjKNKA%2F-MJgYi8MwG-QEqdvZ63Q%2Fimage.png?alt=media&token=d82387cf-bc09-47a2-a9d0-7ff0eb0cd2b2)

这里你们可能已经注意到了，第一个指令位于地址0x80000000，对应的是一个RISC-V指令：auipc指令。有人知道第二列，例如0x0000a117、0x83010113、0x6505，是什么意思吗？有人想来回答这个问题吗？
学生回答：这是汇编指令的16进制表现形式对吗？
是的，完全正确。所以这里0x0000a117就是auipc，这里是二进制编码后的指令。因为每个指令都有一个二进制编码，kernel的asm文件会显示这些二进制编码。当你在运行gdb时，如果你想知道具体在运行什么，你可以看具体的二进制编码是什么，有的时候这还挺方便的。
接下来，让我们不带gdb运行XV6（make会读取Makefile文件中的指令）。这里会编译文件，然后调用QEMU（qemu-system-riscv64指令）。这里本质上是通过C语言来模拟仿真RISC-V处理器。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgd7lBxdCSukMsxDGi%2Fimage.png?alt=media&token=bfcf8e8d-34d8-44a6-b84b-e378bde5bc09)

我们来看传给QEMU的几个参数：
-kernel：这里传递的是内核文件（kernel目录下的kernel文件），这是将在QEMU中运行的程序文件。
-m：这里传递的是RISC-V虚拟机将会使用的内存数量
-smp：这里传递的是虚拟机可以使用的CPU核数
-drive：传递的是虚拟机使用的磁盘驱动，这里传入的是fs.img文件
这样，XV6系统就在QEMU中启动了。

## 3.8 QEMU


QEMU表现的就像一个真正的计算机一样。当你想到QEMU时，你不应该认为它是一个C程序，你应该把它想成是下图，一个真正的主板。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJghmTlvFOKAnfGeveB%2Fimage.png?alt=media&token=2e8f081f-1f48-43e8-8486-0352732fd28e)

图中是一个我办公室中的RISC-V主板，它可以启动一个XV6。当你通过QEMU来运行你的内核时，你应该认为你的内核是运行在这样一个主板之上。主板有一个开关，一个RISC-V处理器，有支持外设的空间，比如说一个接口是连接网线的，一个是PCI-E插槽，主板上还有一些内存芯片，这是一个你可以在上面编程的物理硬件，而XV6操作系统管理这样一块主板，你在你的脑海中应该有这么一张图。
对于RISC-V，有完整的文档介绍，比如说下图是一个RISC-V的结构图：

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgYxe3Ki7wfpMQgXe-%2F-MJgjgTH1N3waADySy12%2Fimage.png?alt=media&token=a66a77d5-f019-4ddb-921e-1a74fd055567)

这个图里面有：
4个核：U54 Core 1-4
L2 cache：Banked L2
连接DRAM的连接器：DDR Controller
各种连接外部设备的方式，比如说UART0，一端连接了键盘，另一端连接了terminal。
以及连接了时钟的接口：Clock Generation
我们后面会讨论更多的细节，但是这里基本上就是RISC-V处理器的所有组件，你通过它与实际的硬件交互。
实际上抛开一些细节，通过QEMU模拟的计算机系统或者说计算机主板，与这里由SiFive生产的计算机主板非常相似。本来想给你们展示一下这块主板的，但是我刚刚说过它在我的办公室，而我已经很久没去过办公室了，或许它已经吃了很多灰了。当你们在运行QEMU时，你们需要知道，你们基本上跟在运行硬件是一样的，只是说同样的东西，QEMU在软件中实现了而已。
当我们说QEMU仿真了RISC-V处理器时，背后的含义是什么？
直观来看，QEMU是一个大型的开源C程序，你可以下载或者git clone它。但是在内部，在QEMU的主循环中，只在做一件事情：
读取4字节或者8字节的RISC-V指令。
解析RISC-V指令，并找出对应的操作码（op code）。我们之前在看kernel.asm的时候，看过一些操作码的二进制版本。通过解析，或许可以知道这是一个ADD指令，或者是一个SUB指令。
之后，在软件中执行相应的指令。
这基本上就是QEMU的全部工作了，对于每个CPU核，QEMU都会运行这么一个循环。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgoATd2oLIEq69pgjA%2F-MJivBPG66Wv8SL-HP-D%2Fimage.png?alt=media&token=4e336562-9790-41bc-b770-acfa6da1532c)

为了完成这里的工作，QEMU的主循环需要维护寄存器的状态。所以QEMU会有以C语言声明的类似于X0，X1寄存器等等。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJgoATd2oLIEq69pgjA%2F-MJivpmcbCh1TaxGomhZ%2Fimage.png?alt=media&token=16d5ab23-cbdd-4960-b9a3-6f46caa19ebe)

当QEMU在执行一条指令，比如(ADD a0, 7, 1)，这里会将常量7和1相加，并将结果存储在a0寄存器中，所以在这个例子中，寄存器X0会是7。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJliet0hZBL75tlaCZM%2F-MJljj-HF66Hcx_5N-6M%2Fimage.png?alt=media&token=cb09f40c-ed00-494f-a5ce-1d503f48eb80)

之后QEMU会执行下一条指令，并持续不断的执行指令。除了仿真所有的普通权限指令之外，QEMU还会仿真所有的特殊权限指令，这就是QEMU的工作原理。对于你们来说，你们只需要认为你们跑在QEMU上的代码跟跑在一个真正的RISC-V处理器上是一样的，就像你们在6.004这门课程中使用过的RISC-V处理器一样。
这里有什么问题吗？
学生提问：我想知道，QEMU有没有什么欺骗硬件的实现，比如说overlapping instruction？
Frans教授：并没有，真正的CPU运行在QEMU的下层。当你运行QEMU时，很有可能你是运行在一个x86处理器上，这个x86处理器本身会做各种处理，比如顺序解析指令。所以QEMU对你来说就是个C语言程序。
学生提问：那多线程呢？程序能真正跑在4个核上吗？还是只能跑在一个核上？如果能跑在多个核上，那么QEMU是不是有多线程？
Frans教授：我们在Athena上使用的QEMU还有你们下载的QEMU，它们会使用多线程。QEMU在内部通过多线程实现并行处理。所以，当QEMU在仿真4个CPU核的时候，它是并行的模拟这4个核。我们在后面有个实验会演示这里是如何工作的。所以，（当QEMU仿真多个CPU核时）这里真的是在不同的CPU核上并行运算。

## 3.9 XV6 启动过程

接下来，我会系统的介绍XV6，让你们对XV6的结构有个大概的了解。在后面的课程，我们会涉及到更多的细节。
首先，我会启动QEMU，并打开gdb。本质上来说QEMU内部有一个gdb server，当我们启动之后，QEMU会等待gdb客户端连接。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJlqR3qqukigoqwoSGg%2Fimage.png?alt=media&token=8d691ec9-91a5-40b4-9be2-1e3bd9c694b0)

我会在我的计算机上再启动一个gdb客户端，这里是一个RISC-V 64位Linux的gdb，有些同学的电脑可能是multi-arch或者其他版本的的gdb，但是基本上来说，这里的gdb是为RISC-V 64位处理器编译的。
在连接上之后，我会在程序的入口处设置一个端点，因为我们知道这是QEMU会跳转到的第一个指令。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJls4laZcPl0Xuf6XVJ%2Fimage.png?alt=media&token=67a12dee-4c16-43ad-b4f1-0b9dc06881b7)

设置完断点之后，我运行程序，可以发现代码并没有停在0x8000000（见3.7 kernel.asm中，0x80000000是程序的起始位置），而是停在了0x8000000a。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJlsbSgYVFkERqIERkM%2Fimage.png?alt=media&token=a1a5cfd2-450f-4a48-bc32-3d16b09df6b8)

如果我们查看kernel的汇编文件，

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJlsqMcdt1J3vjsW88_%2Fimage.png?alt=media&token=81b6d55b-b7ef-4092-9c47-b5a47b4f5368)

我们可以看到，在地址0x8000000a读取了控制系统寄存器（Control System Register）mhartid，并将结果加载到了a1寄存器。所以QEMU会模拟执行这条指令，之后执行下一条指令。
地址0x80000000是一个被QEMU认可的地址。也就是说如果你想使用QEMU，那么第一个指令地址必须是它。所以，我们会让内核加载器从那个位置开始加载内核。如果我们查看kernel.ld，

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJlum8m8tZjc6IfaDfw%2Fimage.png?alt=media&token=0cbebbd4-d438-4107-812c-b48cad22cff3)

我们可以看到，这个文件定义了内核是如何被加载的，从这里也可以看到，内核使用的起始地址就是QEMU指定的0x80000000这个地址。这就是我们操作系统最初运行的步骤。
回到gdb，我们可以看到gdb也显示了指令的二进制编码

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlp5nnqbQKpEdsEdt7%2F-MJlvpQK6UN3ZE0x6K10%2Fimage.png?alt=media&token=35cf4387-181c-422b-8a05-80f13bd954dd)

可以看出，csrr是一个4字节的指令，而addi是一个2字节的指令。
我们这里可以看到，XV6从entry.s开始启动，这个时候没有内存分页，没有隔离性，并且运行在M-mode（machine mode）。XV6会尽可能快的跳转到kernel mode或者说是supervisor mode。我们在main函数设置一个断点，main函数已经运行在supervisor 
mode了。接下来我运行程序，代码会在断点，也就是main函数的第一条指令停住

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoBrC5i9D26QWYxztK%2Fimage.png?alt=media&token=0adacfc6-2b45-4dbf-83d3-aa0239ac7d94)
。
上图中，左下是gdb的断点显示，右边是main函数的源码。接下来，我想运行在gdb的layout split模式：
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoCZsIhe8t7K-YbT-2%2Fimage.png?alt=media&token=e6ffae56-9a69-403b-914b-622f1768625d)

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoCnirrECVo-lvg7dS%2Fimage.png?alt=media&token=a20802f5-715b-41b5-a942-2d571ea78584)

从这个视图可以看出gdb要执行的下一条指令是什么，断点具体在什么位置。
这里我只在一个CPU上运行QEMU（见最初的make参数），这样会使得gdb调试更加简单。因为现在只指定了一个CPU核，QEMU只会仿真一个核，我可以单步执行程序（因为在单核或者单线程场景下，单个断点就可以停止整个程序的运行）。
通过在gdb中输入n，可以挑到下一条指令。这里调用了一个名为consoleinit的函数，它的工作与你想象的完全一样，也就是设置好console。一旦console设置好了，接下来可以向console打印输出（代码16、17行）。执行完16、17行之后，我们可以在QEMU看到相应的输出。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoEyhIx-LcbBXhSi_p%2Fimage.png?alt=media&token=ad4b03fa-8bad-401f-8682-bbfdd57e8cbb)

除了console之外，还有许多代码来做初始化。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoIpUzQSmCleh6RUZF%2Fimage.png?alt=media&token=ef7b53b5-aea4-41f6-8dc8-e62d7696aa9e)

kinit：设置好页表分配器（page allocator）
kvminit：设置好虚拟内存，这是下节课的内容
kvminithart：打开页表，也是下节课的内容
processinit：设置好初始进程或者说设置好进程表单
trapinit/trapinithart：设置好user/kernel mode转换代码
plicinit/plicinithart：设置好中断控制器PLIC（Platform Level Interrupt Controller），我们后面在介绍中断的时候会详细的介绍这部分，这是我们用来与磁盘和console交互方式
binit：分配buffer cache
iinit：初始化inode缓存
fileinit：初始化文件系统
virtio_disk_init：初始化磁盘
userinit：最后当所有的设置都完成了，操作系统也运行起来了，会通过userinit运行第一个进程，这里有点意思，接下来我们看一下userinit
在继续之前，这里有什么问题吗？
学生提问：这里的初始化函数的调用顺序重要吗？
Frans教授：重要，哈哈。一些函数必须在另一些函数之后运行，某几个函数的顺序可能不重要，但是对它们又需要在其他的一些函数之后运行。
可以通过gdb的s指令，跳到userinit内部。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoLEM84J5GQs0ERl8K%2Fimage.png?alt=media&token=859a93b3-5e2a-4747-8f91-8e087eaaace4)

上图是userinit函数，右边是源码，左边是gdb视图。userinit有点像是胶水代码/Glue code（胶水代码不实现具体的功能，只是为了适配不同的部分而存在），它利用了XV6的特性，并启动了第一个进程。我们总是需要有一个用户进程在运行，这样才能实现与操作系统的交互，所以这里需要一个小程序来初始化第一个用户进程。这个小程序定义在initcode中。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoM_1o5pdIpLDMhOPF%2Fimage.png?alt=media&token=e8cd60e8-58ea-4f43-9cc1-4ceba5c5da7c)

这里直接是程序的二进制形式，它会链接或者在内核中直接静态定义。实际上，这段代码对应了下面的汇编程序。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoMxZ6GgCX-d41bKUJ%2Fimage.png?alt=media&token=4064b251-179c-405e-9ff6-0eeb129248ed)

这个汇编程序中，它首先将init中的地址加载到a0（la a0, init），argv中的地址加载到a1（la a1, argv），exec系统调用对应的数字加载到a7（li a7, SYS_exec），最后调用ECALL。所以这里执行了3条指令，之后在第4条指令将控制权交给了操作系统。
如果我在syscall中设置一个断点，

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoOCq5dfumR1y4X1S0%2Fimage.png?alt=media&token=090419af-cb12-48a4-ba1b-d0664e851503)

并让程序运行起来。userinit会创建初始进程，返回到用户空间，执行刚刚介绍的3条指令，再回到内核空间。这里是任何XV6用户会使用到的第一个系统调用。让我们来看一下会发生什么。通过在gdb中执行c，让程序运行起来，我们现在进入到了syscall函数。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoP-kYebjEA8ER-Gc6%2Fimage.png?alt=media&token=852aefb9-a988-4141-ab65-4db6e3c97c59)

我们可以查看syscall的代码，

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoQlt5JMbIVG4cHGw1%2Fimage.png?alt=media&token=c23196d7-8eeb-4b3d-a9be-d7cdb4cb636a)

num = p->trapframe->a7 会读取使用的系统调用对应的整数。当代码执行完这一行之后，我们可以在gdb中打印num，可以看到是7。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoRWZ9T4-zts6PEHww%2Fimage.png?alt=media&token=e78dbdf9-ad4c-4453-9361-4b322188c7d1)

如果我们查看syscall.h，可以看到7对应的是exec系统调用。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoRjS9TiHNsg8ERUIz%2Fimage.png?alt=media&token=fb923ba7-f2cc-4531-baa4-6f10c19642e0)

所以，这里本质上是告诉内核，某个用户应用程序执行了ECALL指令，并且想要调用exec系统调用。
p->trapframe->a0 = syscall[num]() 这一行是实际执行系统调用。这里可以看出，num用来索引一个数组，这个数组是一个函数指针数组，可以预期的是syscall[7]对应了exec的入口函数。我们跳到这个函数中去，可以看到，我们现在在sys_exec函数中。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoTR9fhh86mX_a-Pmr%2Fimage.png?alt=media&token=50d3ba16-ad8b-41a8-8499-d646937ab9c4)

sys_exec中的第一件事情是从用户空间读取参数，它会读取path，也就是要执行程序的文件名。这里首先会为参数分配空间，然后从用户空间将参数拷贝到内核空间。之后我们打印path，
可以看到传入的就是init程序。所以，综合来看，initcode完成了通过exec调用init程序。让我们来看看init程序，

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoUWGAs71Xkzmd_VMj%2Fimage.png?alt=media&token=fb172843-5060-4770-8087-3062fafdeba3)

init会为用户空间设置好一些东西，比如配置好console，调用fork，并在fork出的子进程中执行shell。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoUuazVQO0JY5JEpLc%2Fimage.png?alt=media&token=7816fac5-f09f-4ae7-94da-765d8f599f45)

最终的效果就是Shell运行起来了。如果我再次运行代码，我还会陷入到syscall中的断点，并且同样也是调用exec系统调用，只是这次是通过exec运行Shell。当Shell运行起来之后，我们可以从QEMU看到Shell。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MJlxDELJZKUZ7wmW49T%2F-MJoVSJhTglzqYodbfOS%2Fimage.png?alt=media&token=fbef4537-4aae-4a95-a721-b29fb02c413d)

这里简单的介绍了一下XV6是如何从0开始直到第一个Shell程序运行起来。并且我们也看了一下第一个系统调用是在什么时候发生的。我们并没有看系统调用背后的具体机制，这个在后面会介绍。但是目前来说，这些对于你们完成这周的syscall lab是足够了。这些就是你们在实验中会用到的部分。这里有什么问题吗？
学生提问：我们会处理网络吗，比如说网络相关的实验？
Frans教授：是的，最后一个lab中你们会实现一个网络驱动。你们会写代码与硬件交互，操纵连接在RISC-V主板上网卡的驱动，以及寄存器，再向以太网发送一些网络报文。
好的，最后让我总结一下。因为没有涉及到太多的细节，我认为syscall lab可能会比上一个utils lab简单些，但是下一个实验会更加的复杂。要想做好实验总是会比较难，别总是拖到最后才完成实验，这样有什么奇怪的问题我们还能帮帮你。好了就这样，我退了，下节课再见~