# Lec06 Isolation & system call entry & exit

## 准备工作

[阅读第4章](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)
[阅读RISCV.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h)
[阅读trampoline.S](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S)
[阅读trap.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c)

## 6.1 Trap 机制

### 6.1.1 Trap机制

程序运行是完成用户空间和内核空间的切换，出现下述情况，都会发生这样的切换：

- 程序执行系统调用
- 程序出现了类似page fault、运算时除以0的错误
- 一个设备触发了中断使得当前程序运行需要响应内核设备驱动

**trap机制**：用户空间和内核空间的切换通常被称为trap，而trap涉及了许多设计和重要的细节，这些细节对于实现安全隔离和性能来说非常重要。trap机制要尽可能的简单，这一点非常重要。

### 6.1.2 实例：shell执行系统调用

> **Shell 执行系统调用**

`Shell`可能会执行系统调用，将程序运行切换到内核。比如`XV6`启动之后`Shell`输出的一些提示信息，就是通过执行`write`系统调用来输出的。

> **执行系统调用涉及的硬件**

我们需要清楚如何让程序的运行，从只拥有`user`权限并且位于用户空间的Shell，切换到拥有`supervisor`权限的内核。

在这个过程中，硬件的状态将会非常重要，因为该过程中需要将**硬件从适合运行用户应用程序的状态，改变到适合运行内核代码的状态**。

最关心的状态可能是**32个用户寄存器**，这在上节课中有介绍。`RISC-V`总共有32个比如`a0，a1`这样的寄存器，**用户应用程序可以使用全部的寄存器**，并且使用寄存器的指令性能是最好的。这里的很多寄存器都有特殊的作用。

- `stack pointer`（也叫做堆栈寄存器 `stack register`）
- 在硬件中还有一个寄存器叫做程序计数器（`Program Counter Register`）
- 表明当前`mode`的标志位，这个标志位表明了当前是`supervisor mode`还是`user mode`。当运行`Shell`的时候，自然是在`user mode`
- 还有**控制CPU工作方式的寄存器**，如`SATP`（`Supervisor Address Translation and Protection`）寄存器，它包含了指向`page table`的物理内存地址（详见4.3）
- 还有一些对于今天非常重要的寄存器，如`STVEC`（`Supervisor Trap Vector Base Address Register`）寄存器，它指向了内核中处理`trap`的指令的起始地址
- `SEPC`（`Supervisor Exception Program Counter`）寄存器，在`trap`的过程中保存程序计数器的值
- `SSRATCH`（`Supervisor Scratch Register`）寄存器，这也是个非常重要的寄存器（详见6.5）

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK04CleJKuURh9XXWGG%2F-MK0Jg73bN-fwYa4_qZ_%2Fimage.png?alt=media&token=0c0df379-44cc-4f81-926e-8855fdcbe9c3)

> **Trap开始时，执行的硬件操作**

在`trap`的最开始，CPU的所有状态都设置成运行用户代码而不是内核代码。

在`trap`处理的过程中，实际上需要更改一些这里的状态，或者对状态做一些操作。这样才可以运行系统内核中普通的C程序，接下来先来预览一下需要做的操作：

- 首先，需要保存32个用户寄存器。因为很显然`Trap`结束后需要恢复用户应用程序的执行，尤其是当用户程序被设备中断所打断时，希望内核能够响应中断，之后在用户程序完全无感知的情况下再恢复用户代码的执行。所以这意味着32个用户寄存器不能被内核弄乱。但是这些寄存器又要被内核代码所使用，所以在trap之前，必须先在某处保存这32个用户寄存器
- 程序计数器也需要在某个地方保存，它几乎跟一个用户寄存器的地位是一样的，我们需要能够在用户程序运行中断的位置继续执行用户程序
- 需要将`mode`改成`supervisor mode`，因为想要使用内核中的各种各样的特权指令
- `SATP`寄存器现在正指向`user page table`，而`user page table`只包含了用户程序所需要的内存映射和一两个其他的映射，它并没有包含整个内核数据的内存映射。所以在运行内核代码之前，我们需要将`SATP`指向`kernel page table`。
- 需要将**堆栈寄存器**指向位于内核的一个地址，因为我们需要一个堆栈来调用内核的C函数。

通过上述设置之后，所有的硬件状态都适合在内核中使用，之后将跳入内核的C代码。

> **用户空间切换到内核空间的目标**

操作系统的一些high-level的目标能帮我们过滤一些实现选项。

- 其中一个目标是**安全和隔离**，我们不想让用户代码介入到这里的`user/kernel`切换，否则有可能会破坏安全性。这意味着，**trap中涉及到的硬件和内核机制不能依赖任何来自用户空间东西**。比如不能依赖32个用户寄存器，它们可能保存的是恶意的数据，所以，`XV6`的`trap`机制不会查看这些寄存器，而只是将它们保存起来。
- 在操作系统的trap机制中，我们仍然想**保留隔离性并防御来自用户代码的可能的恶意攻击**。另一方面，想要让`trap`机制对用户代码是透明的，也就是说想要执行`trap`，然后在内核中执行代码，同时用户代码并不用察觉到任何事情。这样也更容易写用户代码。

- 需要注意的是，虽然我们这里关心隔离和安全，但是今天我们只会讨论从用户空间切换到内核空间相关的安全问题。当然，系统调用的具体实现，比如说write在内核的具体实现，以及内核中任何的代码，也必须小心并安全的写好。所以，即使从用户空间到内核空间的切换十分安全，整个内核的其他部分也必须非常安全，并时刻小心用户代码可能会尝试欺骗它。

> **切换到supervisor mode后的权限**

在前面介绍的寄存器中，有一个特殊的寄存器需要讨论一下，也就是`mode`标志位。这里的`mode`表明当前是`user mode`还是`supervisor mode`。

- 当我们在用户空间时，这个标志位对应的是`user mode`，当我们在内核空间时，这个标志位对应`supervisor mode`。
- 当这个标志位从`user mode`变更到`supervisor mode`时，我们能得到什么样的权限。实际上，这里获得的额外权限实在是有限。也就是说，你可以在`supervisor mode`完成，但是不能在`user mode`完成的工作，或许并没有你想象的那么有特权。

**切换到supervisor mode后的权限：**

- 1. 读写控制寄存器，在`supervisor mode`可以读写如下寄存器，而`user mode`不能做这样的操作：
  - 读写`SATP`寄存器，也就是`page table`的指针；
  - `STVEC`，也就是处理`trap`的内核指令地址；
  - `SEPC`，保存当发生`trap`时的程序计数器；如`SSCRATCH`。

- 2. `supervisor mode`可以使用`PTE_U`标志位为`0`的`PTE`。当`PTE_U`标志位为`1`的时候，表明用户代码可以使用这个页表；如果这个标志位为`0`，则只有`supervisor mode`可以使用这个页表。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKAO-jBu6EvLnPok12b%2F-MKAZ_LB0s1zfIYYJx6E%2Fimage.png?alt=media&token=1976aa45-18bb-48f8-abbe-1e9df2b77b3b)

- 3. **需要特别指出的是：** `supervisor mode`中的代码并不能读写任意物理地址。在`supervisor mode`中，就像普通的用户代码一样，也需要通过`page table`来访问内存。如果一个虚拟地址并不在当前由`SATP`指向的`page table`中，又或者`SATP`指向的`page table`中`PTE_U=1`，那么`supervisor mode`不能使用那个地址。所以，即使我们在`supervisor mode`，我们还是受限于当前`page table`设置的虚拟地址。

## 6.2 Trap代码执行流程

从Shell的角度来说，Trap代码执行流程就是Shell代码中的C函数调用，但是实际上，`write`通过执行`ECALL`指令来执行系统调用。`ECALL`指令会切换到具有`supervisor mode`的内核中。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKFsfImgYCtnwA1d2hO%2F-MKHxleUqYy-y0mrS48w%2Fimage.png?alt=media&token=ab7c66bc-cf61-4af4-90fd-1fefc96c7b5f)

- 在这个过程中，**内核中执行的第一个指令是一个由汇编语言写的函数`uservec`**，该函数是内核代码`trampoline.s`文件的一部分。所以执行的第一个代码就是这个`uservec`汇编函数。
- 在汇编函数`uservec`中，代码执行跳转到了由C语言实现的函数`usertrap`中，这个函数在`trap.c`中
- 在`usertrap`这个C函数中，执行了一个叫做`syscall`的函数。
- 函数`syscall`会在一个表单中根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。在本例中这个函数就是`sys_write`
- `sys_write`会将要显示数据输出到`console`上，当它完成了之后，它会返回给`syscall`函数
- 因为现在相当于在`ECALL`之后中断了用户代码的执行，**为了用户空间的代码恢复执行**，需要做一系列的事情。在`syscall`函数中，会调用一个函数叫做`usertrapret`，它也位于`trap.c`中，这个函数完成了部分方便在C代码中实现的返回到用户空间的工作。
- 除此之外，最终还有一些工作只能在汇编语言中完成。这部分工作通过汇编语言实现，并且存在于`trampoline.s`文件中的`userret`函数中。
- 最终，在这个汇编函数中会调用机器指令返回到用户空间，并且恢复`ECALL`之后的用户程序的执行。

> **问答**

- 学生提问：vm.c运行在什么mode下？
  - Robert教授：vm.c中的所有函数都是内核的一部分，所以运行在supervisor mode。
- 学生提问：为什么这些函数叫这些名字？
  - Robert教授：现在的函数命名比较乱，明年我会让它们变得更加合理一些。（助教说）我认为命名与寄存器的名字有关。
- 学生提问：难道vm.c里的函数不是要直接访问物理内存吗？
  - Robert教授：是的，这些函数能这么做的原因是，内核小心的在page table中设置好了各个PTE。这样当内核收到了一个读写虚拟内存地址的请求，会通过kernel page table将这个虚拟内存地址翻译成与之等价物理内存地址，再完成读写。所以，一旦使用了kernel page table，就可以非常方便的在内核中使用所有这些直接的映射关系。但是直到trap机制切换到内核之前，这些映射关系都不可用。直到trap机制将程序运行切换到内核空间之前，我们使用的仍然是没有这些方便映射关系的user page table。
- 学生提问：这个问题或许并不完全相关，read和write系统调用，相比内存的读写，他们的代价都高的多，因为它们需要切换模式，并来回捣腾。有没有可能当你执行打开一个文件的系统调用时， 直接得到一个page table映射，而不是返回一个文件描述符？这样只需要向对应于设备的特定的地址写数据，程序就能通过page table访问特定的设备。你可以设置好限制，就像文件描述符只允许修改特定文件一样，这样就不用像系统调用一样在用户空间和内核空间来回捣腾了。
  - Robert教授：这是个很好的想法。实际上很多操作系统都提供这种叫做内存映射文件（Memory-mapped file access）的机制，在这个机制里面通过page table，可以将用户空间的虚拟地址空间，对应到文件内容，这样你就可以通过内存地址直接读写文件。实际上，你们将在mmap 实验中完成这个机制。对于许多程序来说，这个机制的确会比直接调用read/write系统调用要快的多。

## 6.3 GDB跟踪代码

### 6.3.1 `ECALL`之前的状态

> **`sh.c`调用`write`系统调用**

下列代码是一个`write`系统调用，它将`$`写入到文件描述符`2`。

```c
int
getcmd(char *buf, int nbuf)
{
  fprintf(2, "$ ");
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

> **打开GDB并启动XV6**

- 启动xv6

```sh
(base) iiixv@IIIXVdeMacBook-Air 开发 % cd xv6-riscv 
(base) iiixv@IIIXVdeMacBook-Air xv6-riscv % make qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::25501
```

- 启动gbd

```sh
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
warning: File "/Users/iiixv/开发/xv6-riscv/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
	add-auto-load-safe-path /Users/iiixv/开发/xv6-riscv/.gdbinit
line to your configuration file "/Users/iiixv/.gdbinit".
To completely disable this security protection add
	set auto-load safe-path /
line to your configuration file "/Users/iiixv/.gdbinit".
For more information about this security protection see the
--Type <RET> for more, q to quit, c to continue without paging--c
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
	info "(gdb)Auto-loading safe path"
(gdb) source .gdbinit
The target architecture is set to "riscv:rv64".
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000001000 in ?? ()
(gdb) 
```

> **`Shell`调用`write`时关联的库函数**

作为用户代码的`Shell`调用`write`时，实际上调用的是关联到`Shell`的一个库函数。你可以查看这个库函数的源代码，在`usys.s`。

下列代码就是实际被调用的`write`函数的实现：

```s
write:
 li a7, SYS_write
 ecall
 ret
 ```

- 它首先将`SYS_write`加载到`a7`寄存器，`SYS_write`是常量`16`。这里告诉内核，我想要运行第16个系统调用，而这个系统调用正好是write。
- 之后这个函数中执行了`ecall`指令，从这里开始代码执行跳转到了内核。
- 内核完成它的工作之后，代码执行会返回到用户空间，继续执行`ecall`之后的指令，也就是`ret`
- 最终返回到`Shell`中，所以`ret`从`write`库函数返回到了`Shell`中。

> **`ecall`指令处设置断点**

为了展示这里的系统调用，需要在`ecall`指令处放置一个断点。

```sh
(gdb) b *0xe00
Breakpoint 1 at 0xe00
```

为了能设置断点，需要知道`ecall`指令的地址，我们可以通过查看由`XV6`编译产生的`sh.asm`找出这个地址。`sh.asm`是带有指令地址的汇编代码（注：`asm`文件3.7有介绍）。通过查看这条指令的地址是`0xe00`。

```asm
0000000000000dfe <write>:
.global write
write:
 li a7, SYS_write
     dfe:	48c1                	li	a7,16
 ecall
     e00:	00000073          	ecall
 ret
     e04:	8082                	ret
 ```

> **查看代码所在位置**

现在，让XV6开始运行，我们期望的是XV6在Shell代码中正好在执行ecall之前就会停住。

- 从`gdb`可以看出，我们下一条要执行的指令就是`ecall`。我们来检验一下我们真的在我们期望自己在的位置，让我们来打印程序计数器（`Program Counter`），正好我们期望在的位置`0xde6`。

```sh
(gdb) b *0xe00
Breakpoint 1 at 0xe00
(gdb) c
Continuing.
[Switching to Thread 1.2]

Thread 2 hit Breakpoint 1, 0x0000000000000e00 in ?? ()
=> 0x0000000000000e00:	73 00 00 00	ecall
(gdb) print $pc
$1 = (void (*)()) 0xe00
(gdb) 
 ```

- 输入`info reg`打印全部32个用户寄存器
  - `a0，a1，a2`是Shell传递给write系统调用的参数：
  - `a0`是文件描述符2；
  - `a1`是Shell想要写入字符串的指针；
  - `a2`是想要写入的字符数

  ```sh
  (gdb) info reg
  ra             0x24	0x24
  sp             0x4f70	0x4f70
  gp             0x505050505050505	0x505050505050505
  tp             0x505050505050505	0x505050505050505
  t0             0x505050505050505	361700864190383365
  t1             0x505050505050505	361700864190383365
  t2             0x505050505050505	361700864190383365
  fp             0x4f90	0x4f90
  s1             0x2020	8224
  a0             0x2	2
  a1             0x1300	4864
  a2             0x2	2
  a3             0x505050505050505	361700864190383365
  a4             0x505050505050505	361700864190383365
  a5             0x2	2
  a6             0x505050505050505	361700864190383365
  a7             0x10	16
  s2             0x64	100
  s3             0x20	32
  s4             0x2023	8227
  s5             0x1400	5120
  s6             0x505050505050505	361700864190383365
  s7             0x505050505050505	361700864190383365
  ```

- 打印`Shell`想要写入的字符串内容，进一步证明断点停在我们期望停在的位置

```sh
(gdb) x/2c $a1
0x1300:	36 '$'	32 ' '
 ```

上图的寄存器中，程序计数器（pc）和堆栈指针（sp）的地址现在都在距离0比较近的地址，这进一步印证了当前代码运行在用户空间，因为用户空间中所有的地址都比较小。但是一旦我们进入到了内核，内核会使用大得多的内存地址。

> **查看`STAP`寄存器**

系统调用的时间点会有大量状态的变更，其中一个最重要的需要变更的状态，且在它变更之前我们对它还有依赖的，就是当前的`page table`

```sh
(gdb) print/x $satp
$2 = 0x8000000000087f5f
 ```

> **`QEMU`打印完整的`page table`**

上边`satp`寄存器输出的是**物理内存地址**，它并没有告诉我们有关`page table`中的映射关系是什么，`page table`长什么样。
但是幸运的是，在`QEMU`中有一个方法可以打印当前的`page table`。从`QEMU`界面，输入`ctrl a + c`可以进入到QEMU的`console`，之后输入`info mem`，`QEMU`会打印完整的`page table`。

```sh
(qemu) info mem
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
0000000000000000 0000000087f5c000 0000000000001000 r-xu-a-
0000000000001000 0000000087f59000 0000000000001000 r-xu---
0000000000002000 0000000087f58000 0000000000001000 rw-u---
0000000000003000 0000000087f57000 0000000000001000 rw-----
0000000000004000 0000000087f56000 0000000000001000 rw-u-ad
0000003fffffe000 0000000087f6d000 0000000000001000 rw---ad
0000003ffffff000 0000000080007000 0000000000001000 r-x--a-
 ```

上边`page table`是用户程序`Shell`的`page table`，它非常小，只包含了6条映射关系。这6条映射关系包括:

- 有关`Shell`的指令和数据
- 一个无效的`page`用来作为`guard page`，以防止`Shell`尝试使用过多的`stack page`。我们可以看出这个`page`是无效的，因为在attr这一列它并没有设置`u`标志位（第三行）。

`attr`这一列是PTE的标志位:

- 第三行的标志位是`rwx`表明这个page可以读，可以写，也可以执行指令。
- 之后的是`u`标志位，它表明`PTE_u`标志位是否被设置，用户代码只能访问`u`标志位设置了的PTE。
- 再下一个标志位我也不记得是什么了（注，从4.3可以看出，这个标志位是Global）。
- 再下一个标志位是`a`（`Accessed`），表明这条`PTE`是不是被使用过。
- 再下一个标志位`d`（`Dirty`）表明这条`PTE`是不是被写过。

最后两条`PTE`的虚拟地址非常大，非常接近虚拟地址的顶端，这两个`page`分别是`trapframe page`和`trampoline page`。可以看到，它们都没有设置`u`标志，所以用户代码不能访问这两条`PTE`。一旦进入到了`supervisor mode`，就可以访问这两条`PTE`了。

对于这里`page table`，有一件事情需要注意：它并没有包含任何内核部分的地址映射，这里既没有对于`kernel data`的映射，也没有对于`kernel`指令的映射。**除了最后两条`PTE`，这个`page table`几乎是完全为用户代码执行而创建，所以它对于在内核执行代码并没有直接特殊的作用。**

> **学生提问**

- 学生提问：`PTE`中`a`标志位是什么意思？
  - Robert教授：**这表示这条PTE是不是被代码访问过，是不是曾经有一个被访问过的地址包含在这个PTE的范围内**。d标志位表明是否曾经有写指令使用过这条PTE。这些标志位由硬件维护以方便操作系统使用。对于比XV6更复杂的操作系统，当物理内存吃紧的时候，可能会通过将一些内存写入到磁盘来，同时将相应的PTE设置成无效，来释放物理内存page。你可以想到，这里有很多策略可以让操作系统来挑选哪些page可以释放。我们可以查看a标志位来判断这条PTE是否被使用过，如果它没有被使用或者最近没有被使用，那么这条PTE对应的page适合用来保存到磁盘中。类似的，d标志位告诉内核，这个page最近被修改过。不过XV6没有这样的策略。

### 6.3.2 `ECALL`之后的状态

> **执行`ECALL`指令**

```sh
(gdb) stepi
0x0000003ffffff000 in ?? ()
=> 0x0000003ffffff000:	73 10 05 14	csrw	sscratch,a0
 ```

第一个问题，执行完了ecall之后我们现在在哪？我们可以打印程序计数器（Program Counter）来查看。

> **打印程序计数器（Program Counter）来查看位置**

可以看到程序计数器的值变化了，之前我们的程序计数器还在一个很小的地址0xe00，但是现在在一个大得多的地址

```sh
(gdb) print $pc
$1 = (void (*)()) 0x3ffffff000
 ```

> **查看`page table`**

我们还可以查看page table，我通过在QEMU中执行info mem来查看当前的page table，可以看出，这还是与之前完全相同的page table，所以page table没有改变。

```sh
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
0000000000000000 0000000087f5c000 0000000000001000 r-xu-a-
0000000000001000 0000000087f59000 0000000000001000 r-xu---
0000000000002000 0000000087f58000 0000000000001000 rw-u---
0000000000003000 0000000087f57000 0000000000001000 rw-----
0000000000004000 0000000087f56000 0000000000001000 rw-u-ad
0000003fffffe000 0000000087f6d000 0000000000001000 rw---ad
0000003ffffff000 0000000080007000 0000000000001000 r-x--a-
(qemu) info mem
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
0000000000000000 0000000087f5c000 0000000000001000 r-xu-a-
0000000000001000 0000000087f59000 0000000000001000 r-xu---
0000000000002000 0000000087f58000 0000000000001000 rw-u---
0000000000003000 0000000087f57000 0000000000001000 rw-----
0000000000004000 0000000087f56000 0000000000001000 rw-u-ad
0000003fffffe000 0000000087f6d000 0000000000001000 rw---ad
0000003ffffff000 0000000080007000 0000000000001000 r-x--a-
```

> **查看一下现在将要运行的指令**

```sh
(gdb) x/6i 0x0000003ffffff000
=> 0x3ffffff000:	csrw	sscratch,a0
   0x3ffffff004:	lui	a0,0x2000
   0x3ffffff008:	addiw	a0,a0,-1
   0x3ffffff00a:	slli	a0,a0,0xd
   0x3ffffff00c:	sd	ra,40(a0)
   0x3ffffff010:	sd	sp,48(a0)
```

根据现在的程序计数器，代码正在`trampoline page`的最开始，这是用户内存中一个非常大的地址。所以现在我们的指令正运行在内存的`trampoline page`中。

这些指令是内核在`supervisor mode`中将要执行的最开始的几条指令，也是在`trap`机制中最开始要执行的几条指令。因为`gdb`有一些奇怪的行为，我们实际上已经执行了位于`trampoline page`最开始的一条指令（注，也就是`csrrw`指令），我们将要执行的是第二条指令。

> **查看寄存器**

通过查看寄存器，对比`ECALL`指令之前的寄存器状态，**寄存器的值并没有改变**，**还是用户程序拥有的一些寄存器内容**，并且这些数据也还只保存在这些寄存器中。

```sh
info reg
ra             0x24	0x24
sp             0x4f70	0x4f70
gp             0x505050505050505	0x505050505050505
tp             0x505050505050505	0x505050505050505
t0             0x505050505050505	361700864190383365
t1             0x505050505050505	361700864190383365
t2             0x505050505050505	361700864190383365
fp             0x4f90	0x4f90
s1             0x2020	8224
a0             0x2	2
a1             0x1300	4864
a2             0x2	2
a3             0x505050505050505	361700864190383365
a4             0x505050505050505	361700864190383365
a5             0x2	2
a6             0x505050505050505	361700864190383365
a7             0x10	16
s2             0x64	100
s3             0x20	32
s4             0x2023	8227
s5             0x1400	5120
s6             0x505050505050505	361700864190383365
s7             0x505050505050505	361700864190383365
s8             0x505050505050505	361700864190383365
s9             0x505050505050505	361700864190383365
s10            0x505050505050505	361700864190383365
s11            0x505050505050505	361700864190383365
t3             0x505050505050505	361700864190383365
t4             0x505050505050505	361700864190383365
```

所以在将寄存器数据保存在某处之前，**在这个时间点不能使用任何寄存器**，否则的话我们是没法恢复寄存器数据的。如果内核在这个时间点使用了任何一个寄存器，内核会覆盖寄存器内的用户数据，之后如果我们尝试要恢复用户程序，我们就不能恢复寄存器中的正确数据，用户程序的执行也会相应的出错。

- 学生提问：我想知道`csrrw`指令是干什么的？
  - Robert教授：我们过几分钟会讨论这部分。但是对于你的问题的答案是，这条指令交换了寄存器`a0`和`sscratch`的内容。这个操作超级重要，它回答了这个问题：内核的`trap`代码如何能够在不使用任何寄存器的前提下做任何操作。这条指令将`a0`的数据保存在了`sscratch`中，同时又将`sscratch`内的数据保存在`a0`中。之后内核就可以任意的使用`a0`寄存器了。

> **查看`STVEC`寄存器**

现在地址是`0x3ffffff000`，也就是上面`page table`输出的最后一个`page`，这是`trampoline page`，即目前正在`trampoline page`中执行程序。**`trampoline page`包含了内核的`trap`处理代码**。

因为`ecall`并不会切换`page table`，所以`trap`处理代码必须存在于每一个`user page table`中。我们需要在`user page table`中的某个地方来执行最初的内核代码。而这个`trampoline page`，是由内核小心的映射到每一个`user page table`中，以使得当我们仍然在使用`user page table`时，内核在一个地方能够执行`trap`机制的最开始的一些指令。

**这里的控制是通过`STVEC`寄存器完成的**，这是一个只能在`supervisor mode`下读写的特权寄存器。在从内核空间进入到用户空间之前，内核会设置好`STVEC`寄存器指向内核希望`trap`代码运行的位置。

```sh
(gdb) print/x $stvec
$3 = 0x3ffffff000
```

所以如你所见，内核已经事先设置好了`STVEC`寄存器的内容为`0x3ffffff000`，这就是`trampoline page`的起始位置。

最后，我想提示你们，即使`trampoline page`是在用户地址空间的`user page table`完成的映射，用户代码不能写它，因为这些`page`对应的`PTE`并没有设置`PTE_u`标志位。这也是为什么`trap`机制是安全的。

> **ECALL指令实际改变的事情**

上边我们一直在说，现在已经在`supervisor mode`了，但是实际上并没有任何能直接确认当前在哪种`mode`下的方法。

不过我的确发现程序计数器现在正在`trampoline page`执行代码，而这些`page`对应的PTE并没有设置`PTE_u`标志位。**所以现在只有当代码在`supervisor mode`时，才可能在程序运行的同时而不崩溃**。所以，我从代码没有崩溃和程序计数器的值推导出我们必然在`supervisor mode`。

通过`ecall`走到`trampoline page`的，而`ecall`实际上只会改变三件事情：

- `ecall`将代码从`user mode`改到`supervisor mode`
- `ecall`将程序计数器的值保存在了`SEPC`寄存器
  - 可以**通过打印程序计数器看到这里的效果**，尽管其他的寄存器还是还是用户寄存器，但是这里的程序计数器明显已经不是用户代码的程序计数器。这里的程序计数器是从`STVEC`寄存器拷贝过来的值。
  
    ```sh
    (gdb) stepi
      0x0000003ffffff004 in ?? ()
      => 0x0000003ffffff004:	37 05 00 02	lui	a0,0x2000
    ```

  - 我们也可以打印`SEPC`（`Supervisor Exception Program Counter`）寄存器，这是`ecall`保存用户程序计数器的地方。这个寄存器里面有熟悉的地址`0xe00`，这是`ecall`指令在用户空间的地址。所以`ecall`至少保存了程序计数器的数值。
  
    ```sh
    (gdb) print/x $sepc
    $5 = 0xe00
    ```

- `ecall`会跳转到`STVEC`寄存器指向的指令。

> **接下来的工作**

所以现在，ecall帮我们做了一点点工作，但是实际上我们离执行内核中的C代码还差的很远。接下来：

- **需要保存32个用户寄存器的内容**，这样当我们想要恢复用户代码执行时，我们才能恢复这些寄存器的内容
- 因为现在我们还在`user page table`，我们需要切换到`kernel page table`。
- 我们需要创建或者找到一个`kernel stack`，并将`Stack Pointer`寄存器的内容指向那个`kernel stack`。这样才能给C代码提供栈。
- 我们还需要跳转到内核中C代码的某些合理的位置。

ecall并不会为我们做这里的任何一件事。

> **其他**

当然，我们可以通过修改硬件让ecall为我们完成这些工作，而不是交给软件来完成。并且，我们也将会看到，在软件中完成这些工作并不是特别简单。

所以你现在就会问，为什么`ecall`不多做点工作来将代码执行从用户空间切换到内核空间呢？为什么`ecall`不会保存用户寄存器，或者切换`page table`指针来指向`kernel page table`，或者自动的设置`Stack Pointer`指向`kernel stack`，或者直接跳转到`kernel`的`C`代码，而不是在这里运行复杂的汇编代码？

实际上，有的机器在执行系统调用时，会在硬件中完成所有这些工作。但是RISC-V并不会，RISC-V秉持了这样一个观点：**ecall只完成尽量少必须要完成的工作，其他的工作都交给软件完成。**这里的原因是，RISC-V设计者想要为软件和操作系统的程序员提供最大的灵活性，这样他们就能按照他们想要的方式开发操作系统。所以你可以这样想，尽管XV6并没有使用这里提供的灵活性，但是一些其他的操作系统用到了。

- 举个例子，因为这里的`ecall`是如此的简单，或许某些操作系统可以在不切换`page table`的前提下，执行部分系统调用。切换`page table`的代价比较高，如果`ecall`打包完成了这部分工作，那就不能对一些系统调用进行改进，使其不用在不必要的场景切换`page table`。
- 某些操作系统同时将`user`和`kernel`的虚拟地址映射到一个`page table`中，这样在`user`和`kernel`之间切换时根本就不用切换`page table`。对于这样的操作系统来说，如果`ecall`切换了`page table`那将会是一种浪费，并且也减慢了程序的运行。
- 或许在一些系统调用过程中，一些寄存器不用保存，而哪些寄存器需要保存，哪些不需要，取决于于软件，编程语言，和编译器。通过不保存所有的32个寄存器或许可以节省大量的程序运行时间，所以你不会想要ecall迫使你保存所有的寄存器。
- 最后，对于某些简单的系统调用或许根本就不需要任何stack，所以对于一些非常关注性能的操作系统，ecall不会自动为你完成stack切换是极好的。

所以，ecall尽量的简单可以提升软件设计的灵活性。

> **学生提问**

- 学生提问：为什么我们在`gdb`中看不到`ecall`的具体内容？或许我错过了，但是我觉得我们是直接跳到`trampoline`代码的。
  - Robert教授：`ecall`只会更新`CPU`中的`mode`标志位为`supervisor`，并且设置程序计数器成`STVEC`寄存器内的值。在进入到用户空间之前，内核会将`trampoline page`的地址存在`STVEC`寄存器中。所以`ecall`的下一条指令的位置是`STVEC`指向的地址，也就是`trampoline page`的起始地址。（注，**实际上`ecall`是`CPU`的指令，自然在`gdb`中看不到具体内容**）

### 6.3.3 `uservec`函数

回到`XV6`和`RISC-V`，现在程序位于`trampoline page`的起始，也是`uservec`函数的起始。我们现在需要做的第一件事情就是**保存寄存器的内容**。

> **保存寄存器内容的常见操作**

在RISC-V上，如果不能使用寄存器，基本上不能做任何事情。所以，对于保存这些寄存器，我们有什么样的选择呢？

- 在一些其他的机器中，**直接将32个寄存器中的内容写到物理内存中某些合适的位置**。但是我们不能在RISC-V中这样做，因为在RISC-V中，`supervisor mode`下的代码不允许直接访问物理内存。所以我们只能使用`page table`中的内容，但是从前面的输出来看，`page table`中也没有多少内容。
- **直接将SATP寄存器指向`kernel page table`**，之后我们就可以直接使用所有的`kernel mapping`来帮助我们存储用户寄存器。这是合法的，因为`supervisor mode`可以更改SATP寄存器。但是在`trap`代码当前的位置，也就是`trap`机制的最开始，我们并不知道`kernel page table`的地址。并且更改SATP寄存器的指令，要求写入SATP寄存器的内容来自于另一个寄存器。所以，为了能执行更新`page table`的指令，我们需要一些空闲的寄存器，这样我们才能先将`page table`的地址存在这些寄存器中，然后再执行修改SATP寄存器的指令。

> **XV6实现保存寄存器内容**

**对于保存用户寄存器，XV6在RISC-V上的实现包括了两个部分**。

- 第一个部分是：XV6在每个`user page table`映射了`trapframe page`，这样每个进程都有自己的`trapframe page`。即在`trap`处理代码中，在`user page table`中包含一个之前由`kernel`设置好的映射关系，这个映射关系指向了一个可以用来存放这个进程的用户寄存器的内存位置。这个位置的虚拟地址总是`0x3ffffffe000`。可以通过`proc.h`中的`trapframe`结构体查看`trapframe page`中存放了什么
  
  ```h
  struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
  };
  ```

- **另一半的答案在于我们之前提过的`SSCRATCH`寄存器**。在进入到`user space`之前，内核会将`trapframe page`的地址（`0x3fffffe000`）保存在由RISC-V提供的`SSCRATCH`寄存器中。更重要的是，RISC-V有一个指令允许交换任意两个寄存器的值。而`SSCRATCH`寄存器的作用就是保存另一个寄存器的值，并将自己的值加载给另一个寄存器。

查看`trampoline.S`代码：

- 首先，执行`csrrw`指令，这个指令交换了a0和sscratch两个寄存器的内容。可以通过打印`a0`和`SSCRATCH`寄存器内容。`a0`现在的值是`0x3fffffe000`，(`trapframe page`的虚拟地址)。它之前保存在`SSCRATCH`寄存器中，但是我们现在交换到了`a0`中。`SSCRATCH`寄存器现在的内容是2，这是`a0`寄存器之前的值。a0寄存器保存的是write函数的第一个参数，在这个场景下，是Shell传入的文件描述符2。所以我们现在将a0的值保存起来了，并且我们有了指向`trapframe page`的指针。*(注意此处源码与视频中内容有出入)*
  
  ```sh
  (gdb) print/x $a0
  $11 = 0x3fffffe000
  (gdb) print/x $sscratch
  $12 = 0x2
  ```

- `trampoline.S`中接下来30多个奇怪指令的工作。这些指令就是的执行`sd`，将每个寄存器保存在`trapframe`的不同偏移位置。因为`a0`在交换完之后包含的是`trapframe page`地址，也就是`0x3fffffe000`。所以，**每个寄存器被保存在了偏移量`+a0`的位置**。

```s
.globl uservec
uservec:    
 #
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #

        # save user a0 in sscratch so
        # a0 can be used to get at TRAPFRAME.
        csrw sscratch, a0

        # each process has a separate p->trapframe memory area,
        # but it's mapped to the same virtual address
        # (TRAPFRAME) in every process's user page table.
        li a0, TRAPFRAME
        
        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # initialize kernel stack pointer, from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), from p->trapframe->kernel_trap
        ld t0, 16(a0)


        # fetch the kernel page table address, from p->trapframe->kernel_satp.
        ld t1, 0(a0)

        # wait for any previous memory operations to complete, so that
        # they use the user page table.
        sfence.vma zero, zero

        # install the kernel page table.
        csrw satp, t1

        # flush now-stale user entries from the TLB.
        sfence.vma zero, zero

        # jump to usertrap(), which does not return
        jr t0

```

> **学生提问**

- 学生提问：当与`a0`寄存器进行交换时，`trapframe`的地址是怎么出现在`SSCRATCH`寄存器中的？
  - Robert教授：在内核前一次切换回用户空间时，内核会执行`set sscratch`指令，将这个寄存器的内容设置为`0x3fffffe000`，也就是`trapframe page`的虚拟地址。所以，当我们在运行用户代码，比如运行`Shell`时，`SSCRATCH`保存的就是指向trapframe的地址。之后，Shell执行了ecall指令，跳转到了trampoline page，这个page中的第一条指令会交换a0和SSCRATCH寄存器的内容。所以，SSCRATCH中的值，也就是指向trapframe的指针现在存储与a0寄存器中。
- 同一个学生提问：这是发生在进程创建的过程中吗？这个SSCRATCH寄存器存在于哪？
  - Robert教授：这个寄存器存在于CPU上，这是CPU上的一个特殊寄存器。内核在什么时候设置的它呢？这有点复杂。它被设置的实际位置，选中的代码是内核在返回到用户空间之前执行的最后两条指令。
  
    ```sh
    # restore user a0
    ld a0, 112(a0)
    
    # return to user mode and user pc.
    # usertrapret() set up sstatus and sepc.
    sret
    ```

    在内核返回到用户空间时，会恢复所有的用户寄存器。之后会再次执行交换指令，`csrrw`。因为之前内核已经设置了a0保存的是trap frame地址，经过交换之后SSCRATCH仍然指向了trapframe page地址，而a0也恢复成了之前的数值。最后sret返回到了用户空间。

    你或许会好奇，a0是如何有trapframe page的地址。我们可以查看trap.c代码

    ```c
      // jump to userret in trampoline.S at the top of memory, which 
    // switches to the user page table, restores user registers,
    // and switches to user mode with sret.
    uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
    ((void (*)(uint64))trampoline_userret)(satp);
    ```

    这是内核返回到用户空间的最后的C函数。C函数做的最后一件事情是调用`trampoline_userret`函数，传递的参数是`TRAMFRAME`和`user page table`。在C代码中，当你调用函数，第一个参数会存在`a0`，这就是为什么`a0`里面的数值是指向`trapframe`的指针。`fn`函数是就是刚刚我向你展示的位于`trampoline.S`中的代码。

- 学生提问：当你启动一个进程，之后进程在运行，之后在某个时间点进程执行了`ecall`指令，那么你是在什么时候执行上一个问题中的`fn`函数呢？因为这是进程的第一个`ecall`指令，所以这个进程之前应该没有调用过`fn`函数吧。
  - Robert教授：好的，或许对于这个问题的一个答案是：一台机器总是从内核开始运行的，当机器启动的时候，它就是在内核中。 任何时候，不管是进程第一次启动还是从一个系统调用返回，进入到用户空间的唯一方法是就是执行`sret`指令。`sret`指令是由RISC-V定义的用来从`supervisor mode`转换到`user mode`。**所以，在任何用户代码执行之前，内核会执行fn函数，并设置好所有的东西，例如SSCRATCH，STVEC寄存器。**
- 学生提问：当我们在汇编代码中执行`ecall`指令，是什么触发了`trampoline`代码的执行，是CPU中的从user到supervisor的标志位切换吗？
  - Robert教授：在我们的例子中，Shell在用户空间执行了ecall指令。ecall会完成几件事情，ecall指令会设置当前为`supervisor mode`，保存程序计数器到`SEPC`寄存器，并且将程序计数器设置成控制寄存器`STVEC`的内容。`STVEC`是内核在进入到用户空间之前设置好的众多数据之一，内核会将其设置成`trampoline page`的起始位置。所以，当ecall指令执行时，ecall会将STVEC拷贝到程序计数器。之后程序继续执行，但是却会在当前程序计数器所指的地址，也就是`trampoline page`的起始地址执行。
- 学生提问：寄存器保存在了`trapframe page`，但是这些寄存器用户程序也能访问，为什么我们要使用内存中一个新的区域（指的是`trapframe page`），而不是使用程序的栈？
  - Robert教授：好的，这里或许有两个问题。第一个是，为什么我们要保存寄存器？为什么内核要保存寄存器的原因，是因为内核即将要运行会覆盖这些寄存器的C代码。如果我们想正确的恢复用户程序，我们需要将这些寄存器恢复成它们在ecall调用之前的数值，所以我们需要将所有的寄存器都保存在trapframe中，这样才能在之后恢复寄存器的值。
  另一个问题是，为什么这些寄存器保存在trapframe，而不是用户代码的栈中？这个问题的答案是，我们不确定用户程序是否有栈，必然有一些编程语言没有栈，对于这些编程语言的程序，Stack Pointer不指向任何地址。当然，也有一些编程语言有栈，但是或许它的格式很奇怪，内核并不能理解。比如，编程语言以堆中以小块来分配栈，编程语言的运行时知道如何使用这些小块的内存来作为栈，但是内核并不知道。所以，如果我们想要运行任意编程语言实现的用户程序，内核就不能假设用户内存的哪部分可以访问，哪部分有效，哪部分存在。所以内核需要自己管理这些寄存器的保存，这就是为什么内核将这些内容保存在属于内核内存的trapframe中，而不是用户内存。

> **将`kernel_sp`加载到`Stack Pointer`**

程序现在仍然在`trampoline`的最开始，也就是`uservec`函数的最开始，基本上还没有执行任何内容。我在寄存器拷贝的结束位置设置了一个断点，我们在gdb中让代码继续执行，现在我们停在了下面这条ld（load）指令。

```sh
(gdb) b *0x3ffffff076
Breakpoint 2 at 0x3ffffff076
(gdb) c
Continuing.

Thread 3 hit Breakpoint 2, 0x0000003ffffff076 in ?? ()
=> 0x0000003ffffff076:	f3 22 00 14	csrr	t0,sscratch
```

这条指令正在将`a0`指向的内存地址往后数的第8个字节开始的数据加载到`Stack Pointer`寄存器。`a0`的内容现在是`trapframe page`的地址，从本节第一张图中，trapframe的格式可以看出，**第8个字节开始的数据是内核的`Stack Pointer（kernel_sp）`**。

`trapframe`中的`kernel_sp`是由`kernel`在进入用户空间之前就设置好的，它的值是这个进程的`kernel stack`。

**所以这条指令的作用是初始化`Stack Pointer`指向这个进程的`kernel stack`的最顶端**。

此时打印一下当前的`Stack Pointer`寄存器，
这是这个进程的kernel stack。因为XV6在每个kernel stack下面放置一个guard page，所以kernel stack的地址都比较大。

```sp
print/x $sp
$3 = 0x3fffffc000
```

> **向tp寄存器写入数据**

因为在RISC-V中，没有一个直接的方法来确认当前运行在多核处理器的哪个核上，XV6会将CPU核的编号也就是`hartid`保存在`tp`寄存器。在内核中好几个地方都会使用了这个值，例如，内核可以通过这个值确定某个CPU核上运行了哪些进程。我们执行这条指令，并且打印`tp`寄存器。

```sh
(gdb) print/x $tp
$3 = 0x0
```

我们现在运行在CPU核0，这说的通，因为我之前配置了QEMU只给XV6分配一个核，所以我们只能运行在核0上。

> **向t0寄存器写入数据**

这里写入的是我们将要执行的第一个C函数的指针，也就是函数`usertrap`的指针。我们在后面会使用这个指针。

```sh
(gdb) print/x $t0
$5 = 0x800027e0
```

> **向t1寄存器写入数据**

这里写入的是`kernel page table`的地址，我们可以打印t1寄存器的内容。

```sh
(gdb) print/x $t1
$7 = 0x8000000000087fff
```

实际上严格来说，`t1`的内容并不是`kernel page table`的地址，这是你需要向SATP寄存器写入的数据。它包含了`kernel page table`的地址，但是移位了（注，详见4.3），并且包含了各种标志位。

> **交换SATP和t1寄存器**

这条指令执行完成之后，当前程序会从`user page table`切换到`kernel page table`。现在我们在QEMU中打印`page table`，可以看出与之前的`page table`完全不一样。

```sh
(qemu) info mem
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
000000000c000000 000000000c000000 0000000000001000 rw---ad
000000000c001000 000000000c001000 0000000000001000 rw-----
000000000c002000 000000000c002000 0000000000001000 rw---ad
000000000c003000 000000000c003000 00000000001fe000 rw-----
000000000c201000 000000000c201000 0000000000001000 rw---ad
000000000c202000 000000000c202000 00000000001fe000 rw-----
0000000010000000 0000000010000000 0000000000001000 rw---a-
0000000010001000 0000000010001000 0000000000001000 rw---ad
0000000080000000 0000000080000000 0000000000007000 r-x--a-
0000000080007000 0000000080007000 0000000000001000 r-x----
0000000080008000 0000000080008000 0000000000002000 rw---ad
000000008000a000 000000008000a000 0000000000006000 rw-----
0000000080010000 0000000080010000 0000000000012000 rw---ad
0000000080022000 0000000080022000 0000000007f4c000 rw-----
0000000087f6e000 0000000087f6e000 000000000000a000 rw---ad
0000000087f78000 0000000087f78000 0000000000088000 rw-----
0000003ffff7f000 0000000087f78000 000000000003f000 rw-----
0000003fffffd000 0000000087fb7000 0000000000001000 rw---ad
0000003ffffff000 0000000080007000 0000000000001000 r-x--a-
```

现在这里输出的是由内核设置好的巨大的kernel page table。所以现在我们成功的切换了page table，我们在这个位置进展的很好，Stack Pointer指向了kernel stack；我们有了kernel page table，可以读取kernel data。我们已经准备好了执行内核中的C代码了。

> **代码为什么没崩溃？**

这里还有个问题，为什么代码没有崩溃？

毕竟我们在内存中的某个位置执行代码，**程序计数器保存的是虚拟地址**，如果我们切换了`page table`，为什么同一个虚拟地址不会通过新的`page table`寻址走到一些无关的page中？看起来我们现在没有崩溃并且还在执行这些指令。有人来猜一下原因吗？

学生回答：因为我们还在`trampoline`代码中，而`trampoline`代码在用户空间和内核空间都映射到了同一个地址。

完全正确。我不知道你们是否还记得user page table的内容，trampoline page在user page table中的映射与kernel page table中的映射是完全一样的。这两个page table中其他所有的映射都是不同的，只有trampoline page的映射是一样的，因此我们在切换page table时，寻址的结果不会改变，我们实际上就可以继续在同一个代码序列中执行程序而不崩溃。这是trampoline page的特殊之处，它同时在user page table和kernel page table都有相同的映射关系。
之所以叫trampoline page，是因为你某种程度在它上面“弹跳”了一下，然后从用户空间走到了内核空间。

> **最后一条指令是jr t0**

执行了这条指令，我们就要从`trampoline`跳到内核的C代码中。这条指令的作用是**跳转到t0指向的函数中**。我们打印t0对应的一些指令，
可以看到t0的位置对应于一个叫做usertrap函数的开始。接下来我们就要以kernel stack，kernel page table跳转到usertrap函数。

### 6.3.4 `usertrap`函数

usertrap函数是位于trap.c文件的一个函数。
既然我们已经运行在C代码中，接下来，我在gdb中输入tui enable打开对于C代码的展示。
我们现在在一个更加正常的世界中，我们正在运行C代码，应该会更容易理解。我们仍然会读写一些有趣的控制寄存器，但是环境比起汇编语言来说会少了很多晦涩。

有很多原因都可以让程序运行进入到usertrap函数中来，比如系统调用，运算时除以0，使用了一个未被映射的虚拟地址，或者是设备中断。usertrap某种程度上存储并恢复硬件状态，但是它也需要检查触发trap的原因，以确定相应的处理方式，我们在接下来执行usertrap的过程中会同时看到这两个行为。

> **更改`STVEC`寄存器**

`usertrap`函数做的第一件事情是更改`STVEC`寄存器。

`XV6`处理`trap`的方法是不一样的。取决于`trap`是来自于用户空间还是内核空间，实际上目前为止，我们只讨论过当trap是由用户空间发起时会发生什么。如果trap从内核空间发起，将会是一个非常不同的处理流程，因为从内核发起的话，程序已经在使用kernel page table。所以当trap发生时，程序执行仍然在内核的话，很多处理都不必存在。

在内核中执行任何操作之前，usertrap中先将STVEC指向了kernelvec变量，这是内核空间trap处理代码的位置，而不是用户空间trap处理代码的位置。
出于各种原因，我们需要知道当前运行的是什么进程，我们通过调用myproc函数来做到这一点。myproc函数实际上会查找一个根据当前CPU核的编号索引的数组，CPU核的编号是hartid，如果你还记得，我们之前在uservec函数中将它存在了tp寄存器。这是myproc函数找出当前运行进程的方法。

> **保存用户程序计数器**

接下来我们要保存用户程序计数器，它仍然保存在`SEPC`寄存器中，但是可能发生这种情况：当程序还在内核中执行时，我们可能切换到另一个进程，并进入到那个程序的用户空间，然后那个进程可能再调用一个系统调用进而导致SEPC寄存器的内容被覆盖。所以，我们需要保存当前进程的SEPC寄存器到一个与该进程关联的内存中，这样这个数据才不会被覆盖。这里我们使用trapframe来保存这个程序计数器。

> **确定进入`usertrap`函数的原因**

接下来我们需要找出我们现在会在usertrap函数的原因。根据触发trap的原因，RISC-V的SCAUSE寄存器会有不同的数字。数字8表明，我们现在在trap代码中是因为系统调用。可以打印SCAUSE寄存器，它的确包含了数字8，我们的确是因为系统调用才走到这里的。

所以，我们可以进到这个if语句中。接下来第一件事情是检查是不是有其他的进程杀掉了当前进程，但是我们的Shell没有被杀掉，所以检查通过。

在RISC-V中，存储在SEPC寄存器中的程序计数器，是用户程序中触发trap的指令的地址。但是当我们恢复用户程序时，我们希望在下一条指令恢复，也就是ecall之后的一条指令。所以对于系统调用，我们对于保存的用户程序计数器加4，这样我们会在ecall的下一条指令恢复，而不是重新执行ecall指令。
XV6会在处理系统调用的时候使能中断，这样中断可以更快的服务，有些系统调用需要许多时间处理。中断总是会被RISC-V的trap硬件关闭，所以在这个时间点，我们需要显式的打开中断。

> **调用`syscall`函数**

下一行代码中，我们会调用syscall函数。这个函数定义在syscall.c。
它的作用是从syscall表单中，根据系统调用的编号查找相应的系统调用函数。如果你还记得之前的内容，Shell调用的write函数将a7设置成了系统调用编号，对于write来说就是16。所以syscall函数的工作就是获取由trampoline代码保存在trapframe中a7的数字，然后用这个数字索引实现了每个系统调用的表单。
我们可以打印num，的确是16。这与Shell调用的write函数写入的数字是一致的。
之后查看通过num索引得到的函数，正是sys_write函数。sys_write函数是内核对于write系统调用的具体实现。这里再往后的代码执行就非常复杂了，我就不具体介绍了。在这节课中，对于系统调用的实现，我只对进入和跳出内核感兴趣。这里我让代码直接执行sys_write函数。
这里有件有趣的事情，系统调用需要找到它们的参数。你们还记得write函数的参数吗？分别是文件描述符2，写入数据缓存的指针，写入数据的长度2。syscall函数直接通过trapframe来获取这些参数，就像这里刚刚可以查看trapframe中的a7寄存器一样，我们可以查看a0寄存器，这是第一个参数，a1是第二个参数，a2是第三个参数。
现在syscall执行了真正的系统调用，之后sys_write返回了。

这里向trapframe中的a0赋值的原因是：所有的系统调用都有一个返回值，比如write会返回实际写入的字节数，而RISC-V上的C代码的习惯是函数的返回值存储于寄存器a0，所以为了模拟函数的返回，我们将返回值存储在trapframe的a0中。之后，当我们返回到用户空间，trapframe中的a0槽位的数值会写到实际的a0寄存器，Shell会认为a0寄存器中的数值是write系统调用的返回值。执行完这一行代码之后，我们打印这里trapframe中a0的值，可以看到输出2。
这意味这sys_write的返回值是2，符合传入的参数，这里只写入了2个字节。
从syscall函数返回之后，我们回到了trap.c中的usertrap函数。
我们再次检查当前用户进程是否被杀掉了，因为我们不想恢复一个被杀掉的进程。当然，在我们的场景中，Shell没有被杀掉。

最后，`usertrap`调用了一个函数`usertrapret`。

### 6.3.5 `usertrapret`函数

usertrap函数的最后调用了usertrapret函数，来设置好在返回到用户空间之前内核要做的工作。我们可以查看这个函数的内容。

> **关闭中断**

它首先关闭了中断。我们之前在系统调用的过程中是打开了中断的，这里关闭中断是因为我们将要更新STVEC寄存器来指向用户空间的trap处理代码，而之前在内核中的时候，我们指向的是内核空间的trap处理代码（6.6）。我们关闭中断因为当我们将STVEC更新到指向用户空间的trap处理代码时，我们仍然在内核中执行代码。如果这时发生了一个中断，那么程序执行会走向用户空间的trap处理代码，即便我们现在仍然在内核中，出于各种各样具体细节的原因，这会导致内核出错。所以我们这里关闭中断。

> **设置了STVEC寄存器指向trampoline代码**

在下一行我们设置了STVEC寄存器指向trampoline代码，在那里最终会执行sret指令返回到用户空间。位于trampoline代码最后的sret指令会重新打开中断。这样，即使我们刚刚关闭了中断，当我们在执行用户代码时中断是打开的。
接下来的几行填入了trapframe的内容，这些内容对于执行trampoline代码非常有用。这里的代码就是：
存储了kernel page table的指针
存储了当前用户进程的kernel stack
存储了usertrap函数的指针，这样trampoline代码才能跳转到这个函数（注，详见6.5中 ld t0 (16)a0 指令）
从tp寄存器中读取当前的CPU核编号，并存储在trapframe中，这样trampoline代码才能恢复这个数字，因为用户代码可能会修改这个数字
现在我们在usertrapret函数中，我们正在设置trapframe中的数据，这样下一次从用户空间转换到内核空间时可以用到这些数据。

> **学生提问**

- 学生提问：为什么trampoline代码中不保存SEPC寄存器？
  - Robert教授：可以存储。trampoline代码没有像其他寄存器一样保存这个寄存器，但是我们非常欢迎大家修改XV6来保存它。如果你还记得的话（详见6.6），这个寄存器实际上是在C代码usertrap中保存的，而不是在汇编代码trampoline中保存的。我想不出理由这里哪种方式更好。用户寄存器（User Registers）必须在汇编代码中保存，因为任何需要经过编译器的语言，例如C语言，都不能修改任何用户寄存器。所以对于用户寄存器，必须要在进入C代码之前在汇编代码中保存好。但是对于SEPC寄存器（注，控制寄存器），我们可以早点保存或者晚点保存。

> **设置`SSTATUS`寄存器**
接下来我们要设置SSTATUS寄存器，这是一个控制寄存器。这个寄存器的SPP bit位控制了sret指令的行为，该bit为0表示下次执行sret的时候，我们想要返回user mode而不是supervisor mode。这个寄存器的SPIE bit位控制了，在执行完sret之后，是否打开中断。因为我们在返回到用户空间之后，我们的确希望打开中断，所以这里将SPIE bit位设置为1。修改完这些bit位之后，我们会把新的值写回到SSTATUS寄存器。

我们在trampoline代码的最后执行了sret指令。这条指令会将程序计数器设置成SEPC寄存器的值，所以现在我们将SEPC寄存器的值设置成之前保存的用户程序计数器的值。在不久之前，我们在usertrap函数中将用户程序计数器保存在trapframe中的epc字段。

> **根据user page table地址生成相应的SATP值**

接下来，我们根据user page table地址生成相应的SATP值，这样我们在返回到用户空间的时候才能完成page table的切换。实际上，我们会在汇编代码trampoline中完成page table的切换，并且也只能在trampoline中完成切换，因为只有trampoline中代码是同时在用户和内核空间中映射。但是我们现在还没有在trampoline代码中，我们现在还在一个普通的C函数中，所以这里我们将page table指针准备好，并将这个指针作为第二个参数传递给汇编代码，这个参数会出现在a1寄存器。
倒数第二行的作用是计算出我们将要跳转到汇编代码的地址。我们期望跳转的地址是tampoline中的userret函数，这个函数包含了所有能将我们带回到用户空间的指令。所以这里我们计算出了userret函数的地址。
倒数第一行，将fn指针作为一个函数指针，执行相应的函数（也就是userret函数）并传入两个参数，两个参数存储在a0，a1寄存器中。

### 6.3.6 `userret`函数

现在程序执行又到了trampoline代码。

> **切换`page table`**

第一步是切换page table。在执行csrw satp, a1之前，page table应该还是巨大的kernel page table。这条指令会将user page table（在usertrapret中作为第二个参数传递给了这里的userret函数，所以存在a1寄存器中）存储在SATP寄存器中。执行完这条指令之后，page table就变成了小得多的user page table。但是幸运的是，user page table也映射了trampoline page，所以程序还能继续执行而不是崩溃。（注，sfence.vma是清空页表缓存，详见4.4）。
在uservec函数中，第一件事情就是交换SSRATCH和a0寄存器。而这里，我们将SSCRATCH寄存器恢复成保存好的用户的a0寄存器。在这里a0是trapframe的地址，因为C代码usertrapret函数中将trapframe地址作为第一个参数传递过来了。112是a0寄存器在trapframe中的位置。（注，这里有点绕，本质就是通过当前的a0寄存器找出存在trapframe中的a0寄存器）我们先将这个地址里的数值保存在t0寄存器中，之后再将t0寄存器的数值保存在SSCRATCH寄存器中。
为止目前，所有的寄存器内容还是属于内核。

> **将保存的寄存器值恢复到对应的各个寄存器**

接下来的这些指令将a0寄存器指向的trapframe中，之前保存的寄存器的值加载到对应的各个寄存器中。之后，我们离能真正运行用户代码就很近了。

学生提问：现在trapframe中的a0寄存器是我们执行系统调用的返回值吗？
Robert教授：是的，系统调用的返回值覆盖了我们保存在trapframe中的a0寄存器的值（详见6.6）。我们希望用户程序Shell在a0寄存器中看到系统调用的返回值。所以，trapframe中的a0寄存器现在是系统调用的返回值2。相应的SSCRATCH寄存器中的数值也应该是2，可以通过打印寄存器的值来验证。
现在我们打印所有的寄存器，
我不确定你们是否还记得，但是这些寄存器的值就是我们在最最开始看到的用户寄存器的值。例如SP寄存器保存的是user stack地址，这是一个在较小的内存地址；a1寄存器是我们传递给write的buffer指针，a2是我们传递给write函数的写入字节数。
a0寄存器现在还是个例外，它现在仍然是指向trapframe的指针，而不是保存了的用户数据。

> **交换`SSCRACTH`和`a0`寄存器的值**

接下来，在我们即将返回到用户空间之前，我们交换SSCRATCH寄存器和a0寄存器的值。前面我们看过了SSCRATCH现在的值是系统调用的返回值2，a0寄存器是trapframe的地址。交换完成之后，a0持有的是系统调用的返回值，SSCRATCH持有的是trapframe的地址。之后trapframe的地址会一直保存在SSCRATCH中，直到用户程序执行了另一次trap。现在我们还在kernel中。

> **执行最后一条指令`sret`**

sret是我们在kernel中的最后一条指令，当我执行完这条指令：
程序会切换回user mode
SEPC寄存器的数值会被拷贝到PC寄存器（程序计数器）
重新打开中断
现在我们回到了用户空间。打印PC寄存器，
这是一个较小的指令地址，非常像是在用户内存中。如果我们查看sh.asm，可以看到这个地址是write函数的ret指令地址。
所以，现在我们回到了用户空间，执行完ret指令之后我们就可以从write系统调用返回到Shell中了。或者更严格的说，是从触发了系统调用的write库函数中返回到Shell中。

学生提问：你可以再重复一下在sret过程中，中断会发生什么吗？
Robert教授：sret打开了中断。所以在supervisor mode中的最后一个指令，我们会重新打开中断。用户程序可能会运行很长时间，最好是能在这段时间响应例如磁盘中断。

## 6.4 总结

最后总结一下，系统调用被刻意设计的看起来像是函数调用，但是背后的user/kernel转换比函数调用要复杂的多。之所以这么复杂：

- 很大一部分原因是要保持user/kernel之间的隔离性，内核不能信任来自用户空间的任何内容。
- 另一方面，XV6实现trap的方式比较特殊，XV6并不关心性能。但是通常来说，操作系统的设计人员和CPU设计人员非常关心如何提升trap的效率和速度。必然还有跟

我们这里不一样的方式来实现trap，当你在实现的时候，可以从以下几个问题出发：

- 硬件和软件需要协同工作，你可能需要重新设计XV6，重新设计RISC-V来使得这里的处理流程更加简单，更加快速。
- 另一个需要时刻记住的问题是，恶意软件是否能滥用这里的机制来打破隔离性。

