# Paging Tables

页表是**操作系统**为每个**进程**提供**私有地址空间**和**内存**的机制。

- 页表决定了内存地址的含义，以及物理内存的哪些部分可以访问。
- 页表允许xv6**隔离不同进程的地址空间**，并将复用到单个物理内存上。
- 页表还提供了一层抽象（a level of indirection），这允许xv6执行一些特殊操作：映射相同的内存到不同的地址空间中（a trampoline page），并用一个未映射的页面保护内核和用户栈区。

## 3.1 Paging Hardware

> **页表硬件的作用**

**RISC-V 页表硬件的作用**：RISC-V 指令操作的是**虚拟地址**，而机器的RAM或物理内存是由**物理地址索引**。RISC-V 页表硬件通过**将每个虚拟地址映射到物理地址来为这两种地址建立联系**

> **单级页表**

XV6基于`Sv39 RISC-V`运行————既只使用64位虚拟地址的低39位，而**虚拟地址的高25位不用于转换**，将来RISC-V可能会使用那些位来定义更多级别的转换。。

在这种Sv39配置中，**`RISC-V`页表在逻辑构成**详解如下：

- 分页硬件通过使用虚拟地址`39`位中的**前`27`位索引页表**，即索引由 $2^{27}(134217728)$ 个页表条目（`Page Table Entries/PTE`）组成的页表，找到该虚拟地址对应的一个`PTE`。
- 每个`PTE`包含一个 **`44`位的物理页码**（`Physical Page Number/PPN`）和一些标志(`flags`)。（注：此处PTE格式中有空间让物理地址长度再增长10个比特位）
- 然后生成一个`56`位的**物理地址**，其前`44`位来自`PTE`中的`PPN`，其后12位来自**原始虚拟地址(offset)**。

图3.1（逻辑视图是一个简单的PTE数组）显示了上述过程。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p1.png)

页表使操作系统能够以 `4096` ( $2^{12}$ ) 字节的对齐块的粒度**控制虚拟地址到物理地址**的转换，这样的块称为**页（page）**。

$2^{39}$ 字节是 512 GB，这应该足够让应用程序运行在 RISC-V 计算机上。 $2^{56}$ 的物理内存空间在不久的将来足以容纳可能的 I/O 设备和 DRAM 芯片。 如果需要更多，RISC-V 设计人员定义了具有 `48` 位虚拟地址的 `Sv48`

> **多级页表**

如图3.2所示，实际的转换分三个步骤进行。**页表以三级的树型结构存储**在物理内存中。

- 该树的根是一个 **`4096`字节的页表页**，其中包含`512`个PTE，每个`PTE`中包含该树下一级页表页的物理地址。这些页中的每一个`PTE`都包含该树最后一级的`512`个PTE（也就是说每个PTE占8个字节，正如图3.2最下面所描绘的）。
- 分页硬件使用`27`位中的前`9`位在**根页表页面中选择PTE**，**中间`9`位**在树的**下一级页表**页面中选择`PTE`，**最后`9`位**选择最终的`PTE`。
- 如果转换地址所需的三个`PTE`中的任何一个不存在，页式硬件就会引发页面故障异常（page-fault exception），并让内核来处理该异常（参见第4章）。
- 每个`PTE`**包含标志位**，这些标志位告诉分页硬件允许如何使用关联的虚拟地址。
  - `PTE_V`指示PTE是否存在：如果它没有被设置，对页面的引用会导致异常（即不允许）。
  - `PTE_R`控制是否允许指令读取到页面。
  - `PTE_W`控制是否允许指令写入到页面。
  - `PTE_X`控制CPU是否可以将页面内容解释为指令并执行它们。
  - `PTE_U`控制用户模式下的指令是否被允许访问页面；如果没有设置PTE_U，PTE只能在管理模式下使用。
- 所有flags以及页表硬件相关的结构都定义在`kernel/riscv.h`中
  
图3.2显示了它是如何工作的。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p2.png)

> **多级页表的优缺点**

**优点**：图 3.2 的三级结构使用了一种**更节省内存**的方式来记录 `PTE`。在大范围的虚拟地址没有被映射的常见情况下，三级结构可以忽略整个页面目录。

举个例子，如果一个应用程序只使用了一个页面，那么顶级页面目录将只使用条目`0`，条目 `1` 到 `511` 都将被忽略，因此内核不必为这`511`个条目所对应的中间页面目录分配页面，也就更不必为这 511 个中间页目录分配底层页目录的页。 所以，在这个例子中，三级设计仅使用了三个页面，共占用 $3\times4096$个字节。

**缺点**：因为 CPU 在执行转换时会在硬件中遍历三级结构，所以缺点是 CPU 必须从内存中加载三个 PTE 以将虚拟地址转换为物理地址。为了减少从物理内存加载 PTE 的开销，RISC-V CPU 将页表条目缓存在 `Translation Look-aside Buffer (TLB)` 中。

> **stap寄存器**

**`satp`的作用**:存放根页表页在物理内存中的地址，为了告诉硬件使用页表，内核必须将根页表页的物理地址写入到`satp`寄存器中。

每个CPU都有自己的satp，一个CPU将使用自己的satp指向的页表转换后续指令生成的所有地址。每个CPU都有自己的satp，因此不同的CPU就可以运行不同的进程，每个进程都有自己的页表描述的私有地址空间。

通常，内核将所有物理内存映射到其页表中，以便它可以使用加载/存储指令读取和写入物理内存中的任何位置。 由于**页目录位于物理内存中**，内核可以通过**使用标准存储指令**写入 `PTE` 的虚拟地址来**对页目录中的 `PTE` 内容进行编程**。

关于术语的一些注意事项。

- 物理内存是指DRAM中的存储单元。物理内存以一个字节为单位划为地址，称为物理地址。
- 指令只使用虚拟地址，分页硬件将其转换为物理地址，然后将其发送到DRAM硬件来进行读写。
- 物理内存和虚拟地址不同，**虚拟内存**不是物理对象，而是指**内核提供的管理物理内存和虚拟地址的抽象和机制的集合。**

***

## 3.2 Kernel Address Space

- Xv6为每个进程维护一个页表，用以**描述每个进程的用户地址空间**
- 外加一个单独**描述内核地址空间的页表**。内核配置其地址空间的布局，以允许自己以可预测的虚拟地址访问物理内存和各种硬件资源。
- 内核页表中的内容为所有进程共享，每个进程都有自己的进程页表

图3.3显示了这种布局如何将**内核虚拟地址映射到物理地址**。文件(`kernel/memlayout.h`) 声明了xv6内核内存布局的常量。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p3.png)

QEMU模拟了一台计算机:

- 它包括从物理地址`0x80000000`开始并至少到`0x86400000`结束的RAM（物理内存）`xv6`称结束地址为`PHYSTOP`。
- `QEMU`模拟还包括`I/O`设备，如磁盘接口。QEMU将设备接口作为**内存映射控制寄存器暴露给软件**，这些寄存器位于物理地址空间0x80000000以下。内核可以通过读取/写入这些特殊的物理地址与设备交互；这种读取和写入与设备硬件而不是RAM通信。第4章解释了xv6如何与设备进行交互。
- 内核使用 **“直接映射”**获取内存和内存映射设备寄存器；也就是说，**将资源映射到等于物理地址的虚拟地址**。例如，内核本身在虚拟地址空间和物理内存中都位于`KERNBASE=0x80000000`。直接映射简化了读取或写入物理内存的内核代码。例如，当`fork`为子进程分配用户内存时，分配器返回该内存的物理地址；`fork`在将父进程的用户内存复制到子进程时直接将该地址用作虚拟地址。
- 有几个内核虚拟地址**不是直接映射**：
  - **蹦床页面**(`trampoline page`)。它映射在虚拟地址空间的顶部；用户页表具有相同的映射。第4章讨论了蹦床页面的作用，但我们在这里看到了一个有趣的页表用例；一个物理页面（持有蹦床代码）在内核的虚拟地址空间中映射了两次：一次在虚拟地址空间的顶部，一次直接映射。
  - **内核栈页面(`Kstack`)**。每个进程都有自己的内核栈，它将映射到偏高一些的地址，这样xv6在它之下就可以留下一个未映射的保护页(`guard page`)。保护页的PTE是无效的（也就是说PTE_V没有设置），所以如果内核溢出内核栈就会引发一个异常，内核触发panic。如果没有保护页，栈溢出将会覆盖其他内核内存，引发错误操作。恐慌崩溃（panic crash）是更可取的方案。（注：Guard page不会浪费物理内存，它只是占据了虚拟地址空间的一段靠后的地址，但并不映射到物理地址空间。）

虽然内核通过高地址内存映射使用内核栈，但是它们也可以通过直接映射的地址进入内核。

另一种设计可能只有直接映射，并在直接映射的地址使用栈。然而，在这种安排中，提供保护页将涉及取消映射虚拟地址，否则虚拟地址将引用物理内存，这将很难使用。

内核在权限`PTE_R`和`PTE_X`下**映射蹦床页面和内核文本页面**。内核从这些页面读取和执行指令。

内核在权限`PTE_R`和`PTE_W`下**映射其他页面**，这样它就可以读写那些页面中的内存。对于保护页面的映射是无效的。

***

## 3.3 Code: Creating an Address Space

> **概述：**

大多数用于操作地址空间和页表的xv6代码都写在`vm.c (kernel/vm.c:1)` 中。

- 其核心数据结构是`pagetable_t`，它实际上是指向RISC-V根页表页的指针；一个`pagetable_t`可以是内核页表，也可以是一个进程页表。
- 最核心的函数是`walk`（为虚拟地址找到PTE）和`mappages`（为新映射装载PTE）。
- 名称以`kvm`开头的函数操作内核页表；以`uvm`开头的函数操作用户页表；其他函数供二者调用。
- `copyout`和`copyin`复制数据到用户虚拟地址或从用户虚拟地址复制数据，这些虚拟地址作为系统调用参数提供; 由于它们需要显式地翻译这些地址，以便找到相应的物理内存，故将它们写在vm.c中。

> **执行流程：**

- **在启动序列的前期**，`main` 调用 `kvminit` (`kernel/vm.c:54`) 以使用 `kvmmake` (`kernel/vm.c:20`) **创建内核的页表**。此调用发生在 `xv6` 启用 `RISC-V` 上的分页之前，因此**地址直接引用物理内存**。

- `kvmmake` 首先**分配一个物理内存页来保存根页表页**。

- 然后`kvmmake`调用`kvmmap`来装载内核需要的转换。转换包括内核的指令和数据、物理内存的上限到 `PHYSTOP`，并包括实际上是设备的内存。`proc_mapstacks (kernel/proc.c:33)` 为每个进程分配一个内核堆栈。`proc_mapstacks`调用 `kvmmap` 将每个堆栈映射到由 `KSTACK` 生成的虚拟地址，从而为无效的堆栈保护页面留出空间。

- `kvmmap(kernel/vm.c:127)`调用`mappages(kernel/vm.c:138)`，`mappages`将范围虚拟地址到同等范围物理地址的映射装载到一个页表中。它以页面大小为间隔，为范围内的每个虚拟地址单独执行此操作。
对于要映射的每个虚拟地址，`mappages`调用`walk`来查找该地址的`PTE`地址。然后，初始化`PTE`用于保存相关的物理页号、所需权限（`PTE_W、PTE_X和/或PTE_R`）以及用于标记`PTE`有效的`PTE_V(kernel/vm.c:153)`。

- 在查找`PTE`中的虚拟地址（参见图3.2）时，`walk(kernel/vm.c:72)`模仿`RISC-V`分页硬件。`walk`一次从3级页表中获取9个比特位。它使用上一级的9位虚拟地址来查找下一级页表或最终页面的`PTE (kernel/vm.c:78)`。如果`PTE`无效，则所需的页面还没有分配；如果设置了`alloc`参数，`walk`就会分配一个新的页表页面，并将其物理地址放在`PTE`中。它返回树中最低一级的`PTE`地址(`kernel/vm.c:88`)。

  >上面的代码**依赖于直接映射到内核虚拟地址空间中的物理内存**。例如，当`walk`降低页表的级别时，它从`PTE (kernel/vm.c:80)`中提取下一级页表的（物理）地址，然后使用该地址作为虚拟地址来获取下一级的`PTE (kernel/vm.c:78)。`

- `main`调用`kvminithart (kernel/vm.c:53)`来安装内核页表。`kvminithart`将根页表页的物理地址写入寄存器`satp`。之后，`CPU`将使用内核页表转换地址。由于内核使用标识映射，下一条指令的当前虚拟地址将映射到正确的物理内存地址。

- `main`中调用的`procinit (kernel/proc.c:26)`为每个进程分配一个内核栈。它将每个栈映射到`KSTACK`生成的虚拟地址，这为无效的栈保护页面留下了空间。`kvmmap`将映射的`PTE`添加到内核页表中，对`kvminithart`的调用将内核页表重新加载到`satp`中，以便硬件知道新的`PTE`。

每个`RISC-V CPU`都将页表条目缓存在转译后备缓冲器（快表/TLB）中，当`xv6`更改页表时，它必须告诉`CPU`使相应的缓存`TLB`条目无效。如果没有这么做，那么在某个时候`TLB`可能会使用旧的缓存映射，指向一个在此期间已分配给另一个进程的物理页面，这样会导致一个进程可能能够在其他进程的内存上涂鸦。

`RISC-V`有一个指令`sfence.vma`，用于刷新当前`CPU`的`TLB`。`xv6`在重新加载`satp`寄存器后，在`kvminithart`中执行`sfence.vma`，并在返回用户空间之前在用于切换至一个用户页表的`trampoline`代码中执行`sfence.vma (kernel/trampoline.S:79)`。

***

## 3.4 Physical Memory Allocation

内核必须在运行时为页表、用户内存、内核栈和管道缓冲区分配和释放物理内存。

xv6使用内核末尾到`PHYSTOP`之间的物理内存进行**运行时分配**。

它一次分配和释放整个`4096`字节的页面。它**使用链表的数据结构将空闲页面记录下来**。分配时需要从链表中删除页面；释放时需要将释放的页面添加到链表中。

***

## 3.5 Code: Physical Meomory Allocator

分配器(`allocator`)位于`kalloc.c(kernel/kalloc.c:1)`中。

**分配器的数据结构：**

可供分配的物理内存页的空闲列表。每个空闲页的列表元素是一个`struct run(kernel/kalloc.c:17)`。

```c
struct run {
  struct run *next;
};
```

**分配器从哪里获得内存来填充该数据结构呢？**

它将每个空闲页的`run`结构存储在空闲页本身，因为在那里没有存储其他东西。空闲列表受到自旋锁（`spin lock`）的保护`(kernel/kalloc.c:21-24)`。**列表和锁被封装在一个结构体中**，以明确锁在结构体中保护的字段。现在，忽略锁以及对`acquire`和`release`的调用；第6章将详细查看有关锁的细节。

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```

> **Tips:**
>  
> 1. **对于互斥锁**，如果资源已经被占用，资源申请者只能进入睡眠状态。但是**自旋锁**不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。
> 2. **自旋锁**比较适用于**锁使用者保持锁时间比较短的情况**。正是由于自旋锁使用者一般保持锁时间非常短，因此选择自旋而不是睡眠是非常必要的，自旋锁的效率远高于互斥锁。

**函数`kinit(kernel/kalloc.c:27)`**

`main`函数调用`kinit(kernel/kalloc.c:27)`来**初始化分配器**。

`kinit`初始化空闲列表以保存从内核结束到`PHYSTOP`之间的每一页。`xv6`应该**通过解析硬件提供的配置信息来确定有多少物理内存可用**。

`xv6`假设机器有`128`兆字节的RAM。`kinit`调用`freerange`将内存添加到空闲列表中，在`freerange`中每页都会调用`kfree`。`PTE`只能引用在`4096`字节边界上对齐的物理地址（是4096的倍数），所以`freerange`使用`PGROUNDUP`来确保它只释放对齐的物理地址。分配器开始时没有内存；通过调用`kfree`给了它一些管理空间。

**分配器代码充满C类型转换的原因：**

- 主要原因：分配器**有时将地址视为整数**，以便对其执行算术运算（例如，在`freerange`中遍历所有页面），**有时将地址用作读写内存的指针**（例如，操纵存储在每个页面中的run结构）。这种地址的双重用途是分配器代码充满C类型转换的主要原因。
- 另一个原因是释放和分配从本质上改变了内存的类型。

**函数`kfree (kernel/kalloc.c:47)`:**

- 首先将内存中的每一个字节设置为1。这将导致使用释放后的内存的代码（使用“悬空引用”）读取到垃圾信息而不是旧的有效内容，从而希望这样的代码更快崩溃。
- 然后`kfree`将页面前置（头插法）到空闲列表中：它将`pa`转换为一个指向`struct run`的指针`r`，在`r->next`中记录空闲列表的旧开始，并将空闲列表设置为等于r。

**函数`kalloc(kernel/kalloc.c:69)`:**

`kalloc`删除并返回空闲列表中的第一个元素。

***

## 3.6 Process Address Space

**每个进程都有一个单独的页表，当xv6在进程之间切换时，也会更改页表。**

> **进程虚拟地址空间分布**

- 如图2.3所示，一个进程的用户内存从虚拟地址零开始，可以增长到MAXVA (`#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))`)，原则上允许一个进程内存寻址空间为`256G`。

- 首先，**不同进程的页表将用户地址转换为物理内存的不同页面**，这样每个进程都拥有私有内存。
- 第二，每个进程看到的自己的内存空间都是以0地址起始的连续虚拟地址，而进程的物理内存可以是非连续的。
- 第三，内核在用户地址空间的顶部映射一个带有蹦床（trampoline）代码的页面，这样在所有地址空间都可以看到一个单独的物理内存页面。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p5.png)

当进程向xv6**请求更多的用户内存**时:

- xv6首先使用`kalloc`来分配物理页面。
- 然后，它将`PTE`添加到进程的页表中，指向新的物理页面。`Xv6`在这些`PTE`中设置`PTE_W、PTE_X、PTE_R、PTE_U和PTE_V`标志。大多数进程不使用整个用户地址空间；`xv6`在未使用的`PTE`中留空`PTE_V`。

> **xv6中执行态度进程的用户内存布局**

- 栈是**单独一个页面**，显示的是由`exec`创建后的初始内容。包含**命令行参数的字符串**以及**指向它们的指针数组**位于栈的最顶部。再往下是允许程序在`main`处开始启动的值（即`main`的地址、`argc、argv`），这些值产生的效果就像刚刚调用了`main(argc, argv)`一样。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p6.png)

为了检测用户栈**是否溢出了所分配栈内存**，`xv6`在栈正下方放置了一个无效的保护页（`guard page`）。如果用户栈溢出并且进程试图使用栈下方的地址，那么由于映射无效（`PTE_V为0`）硬件将生成一个页面故障异常。当用户栈溢出时，实际的操作系统可能会自动为其分配更多内存。

***

## 3.7 Code:sbrk

`sbrk`是一个用于进程**减少或增长其内存**的系统调用。

这个系统调用由函数`growproc`实现(`kernel/proc.c:239`)。

```c
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
```

`growproc`根据n是正的还是负的调用`uvmalloc`（正）或`uvmdealloc`（负）。

- `uvmalloc(kernel/vm.c:229)`调用`kalloc`分配物理内存，并用`mappages`将`PTE`添加到用户页表中。
- `uvmdealloc`调用`uvmunmap(kernel/vm.c:174)`，`uvmunmap`使用`walk`来查找对应的`PTE`，并使用`kfree`来释放`PTE`引用的物理内存。

XV6使用进程的页表，**不仅是**告诉硬件如何映射用户虚拟地址，也是明晰哪一个物理页面已经被分配给该进程的唯一记录。这就是为什么释放用户内存（在`uvmunmap`中）需要检查用户页表的原因。

***

## 3.9 Code:exec

exec是**创建用户地址空间**的系统调用。它使用一个存储在文件系统中的**文件****初始化地址空间的用户部分**。

`exec(kernel/exec.c:13)`使用`namei (kernel/exec.c:26)`打开指定的二进制`path`，这在第8章中有解释。

```c
struct inode*
namei(char *path)
{
  char name[DIRSIZ];
  return namex(path, 0, name);
}
```

然后，读取`ELF`头。`Xv6`应用程序以广泛使用的`ELF`格式描述，定义于(`kernel/elf.h`)。ELF二进制文件由以下部分组成：

- ELF header
- struct elfhdr(`kernel/elf.h:6`)
- 后面一系列的程序节头（section headers）
- struct proghdr(`kernel/elf.h:25`)每个`proghdr`描述程序中必须加载到内存中的一节（`section`）；xv6程序只有一个程序节头，但是其他系统对于指令和数据部分可能各有单独的节。

> Note
> ELF文件格式：在计算机科学中，是一种用于二进制文件、可执行文件、目标代码、共享库和核心转储格式文件。ELF是UNIX系统实验室（USL）作为应用程序二进制接口（Application Binary Interface，ABI）而开发和发布的，也是Linux的主要可执行文件格式。ELF文件由4部分组成，分别是ELF头（ELF header）、程序头表（Program header table）、节（Section）和节头表（Section header table）。实际上，一个文件中不一定包含全部内容，而且它们的位置也未必如同所示这样安排，只有ELF头的位置是固定的，其余各部分的位置、大小等信息由ELF头中的各项值来决定。

**第一步**是快速检查文件可能包含`ELF`二进制的文件。`ELF`二进制文件以四个字节的“魔数”(`magic number`)`0x7F、“E”、“L”、“F”`或`ELF_MAGIC`开始(`kernel/elf.h:3`)。如果`ELF`头有正确的魔数，那么`exec`假设二进制文件格式良好。

`exec`使用`proc_pagetable (kernel/exec.c:38)`分配一个没有用户映射的新页表；使用`uvmalloc (kernel/exec.c:52)`为**每个`ELF`段分配内存**；并使用`loadseg (kernel/exec.c:10)`将每个段加载到内存中。`loadseg`使用`walkaddr`找到分配内存的物理地址，在该地址写入`ELF`段的每一页，并使用`readi`从文件中读取。

使用`exec`创建的第一个用户程序`/init`的程序节标题如下：

```c
 # objdump -p _init 
 user/_init: file format elf64-littleriscv 
 Program Header: 
     LOAD off 0x00000000000000b0 vaddr 0x0000000000000000 
                                    paddr 0x0000000000000000 align 2**3 
          filesz 0x0000000000000840 memsz 0x0000000000000858 flags rwx 
     STACK off 0x0000000000000000 vaddr 0x0000000000000000 
                                    paddr 0x0000000000000000 align 2**4 
          filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
```

程序节头的`filesz`可能小于`memsz`，这表明它们之间的间隙应该用零来填充（对于C全局变量），而不是从文件中读取。对于`/init，filesz`是`2112`字节，`memsz`是`2136`字节，因此`uvmalloc`分配了足够的物理内存来保存`2136`字节，但只从文件`/init`中读取`2112`字节。

现在`exec`**分配并初始化用户栈**。它只分配一个栈页面。`exec`一次将参数中的一个字符串复制到栈顶，并在`ustack`中记录指向它们的指针。它在传递给`main`的`argv`列表的末尾放置一个空指针。`ustack`中的前三个条目是**伪返回程序计数器**（`fake return program counter`）、`**argc**`和 **`argv`** 指针。

`exec`在**栈页面的正下方放置了一个不可访问的页面**，这样试图使用超过一个页面的程序就会出错。这个不可访问的页面还允许`exec`处理过大的参数；在这种情况下，被`exec`用来将参数复制到栈的函数`copyout(kernel/vm.c:355)` 将会注意到目标页面不可访问，并返回`-1`。

在准备新内存映像的过程中，如果`exec`**检测到像无效程序段这样的错误**，它会跳到标签`bad`，释放新映像，并返回`-1`。`exec`必须等待系统调用会成功后再释放旧映像：因为如果旧映像消失了，系统调用将无法返回`-1`。`exec`中唯一的错误情况发生在映像的创建过程中。一旦映像完成，`exec`就可以提交到新的页表`(kernel/exec.c:113)`并释放旧的页表`(kernel/exec.c:117)`。

`exec`将`ELF`文件中的字节加载到`ELF`文件指定地址的内存中。用户或进程可以将他们想要的任何地址放入`ELF`文件中。因此`exec`是有风险的，因为`ELF`文件中的地址可能会意外或故意的引用内核。对一个设计拙劣的内核来说，后果可能是一次崩溃，甚至是内核的隔离机制被恶意破坏（即安全漏洞）。xv6执行许多检查来避免这些风险。例如，`if(ph.vaddr + ph.memsz < ph.vaddr)`检查总和是否溢出64位整数，危险在于用户可能会构造一个`ELF`二进制文件，其中的`ph.vaddr`指向用户选择的地址，而`ph.memsz`足够大，使总和溢出到`0x1000`，这看起来像是一个有效的值。在`xv6`的旧版本中，用户地址空间也包含内核（但在用户模式下不可读写），用户可以选择一个与内核内存相对应的地址，从而将`ELF`二进制文件中的数据复制到内核中。在xv6的RISC-V版本中，这是不可能的，因为内核有自己独立的页表；`loadseg`加载到进程的页表中，而不是内核的页表中。

内核开发人员很容易省略关键的检查，而现实世界中的内核有很长一段丢失检查的历史，用户程序可以利用这些检查的缺失来获得内核特权。xv6可能没有完成验证提供给内核的用户级数据的全部工作，恶意用户程序可以利用这些数据来绕过xv6的隔离。

***

## 3.10 Real word

像大多数操作系统一样，xv6使用**分页硬件进行内存保护和映射**。大多数操作系统通过**结合分页和页面故障异常使用分页**，比xv6复杂得多，我们将在第4章讨论这一点。

内核通过使用虚拟地址和物理地址之间的直接映射，以及假设在地址`0x8000000`处有物理`RAM` (内核期望加载的位置) ，`Xv6`得到了简化。这在`QEMU`中很有效，但在实际硬件上却是个坏主意；实际硬件将`RAM`和设备置于不可预测的物理地址，因此（例如）在`xv6`期望能够存储内核的`0x8000000`地址处可能没有`RAM`。更严肃的内核设计利用**页表将任意硬件物理内存布局转换为可预测的内核虚拟地址布局**。

`RISC-V`支持物理地址级别的保护，但`xv6`没有使用这个特性。

在有大量内存的机器上，使用`RISC-V`对“超级页面”的支持可能很有意义。而当物理内存较小时，小页面更有用，这样可以以精细的粒度向磁盘分配和输出页面。例如，如果一个程序只使用`8KB`内存，给它一个`4MB`的物理内存超级页面是浪费。在有大量内存的机器上，较大的页面是有意义的，并且可以减少页表操作的开销。

`xv6`内核缺少一个类似`malloc`可以为小对象提供内存的分配器，这使得**内核无法使用需要动态分配的复杂数据结构**。

内存分配是一个长期的热门话题，基本问题是**有效使用有限的内存并为将来的未知请求做好准备**。今天，人们更关心速度而不是空间效率。此外，一个更复杂的内核可能会分配许多不同大小的小块，而不是（如xv6中）只有4096字节的块；一个真正的内核分配器需要处理小分配和大分配。

***

## 3.11 Exercises

1. 分析RISC-V的设备树以找到计算机拥有的物理内存量。
2. 编写一个用户程序，通过调用sbrk(1)为其地址空间增加一个字节。运行该程序并研究调用sbrk之前和调用sbrk之后该程序的页表。内核分配了多少空间？新内存的PTE包含什么？
3. 修改xv6来为内核使用超级页面。
4. 修改xv6，这样当用户程序解引用空指针时会收到一个异常。也就是说，修改xv6使得虚拟地址0不被用户程序映射。
5. 传统上，exec的Unix实现包括对shell脚本的特殊处理。如果要执行的文件以文本#!开头, 那么第一行将被视为解释此文件的程序来运行。例如，如果调用exec来运行myprog arg1，而myprog的第一行是#!/interp，那么exec将使用命令行/interp myprog arg1运行 /interp。在xv6中实现对该约定的支持。
6. 为内核实现地址空间随机化
