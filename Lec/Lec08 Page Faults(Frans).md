# Lec08 Page Faults(Frans)

## 8.1 Page Fault Basics


> **准备工作**

[阅读4.6节](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)

> **课程内容**


今天课程的内容对应了后面几个实验，包括如下内容：

- lazy allocation，下一个lab的内容
- copy-on-write fork
- demand paging
- memory mapped files

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMB4xWfIo1Jb_zgmcjc%2F-MMBCAgW3JRqkqu36wni%2Fimage.png?alt=media&token=27d19200-0834-46c3-a4b4-a78eba82c89f)

注：
- 在XV6中，上述功能都没实现。在XV6中采用了非常保守的处理方式，**一旦用户空间进程触发了page fault，会导致进程被杀掉**。
- 因此，本节课对于代码的讲解会比较少，相应的在设计层面会有更多的内容，毕竟我们也没有代码可以讲解（XV6中没有实现）。

> **回顾虚拟内存**

**虚拟内存的两个主要的优点：**

- 隔离性（**Isolation**）：虚拟内存使得操作系统可以为每个应用程序提供属于它们自己的地址空间。所以一个应用程序不可能有意或者无意的修改另一个应用程序的内存数据。虚拟内存同时也提供了用户空间和内核空间的隔离性，并且通过`page table lab`也可以理解虚拟内存的隔离性。
- level of indirection，提供了一层抽象。处理器和所有的指令都可以使用虚拟地址，而内核会定义从虚拟地址到物理地址的映射关系。这一层抽象是本节课讨论功能的基础。不过到目前为止，在XV6中内存地址的映射都比较无聊，实际上**在内核中基本上是直接映射**（注，也就是虚拟地址等于物理地址）。当然也有几个比较有意思的地方：
  - `trampoline page`，它使得内核可以将一个物理内存`page`映射到多个用户地址空间中。
  - `guard page`，它同时在内核空间和用户空间用来保护`Stack`。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMBIijuc4KoC6PEGguT%2F-MMDXGaRjS5MKB1G8rjM%2Fimage.png?alt=media&token=f70bb5cf-982a-4648-9da4-f89a8e9494b7)

> **Page Fault**

到目前为止，**内存地址映射相对来说比较静态**。不管是`user page table`还是`kernel page table`，都是在最开始的时候设置好，之后就不会再做任何变动。`page fault`可以让地址映射关系变得动态起来。**通过`page fault`，内核可以更新`page table`，这是一个非常强大的功能**。因为现在可以动态的更新虚拟地址这一层抽象，结合`page table`和`page fault`，内核将会有巨大的灵活性。

接下来会看到各种各样利用动态变更`page table`实现的有趣的功能。

**当发生page fault时，内核需要什么样的信息才能够响应page fault?**

- **出错的虚拟地址，或者触发`page fault`的源**。当出现`page fault`时，XV6内核会打印出错的虚拟地址，并且这个地址会被保存在`STVAL`寄存器中。所以，当一个用户应用程序触发了`page fault`，page fault会使用与上节相同的`trap`机制，将程序运行切换到内核，同时将出错的地址存放在`STVAL`寄存器中。
- **出错的原因**。我们需要对不同场景的`page fault`有不同的响应。不同的场景是指:因为`load`指令触发的`page fault`，因为`store`指令触发的`page fault`又或者是因为`jump`指令触发的`page fault`。所以实际上如果你查看RISC-V的文档，在`SCAUSE`寄存器的介绍（注，`Supervisor cause`寄存器，保存了`trap`机制中进入到`supervisor mode`的原因），有多个与`page fault`相关的原因。基本上来说，`page fault`和其他的异常使用与系统调用相同的`trap`机制（注，详见lec06）来从用户空间切换到内核空间。如果是因为`page fault`触发的`trap`机制并且进入到内核空间，`STVAL`寄存器和`SCAUSE`寄存器都会有相应的值。
  ![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMD_TK8Ar4GqWE6xfWV%2F-MMNmVfRDZSAOKze10lZ%2Fimage.png?alt=media&token=4bbfdfa6-1491-4ab8-8248-03bd0e36a8e9)
  
  - 13表示是因为load引起的page fault；15表示是因为store引起的page fault；12表示是因为指令执行引起的page fault。
  - 因此第二个信息存在SCAUSE寄存器中，其中总共有3个类型的原因与page fault相关，分别是读、写和指令。
  - ECALL进入到supervisor mode对应的是8，这是我们在上节课中应该看到的SCAUSE值。

- **触发`page fault`的指令的地址**。从上节课可以知道，作为`trap`处理代码的一部分，这个地址存放在`SEPC（Supervisor Exception Program Counter）`寄存器中，并同时会保存在`trapframe->epc`（注，详见lec06）中。以便在`page fault handler`中修复`page table`，并重新执行对应的指令。理想情况下，修复完`page table`之后，指令就可以无错误的运行了。


![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMNpt2e_uc39XgzmDVX%2F-MMNr1p3hOfGCXnPFEAx%2Fimage.png?alt=media&token=5a99af6a-d462-474e-9388-3bfbbe1caa26)


**从硬件和XV6的角度来说，当出现了page fault，现在有了3个极其有价值的信息，分别是：**

- 引起`page fault`的内存地址
- 引起`page fault`的原因类型
- 引起`page fault`时的程序计数器值，表明了`page fault`在用户空间发生的位置


接下来我们将查看不同虚拟内存功能的实现机制，来帮助我们理解如何利用`page fault handler`修复`page table`并做一些有趣的事情。

## 8.2 Lazy Page allocation

### 8.2.1 内存allocation（Sbrk）

`sbrk`是XV6提供的系统调用，它使得用户应用程序能扩大自己的`heap`。

- 当一个应用程序启动的时候，`sbrk`指向的是`heap`的最底端，同时也是`stack`的最顶端。这个位置通过代表进程的数据结构中的`sz`字段表示，这里以`p->sz`表示。
- 当调用sbrk时，它的参数是整数，代表了想要申请的`page`数量（注，原视频说的是page，但是根据Linux man page，实际中sbrk的参数是字节数）。**sbrk会扩展heap的上边界**（也就是会扩大heap）。类似的，应用程序还可以通过给`sbrk`**传入负数作为参数**，来减少或者压缩它的地址空间。不过在这节课我们只关注增加内存的场景。
- 当`sbrk`实际发生或者被调用的时候，内核会分配一些物理内存，并将这些内存映射到用户应用程序的地址空间，然后将内存内容初始化为0，再返回`sbrk`系统调用。这样，应用程序可以通过多次`sbrk`系统调用来增加它所需要的内存。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMSxponnGmjT-9o9zTI%2F-MMSzCvRiaVMkJF9feJx%2Fimage.png?alt=media&token=493295ce-507e-44a9-a4d4-d400b9f14aee)


在XV6中，**`sbrk`的实现默认是`eager allocation`**。这表示了，一旦调用了`sbrk`，内核会立即分配应用程序所需要的物理内存。但是实际上，对于应用程序来说很难预测自己需要多少内存，所以通常来说，应用程序倾向于申请多于自己所需要的内存。这意味着，进程的内存消耗会增加许多，但是有部分内存永远也不会被应用程序所使用到。

### 8.2.2 Lazy allocation

原则上来说，`eager allocation`不是一个大问题。但是使用虚拟内存和`page fault handler`，完全可以用某种更聪明的方法来解决这里的问题，**即`lazy allocation`**。

> **核心思想**

`sbrk`系统调基本上不做任何事情，唯一需要做的事情就是提升`p->sz`，将`p->sz`增加`n`，其中`n`是需要新分配的内存`page`数量（字节数）。**但是内核在这个时间点并不会分配任何物理内存**。

之后在某个时间点，应用程序使用到了新申请的那部分内存，这时会触发`page fault`，因为我们还没有将新的内存映射到`page table`。所以，如果我们解析一个大于旧的`p->sz`，但是又小于新的`p->sz`（注，也就是旧的`p->sz + n`）的虚拟地址，我们希望内核能够分配一个内存`page`，并且重新执行指令。

> **对由`Lazy allocation`引起的`Page Fault`的响应**

所以，当看到了一个`page fault`，相应的虚拟地址小于当前`p->sz`，同时大于`stack`，那么我们就知道这是一个来自于`heap`的地址，但是内核还没有分配任何物理内存。

**所以对于这个`page fault`的响应也理所当然的直接明了**：
- 在`page fault handler`中，通过`kalloc`函数分配一个内存`page`；
- 初始化这个`page`内容为`0`；
- 将这个内存`page`映射到`user page table`中；
- 最后重新执行指令。

比方说，如果是`load`指令，或者`store`指令要访问属于当前进程但是还未被分配的内存，在我们映射完新申请的物理内存`page`之后，重新执行指令应该就能通过了。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMY1H3f9LVnmwGw1VOM%2F-MMY6HDdvPGIZ1hT4xtz%2Fimage.png?alt=media&token=d0ad00dd-0c2c-44d9-966e-085aeca0c970)

> **学生提问**

- 学生提问：在`eager allocation`的场景，一个进程可能消耗了太多的内存进而耗尽了物理内存资源。如果我们不使用`eager allocation`，而是使用`lazy allocation`，应用程序怎么才能知道当前已经没有物理内存可用了？
  - Frans教授：这是个非常好的问题。从应用程序的角度来看，会有一个错觉：存在无限多可用的物理内存。但是在某个时间点，应用程序可能会用光了物理内存，之后如果应用程序再访问一个未被分配的`page`，但这时又没有物理内存，这时内核可以有两个选择，我稍后会介绍更复杂的那个。你们在`lazy lab`中要做的是，返回一个错误并杀掉进程。因为现在已经`OOM（Out Of Memory）`了，内核也无能为力，所以在这个时间点可以杀掉进程。在这节课稍后的部分会介绍，可以有更加聪明的解决方案。
- 学生提问：如何判断一个地址是新分配的内存还是一个无效的地址？
  - Frans教授：在地址空间中，我们有`stack`，`data`和`text`。通常来说我们将`p->sz`设置成一个更大的数，新分配的内存位于旧的`p->sz`和新的`p->sz`之间，但是这部分内存还没有实际在物理内存上进行分配。如果使用的地址低于`p->sz`，那么这是一个用户空间的有效地址。如果大于`p->sz`，对应的就是一个程序错误，这意味着用户应用程序在尝试解析一个自己不拥有的内存地址。希望这回答了你的问题。
- 学生提问：为什么我们需要杀掉进程？操作系统不能只是返回一个错误说现在已经`OOM`了，尝试做一些别的操作吧。
  - Frans教授：让我们稍后再回答这个问题。在`XV6`的`page fault`中，我们默认会直接杀掉进程，但是这里的处理可以更加聪明。实际的操作系统的处理都会更加聪明，尽管如此，如果最终还是找不到可用内存，实际的操作系统还是可能会杀掉进程。

### 8.2.3 `Lazy allocation`相关的代码

#### 8.2.3.1 page fault

> **修改`sys_sbrk`函数**

**`sys_sbrk`功能**：完成实际增加应用程序的地址空间，分配内存等等

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0,&n) < 0)
    return -1;
  addr = myproc() -> sz;
  if(growproc(n) < 0)
    return -1;
  return addr;
}
```

**修改内容：** 修改这个函数，让它只对`p->sz`加`n`，并不执行增加内存的操作。

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0,&n) < 0)
    return -1;
  addr = myproc() -> sz;
  myproc() -> sz = myproc() -> sz + n;
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

> **启动XV6**


修改完之后启动XV6，并且执行`“echo hi”`，会得到一个`page fault`。**出现`page fault`的原因**：在`Shell`中执行程序，`Shell`会先`fork`一个子进程，子进程会通过`exec`执行`echo`（注，详见1.9）。在这个过程中，`Shell`会申请一些内存，**所以`Shell`会调用`sys_sbrk`**，然后就出错了（注，因为前面修改了代码，调用sys_sbrk不会实际分配所需要的内存）。

**输出错误信息：**

- 输出了`SCAUSE`寄存器内容，我们可以看到它的值是`15`，表明这是一个`store page fault`（详见8.1）。
- 进程的`pid`是`3`，这极可能是Shell的pid。
- 还可以看到`SEPC`寄存器的值，是`0x12a4`。
- 最后还可以看到出错的虚拟内存地址，也就是`STVAL`寄存器的内容，是`0x4008`

**查看`Shell`的 汇编代码**

- 搜索`SEPC`对应的地址，可以看到这的确是一个`store`指令。看起来就是出现`page fault`的位置。
- 如果向前看汇编代码，可**以看到`page fault`是出现在`malloc`的实现代码中**。在`malloc`的实现中，使用`sbrk`系统调用来获得一些内存，之后会初始化刚刚获取到的内存，在`0x12a4`位置，刚刚获取的内存中写入数据，**但是实际上在向未被分配的内存写入数据。**

**另一个证明内存还未被分配的点：** `XV6`中`Shell`通常是有`4个page`，包含了`text`和`data`。**出错的地址在`4个page`之外，也就是第`5个page`**，(在4个page之外8个字节)。这也合理，因为在`0x12a4`对应的指令中，`a0`持有的是`0x4000`，而`8`相对`a0`的偏移量。**偏移之后的地址就是我们想要使用的地址**（出错的地址）。

#### 8.2.3.2 如何处理 page fault

> **查看`trap.c`中的`usertrap`函数**

在`usertrap`中根据不同的`SCAUSE`完成不同的操作。

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  // in lec06 ，是因为`SCAUSE == 8`进入的`trap`，这是我们处理普通系统调用的代码。
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){    // 如果`SCAUSE != 8`，检查是否有任何的设备中断;如果有的话处理相关的设备中断。
    // ok
  } else {     //如果两个条件都不满足，**会打印一些信息，并且杀掉进程**。
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

现在需要增加一个检查，判断`SCAUSE == 15`，如果符合条件，需要一些定制化的处理。这里的定制化处理是什么样呢？

- 学生回答：想要检查`p->sz`是否大于当前存在`STVAL`寄存器中的虚拟地址。如果大于的话，就实际分配物理内存。
- 这是一种处理方式

这里会以**演示为目的简单的处理一下**，在`lazy lab`中需要完成更多的工作。

```c

//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scaues() == 15){
    uint64 va = r_stval();
    printf("page falut %p \n", va);
    uint64 ka = (unint64) kalloc();
    if(ka == 0){
      p -> killed = 1;
    } else {
      memset((void *) ka, 0, PGSIZE);
      va = PGROUNDDOWN(va);
      if (mappages(p -> pagetable, va, PGSIZE, ka, PTE_W|PTE_U|PTE_R) != 0){
        kfree((void *) ka);
        p -> killed = 1;
      }
    }
  }
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

上述代码中：
- 首先打印一些调试信息。
- 然后分配一个物理内存`page`，如果`ka`等于`0`，表明没有物理内存, **现在`OOM`了，我们会杀掉进程**。
- 如果有物理内存，**首先会将内存内容设置为0**，**之后将物理内存`page`指向用户地址空间中合适的虚拟内存地址**。具体来说:
  - 首先**将虚拟地址向下取整**，这里引起`page fault`的虚拟地址是`0x4008`，向下取整之后`是0x4000`。
  - 之后**将物理内存地址跟取整之后的虚拟内存地址的关系添加到`page table`中**。对应的`PTE`需要设置常用的权限标志位，在这里是`u，w，r` bit位。

> **重新编译`xv6`并启动**

先重新编译`XV6`，再执行`“echo hi”`，或许可以乐观的认为现在可以正常工作了。

```sh

```

这里有两个`page fault`:
- 第一个对应的虚拟内存地址是`0x4008`
- 但是很明显在处理上述`page fault`时，又产生了另一个`page fault 0x13f48`。现在唯一的问题是，`uvmunmap`在报错，一些它尝试`unmap`的`page`并不存在。这里`unmap`的内存是什么？
  - 学生回答：之前`lazy allocation`但是又没有实际分配的内存。
  - 完全正确。这里`unmap`的是之前`lazy allocated`，但是又还没有用到的地址。所以对于这个内存，并没有对应的物理内存。所以在`uvmunmap`函数中，当`PTE`的`v`标志位为`0`并且没有对应的`mapping`，这并不是一个实际的`panic`，这是我们预期的行为。

```c
void 
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free){

}
```

实际上，对于这个`page`我们并不用做任何事情，我们可以直接`continue`跳到下一个`page`。

```c
void 
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free){

}
```

**再重新编译`XV6`**，并执行`“echo hi”`。

```c

```

可以看到2个`page fault`，但是`echo hi`正常工作了。现在，我们一定程度上有了最基本最简单的`lazy allocation`。这里有什么问题吗？

> **学生提问**

- 学生提问：我并不能理解为什么在`uvmunmap`中可以直接改成`continue`？
  - Frans教授：之前的`panic`表明，我们尝试在释放一个并没有`map`的`page`。怎么会发生这种情况呢？唯一的原因是`sbrk`增加了`p->sz`，但是应用程序还没有使用那部分内存。因为对应的物理内存还没有分配，所以这部分新增加的内存的确没有映射关系。我们现在是`lazy allocation`，我们只会为需要的内存分配物理内存`page`。如果我们不需要这部分内存，那么就不会存在`map`关系，这非常的合理。相应的，我们对于这部分内存也不能释放，因为没有实际的物理内存可以释放，所以这里最好的处理方式就是`continue`，跳过并处理下一个`page`。
- 学生提问：在`uvmunmap`中，我认为之前的`panic`存在是有理由的，我们是不是应该判断一下，然后对于特定的场景还是`panic`？
  - Frans教授：为什么之前的panic会存在？对于未修改的XV6，永远也不会出现用户内存未map的情况，所以一旦出现这种情况需要panic。但是现在我们更改了XV6，所以我们需要去掉这里的panic，因为之前的不可能变成了可能。

这部分内容对于下一个实验有很大的帮助，实际上这是下一个实验3个部分中的一个，但是很明显这部分不足以完成下一个`lazy lab`。我们这里做了一些修改，但是很多地方还是有可能出错。就像有人提到的，我这里并没有检查触发`page fault`的虚拟地址是否小于`p->sz`。**还有其他的可能出错的地方吗？**
- 学生回答：通过`sbrk`增加的用户进程的内存数是一个整型数而不是一个无符号整型数，可能会传入负数。
- 是的，可能会使用负数，这意味着缩小用户内存。当我们在缩小用户内存时，我们也需要小心一些。实际上，在一个操作系统中，我们可能会在各种各样的用户场景中使用这里的PTE，对于不同的用户场景我们或许需要稍微修改XV6，这就是接下来的lazy lab的内容。你们需要完成足够多的修改，才能通过所有的测试用例。

## 8.3 Zero Fill On Demand

> **Zero Fill on Demand**

当查看一个用户程序的地址空间时，存在三个区域，并且当编译器在生成二进制文件时，编译器会填入这三个区域。
- `text`区域：`text`区域是程序的指令
- `data`区域：`data`区域存放的是初始化了的全局变量
- `BSS`区域：`BSS`区域包含了未被初始化或者初始化为0的全局或者静态变量），之所以这些变量要单独列出来，是因为例如你在C语言中定义了一个大的矩阵作为全局变量，它的元素初始值都是0，为什么要为这个矩阵分配内存呢？其实只需要记住这个矩阵的内容是0就行。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjMqer7xmB42tDWRuE%2Fimage.png?alt=media&token=69bc2287-687f-4acd-a1e4-aa4fe901754f)

> **针对`BBS`的调优**

在一个正常的操作系统中，如果执行`exec`，`exec`会申请地址空间，里面会存放`text`和`data`。因为`BSS`里面保存了未被初始化的全局变量，这里或许有许多许多个`page`，但是所有的`page`内容都为`0`。
**通常可以调优的地方是:**，对如此多的内容全是`0`的`page`，**在物理内存中，只需要分配一个`page`，这个`page`的内容全是`0`**。然后将**所有虚拟地址空间的全`0`的`page`都`map`到这一个物理`page`上**。这样至少在程序启动的时候能节省大量的物理内存分配。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjOUZgcx_hv79eQ_1r%2Fimage.png?alt=media&token=9411955a-324a-4db8-b4ea-be97bcd411df)

当然这里的`mapping`需要非常的小心，**不能允许对于这个`page`执行写操作**，因为所有的虚拟地址空间`page`都期望`page`的内容是全`0`，**所以这里的`PTE`都是只读的**。

之后在某个时间点，**应用程序尝试写`BSS`中的一个`page`时**，比如说需要更改一两个变量的值，我们会得到`page fault`。那么，对于这个特定场景中的`page fault`我们该做什么呢？
- 学生回答：应该创建一个新的`page`，将其内容设置为`0`，并重新执行指令。

是的，完全正确。假设`store`指令发生在`BSS`最顶端的`page`中。我们想要做的是，在物理内存中申请一个新的内存`page`，将其内容设置为`0`，因为这个内存的预期内容为0。之后需要更新这个`page`的`mapping`关系，**首先`PTE`要设置成可读可写，然后将其指向新的物理`page`。这里相当于更新了`PTE`，之后我们可以重新执行指令。**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjR5Fl20ATXIG70FFk%2Fimage.png?alt=media&token=6d3c6a13-7fa4-4ce6-b569-ceec31fa9694)

**调优的好处**

- 这样节省一部分内存。可以在需要的时候才申请内存。这里类似于`lazy allocation`。假设程序申请了一个大的数组，来保存可能的最大的输入，并且这个数组是全局变量且初始为`0`。但是最后或许只有一小部分内容会被使用。
- 在`exec`中需要做的工作变少了。程序可以启动的更快，可以获得更好的交互体验，因为只需要分配一个内容全是`0`的物理`page`。所有的虚拟`page`都可以映射到这一个物理`page`上。

**引发的`page fault`的代价**

- 学生提问：但是因为每次都会触发一个`page fault`，`update`和`write`会变得更慢吧？
- Frans教授：是的，这是个很好的观点，所以这里是实际上我们将一些操作推迟到了`page fault`再去执行。并且我们期望并不是所有的`page`都被使用了。如果一个`page`是`4096`字节，我们只需要对每`4096`个字节消耗一次`page fault`即可。但是这里是个好的观点，我们的确增加了一些由`page fault`带来的代价。
  
- `page fault`的代价是多少呢？我们该如何看待它？这是一个与`store`指令相当的代价，还是说代价要高的多？
- 学生回答：代价要高的多。store指令可能需要消耗一些时间来访问RAM，但是page fault需要走到内核。

是的，在lec06中你们已经看到了，仅仅是在trap处理代码中，就有至少有100个store指令用来存储当前的寄存器。除此之外，还有从用户空间转到内核空间的额外开销。所以，page fault并不是没有代价的，之前问的那个问题是一个非常好的问题。

## 8.4 Copy On Write Fault

## 8.5 Demand Paging 

## 8.6 Memory Mapped Files


