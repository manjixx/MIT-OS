# Lecture 01. Introduction and Examples(Robert)

- [Lecture 01. Introduction and Examples(Robert)](#lecture-01-introduction-and-examplesrobert)
  - [1.1 课程内容简介](#11-课程内容简介)
  - [1.2 操作系统结构](#12-操作系统结构)
  - [1.3 WHY HARD/INTERESTING?](#13-why-hardinteresting)
  - [1.4 系统调用](#14-系统调用)
    - [1.4.1 read, write, exit 系统调用](#141-read-write-exit-系统调用)
      - [1. read 系统调用](#1-read-系统调用)
      - [2. write 系统调用](#2-write-系统调用)
      - [3. exit 系统调用](#3-exit-系统调用)
    - [1.4.2 open系统调用](#142-open系统调用)
      - [1. open 详解](#1-open-详解)
      - [2. write系统调用](#2-write系统调用)
      - [3. 文件描述符核心机制](#3-文件描述符核心机制)
    - [1.4.3 shell](#143-shell)
      - [1. shell 执行 `ls` 原理](#1-shell-执行-ls-原理)
      - [2. IO 重定向原理](#2-io-重定向原理)
    - [1.4.4 fork](#144-fork)
    - [1.4.5 exec、wait系统调用](#145-execwait系统调用)
      - [1.4.5.1 exec系统调用](#1451-exec系统调用)
      - [1.4.5.2 fork/exec系统调用](#1452-forkexec系统调用)
    - [1.4.6 I/O Redirect](#146-io-redirect)
    - [1.4.3 Shell 工作机制](#143-shell-工作机制)
    - [1.4.4 fork 系统调用](#144-fork-系统调用)
      - [进程克隆演示](#进程克隆演示)
      - [运行现象解析](#运行现象解析)
    - [1.4.5 exec 与 wait 系统调用](#145-exec-与-wait-系统调用)
      - [进程替换模式](#进程替换模式)
      - [执行结果](#执行结果)
      - [系统调用协作模式](#系统调用协作模式)
    - [1.4.6 I/O 重定向实现](#146-io-重定向实现)
      - [文件描述符重定向示例](#文件描述符重定向示例)
      - [关键技术点](#关键技术点)
    - [附录：关键概念对比表](#附录关键概念对比表)

## 1.1 课程内容简介

> **课程目标**

- O/S DESIGN
- HANDS-ON EXP

> **O/S PURPOSE**

- `ABSTRACT H/W`抽象硬件
- `MULTIPLEX`:复用硬件
- `ISOLATION`:多个程序件互不干扰
- `SHARING`
- `SECURITY/PERMISSION SYSTEM/ACCESS CONTROL`
- `GOOD PERFORMANCE`
- `RANGE OF USER`

## 1.2 操作系统结构

重点内容：

- `Kernel`
- `Kernal`和用户空间（`USERSPACE`）程序间的接口
- `Kernel`内软件的架构

> **O/S ORAGNIZATION（操作系统结构）**

- `USERSPACE`:用户空间，运行各种各样的应用程序，如文本编辑器vi，C编译器(cc)
- `KERNAL`: **计算机资源守护者**。当启动计算机时，`Kernel`总是第一个被启动且只有一个。它维护数据管理每一个用户空间进程。`Kernel`同时还维护了大量的数据结构来实现管理各种各样的硬件资源，以供用户空间的程序使用。`Kernel`同时还有大量内置的服务。**`Kernel` 组成如下：**
  - `FS（文件系统）`:文件系统通常有一些逻辑分区。目前而言，文件系统的作用是**管理文件内容并找出文件在磁盘中的具体位置**。文件系统还维护了一个独立的命名空间，其中每个文件都有文件名，并且命名空间中有一个层级的目录，每个目录包含了一些文件。所有这些都被文件系统所管理。
  - `PROCESS（进程管理系统）`:每一个用户空间程序都被称为一个进程，它们有自己的内存，同时会共享的CPU时间。
  - `MEM,ALLCO（内存分配管理）`:不同的进程需要不同数量的内存，Kernel会复用内存、划分内存，并为所有的进程分配内存。    - `ACCESS CTL（读取控制）`:当一个进程想要使用某些资源时，如读取磁盘中的数据，使用某些内存，`Kernel`中的`Access Control`会决定是否允许这样的操作。
- 硬件底层资源：CPU、RAM、DISK、NET

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MH_0vtckm44OL-Ry80u%2F-MHfa6qidpjQGh_XpuRm%2Fimage.png?alt=media&token=13f0cc46-16b5-4e7e-bfc6-498ff3c6449a)

> **API-KERNAL**

应用程序通过**系统调用（`System Call`）** 来访问`Kernel`。系统调用与程序中的函数调用看起来是一样的，但区别是系统调用是由`Kernel`实现的`API`，实际会运行到系统内核中，并执行内核中对于系统调用的实现。

- 🙋🌰 **`fd = open("out",1)`：打开文件**
  - `open`是一个系统调用，`open`会跳转到`Kernel`，`Kernel`可以获取到`open`参数，执行一些实现`open`的`kernel`代码；**返回一个文件描述符对象。**
  - `fd`全称为`file descriptor`,应用程序可以使用这个文件描述符作为`handle`，来表示相应打开的文件。*（注：handle（句柄） 是一种抽象概念，表示对某个系统资源（如文件、网络连接、内存块、图形对象等）的引用或标识符。它并不是资源本身，而是应用程序通过操作系统或底层框架间接操作资源的一种方式。）*
- 🙋🌰 **`write(fd,"hello\n",6)`：向文件写入数据**
  - **功能**：告诉内核，将内存中这个地址起始的6个字节数据写入到fd对应的文件中。
  - `param1`：由`open`返回的文件描述符。
  - `param2`：指向要写入数据的指针（数据通常是`char`型序列）,即内存中的地址。
  - `param3`：想要写入字符的数量
- 🙋🌰 `pid = fork();`
  - **功能：** 创建一个调用进程完全相同的新进程，返回新进程的`process ID/pid`

> **Q：系统调用和程序里的函数的区别？**

**Robert教授**：**`Kernel`的代码总是有特殊的权限**。当机器启动`Kernel`时，`Kernel`会有特殊的权限能直接访问各种各样的硬件，例如磁盘。而普通的用户程序是没有办法直接访问这些硬件的。所以，当你执行一个普通的函数调用时，你所调用的函数并没有对于硬件的特殊权限。然而，如果你触发系统调用到内核中，内核中的具体实现会具有这些特殊的权限，这样就能修改敏感的和被保护的硬件资源，比如访问硬件磁盘。我们之后会介绍更多有关的细节。

## 1.3 WHY HARD/INTERESTING?
  
- `UNFORGNING`:内核编程环境较为复杂困难
- `TENSIONS`
  - EFFICIENT vs ABSTRACT
  - POWERFUL vs SIMPLE
  - FLEXIBLE vs SECURITY
- `INTERACTION`：系统提供服务间的交互
  - `fd = open()`与 `pid = fork()`

> **课程环境**

- `OS-XV6`：类unix操作系统
- `RISC-V`：微处理器，`RISC-V`指令集
- `QEMU`：硬件仿真，模拟`RISC-V`

```zsh
make clean
make qemu
```

## 1.4 系统调用

XV6书籍的第二章有一个表格，列举了所有的系统调用的参数和返回值。

```zsh
cd xv6-riscv

// 清除编译
make clean

// 编译
make qemu
```

### 1.4.1 read, write, exit 系统调用

```c
// copy.c: 实现输入到输出的逐字节拷贝
#include "kernel/types.h"
#include "user/user.h"

int main()
{
    char buf[64];  // 在栈空间分配64字节缓冲区
    
    while(1) {
        int n = read(0, buf, sizeof(buf));  // 系统调用1: read
        if(n <= 0) break;                  // 处理终止条件
        write(1, buf, n);                   // 系统调用2: write
    }
    exit(0);                                // 系统调用3: exit
}
```

#### 1. read 系统调用

`read` 是一个从文件描述符中读取数据的系统调用，它接受三个参数：

- **文件描述符（fd）**：

  - 第一个参数 `0` 表示标准输入（console）。
  - 在类 Unix 系统中，文件描述符 `0`、`1`、`2` 分别对应**标准输入、标准输出和标准错误输出**，遵循 UNIX 风格。

- **缓冲区指针**：

  - 第二个参数 `buf` 是一个字符数组指针，指向预分配的内存区域。
  - 在本程序中，`buf` 是在栈上分配的 64 字节大小的缓冲区，`read` 会将读取的数据存储在该缓冲区中。

- **读取的最大字节数**：

  - 第三个参数 `sizeof(buf)` 表示最多读取 64 字节的数据。

`read` 返回值：

- **大于 0**：表示读取的字节数。
- **等于 0**：表示已到达文件末尾（EOF）。
- **小于 0**：表示发生错误（如无效文件描述符）。

#### 2. write 系统调用

`write` 是一个向文件描述符中写入数据的系统调用。

- 第一个参数 `1` 表示标准输出（console）。
- 第二个参数是指向待写入数据的缓冲区指针（`buf`）。
- 第三个参数是写入的数据长度（`n`，即 read 读取的字节数）。

#### 3. exit 系统调用

`exit` 用于终止当前进程。参数 `0` 表示正常退出。

> ⚠️ **说明**

- `copy`只关心字节流的复制，不关心数据的格式。它是纯粹的字节流复制工具。
- 操作系统只处理数据为 8 位字节流，数据如何解析由应用程序决定。

> **Q：如果 read 的第三个参数设置为 `1 + sizeof(buf)` 会怎样？**

**答：** 如果将第三个参数设为 65 字节，系统会尝试将 65 字节数据写入缓冲区。然而缓冲区只有 64 字节，这种写入将导致缓冲区溢出，可能会覆盖其他栈上的数据，从而引发程序崩溃或产生异常行为。防御策略：严格保持 `count ≤ sizeof(buf)`，C语言不自动检查边界，需程序员显式保证

### 1.4.2 open系统调用

最直接的创建文件描述符的方法是open系统调用。

> **open.c**

创建一个叫做`output.txt`的新文件，并向它写入一些数据，最后退出。我们看不到任何输出，因为它只是向打开的文件中写入数据。

```c
// open.c: 创建文件并写入数据
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main() {
    int fd = open("output.txt", O_WRONLY | O_CREATE);  // 系统调用1: open
    write(fd, "ooo\n", 4);                             // 系统调用2: write
    exit(0);                                           // 系统调用3: exit
}
```

```zsh
$ cat output.txt
ooo
```

#### 1. open 详解

`open` 参数解析：

- 参数1：目标文件名（output.txt）
- 参数2：标志位组合：`O_WRONLY`（只写模式） + `O_CREATE`（不存在时创建文件）

返回值：

- 新分配的文件描述符（通常为一个小的数字 3,4或其他数字，0-2已被标准流占用）

#### 2. write系统调用

`write` 参数解析：

- 参数1：文件描述符
- 参数2：数据的指针
- 参数3：要写入的字节数。数据被写入到了文件描述符对应的文件中。注意与上述`copy`的区别，`copy`中相当于将数据写入控制台

#### 3. 文件描述符核心机制

1. **文件描述符本质：内核维护的映射表**：
   - 每个进程拥有独立文件描述符表
   - 表的key为文件描述符，表项存储实际文件对象的指针（包含读写位置、访问模式等信息）

2. **跨进程独立性**：每个进程都有自己独立的文件描述符空间，两个进程打开文件时，可能会得到相同文件描述符，但内核为每个进程都维护了一个独立的文件描述符空间，**所以相同数字的文件描述符可能会对应到不同的文件。**
   - 进程A的fd3可能指向文件X
   - 进程B的fd3可能指向网络套接字
  
```sh
+----------------+       +-----------------+
| 进程A          |       | 进程B           |
|   fd0 → stdin  |       |   fd0 → stdin   |
|   fd1 → stdout |       |   fd1 → fileX   |
|   fd3 → fileY ←|———————|→ fd2 → fileY    |
+----------------+       +-----------------+
```

### 1.4.3 shell

`Shell` 是一种命令行接口，专为 `Unix` 系统管理而设计。它提供了多种工具来管理文件、编写程序和脚本。其本质为：

- **命令解释器**：将用户输入转化为程序执行
- **进程管理器**：通过`fork`+`exec`执行目标程序
- **环境定制工具**：管理环境变量与I/O重定向

#### 1. shell 执行 `ls` 原理

Shell最常见的功能：输入指定内容，Shell运行相应的程序。当输入ls时，实际的意义是要求**Shell运行名为ls的程序**，文件系统中会有一个文件名为ls，这个文件中包含了一些计算机指令，Shell 会运行位于文件ls内的这些计算机指令。

- `Shell`在文件系统中定位名为`ls`的二进制文件
- 创建子进程加载该文件指令
- 执行结果显示在控制台（除非进行重定向）

#### 2. IO 重定向原理

`Shell` 提供了标准输入（`stdin`）、标准输出（`stdout`）和标准错误（`stderr`）三种数据流，并**支持通过重定向操作将它们定向到文件或其他程序**。

> **输出重定向**

```bash
# 标准输出重定向 >
$ ls > out.txt         # 将ls输出写入文件

```

**实现步骤**：

- Shell通过`fork`创建子进程
- 子进程修改文件描述符映射（如将fd1重定向到文件）
  - 子进程关闭标准输出`fd1`
  - 打开目标文件获得新的`fd1`
- 子进程通过`exec`执行目标程序`ls`，输出自然流向文件

```zsh
$ ls > out
$ cat out
.              1 1 1024
..             1 1 1024
README         2 2 2226
cat            2 3 24232
copy           2 4 22632
echo           2 5 23048
forktest       2 6 13272
grep           2 7 27528
init           2 8 23792
kill           2 9 22992
ln             2 10 22848
ls             2 11 26424
mkdir          2 12 23152
open           2 13 22480
rm             2 14 23128
sh             2 15 41952
stressfs       2 16 23984
usertests      2 17 157032
grind          2 18 38160
wc             2 19 25312
zombie         2 20 22384
console        3 21 0
output.txt     2 22 4
out            2 23 578
```

**上述指令的实际意义是:** 要求Shell运行ls命令，并**将输出重定向到一个叫做out的文件中**。这里执行完成之后在控制台看不到任何的输出，因为输出都送到了`out`文件。可以通过`cat`指令读取一个文件，并显示文件的内容。

> **输入重定向**

```sh
# 标准输入重定向 <
$ grep "key" < in.txt  # 从文件读取输入
```

**实现机制**

- Shell通过`fork`创建子进程
- 子进程修改文件描述符映射
  - 子进程关闭标准输入`fd=0`
  - 打开`log.txt`获得新`fd=0`
- 子进程通过`exec`执行目标程序`grep从新输入源读取数据

```zsh
$ grep x # 搜索输入中包含x的行

$ grep x < out # 将输入定向到文件out，查看out中的x
output.txt     2 22 4

$ grep l < out # 将输入定向到文件out，查看out中的l
kill           2 9 22992
ln             2 10 22848
ls             2 11 2642
```

> **Q：编译器如何处理系统调用？生成的汇编语言是不是会调用一些由操作系统定义的代码段？**

Robert教授：有一个特殊的RISC-V指令，程序可以调用这个指令，并将控制权交给内核。所以，实际上当运行C语言并执行`open`或者`write`的系统调用时，从技术上来说，`open`是一个 C 函数，但是这个函数内的指令实际上是机器指令，即调用的`open`函数并不是一个`C`语言函数，而是由汇编语言实现。组成这个系统调用的汇编语言实际上在`RISC-V`中被称为`ecall`。这个特殊的指令将控制权转给内核。之后内核检查进程的内存和寄存器，并确定相应的参数。

A：（Robert教授）系统调用通过特殊机器指令实现：

- 程序中的系统调用（如 open、write）在 C 语言中表现为函数调用。但其本质为C标准库函数封装ecall指令，编译器将这些系统调用转换为底层的汇编代码.

- 执行ecall触发硬件中断（模式切换至内核态），将控制权交给操作系统内核。

- 内核通过寄存器获取调用参数，并根据传递的参数执行相应的操作。

- 执行完成后返回用户态继续运行

```mermaid
graph TD
    A[用户程序调用C库函数 open/write] --> B[C库函数内部处理]
    
    subgraph 用户空间
    B --> C[准备系统调用参数]
    C --> D[将系统调用号存入a7寄存器]
    D --> E[参数存入a0-a6寄存器]
    E --> F[执行ecall指令]
    end

    subgraph 内核空间
    F --> G{触发陷阱机制}
    G --> H[硬件自动完成以下操作]
    H --> H1[切换到内核模式]
    H1 --> H2[保存pc到mepc寄存器]
    H2 --> H3[跳转到mtvec指定的陷阱向量]
    
    H3 --> I[保存用户上下文]
    I --> J[读取a7寄存器获取系统调用号]
    J --> K[检查参数合法性]
    
    K --> L{参数有效?}
    L --> |Yes| M[执行对应系统调用服务例程]
    L --> |No| N[设置错误码]
    
    M --> O[操作硬件/文件系统等]
    O --> P[返回结果到a0寄存器]
    N --> Q[设置a0为-1]
    
    P & Q --> R[恢复用户上下文]
    R --> S[执行mret指令]
    end

    subgraph 返回用户空间
    S --> T[硬件自动完成以下操作]
    T --> T1[切换回用户模式]
    T1 --> T2[从mepc恢复pc]
    T2 --> U[继续执行用户程序]
    end

    style A fill:#c0ffc0,stroke:#333
    style B fill:#c0ffc0,stroke:#333
    style C fill:#c0ffc0,stroke:#333
    style D fill:#c0ffc0,stroke:#333
    style E fill:#c0ffc0,stroke:#333
    style F fill:#ffcccb,stroke:#333
    style G fill:#ffeb99,stroke:#333
    style H fill:#ffeb99,stroke:#333
    style H1 fill:#ffeb99,stroke:#333
    style H2 fill:#ffeb99,stroke:#333
    style H3 fill:#ffeb99,stroke:#333
    style I fill:#b0e0e6,stroke:#333
    style J fill:#b0e0e6,stroke:#333
    style K fill:#b0e0e6,stroke:#333
    style L fill:#ffb347,stroke:#333
    style M fill:#98fb98,stroke:#333
    style N fill:#ff6666,stroke:#333
    style O fill:#98fb98,stroke:#333
    style P fill:#98fb98,stroke:#333
    style Q fill:#ff6666,stroke:#333
    style R fill:#b0e0e6,stroke:#333
    style S fill:#ffeb99,stroke:#333
    style T fill:#ffeb99,stroke:#333
    style T1 fill:#ffeb99,stroke:#333
    style T2 fill:#ffeb99,stroke:#333
    style U fill:#c0ffc0,stroke:#333
```


```
用户空间                             内核空间
+-------------------+ ecall        +-------------------+
| 用户模式          | -----------> | 内核模式          |
| 执行ecall指令      |              | 异常处理程序       |
| PC=mepc           | <----------- | 执行mret指令       |
+-------------------+              +-------------------+
```

> **总结**

| 操作          | 文件描述符变化            | 进程影响范围  |
|---------------|--------------------------|-------------|
| `>` 输出重定向 | fd1指向目标文件          | 仅子进程     |
| `<` 输入重定向 | fd0指向源文件            | 仅子进程     |
| `2>` 错误重定向| fd2指向目标文件          | 仅子进程     |
| 管道 `|`      | 创建匿名管道连接fd0/fd1  | 多个子进程   |


### 1.4.4 fork

fork会创建一个新的进程，下面是使用fork的一个简单用例。

> **fork**

```c
// fork.c: create a new process

# include "kernel/types.h"
# include "user/user.h"

int main(){
    int pid;

    pid = fork();

    printf("fork() return %d \n",pid);

    if(pid == 0){
        printf("child\n");
    }else{
        printf("parent\n");
    }
    exit(0);
}
```

- fork会**拷贝当前进程的内存**，并创建一个新的进程，这里的内存包含了进程的指令和数据。之后，我们就有了两个拥有完全一样内存的进程。
- **fork系统调用在两个进程中都会返回**
  - 在原始的进程中，fork系统调用会返回大于0的整数，这个是新创建进程的ID。
  - 在新创建的进程中，fork系统调用会返回0。
- 通过校验pid，如果pid等于0，那么这必然是子进程。调用进程通常称为父进程，父进程看到的pid必然大于0。所以父进程会打印“parent”，子进程会打印“child”。之后两个进程都会退出。

运行fork 之后如下为输出

```zsh
$ fork
ffoorrkk(() )r erteutrnur n4  
0pa re
nt
child
```

上述输出像是乱码，实际状况是：fork系统调用之后，**两个进程都在同时运行**，QEMU实际上是在模拟多核处理器，所以这两个进程实际上就是同时在运行。

当这两个进程在输出的时候，它们会同时一个字节一个字节的输出，两个进程的输出交织在一起，所以你可以看到两个f，两个o等等。

**fork创建了一个新的进程。** 当我们在Shell中运行东西的时候，**Shell实际上会创建一个新的进程来运行我们输入的每一个指令**。

所以，当我输入ls时，我们需要Shell通过fork创建一个进程来运行ls，这里需要某种方式来让这个新的进程来运行ls程序中的指令，加载名为ls的文件中的指令（也就是后面的exec系统调用)。

> **Q:fork产生的子进程是不是总是与父进程是一样的？它们有可能不一样吗？**

Robert教授：**在XV6中**，除了fork的返回值，两个进程是一样的。两个进程的指令是一样的，数据是一样的，栈是一样的，同时，两个进程又有各自独立的地址空间，它们都认为自己的内存从0开始增长，但是在XV6中，父子进程除了fork的返回值，其他都是一样的。文件描述符的表单也从父进程拷贝到子进程。所以如果父进程打开了一个文件，子进程可以看到同一个文件描述符，尽管子进程看到的是一个文件描述符的表单的拷贝。除了拷贝内存以外，fork还会拷贝文件描述符表单这一点还挺重要的，我们接下来会看到。

在一个更加复杂的操作系统，有一些细节我们现在并不关心，这些细节偶尔会导致父子进程不一致。

### 1.4.5 exec、wait系统调用

#### 1.4.5.1 exec系统调用

> **exec代码**
echo是一个非常简单的命令，它接收任何你传递给它的输入，并将输入写到输出。

```c
// exec.c: replace a process with a executable file

# include "kernel/types.h"
# include "user/user.h"

int main(){
    char *argv[] = {"echo","this","is","echo",0};

    exec("echo",argv);

    printf("exec failed!\n");

    exit(0);
}
```

**上述代码会执行exec系统调用**，这个系统调用**会从指定的文件中读取并加载指令**，**并替代当前调用进程的指令**。从某种程度上来说，这样*相当于丢弃了调用进程的内存，并开始执行新加载的指令*

`exec("echo",argv);`:该行会有如下效果

- 操作系统从名为`echo`的文件中**加载指令到当前的进程中**，并替换了当前进程的内存
- **之后开始执行这些新加载的指令**。
- **传入命令行参数**:exec允许你传入一个命令行参数的数组，这里就是一个C语言中的指针数组，在上面代码的第10行设置好了一个字符指针的数组，这里的字符指针本质就是一个字符串（string）。

综上所述此处等价于运行`echo`命令，并携带"this is echo"这三个参数。所以当我们运行exec文件时:

```zsh
$ exec
this is echo
```

可以看到“this is echo”的输出。即此处运行了exec程序，但exec程序实际上调用了exec系统调用，并用echo指令来代替自己，所以这里是**echo命令在产生输出。**

> **exec注意事项**

- **exec系统调用会保留当前的文件描述符表单**。所以任何在exec系统调用之前的文件描述符，例如0，1，2等。它们在新的程序中表示相同的东西。
- **一般而言exec系统调用不会返回**，因为exec会完全替换当前进程的内存，相当于当前进程不复存在了，所以exec系统调用已经没有地方能返回了。
- **exec系统调用只会当出错时才会返回**，因为某些错误会阻止操作系统为你运行文件中的指令，例如程序文件根本不存在，因为exec系统调用不能找到文件，exec会返回-1来表示：出错了，我找不到文件。

> **总结**

以上代码就是实现：一个程序利用另外一个程序代替自己。但实际应用中，我们不希望调用一个程序之后丢失控制权，对于这种还希望能拿回控制权的场景，可以先执行fork系统调用，然后在子进程中调用exec。

#### 1.4.5.2 fork/exec系统调用

> **fork/exec成功调用子进程代码**

```c
// forkexec.c:fork then exec

# include "kernel/types.h"
# include "user/user.h"

int main(){
    int pid,status;

    pid = fork();

    if(pid == 0){
        char *argv[] = {"echo","THIS","IS","ECHO",0};
        exec("echo",argv);
        printf("exec failed!\n");
        exit(1);
    }else{
        printf("parent waiting\n");
        wait(&status);
        printf("the child exited with status %d\n",status);
    }
    exit(0);
}
```

- 在上述代码中，首先调用了`fork`
- 子进程会利用`echo`命令代替自己，echo执行完成后就退出。
- 父进程会重新获得控制权，fork在父进程中返回大于0的值，父进程会继续在19行执行

运行输出:

```zsh
$ forkexec
parent waiting
THIS IS ECHO
the child exited with status 0
```

> **wait系统调用**

Unix提供了一个wait系统调用，**wait会等待之前创建的子进程退出**。

当我们在Shell命令行执行一个指令时，一般会希望Shell等待指令执行完成。所以wait系统调用，使得父进程可以等待任何一个子进程返回。

**wait参数 status**：是一种父进程与子进程之间的一种通信方式，他可以让退出的子进程以一个整数（32bit的数据）与等待的父进程通信。

所以在if语句中的最后一行`exit`的参数是1，操作系统会将1从退出的子进程传递到`wait(&status)`，即父进程的等待处。

`&status`，是将status对应的地址传递给内核，内核会向这个地址写入子进程向exit传入的参数。

> **UNIX中exit的传参风格**

- 如果一个程序成功退出，那么exit的参数即为0；
- 如果一个程序出现错误，那么exit的参数为1。

> **fork/exec调用子进程代码失败示例**

```c
// forkexec.c:fork then exec

# include "kernel/types.h"
# include "user/user.h"

int main(){
    int pid,status;

    pid = fork();

    if(pid == 0){
        char *argv[] = {"echo","THIS","IS","ECHO",0};
        // 调用了不存在的系统调用
        exec("sdfaddfaecho",argv);
        printf("exec failed!\n");
        exit(1);
    }else{
        printf("parent waiting\n");
        wait(&status);
        printf("the child exited with status %d\n",status);
    }
    exit(0);
}
```

运行输出：

```zsh
$ forkexec
parent waiting
exec failed!
the child exited with status 1
```

> **总结**

上述代码中我们使用了一个常用写法，即先调用`fork`,然后在子进程中调用`exec`。

这里需要注意的一点是，fork首先拷贝了整个父进程，之后exec将这个拷贝全部丢弃，某种程度上而言，这里的拷贝操作是浪费的。在大型程序中比较明显。

在后续课程中会针对这个问题实现优化，比如`copy-on-write fork`,这种方式会消除fork的几乎所有的明显的低效，而只拷贝执行exec所需要的内存，这里需要很多涉及到虚拟内存系统的技巧。

同时可以构建一个fork，对于内存实行lazy拷贝，通常来说fork之后立刻是exec，这样你就不用实际的拷贝，因为子进程实际上并没有使用大部分的内存。

### 1.4.6 I/O Redirect

Shell提供了方便的重定向工具，如果运行下述指令则表示:

- Shell会将echo的输出送到文件out
- 之后运行cat指令，并将out文件作为输入

可以看到保存在out文件中的内容就是echo指令的输出。

```zsh
$ echo hello > out
$ cat < out
hello
```

> **redirect.c**

```c
// redirect.c: run a commond with output redirected

# include "kernel/types.h"
# include "user/user.h"
# include "kernel/fcntl.h"


int main(){
    int pid;

    pid = fork();

    if(pid == 0){
        close(1);
        open("output.txt",O_WRONLY|O_CREATE);

        char *argv[] = {"echo","this","is","redirected","echo",0};
        exec("echo",argv);
        prinf("exec failed!\n");
        exit(1);
    }else{
        wait((int *)0);
    }
    exit(0);
}
```

> **代码分析**

Shell之所以具备上述功能，是因为如上述代码所示:

- Shell首先会fork一个子进程，然后在子进程中Shell改变了文件描述符(在这里为"ouput.txt")，默认情况下为1，即console的输出文件符。该方法是Unix中的常见的用来重定向指令的输入输出的方法，这种方法同时又不会影响父进程的输入输出。因为我们不会想要重定向Shell的输出，我们只想重定向子进程的输出。

- 代码`close(1)`的意义是:我们希望文件描述符1指向一个其他的位置，即我们这里关闭原本指向console输出的文件描述符1

- 而因为刚关闭文件描述符1，所以代码`open("output.txt",O_WRONLY|O_CREATE);`一定返回1，因为`open`会返回当前进程未使用的最小文件描述符。而文件描述符`0`对应着console输入，`1`是我们刚刚关闭的最小文件描述符，所以open一定可以返回1，即执行完上述代码后文件描述符`1`与文件`output.txt`关联

- 之后我们执行exec(echo)，echo会输出到文件描述符1，也就是文件output.txt。但echo不清楚发生了什么。

> **运行输出**

```zsh
$ redirect
$ cat < output.txt
this is redirected echo
```

### 1.4.3 Shell 工作机制

### 1.4.4 fork 系统调用

#### 进程克隆演示

```c
// fork.c: 创建进程副本
#include "kernel/types.h"
#include "user/user.h"

int main() {
    int pid = fork();  // 系统调用1: fork
    
    printf("fork() returns %d\n", pid);
    
    if(pid == 0) {
        printf("Child process\n");
    } else {
        printf("Parent process\n");
    }
    exit(0);
}
```

#### 运行现象解析

```bash
$ fork
fork() returns 4
fork() returns 0
Parent process
Child process
```

**关键机制**：

- **写时复制（COW）**：父子进程共享物理内存，仅在修改时创建副本
- **独立地址空间**：每个进程拥有从0开始的虚拟地址空间

---

### 1.4.5 exec 与 wait 系统调用

#### 进程替换模式

```c
// execwait.c: 进程替换与同步
#include "kernel/types.h"
#include "user/user.h"

int main() {
    int pid, status;
    
    pid = fork();
    
    if(pid == 0) {
        char *argv[] = {"echo", "PROCESS REPLACEMENT", 0};
        exec("echo", argv);                 // 系统调用1: exec
        printf("exec failed!\n");           // 仅出错时执行
        exit(1);
    } else {
        wait(&status);                      // 系统调用2: wait
        printf("Exit code: %d\n", status);
    }
    exit(0);
}
```

#### 执行结果

```bash
$ execwait
PROCESS REPLACEMENT
Exit code: 0
```

---

#### 系统调用协作模式

1. **经典组合**：
   - `fork`创建子进程
   - `exec`加载新程序
   - `wait`同步进程状态

2. **状态传递机制**：
   - 子进程通过`exit(code)`返回状态码
   - 父进程通过`wait(&status)`获取状态值

---

### 1.4.6 I/O 重定向实现

#### 文件描述符重定向示例

```c
// redirect.c: 实现输出重定向
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main() {
    int pid = fork();
    
    if(pid == 0) {
        close(1);                              // 关闭标准输出
        open("output.txt", O_WRONLY|O_CREATE); // 新打开的fd会复用1
        
        char *argv[] = {"echo", "REDIRECT TEST", 0};
        exec("echo", argv);
        exit(1);
    } else {
        wait(0);
    }
    exit(0);
}
```

#### 关键技术点

1. **描述符复用规则**：
   - 总是分配当前最小的可用描述符
   - `close(1)`后新打开的文件的描述符必定为1

2. **进程隔离性**：
   - 子进程修改的描述符不影响父进程
   - Shell通过这种方式实现命令级重定向

---

### 附录：关键概念对比表

| 系统调用 | 作用                          | 典型返回值              | 常见使用场景               |
|----------|-------------------------------|-------------------------|---------------------------|
| fork     | 创建进程副本                  | 父进程返回子进程PID<br>子进程返回0 | 并发执行                  |
| exec     | 替换当前进程映像              | 成功无返回<br>失败返回-1 | 加载新程序                |
| wait     | 等待子进程终止                | 返回子进程PID           | 进程同步                  |
| open     | 打开/创建文件                 | 文件描述符（≥3）        | 文件操作                  |
| close    | 关闭文件描述符                | 0成功，-1失败           | 资源释放                  |
