# Lec 04. Page Tables

## 课前准备

- 讲师: Frans
- 课前准备
  - 阅读 book-risv-rev1 中第3章；
  - kernal/memlayout.h
  - kernal/vm.c
  - kernal/kalloc
  - kernal/riscv.h
  - kernal/exec.c

## 4.1 课程内容简介

主题是**虚拟内存（Virtual Memory）**。具体来说会介绍**页表（page tables）**。在后面的课程中，我们还会介绍虚拟内存相关的其他内容。

> **问答**

**Q：** 我想问你们还记得课程6.004和课程6.033中虚拟内存的内容吗？

- **Frans:** 我先来说一下我自己对于虚拟内存或者页表的认知吧。当我还是个学生并第一次听到学到这个词时，我认为它还是很直观简单的。这能有多难呢？无非就是个表单，**将虚拟地址和物理地址映射起来**，实际可能稍微复杂一点，但是应该不会太难。可是当我开始通过代码管理虚拟内存，我才知道虚拟内存比较棘手，比较有趣，功能也很强大。所以，希望在接下来的几节课和几个实验中，你们也能同样的理解虚拟内存。接下来我会问几个同学你们对于虚拟内存的理解是什么。
  
- **学生1**：这就是用来存放虚拟内存到物理内存映射关系的。
  
- **学生2**：这是用来保护硬件设备的。在6.004中介绍的，虚拟地址是12bit，最终会映射到某些16bit的物理地址。
  
- **学生3：** 通过虚拟内存，每个进程都可以有独立的地址空间。通过地址管理单元（Memory Management Unit）或者其他的技术，可以将每个进程的虚拟地址空间映射到物理内存地址。虚拟地址的低bit基本一样，所以映射是以块为单位进行，同时性能也很好。

- **学生4：** 虚拟地址可以让我们对进程隐藏物理地址。通过一些聪明的操控，我们可以读写虚拟地址，最后实际读写物理地址。
- **学生5：** 虚拟内存对于隔离性来说是非常基础的。每个进程都可以认为自己有独立的内存可以使用。

刚刚的回答中，很明显有两件事情是对的。

- 存在某种形式的映射关系；
- 映射关系对于实现隔离性来说有帮助。

> **课程内容**

**隔离性是我们讨论虚拟内存的主要原因**。在接下来的两节课，尤其当我们开始通过代码管理虚拟内存之后，我们可以真正理解它的作用。这节课我们会主要**关注虚拟内存的工作机制**，之后我们会看到如何使用这里的机制来实现非常酷的功能。

**今天的内容主要是3个部分：**

- 首先我会讨论一下地址空间（Address Spaces）。
- 支持虚拟内存的硬件。当然，我介绍的是RISC-V相关的硬件。但是从根本上来说，所有的现代处理器都有某种形式的硬件，来作为实现虚拟内存的默认机制。
- **XV6中的虚拟内存代码**，并看一下**内核地址空间**和**用户地址空间**的结构(layout)。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKNl6g8JRdUm5IhM68c%2Fimage.png?alt=media&token=4f65a8f7-f278-46d6-b027-e20d86db6a40)

## 4.2 地址空间（Address Spaces）

创造虚拟内存的一个出发点是可以**通过它实现隔离性**。如果你正确的设置了`page table`，并且通过代码对它进行正确的管理，那么原则上你可以实现强隔离。

> **回顾：隔离性的最终效果**

在我们一个常出现的图中，我们有一些用户应用程序比如说Shell，cat以及你们自己在lab1创造的各种工具。在这些应用程序下面，我们有操作系统位于内核空间。我们期望的是：

- **用户程序间彼此相互独立**
- **应用程序与内核操作系统相互独立**，这样如果某个应用程序无意或者故意做了一些坏事，也不会影响到操作系统。这是我们对于隔离性的期望。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKNnmJ4XqAfHllLizrD%2Fimage.png?alt=media&token=0e93f4b4-cc25-4ad1-9dc7-a2b7e9f46179)

> **没有内存隔离时**

如果我们不做任何工作，默认情况下我们是没有内存隔离性。

在我们上节课展示的RISC-V主板上，内存是由一些DRAM芯片组成。在这些DRAM芯片中保存了程序的数据和代码。例如内存中的某一个部分是内核，包括了文本，数据，栈等等；

- 如果运行了`Shell`，内存中的某个部分就是`Shell`；
- 如果运行了`cat`程序，内存中的某个部分是`cat`程序。

上述所指都是**物理内存**，它的地址从0开始到某个大的地址结束。结束地址取决于我们的机器现在究竟有多少物理内存。所有程序都必须存在于物理内存中，否则处理器甚至都不能处理程序的指令。

如下场景，假设 **`Shell`存在于内存地址1000-2000之间**。如果`cat`出现了程序错误，**将内存地址1000，也就是`Shell`的起始地址加载到寄存器`a0`中** 。之后执行`sd $7, (a0)`，这里等效于将`7`写入内存地址`1000`。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKNz-f6MsF7yVvGAW0X%2Fimage.png?alt=media&token=7eb372f7-8a32-4651-8c87-1677788a663a)

现在`cat`程序弄乱了`Shell`程序的内存镜像，所以**隔离性被破坏了**，这是我们不想看到的现象。

> **地址空间 (Address Spaces)**

我们想要某种机制，能够将不同程序之间的内存隔离开来，类似的事情就不会发生。**一种实现方式是地址空间（Address Spaces）**。即我们给**包括内核在内的所有程序专属的地址空间**。

**当我们运行`cat`时**，它的地址空间从0到某个地址结束。**当我们运行`Shell`时**，它的地址也从0开始到某个地址结束。如果`cat`程序想要向地址1000写入数据，那么`cat`只会向它自己的地址1000，而不是`Shell`的地址`1000`写入数据。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKO-tmkrzI0URCtzr3X%2Fimage.png?alt=media&token=1f13b9ec-04fc-4b09-9a80-5d43b4664bde)

**每个程序都运行在自己的地址空间，并且这些地址空间彼此之间相互独立**。在这种不同地址空间的概念中，`cat`程序甚至都不具备引用属于`Shell`的内存地址的能力。这是我们想要达成的终极目标，因为这种方式为我们提供了强隔离性，`cat`现在不能引用任何不属于自己的内存。

所以**现在的问题**是**如何在一个物理内存上，创建不同的地址空间**，因为归根到底，我们使用的还是一堆存放了内存信息的DRAM芯片。

> **学生提问**

Q（Student）：我比较好奇物理内存的配置，因为物理内存的数量是有限的，而虚拟地址空间存在最大虚拟内存地址，但是会有很多个虚拟地址空间，所以我们在设计的时候需要将最大虚拟内存地址设置的足够小吗？

- **Frans教授**：并不必要，**虚拟内存可以比物理内存更大，物理内存也可以比虚拟内存更大**。我们马上就会看到这里是如何实现的，其实就是通过`page table`来实现，这里非常灵活。

Q（Student）：同一个学生继续问：如果有太多的进程使用了虚拟内存，有没有可能物理内存耗尽了？

- **Frans教授**：这必然是有可能的。我们接下来会看到如果你有一些大的应用程序，每个程序都有大的`page table`，并且分配了大量的内存，在某个时间你的内存就耗尽了。

Q（Frans教授）：大家们，**在XV6中从哪可以看到内存耗尽了**？如果你们完成了`syscall`实验，你们会知道在`syscall`实验中有一部分是打印剩余内存的数量。

- **Student**：`kalloc`？
- **Frans教授**：是的，`kalloc`保存了空余`page`的列表，如果这个列表为空或者耗尽了，那么`kalloc`会返回一个空指针，内核会妥善处理并将结果返回给用户应用程序。并告诉用户应用程序，要么是对这个应用程序没有额外的内存了，要么是整个机器都没有内存了。
内核的一部分工作就是优雅的处理这些情况，这里的优雅是**指向用户应用程序返回一个错误消息，而不是直接崩溃。**

## 4.3 页表（Page Table）

最常见的方法而且非常灵活的实现地址空间的一种方法就是使用**页表（Page Tables）**。**页表**是在硬件中通过**处理器**和**内存管理单元（Memory Management Unit）** 实现。

> **page table（HW）**

- CPU正在执行指令，例如`sd $7, (a0)`。对于任何一条带有地址的指令，其中的**地址应该认为是虚拟内存地址**而不是物理地址。假设寄存器`a0`中是地址`0x1000`，那么这是一个虚拟内存地址。

- **虚拟内存地址会被转到内存管理单元**（MMU，Memory Management Unit）

- **内存管理单元会将虚拟地址翻译成物理地址**。之后这个物理地址会被用来索引物理内存，并从物理内存加载，或者向物理内存存储数据。

- 为了能够**完成虚拟内存地址到物理内存地址的翻译**，MMU会有一个表单，表单中，一边是**虚拟内存地址**，另一边是**物理内存地址**。举个例子，虚拟内存地址`0x1000`对应了一个物理内存地址`0xFFF0`。这样的表单可以非常灵活。

- **内存地址对应关系的表单保存在内存中**。*CPU中需要有一些寄存器用来存放表单在物理内存中的地址。* 在内存的某个位置保存了地址关系表单，我们假设这个位置的物理内存地址是`0x10`。在 **RISC-V上一个叫做SATP的寄存器会保存地址`0x10`。** 这样，CPU就可以告诉MMU，可以从哪找到将虚拟内存地址翻译成物理内存地址的表单。

- every app has his own map.

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKONgZr-r8W5uRpknWQ%2Fimage.png?alt=media&token=30b484c3-ca16-43f7-a457-aba16023ef0d)

这里的基本想法是**每个应用程序都有自己独立的表单**，并且这个表单定义了应用程序的地址空间。**所以当操作系统将CPU从一个应用程序切换到另一个应用程序时，同时也需要切换SATP寄存器中的内容，从而指向新的进程保存在物理内存中的地址对应表单。** 这样的话，cat程序和Shell程序中相同的虚拟内存地址，就可以翻译到不同的物理内存地址，因为每个应用程序都有属于自己的不同的地址对应表单。

> **学生提问1**

**Q（Student）**：学生提问：所以**MMU并不会保存`page table`，它只会从内存中读取`page table`，然后完成翻译**，是吗？

- **Frans教授**：是的，这就是你们应该记住的。`page table`保存在内存中，MMU只是会去查看`page table`，我们接下来会看到，`page table`比我们这里画的要稍微复杂一些。

**Q（Student）**：刚刚说到`SATP`寄存器会根据进程而修改，我猜每个进程对应的`SATP`值是由kernel保存的？

- **Frans教授**：是的。**内核会写`SATP`寄存器，写`SATP`寄存器是一条特殊权限指令。** 所以，用户应用程序不能通过更新这个寄存器来更换一个地址对应表单，否则的话就会破坏隔离性。所以，只有运行在`kernel mode`的代码可以更新这个寄存器。

> **以page为粒度的地址转换表**

前面都是最基本的介绍，我们在前面画的图还有做的解释都比较初级且存在明显不合理的地方。

- 这里的表单是如何工作的？
- 从刚刚画的图看来，对于每个虚拟地址，在表单中都有一个条目，如果我们真的这么做，表单会有多大？
- 原则上说，在RISC-V上会有多少地址，或者一个寄存器可以保存多少个地址？
- 寄存器是64bit的，所以有多少个地址呢？`2^64`个地址，所以如果我们**以地址为粒度来管理**，表单会变得非常巨大。实际上，所有的内存都会被这里的表单耗尽，所以这一点也不合理。实际情况不可能是一个虚拟内存地址对应page table中的一个条目。

接下来**我将分两步介绍RISC-V中是如何工作的**。

**第一步**：**不要为每个地址创建一条表单条目，而是为每个`page`创建一条表单条目**，因此每一次地址翻译都是针对一个`page`。而`RISC-V`中，一个`page`的大小是`4KB`，也就是`4096Bytes`。

因为RISC-V的寄存器是64bit，所以RISC-V的**虚拟内存地址都是64bit**。但是实际上，**当我们使用的RSIC-V处理器上，并不是所有的64bit都被使用了，高25bit并没有被使用**。

- 这样的结果是限制了虚拟内存地址的数量，**虚拟内存地址的数量现在只有$2^{39}$个，大概是512GB**。当然，如果必要的话，最新的处理器或许可以支持更大的地址空间，只需要将未使用的25bit拿出来做为虚拟内存地址的一部分即可。
- 在剩下的`39bit`中，我们将它划分为两个部分,当MMU在做地址翻译的时候:
  - `27bit`被用来当做`index`，`index`用来查找`page`,通过读取虚拟内存地址中的index可以知道物理内存中的page号，这个page号对应了物理内存中的4096个字节。
  - `12bit`被用来当做`offset`。`offset`对应的是一个`page`中的哪个字节。**`offset`必须是`12bit`，因为对应了一个`page`的`4096`个字节。** 之后虚拟内存地址中的offset指向了page中的4096个字节中的某一个，假设offset是12，那么page中的第12个字节被使用了。将offset加上page的起始地址，就可以得到物理内存地址。

**RISC-V的物理内存,** 在RISC-V中，物理内存地址是`56bit`:

- 其中`44bit`是物理`page`号（PPN，Physical Page Number）
- 剩下`12bit`是`offset`完全继承自虚拟内存地址
- 也就是地址转换时，只需要将虚拟内存中的`27bit`翻译成物理内存中的`44bit`的`page`号，剩下的`12bit`的`offset`直接拷贝过来即可

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKP01z4cRvXo7hN7r3Z%2Fimage.png?alt=media&token=eeb0d866-b4b7-4c8e-8d4b-13ffa7e72a56)

> **学生提问2**

这里有什么问题吗？这些的内容还挺重要的，你们需要掌握这的内容才能做出下一个page table lab。

**Q（Student）**：我想知道`4096`字节作为一个`page`，这在物理内存中是连续的吗？所以`offset`才是`12bit`，这样就足够覆盖`4096`个字节？图中的`56bit`又是根据什么确定的？

- Frans教授：是的，在物理内存中，这是连续的`4096`个字节。所以物理内存是以`4096`为粒度使用的。`page`中的每个字节都可以被`offset`索引到。这是由硬件设计人员决定的。所以RISC-V的设计人员认为` `的物理内存地址是个不错的选择。可以假定，他们是通过技术发展的趋势得到这里的数字。比如说，设计是为了满足5年的需求，可以预测物理内存在5年内不可能超过$2^{56}$这么大。或许，他们预测是的一个小得多的数字，但是为了防止预测错误，他们选择了像$2^{56}$这么大的数字。这里说的通吗？很多同学都问了这个问题。

Q（Student）：如果虚拟内存最多是$2^{27}$（最多应该是$2^{39}$），而物理内存最多是$2^{56}$，这样我们可以有多个进程都用光了他们的虚拟内存，但是物理内存还有剩余，对吗？

- Frans教授：是的，完全正确。

Q（Student）:这是一个64bit的机器，为什么硬件设计人员本可以用64bit但是却用了56bit？

- Frans教授：选择56bit而不是64bit是因为在主板上只需要56根线。

Q（Student）：我们从CPU到MMU之后到了内存，但是不同的进程之间的怎么区别？比如说Shell进程在地址0x1000存了一些数据，ls进程也在地址0x1000也存了一些数据，我们需要怎么将它们翻译成不同的物理内存地址。

- Frans教授：SATP寄存器包含了需要使用的地址转换表的内存地址。所以ls有自己的地址转换表，cat也有自己的地址转换表。每个进程都有完全属于自己的地址转换表。

> **以page为粒度的地址转换表存在的问题**

通过前面的第一步，我们现在是的地址转换表是以page为粒度，而不是以单个内存地址为粒度，现在这个地址转换表已经可以被称为`page table`了。**但是目前的设计还不能满足实际的需求。** 如果每个进程都有自己的page table，那么**每个page table表会有多大呢？**

这个`page table`最多会有$2^{27}$个条目（虚拟内存地址中的`index`长度为27），这是个非常大的数字。**如果每个进程都使用这么大的`page table`，进程需要为`page table`消耗大量的内存，并且很快物理内存就会耗尽。**
所以实际上，硬件并不是按照这里的方式来存储`page table`。从概念上来说，你可以认为`page table`是从0到$2^{27}$，但是实际上并不是这样。

> **多级页表**

实际中，`page table`是一个多级的结构。我们之前提到的虚拟内存地址中的`27bit`的`index`，实际上是由`3`个`9bit`的数字组成（`L2，L1，L0`）

- 前9个bit被用来索引最高级的`page directory`。实际应用中`SATP`寄存器会指向最高一级的`page directory`的物理内存地址，通过`SATP`与`L2`的9个bit得到一个`PPN`，**这个`PPN`指向了中间级的`page directory`。**，即二级页表号
- 然后由高级`page directory`中得到的`PPN`与**虚拟内存地址中的L1部分的9个bit**。得到一个中级`page directory`中的一个`PPN`,**这个`PPN`指向了最低级的`page directory`。**，即三级页表号
- 然后由三级页表号与**虚拟内存地址中的L0部分的9个bit**得到物理页表号。
- 最后由物理页表号+偏移量得到最终物理地址。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKQ-XlNXF40rUbmymdz%2Fimage.png?alt=media&token=7df83c67-2357-45dd-9a0f-1fae410c7ef9)

从某种程度上来说，与之前一种方案还是很相似的，除了实际的索引是由3步，而不是1步完成。**这种方式的主要优点是**，如果地址空间中大部分地址都没有使用，你不必为每一个index准备一个条目。

**举个例子**，如果你的地址空间只使用了一个`page`，`4096Bytes`。除此之外，你没有使用任何其他的地址。现在，你需要多少个`page table entry`，或者`page table directory`来映射这一个`page`？

- **在最高级**，你需要一个`page directory`。在这个`page directory`中，你需要一个数字是`0`的`PTE`，指向中间级`page directory`。
- **在中间级**，你也需要一个`page directory`，里面也是一个数字`0`的PTE，指向最低级`page directory`。所以这里总共需要3个`page directory`（也就是3 * 512个条目）。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKQ1rVeQD0Abge87Lxz%2Fimage.png?alt=media&token=23f4ddfc-9fd7-4319-8f8d-3c0b15db5a45)

**而在前一个方案中**，虽然我们只使用了一个page，还是需要$2^{27}$个PTE。这个方案中，我们只需要$3 * 512$个PTE。所需的空间大大减少了。这是实际上硬件采用这种层次化的3级`page directory`结构的主要原因。

> **学生提问3**

**Q（Student）**：既然每个物理`page`的`PPN`是`44bit`，而物理地址是`56bit`，我们从哪得到缺失的12bit？

- Frans教授：所有的page directory传递的都是PPN，对应的物理地址是44bit的PPN加上12bit的0（注，也就是page的起始地址，因为每个page directory都使用一个完整的page，所以直接从page起始地址开始使用就行）。如果我们查看这里的PTE条目，它们都有相同的格式，其中44bit是PPN，但是寄存器是64bit的，所有有一些bit是留空的。实际上，支持page的硬件在低10bit存了一些标志位用来控制地址权限。
​![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKQ3oLlaUanoFBXrOu6%2F-MKVBwX1eaJLgJ-ezYJN%2Fimage.png?alt=media&token=0b60fbd4-7800-420a-a41c-3b9c6e12192c)
如果你把44bit的PPN和10bit的Flags相加是54bit，也就是说还有10bit未被使用，这10bit被用来作为未来扩展。比如说某一天你有了一个新的RISC-V处理器，它的page table可能略有不同，或许有超过44bit的PPN。如果你看下面这张图，你可以看到，这里有10bit是作为保留字段存在的。
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKQ3oLlaUanoFBXrOu6%2F-MKVD5xKfkcui0853IM6%2Fimage.png?alt=media&token=d4198af2-e6ac-4af4-b1b2-cf600a7bebd1)

> **PTE中的Flag**

接下来，让我们看看PTE中的Flag，因为它也很重要。**每个PTE的低10bit是一堆标志位**：

- **第一个标志位是`Valid`**。如果`Valid bit`位为1，那么表明这是一条合法的`PTE`，你可以用它来做地址翻译。对于刚刚举的应用程序只用了1个page的例子，我们只使用了3个`page directory`，每`个page directory`中只有第0个PTE被使用了，所以只有第0个PTE的Valid bit位会被设置成1，**其他的511个PTE的Valid bit为0。** 这个标志位告诉MMU，你不能使用这条PTE，因为这条PTE并不包含有用的信息。

- 下两个标志位分别是 **`Readable`和`Writable`**。表明你是否可以读/写这个page。
- **`Executable`** 表明你可以从这个page执行指令。
- **`User`** 表明这个`page`可以被运行在用户空间的进程访问。
- 其他标志位并不是那么重要，他们偶尔会出现，前面5个是重要的标志位。

> **学生提问4**

**Q（Student）**：我对于这里的3个page table有个问题。**PPN是如何合并成最终的物理内存地址**？

- Frans教授：我之前或许没有很直接的说这部分（其实是有介绍的）。在最高级的`page directory`中的PPN，包含了下一级page directory的物理内存地址，依次类推。在最低级page directory，我们还是可以得到44bit的PPN，这里包含了我们实际上想要翻译的物理page地址，然后再加上虚拟内存地址的12bit offset，就得到了56bit物理内存地址。

Q（Frans教授）：让我来问自己的一个有趣的问题，为什么是PPN存保存的是物理页面编号？而不是一个虚拟内存地址？

- A（Student）：因为我们需要查找内存，在物理内存中查找下一个page directory的地址。

Q（Frans教授）：是的，我们不能让我们的地址翻译依赖于另一个翻译，否则我们可能会陷入递归的无限循环中。所以`page directory`必须存物理地址。那`SATP`呢？它存的是物理地址还是虚拟地址？

- A（Student）：还是物理地址，因为最高级的`page directory`还是存在物理内存中，对吧。
  
- Frans教授：是的，这里必须是物理地址，因为我们要用它来完成地址翻译，而不是对它进行地址翻译。所以`SATP`需要知道最高一级的`page directory`的物理地址是什么。

Q（Student）： 这里有层次化的3个`page table`，每个`page table`都由虚拟地址的9个bit来索引，所以是由虚拟地址中的`3`个`9bit`来分别索引`3`个`page table`，对吗？

- Frans教授：是的，最高的9个bit用来索引最高一级的page directory，第二个9bit用来索引中间级的page directory，第三个9bit用来索引最低级的page directory。

Q（Student）：当一个进程请求一个虚拟内存地址时，`CPU`会查看`SATP`寄存器得到对应的最高一级`page table`，这级`page table`会使用虚拟内存地址中`27bit index`的最高`9bit`来完成索引，如果索引的结果为空，`MMU`会自动创建一个`page table`吗？

- Frans教授：不会的，MMU会告诉操作系统或者处理器，抱歉我不能翻译这个地址，最终这会变成一个`page fault`。如果一个地址不能被翻译，那就不翻译。就像你在运算时除以0一样，处理器会拒绝那样做。

Q（Student）：我想知道我们是怎么计算`page table`的物理地址，是不是这样，我们从最高级的`page table`得到`44bit`的`PPN`，然后再加上虚拟地址中的`12bit offset`，就得到了完整的`56bit page table`物理地址？

- Frans教授：我们不会加上虚拟地址中的`offset`，这里只是使用了`12bit的0`。所以我们用`44bit`的`PPN`，再加上`12bit`的`0`，这样就得到了下一级`page directory`的`56bit`物理地址。**这里要求每个`page directory`都与物理`page`对齐**（也就是`page directory`的起始地址就是某个`page`的起始地址，所以低`12bit`都为`0`）。

## 4.4 页表缓存（Translation Lookaside Buffer）

> **TLB**

如果我们回想一下page table的结构，可以发现，**当处理器从内存加载或者存储数据时，基本上都要做3次内存查找。** 第一次在最高级的`page directory`,第二次在中间级的`page directory`,最后一次在最低级的`page directory`。对于一个虚拟内存地址的寻址，需要读三次内存，这里代价有点高。

所以所以实际中，几乎所有的处理器都会对于最近使用过的虚拟地址的翻译结果有缓存。这个缓存被称为：**`Translation Lookside Buffer（通常翻译成页表缓存）`**。你会经常看到它的缩写`TLB`。基本上来说，这就是`Page Table Entry`的缓存，也就是`PTE`的缓存。

当处理器第一次查找一个虚拟地址时，硬件通过3级`page table`得到最终的`PPN`，`TLB`会保存虚拟地址到物理地址的映射关系。这样下一次当你访问同一个虚拟地址时，处理器可以查看`TLB`，`TLB`会直接返回物理地址，而不需要通过` `得到结果。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKVKrpl8kcsGqWpnu7W%2F-MKXNac5VI27pJzzCyG6%2Fimage.png?alt=media&token=a9d33391-6989-4f31-b102-5d6d839d1e9c)

> **学生提问**

**Q（Student）：** 前面说TLB会保存虚拟地址到物理地址的对应关系，如果在page级别做cache是不是更加高效？

- Frans教授：有很多种方法都可以实现TLB，对于你们来说最重要的是知道TLB是存在的。TLB实现的具体细节不是我们要深入讨论的内容。**这是处理器中的一些逻辑，对于操作系统来说是不可见的，操作系统也不需要知道TLB是如何工作的。** 你们需要知道TLB存在的唯一原因是，**如果你切换了page table，操作系统需要告诉处理器当前正在切换page table**，处理器会清空TLB。因为本质上来说，如果你切换了`page table`，TLB中的缓存将不再有用，它们需要被清空，否则地址翻译可能会出错。所以操作系统知道TLB是存在的，但只会时不时的告诉操作系统，现在的`TLB`不能用了，因为要切换`page table`了。在`RISC-V`中，清空`TLB`的指令是`sfence_vma`。
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKVKrpl8kcsGqWpnu7W%2F-MKXQkbjRFAud6aDXUd7%2Fimage.png?alt=media&token=c889f66d-b687-4a7b-95c6-1b44157e08e7)

**Q(Student)**：3级的`page table`是由操作系统实现的还是由硬件自己实现的？

- Frans教授：这是由硬件实现的，所以3级 `page table`的查找都发生在硬件中。**MMU是硬件的一部分而不是操作系统的一部分**。在`XV6`中，有一个函数也实现了`page table`的查找，因为时不时的XV6也需要完成硬件的工作，所以`XV6`有这个叫做`walk`的函数，它在软件中实现了`MMU`硬件相同的功能。
  
**Q(Student)**：在这个机制中，`TLB`发生在哪一步，是在地址翻译之前还是之后？

- Frans教授：整个`CPU`和`MMU`都在处理器芯片中，所以在一个`RISC-V`芯片中，有多个`CPU`核，`MMU`和`TLB`存在于每一个`CPU`核里面。`RISC-V`处理器有`L1 cache，L2 Cache`，有些`cache`是根据物理地址索引的，有些`cache`是根据虚拟地址索引的，由虚拟地址索引的`cache`位于`MMU`之前，由物理地址索引的`cache`位于`MMU`之后。

**Q(Student)**：之前提到，硬件会完成3级 `page table`的查找，那为什么我们要在`XV6`中有一个`walk`函数来完成同样的工作？

- **Frans教授**：非常好的问题。这里有几个原因:
  - 首先`XV6`中的`walk`函数设置了最初的`page table`，它需要对`3`级`page table`进行编程所以它首先需要能模拟3级`page table`。
  - 另一个原因或许你们已经在`syscall`实验中遇到了，在`XV6`中，内核有它自己的`page table`，用户进程也有自己的`page table`，用户进程指向`sys_info`结构体的指针存在于用户空间的`page table`，但是内核需要将这个指针翻译成一个自己可以读写的物理地址。如果你查看`copy_in，copy_out`，你可以发现内核会通过用户进程的`page table`，将用户的虚拟地址翻译得到物理地址，这样内核可以读写相应的物理内存地址。这就是为什么在XV6中需要有walk函数的一些原因。

**Q(Student)：** 为什么硬件不开发类似于`walk`函数的接口？这样我们就不用在`XV6`中用软件实现自己的接口，自己实现还容易有`bug`。为什么没有一个特殊权限指令，接收虚拟内存地址，并返回物理内存地址？

- Frans教授：其实这就跟你向一个虚拟内存地址写数据，硬件会自动帮你完成工作一样（工作是指翻译成物理地址，并完成数据写入）。你们在page table实验中会完成相同的工作。我们接下来在看XV6的实现的时候会看到更多的内容。

在我们介绍XV6之前，有关`page table`我还想说一点。用时髦的话说，`page table`提供了一层抽象。我这里说的抽象就是指从虚拟地址到物理地址的映射。这里的映射关系完全由操作系统控制。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKXQyrwtx5ZQyDCznzw%2F-MK_UI6DE5dlgsYM2EGL%2Fimage.png?alt=media&token=67d5eef6-01f3-495c-8482-73082fd52345)

因为操作系统对于这里的地址翻译有完全的控制，它可以实现各种各样的功能。比如，当一个`PTE`是无效的，硬件会返回一个`page fault`，对于这个`page fault`，操作系统可以更新 `page table`并再次尝试指令。所以，通过操纵`page table`，在运行时有各种各样可以做的事情。我们在之后有一节课专门会讲，当出现page fault的时候，操作系统可以做哪些有意思的事情。现在只需要记住，`page table`是一个无比强大的机制，它为操作系统提供了非常大的灵活性。这就是为什么`page table`如此流行的一个原因。

## 4.5 Kernel Page Table

接下来，我们看一下在XV6中，`page table`是如何工作的？首先我们来看一下`kernel page`的分布。下图就是内核中地址的对应关系，**左边是内核的虚拟地址空间**，**右边上半部分是物理内存或者说是DRAM，右边下半部分是I/O设备。**接下来我会首先介绍右半部分，然后再介绍左半部分。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaY9xY8MaH5XTiwuBm%2Fimage.png?alt=media&token=3adbe628-da78-472f-8e7b-3d0b1d3177b5)

### 4.5.1 Physical Address

图中的**右半部分的结构完全由硬件设计者决定**。如你们上节课看到的一样，当操作系统启动时，会从地址`0x80000000`开始运行，这个地址其实也是由硬件设计者决定的。

如果你们看一个主板：中间是RISC-V处理器，我们现在知道了处理器中有4个核，每个核都有自己的MMU和TLB。处理器旁边就是DRAM芯片。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaZik5ig19xs1OK5WX%2Fimage.png?alt=media&token=c5b6e5e2-30a3-4e91-a0e3-2c8ae7cf9b7d)

主板的设计人员决定了，在完成了虚拟到物理地址的翻译之后，如果得到的物理地址大于`0x80000000`会走向`DRAM`芯片，如果得到的物理地址低于`0x80000000`会走向不同的I/O设备。这是由这个主板的设计人员决定的物理结构。如果你想要查看这里的物理结构，你可以阅读主板的手册，手册中会一一介绍物理地址对应关系。地址`0`是保留的，地址`0x10090000`对应以太网，地址`0x80000000`对应DDR内存，处理器外的易失存储（Off-Chip Volatile Memory），也就是主板上的DRAM芯片。

在你们的脑海里应该要记住这张主板的图片，即使我们接下来会基于你们都知道的C语言程序---`QEMU`来做介绍，但是最终所有的事情都是由主板硬件决定的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaa841bcyITdSeXgSv%2Fimage.png?alt=media&token=722903cf-faa3-4e99-8bd3-3198d3a7cf59)

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaaGcY8o95G9RhrbP9%2Fimage.png?alt=media&token=eb60fc6f-da2d-4a8d-9364-c17f5c0f9ac5)

> **学生提问1**

**Q(Student)**：当你说这里是由硬件决定的，硬件是特指CPU还是说CPU所在的主板？

- **Frans教授**：**CPU所在的主板**。CPU只是主板的一小部分，DRAM芯片位于处理器之外。是主板设计者将处理器，DRAM和许多I/O设备汇总在一起。对于一个操作系统来说，CPU只是一个部分，I/O设备同样也很重要。所以当你在写一个操作系统时，你需要同时处理CPU和I/O设备，比如你需要向互联网发送一个报文，操作系统需要调用网卡驱动和网卡来实际完成这个工作。

**回到最初那张图的右侧：物理地址的分布**。

- 可以看到最下面是未被使用的地址，这与主板文档内容是一致的（地址为0）。
- 地址`0x1000`是`boot ROM`的物理地址，当你对主板上电，主板做的第一件事情就是运行存储在`boot ROM`中的代码
- 当`boot`完成之后，会跳转到地址`0x80000000`，操作系统需要确保那个地址有一些数据能够接着启动操作系统。

这里还有一些其他的I/O设备：

- **PLIC是中断控制器**（`Platform-Level Interrupt Controller`）我们下周的课会讲。
- **CLINT（`Core Local Interruptor`）** 也是中断的一部分。所以多个设备都能产生中断，需要中断控制器来将这些中断路由到合适的处理函数。地址`0x02000000`对应CLINT，当你向这个地址执行读写指令，你是向实现了CLINT的芯片执行读写。这里你可以认为你直接在与设备交互，而不是读写物理内存。
- **UART0（`Universal Asynchronous Receiver/Transmitter`）** 负责与`Console`和显示器交互。
- **`VIRTIO disk`**，与磁盘进行交互。
  
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaY9xY8MaH5XTiwuBm%2Fimage.png?alt=media&token=3adbe628-da78-472f-8e7b-3d0b1d3177b5)

> **学生提问2**

**Q(Student)**：确认一下，低于`0x80000000`的物理地址，不存在于`DRAM`中，当我们在使用这些地址的时候，指令会直接走向其他的硬件，对吗？

- **Frans教授**：是的。高于`0x80000000`的物理地址对应DRAM芯片，但是对于例如以太网接口，也有一个特定的低于`0x80000000的`物理地址，我们可以对这个叫做内存映射`I/O`（`Memory-mapped I/O`）的地址执行读写指令，来完成设备的操作。

**Q(Student)**：为什么物理地址最上面一大块标为未被使用？

- Frans教授：物理地址总共有2^56那么多，但是你不用在主板上接入那么多的内存。所以不论主板上有多少DRAM芯片，总是会有一部分物理地址没有被用到。实际上在XV6中，我们限制了内存的大小是128MB。

**Q(Student)**：当读指令从`CPU`发出后，它是怎么路由到正确的`I/O`设备的？比如说，当CPU要发出指令时，它可以发现现在地址是低于`0x80000000`，但是它怎么将指令送到正确的`I/O`设备？

- Frans教授：你可以认为在RISC-V中有一个多路输出选择器（`demultiplexer`）。

### 4.5.2 Virtual Address

接下来我会切换到第一张图的左边，这就是XV6的虚拟内存地址空间。**当机器刚刚启动时，还没有可用的page**，XV6操作系统会设置好内核使用的虚拟地址空间，也就是这张图左边的地址分布。

因为我们想让XV6尽可能的简单易懂，所以这里的**虚拟地址到物理地址的映射，大部分是相等的关系**。比如说内核会按照这种方式设置`page table`，虚拟地址`0x02000000`对应物理地址`0x02000000`。这意味着**左侧低于PHYSTOP的虚拟地址，与右侧使用的物理地址是一样的。**这里的箭头都是水平的，因为这里是**完全相等的映射**。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaY9xY8MaH5XTiwuBm%2Fimage.png?alt=media&token=3adbe628-da78-472f-8e7b-3d0b1d3177b5)

除此之外，这里还有两件重要的事情：

**第一件事情是，有一些page在虚拟内存中的地址很靠后**

比如`kernel stack`在虚拟内存中的地址就很靠后。这是因为在它之下有一个未被映射的`Guard page`，这个`Guard page`对应的`PTE`的`Valid` 标志位没有设置，这样，如果`kernel stack`耗尽了，它会溢出到`Guard page`，但是因为`Guard page`的`PTE`中`Valid`标志位未设置，会导致立即触发`page fault`，这样的结果好过内存越界之后造成的数据混乱。立即触发一个`panic`（也就是`page fault`），你就知道`kernel stack`出错了。同时我们也又不想浪费物理内存给`Guard page`，所以`Guard page`不会映射到任何物理内存，它只是占据了虚拟地址空间的一段靠后的地址。
同时，`kernel stack`被映射了两次，在靠后的虚拟地址映射了一次，在`PHYSTOP`下的`Kernel data`中又映射了一次，但是实际使用的时候用的是上面的部分，因为有`Guard page`会更加安全。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKbZEkzbzbKYgRRedXU%2Fimage.png?alt=media&token=2167acef-e76c-4d0c-81b0-f5175475793f)

这是众多你可以通过`page table`实现的有意思的事情之一。你可以**向同一个物理地址映射两个虚拟地址**，你可以不**将一个虚拟地址映射到物理地址**。可以是一对一的映射，一对多映射，多对一映射。XV6至少在1-2个地方用到类似的技巧。这的`kernel stack`和`Guard page`就是`XV6`基于`page table`使用的有趣技巧的一个例子。

**第二件事情是权限**。

- 例如`Kernel text page`被标位`R-X`，意味着你可以读它，也可以在这个地址段执行指令，但是你不能向`Kernel text`写数据。**通过设置权限我们可以尽早的发现`Bug`从而避免`Bug`。**
- 对于`Kernel data`需要能被写入，所以它的标志位是`RW-`，但是你不能在这个地址段运行指令，所以它的X标志位未被设置。（注，所以，`kernel text`用来存代码，代码可以读，可以运行，但是不能篡改，`kernel data`用来存数据，数据可以读写，但是不能通过数据伪装代码在`kernel`中运行）

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKb_jHb6u2XMiYkH2i0%2F-MKeHhKXkn4VMBskZjDS%2Fimage.png?alt=media&token=64e16ad2-c25b-4535-bb67-cc340934c027)

> **学生提问**

**Q(Student):** 对于不同的进程会有不同的`kernel stack`吗？

- Frans：答案是的。每一个用户进程都有一个对应的`kernel stack`

**Q(Student):** 用户程序的虚拟内存会映射到未使用的物理地址空间吗？

- Frans教授：在kernel page table中，有一段Free Memory，它对应了物理内存中的一段地址。
​![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKb_jHb6u2XMiYkH2i0%2F-MKeJFYcE1NyVZ0QMc67%2Fimage.png?alt=media&token=2bdc7a95-2b87-4c3e-ab06-76120962ec67)
XV6使用这段free memory来存放用户进程的page table，text和data。如果我们运行了非常多的用户进程，某个时间点我们会耗尽这段内存，这个时候fork或者exec会返回错误。

**Q(Student):** 这就意味着，用户进程的虚拟地址空间会比内核的虚拟地址空间小的多，是吗？

- Frans教授：本质上来说，两边的虚拟地址空间大小是一样的。但是用户进程的虚拟地址空间使用率会更低。

**Q(Student):** 如果多个进程都将内存映射到了同一个物理位置，这里会优化合并到同一个地址吗？

- Frans教授：XV6不会做这样的事情，但是`page table`实验中有一部分就是做这个事情。真正的操作系统会做这样的工作。当你们完成了`page table`实验，你们就会对这些内容更加了解。

**Q(Student)**：每个进程都会有自己的3级树状`page table`，通过这个`page table`将虚拟地址翻译成物理地址。所以看起来当我们将内核虚拟地址翻译成物理地址时，我们并不需要`kernel`的`page table`，因为进程会使用自己的树状`page table`并完成地址翻译。

- Frans教授：当`kernel`创建了一个进程，针对这个进程的`page table`也会从`Free memory`中分配出来。内核会为用户进程的`page table`分配几个`page`，并填入`PTE`。在某个时间点，当内核运行了这个进程，内核会将进程的根`page table`的地址加载到`SATP`中。从那个时间点开始，处理器会使用内核为那个进程构建的虚拟地址空间。

**Q(Student)**：所以内核为进程放弃了一些自己的内存，但是进程的虚拟地址空间理论上与内核的虚拟地址空间一样大，虽然实际中肯定不会这么大。

- Frans教授：是的，下图是用户进程的虚拟地址空间分布，与内核地址空间一样，它也是从0到MAXVA。
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKlssQnZeSx7lgksqSn%2F-MKopGK-JjubGvX84-qy%2Fimage.png?alt=media&token=0084006f-eedf-44ac-b93e-a12c936e0cc0)
它有由内核设置好的，专属于进程的page table来完成地址翻译。

**Q(Student)**：但是我们不能将所有的MAXVA地址都使用吧？

- Frans教授：是的我们不能，这样我们会耗尽内存。大多数的进程使用的内存都远远小于虚拟地址空间。

## 4.6 kvminit 函数

接下来，让我们看一看代码，我认为很多东西都会因此变得更加清晰。

首先，我们来做一个的常规操作，启动我们的`XV6`，这里`QEMU`实现了主板，同时我们打开`gdb`。

```bash
Last login: Tue Dec  6 22:37:32 on ttys002
(base) iiixv@IIIXVdeMacBook-Air xv6-riscv % make cpus=1 qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501

xv6 kernel is booting

 ```

```bash
Last login: Wed Dec 21 20:58:44 on ttys002
(base) iiixv@IIIXVdeMacBook-Air xv6-riscv % riscv64-unknown-elf-gdb
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
(gdb) b kvminit
Breakpoint 1 at 0x8000123a: file kernel/vm.c, line 55.
(gdb) c
Continuing.

Thread 1 hit Breakpoint 1, kvminit () at kernel/vm.c:55
55	{
(gdb) 
 ```

上一次我们看了`boot`的流程，我们跟到了`main`函数。`main`函数中调用的一个函数是`kvminit（3.9）`，这个函数会设置好`kernel`的地址空间。`kvminit`的代码如下图所示：

```c
// Make a direct-map page table for the kernel.
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t) kalloc();
  memset(kpgtbl, 0, PGSIZE);

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // PLIC
  kvmmap(kpgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  // map kernel stacks
  proc_mapstacks(kpgtbl);
  
  return kpgtbl;
}

// Initialize the one kernel_pagetable
void
kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```

我们在前一部分看了`kernel`的地址空间长成什么样，这里我们来看一下代码是如何将它设置好的。首先在`kvminit`中设置一个断点，之后运行代码到断点位置。在`gdb`中执行`layout split`，可以看到（从上面的代码也可以看出）函数的第一步是为最高一级`page directory`分配物理`page`（注，调用`kalloc`就是分配物理`page`）。下一行将这段内存初始化为0。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKgiYv2CppKnuZEsKO3%2F-MKjPnUXQpkVeqUpxKel%2Fimage.png?alt=media&token=23dd9a99-c0c5-40d1-932e-b3b312da8dbf)

之后，通过kvmmap函数，将每一个I/O设备映射到内核。例如，下图中高亮的行将UART0映射到内核的地址空间。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKgiYv2CppKnuZEsKO3%2F-MKjRektDaeGzfk110hO%2Fimage.png?alt=media&token=95110bfc-81f8-49eb-bd06-c515c75a2e97)

我们可以查看一个文件叫做memlayout.h，它将4.5中的文档翻译成了一堆常量。在这个文件里面可以看到，UART0对应了地址0x10000000（注，4.5中的文档是真正SiFive RISC-V的文档，而下图是QEMU的地址，所以4.5中的文档地址与这里的不符）。

```c
// Physical memory layout

// qemu -machine virt is set up like this,
// based on qemu's hw/riscv/virt.c:
//
// 00001000 -- boot ROM, provided by qemu
// 02000000 -- CLINT
// 0C000000 -- PLIC
// 10000000 -- uart0 
// 10001000 -- virtio disk 
// 80000000 -- boot ROM jumps here in machine mode
//             -kernel loads the kernel here
// unused RAM after 80000000.

// the kernel uses physical memory thus:
// 80000000 -- entry.S, then kernel text and data
// end -- start of kernel page allocation area
// PHYSTOP -- end RAM used by the kernel

// qemu puts UART registers here in physical memory.
#define UART0 0x10000000L
#define UART0_IRQ 10
```

所以，通过`kvmmap`可以将物理地址映射到相同的虚拟地址（注，因为`kvmmap`的前两个参数一致）。
在`page table`实验中，第一个练习是实现`vmprint`，这个函数会打印当前的`kernel page table`。我们现在跳过这个函数，看一下执行完第一个`kvmmap`时的`kernel page table`。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKgiYv2CppKnuZEsKO3%2F-MKjW0riMeQYsQz76yZu%2Fimage.png?alt=media&token=5efebad5-ab9a-4c19-9073-80b9f0539332)

我们来看一下这里的输出:

- 第一行是最高一级`page directory`的地址，这就是存在`SATP`或者将会存在`SATP`中的地址。
- 第二行可以看到最高一级`page directory`只有一条PTE序号为`0`，它包含了中间级`page directory`的物理地址。
- 第三行可以看到中间级的`page directory`只有一条`PTE`序号为`128`，它指向了最低级`page directory`的物理地址。
- 第四行可以看到最低级的`page directory`包含了PTE指向物理地址。你们可以看到最低一级 `page directory`中`PTE`的物理地址就是`0x10000000`，对应了`UART0`。
前面是物理地址，我们可以从虚拟地址的角度来验证这里符合预期。我们将地址0x10000000向右移位12bit，这样可以得到虚拟地址的高27bit（index部分）。之后我们再对这部分右移位9bit，并打印成10进制数，可以得到128，这就是中间级page directory中PTE的序号。这与之前（4.4）介绍的内容是符合的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKoqX3juAGIEtw1zvSN%2F-MKwkdhVQHoP9SPCHfEW%2Fimage.png?alt=media&token=d336e447-71f3-4ff1-bd56-0dc900390d8e)

从标志位来看（`fl`部分），最低一级`page directory`中的`PTE`有读写标志位，并且`Valid`标志位也设置了（4.3底部有标志位的介绍）。
内核会持续的按照这种方式，调用`kvmmap`来设置地址空间。之后会对`VIRTIO0`、`CLINT`、`PLIC`、`kernel text`、`kernel data`、最后是`TRAMPOLINE`进行地址映射。最后我们还会调用`vmprint`打印完整的`kernel page directory`，可以看出已经设置了很多`PTE`。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKgiYv2CppKnuZEsKO3%2F-MKjdk9613l99xINOywP%2Fimage.png?alt=media&token=9370aef4-86ae-42b8-a687-649170b099db)

这里就不过细节了，但是这些`PTE`构成了我们在`4.5`中看到的地址空间对应关系。

> **学生提问**

**Q(student)**：下面这两行内存不会越界吗？

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKoqX3juAGIEtw1zvSN%2F-MKwKP7Z063SggHz4lMz%2Fimage.png?alt=media&token=bcc6e605-b9ba-4d4f-a35b-f164f4c524f9)

- **Frans**：不会。这里`KERNBASE`是`0x80000000`，这是内存开始的地址。`kvmmap`的第三个参数是`size`，`etext`是`kernel text`的最后一个地址，`etext - KERNBASE`会返回`kernel text`的字节数，我不确定这块有多大，大概是`60-90`个`page`，这部分是`kernel`的`text`部分。`PHYSTOP`是物理内存的最大位置，`PHYSTOP-text`是`kernel`的`data`部分。会有足够的`DRAM`来完成这里的映射。`etext`是内核最后一条指令的地址。

## 4.7 kvminithart 函数

`kvminit`函数返回了，在`main`函数中，我们运行到了`kvminithart`函数。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKgiYv2CppKnuZEsKO3%2F-MKjffmTgmjxQO-BCcin%2Fimage.png?alt=media&token=050d4673-2526-43c7-83aa-6b623e840074)

这个函数首先设置了`SATP`寄存器，`kernel_pagetable`变量来自于`kvminit`第一行。所以这里实际上是内核告诉MMU来使用刚刚设置好的`page table`。当这里这条指令执行之后，下一个指令的地址会发生什么？
在这条指令之前，还不存在可用的`page table`，所以也就不存在地址翻译。执行完这条指令之后，程序计数器（`Program Counter`）增加了`4`。而之后的下一条指令被执行时，程序计数器会被内存中的`page table`翻译。
所以这条指令的执行时刻是一个非常重要的时刻。因为整个地址翻译从这条指令之后开始生效，之后的每一个使用的内存地址都可能对应到与之不同的物理内存地址。因为在这条指令之前，我们使用的都是物理内存地址，这条指令之后`page table`开始生效，所有的内存地址都变成了另一个含义，也就是虚拟内存地址。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKlmAaMCuundOeRCH3F%2F-MKlq_20wJR-qcEYSVzE%2Fimage.png?alt=media&token=56a44d30-52dd-4244-99f5-5fa193ce32e5)

这里能正常工作的原因是值得注意的。因为前一条指令还是在物理内存中，而后一条指令已经在虚拟内存中了。比如，下一条指令地址是`0x80001110`就是一个虚拟内存地址。

为什么这里能正常工作呢？因为`kernel page`的映射关系中，虚拟地址到物理地址是完全相等的。所以，在我们打开虚拟地址翻译硬件之后，地址翻译硬件会将一个虚拟地址翻译到相同的物理地址。所以实际上，我们最终还是能通过内存地址执行到正确的指令，因为经过地址翻译`0x80001110`还是对应`0x80001110`。
管理虚拟内存的一个难点是，一旦执行了类似于`SATP`这样的指令，你相当于将一个`page table`加载到了SATP寄存器，你的世界完全改变了。现在每一个地址都会被你设置好的`page table`所翻译。那么假设你的`page table`设置错误了，会发生什么呢？有人想回答这个问题吗？

- 学生A回答：你可能会覆盖`kernel data`。
- 学生B回答：会产生`page fault`。

是的，因为`page table`没有设置好，虚拟地址可能根本就翻译不了，那么内核会停止运行并`panic`。

所以，如果`page table`中有`bug`，你将会看到奇怪的错误和崩溃，这导致了`page table`实验将会比较难。如果你不够小心，或者你没有完全理解一些细节，你可能会导致`kernel`崩溃，这将会花费一些时间和精力来追踪背后的原因。但这就是管理虚拟内存的一部分，因为对于一个这么强大的工具，如果出错了，相应的你也会得到严重的后果。我并不是要给你们泼凉水，哈哈。另一方面，这也很有乐趣，经过了`page table`实验，你们会真正理解虚拟内存是什么，虚拟内存能做什么。

## 4.8 walk 函数

> **学生提问**

**Q(Student)**：我对于`walk`函数有个问题，从代码看它返回了最高级`page table`的`PTE`，但是它是怎么工作的呢？（注，应该是学生理解有误，`walk`函数模拟了`MMU`，返回的是`va`对应的最低级`page table`的`PTE`）

```c
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

- **Frans教授**：这个函数会返回`page table`的PTE，而内核可以读写PTE。我来画个图，首先我们有一个`page directory`，这个`page directory` 有512个`PTE`。最下面是0，最上面是511。
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKlssQnZeSx7lgksqSn%2F-MKoc0cBkLRLrjnh7STD%2Fimage.png?alt=media&token=6eec7448-4215-4a80-9f01-37c45ef7ba46)
这个函数的作用是返回某一个`PTE`的指针。
![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKlssQnZeSx7lgksqSn%2F-MKocAls4NJRDhJ5zMRk%2Fimage.png?alt=media&token=c2b90fd1-36c8-4c97-8b97-377882385ec3)
这是个虚拟地址，它指向了这个`PTE`。之后内核可以通过向这个地址写数据来操纵这条`PTE`执行的物理`page`。当`page table`被加载到`SATP`寄存器，这里的更改就会生效。
从代码看，这个函数从`level2`走到`level1`然后到`level0`，如果参数`alloc`不为`0`，且某一个`level`的`page table`不存在，这个函数会创建一个临时的`page table`，将内容初始化为0，并继续运行。所以最后总是返回的是最低一级的`page directory`的`PTE`。
如果参数`alloc`没有设置，那么在第一个`PTE`对应的下一级`page table`不存在时就会返回。

**Q(student)**：对于`walk`函数，我有一个比较困惑的地方，在写完`SATP`寄存器之后，内核还能直接访问物理地址吗？在代码里面看起来像是通过`page table`将虚拟地址翻译成了物理地址，但是这个时候`SATP`已经被设置了，得到的物理地址不会被认为是虚拟地址吗？

- **Frans教授**：让我们来看`kvminithart`函数，这里的`kernel_page_table`是一个物理地址，并写入到`SATP`寄存器中。从那以后，我们的代码运行在一个我们构建出来的地址空间中。在之前的`kvminit`函数中，`kvmmap`会对每个地址或者每个`page`调用`walk`函数。所以你的问题是什么？

**Q(student)**：我想知道，在SATP寄存器设置完之后，walk是不是还是按照相同的方式工作？

- Frans：是的。它还能工作的原因是，内核设置了虚拟地址等于物理地址的映射关系，这里很重要，因为很多地方能工作的原因都是因为内核设置的地址映射关系是相同的。

**Q(student)**：每一个进程的`SATP`寄存器存在哪？

- **Frans**：每个CPU核只有一个`SATP`寄存器，但是在每个`proc`结构体，如果你查看`proc.h`，里面有一个指向`page table`的指针，这对应了进程的根`page table`物理内存地址。

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

**Q(student)**：为什么通过3级`page table`会比一个超大的`page table`更好呢？

- **Frans教授**：这是个好问题，这的原因是，3级`page table`中，大量的`PTE`都可以不存储。比如，对于最高级的`page table`里面，如果一个`PTE`为空，那么你就完全不用创建它对应的中间级和最底层`page table`，以及里面的`PTE`。所以，这就是像是在整个虚拟地址空间中的一大段地址完全不需要有映射一样。

**Q(student)**：所以3级`page table`就像是按需分配这些映射块。

- **Frans教授**：是的，就像前面（4.6）介绍的一样。最开始你只有3个`page table`，一个是最高级，一个是中间级，一个是最低级的。随着代码的运行，我们会创建更多的`page table diretory`。