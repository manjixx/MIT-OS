# 课程计划与总结

## 一、课程计划

### Lec 03 OS design

#### 课前准备

- 阅读第2章 Operating system organization
- 阅读 `xv6 code`: `kernel/proc.h, kernel/defs.h, kernel/entry.S, kernel/main.c, user/initcode.S, user/init.c, and skim kernel/proc.c and kernel/exec.c`

#### 实验

Lab syscall: System calls

### Lec 04 page tables

#### 课前准备

- 阅读第 3 章 Page Tables
- 阅读 xv code：`kernel/memlayout.h, kernel/vm.c, kernel/kalloc.c, kernel/riscv.h, and kernel/exec.c`

## 二、课程总结

下面是对 MIT 6.1810 (前身 6.S081/6.828)《Operating System Engineering》的课程架构、教学逻辑，以及各章知识重点的宏观总结。

---

### 2.1  🎯 课程定位与教学理念

- 面向本科生的中级操作系统课程，由 MIT 教授 Robert Morris、Frans Kaashoek 等主讲 ([hackway.org][1])。
- 教学突出“学以致用”：课程结构包含讲授 + 实验，实验主要基于多处理器 RISC‑V 架构的 xv6 操作系统 ([pdos.csail.mit.edu][2])。
- 目标是使学生从系统调用、页表、中断、文件系统等核心模块动手构建操作系统，加深理解系统设计原理。

---

### 2.2 🧩 课程架构总览

课程由讲座和 10 个 lab 实验组成：

| 模块类型                             | 内容                                      |
| -------------------------------- | --------------------------------------- |
| **Lectures**                     | 系统理论+代码分析+设计原则，覆盖从系统调用到网络的核心主题          |
| **Labs (70%)**                   | 实战实验，逐步构建 xv6 的关键系统组件 ([github.com][3]) |
| **Check-offs (20%)**             | 教师面谈深入讨论实验实现细节                          |
| **Homework/Participation (10%)** | 阅读思考题、课程互动                              |

---

### 2.3 📚 Labs 与知识结构篇章

1. **Lab 1 – xv6 & Unix utilities**

   - 熟悉 xv6 构建流程，编写用户命令（如 `sleep`, `pingpong`, `primes`）。
   - **重点**：用户态程序、系统调用接口、Makefile 构建机制 ([github.com][3], [dianhsu.com][4])。

2. **Lab 2 – System Calls**

   - 实现若干系统调用（如 `fork`, `exit`, `write`）。
   - **重点**：syscall 路径、用户态 → 内核态切换、trap 框架 ([github.com][3])。

3. **Lab 3 – Page Tables**

   - 探索内存分页、虚拟地址映射机制。
   - **重点**：多层页表构造、地址转换、中断异常的内存保护策略 ([github.com][3])。

4. **Lab 4 – Traps**

   - 实现异常、系统调用、外设中断处理机制。
   - **重点**：RISC‑V 异常向量、`ecall`、内核 trap 处理流程 ([github.com][3], [juejin.cn][5])。

5. **Lab 5 – Copy-on-Write Fork**

   - 为 `fork()` 实现写时复制 (COW)。
   - **重点**：页错误机制、写保护、内存共享与延迟复制 ([github.com][3])。

6. **Lab 6 – Multithreading**

   - 支持轻量级线程的创建与并发调度。
   - **重点**：线程切换、栈空间管理、互斥机制 ([github.com][3])。

7. **Lab 7 – Network Driver**

   - 实现 E1000 网卡驱动，支持帧发送与接收。
   - **重点**：底层驱动、网卡 DMA 操作、网络协议接入 ([github.com][3])。

8. **Lab 8 – Locks**

   - 构建 xv6 的同步原语（spinlock、sleeplock 等）。
   - **重点**：原子操作、死锁概念、锁的性能优化 ([juejin.cn][5])。

9. **Lab 9 – File System**

   - 为 xv6 实现/优化简单文件系统。
   - **重点**：磁盘块管理、inode、FS 元数据、一致性维护 ([juejin.cn][5])。

10. **Lab 10 – mmap**

    - 支持内存映射文件。
    - **重点**：虚内存管理、页表扩展、文件与内存映射交互 ([juejin.cn][5], [ocw.mit.edu][6])。

---

### 2.4 🔗 微观知识点相互串联

- **系统调用 & Trap**（Lab2、Lab4）：打通用户态与内核态边界，为内核构建入口通道。
- **地址空间 & 页表**（Lab3、Lab5、Lab10）：设计内存访问结构、写时复制共享机制与映射机制。
- **并发执行**（Lab6、Lab8）：实现多线程并发环境，并保障同步正确性。
- **外设与存储管理**（Lab7、Lab9）：从驱动芯片到文件系统，完成数据的可靠存储与传输。

---

### 2.5 🧭 总结思路

MIT 6.1810 的课程设计以 \*\*“动手搭建一个简化的多核操作系统”为主线。\*\*学生逐步理解从 CPU 请求 (trap)、内核处理、到资源管理与硬件交互，再回到用户态的完整流程：

1. 用户请求 → syscall → trap → 内核实现；
2. 虚拟内存与物理内存在页面级的映射、错误以及共享机制；
3. 线程并发与同步确保多任务正确执行；
4. 硬件驱动 + 文件系统协同工作，保障 I/O 间的协同；
5. 最终实现 mmap 提供内存和文件的透明整合；

---

这门课程不仅让你**掌握操作系统内部机制的实现原理**，还通过 xv6 框架和 RISC‑V 架构，给与可操作、可验证、可调试的系统学习体验。如果你有兴趣进一步详细拆解某个实验或机制，我可以继续帮你深入分析。

[1]: https://www.hackway.org/docs/cs/sophomore/operating/cs6828/?utm_source=chatgpt.com "MIT 6.1810 操作系统工程 ⭐️ - HackWay"
[2]: https://pdos.csail.mit.edu/6.1810/2022/overview.html?utm_source=chatgpt.com "6.1810 / Fall 2022 - Massachusetts Institute of Technology"
[3]: https://github.com/sakura-ysy/MIT6.1810?utm_source=chatgpt.com "GitHub - sakura-ysy/MIT6.1810: MIT6.1810 OS公开课个人 ..."
[4]: https://www.dianhsu.com/2022/09/11/6-1810/?utm_source=chatgpt.com "MIT 6.1810 Operating System Engineering - dianhsuのblog"
[5]: https://juejin.cn/column/7276350321094082614?utm_source=chatgpt.com "MIT 6.1810 操作系统 Lab全解 - Cubane的专栏 - 掘金"
[6]: https://ocw.mit.edu/courses/6-1810-operating-system-engineering-fall-2023/?utm_source=chatgpt.com "Operating System Engineering | Electrical ... - MIT OpenCourseWare"
