# lab1: util

## sleep

解题思路：

- 调用 `atoi()` 将 字符参数转换为整型
- 直接调用系统调用`sleep()`
- 注意在 `Makefile` 中添加 `sleep`

```c
#include "kernel/types.h"   // for type int,
#include "user/user.h"      // for sleep(), atoi(), exit()

/*
 * Function: sleep
 * 
 *  A simple program that takes one command-line argument (a number)
 *  and calls the sleep() function to pause execution for the given time.
 *
 *  argc: argument count (should be 2: program name + time)
 *  argv: argument vector (argv[1] is the time in seconds)
 *
 *  returns: 0 on success, 1 on error (wrong number of arguments)
 */
int main(int argc, const char* argv[])
{   
    // check if argument time is provided
    if(argc != 2) {   // param error
        fprintf(2, "usage: sleep <time>\n");
        exit(1);    // exit with error code
    }

    // convert the string argument to an integer
    int time = atoi(argv[1]);
    // system call
    sleep(time);
    // exit program sucessfully
    exit(0);
}
```

## ping-pong

> **解题思路：**

| 过程      | 父进程                  | 子进程                  |
| ------- | -------------------- | -------------------- |
| 创建管道    | `fd_c2p`, `fd_p2c`   | 同上                   |
| 写入 ping | 写 `fd_p2c[WR]`       |                      |
| 读取 ping |                      | 读 `fd_p2c[RD]`       |
| 写入 pong |                      | 写 `fd_c2p[WR]`       |
| 读取 pong | 读 `fd_c2p[RD]`       |                      |
| 打印信息    | `pid: received pong` | `pid: received ping` |

> **注意：**

- 管道是用于单向通信的，因为这个 lab 需要父进程和子进程互相通信，所以应该创建两个管道。
- 使用两个管道进行父子进程通信，需要注意的是如果管道的写端没有close，那么管道中数据为空时对管道的读取将会阻塞。因此对于不需要的管道描述符，要尽可能早的关闭。

> **详细实现**

```c
#include "kernel/types.h"
#include "user/user.h"

#define RD 0  // pipe read end (for reading)
#define WR 1  // pipe write end (for writing)

int main(int argc, const char* argv[])
{
    // Check if argument count is correct
    if (argc != 1) {
        fprintf(2, "usage: pingpong <no parameter>\n");
        exit(1);  // exit with error code
    }

    int exit_status = 0;  // program exit status (0 = OK, 1 = error)
    char buf = 'P';       // the message to transfer (just one char for demo)

    int fd_c2p[2];        // pipe for child to parent
    int fd_p2c[2];        // pipe for parent to child

    // Create pipes
    pipe(fd_c2p);         // fd_c2p[0] = read end, fd_c2p[1] = write end
    pipe(fd_p2c);         // fd_p2c[0] = read end, fd_p2c[1] = write end

    // Create child process
    int pid = fork();

    if (pid < 0) {
        // Fork failed
        fprintf(2, "fork() failed\n");
        // Close pipes before exit
        close(fd_c2p[RD]);
        close(fd_c2p[WR]);
        close(fd_p2c[RD]);
        close(fd_p2c[WR]);
        exit(1);
    } else if (pid == 0) {
        // === Child process ===

        // Close unused ends
        close(fd_p2c[WR]);  // Close write end of parent->child pipe
        close(fd_c2p[RD]);  // Close read end of child->parent pipe

        // Read one byte from parent->child pipe
        if (read(fd_p2c[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child read() error!\n");
            exit_status = 1;
        } else {
            fprintf(1, "%d: received ping\n", getpid());
        }

        // Write response back to parent
        if (write(fd_c2p[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child write() error!\n");
            exit_status = 1;
        }

        // Close used ends
        close(fd_p2c[RD]);
        close(fd_c2p[WR]);
        exit(exit_status);

    } else {
        // === Parent process ===

        // Close unused ends
        close(fd_c2p[WR]);  // Close write end of child->parent pipe
        close(fd_p2c[RD]);  // Close read end of parent->child pipe

        // Write first message to child
        if (write(fd_p2c[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent write() error!\n");
            exit_status = 1;
        }

        // Read response from child
        if (read(fd_c2p[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent read() error!\n");
            exit_status = 1;
        } else {
            fprintf(1, "%d: received pong\n", getpid());
        }

        // Wait for child process to exit
        wait(0);

        // Close used ends
        close(fd_p2c[WR]);
        close(fd_c2p[RD]);
        exit(exit_status);
    }
}
```

> **pipe 实现说明**

`pipe()` 管道创建函数：

```c
int pipe(int pipefd[2]);
```

- 调用 `pipe(pipefd)` 时，内核会：在内核中创建一个“管道对象”（内核缓冲区+状态信息）。
- 参数:`pipefd`：标准 UNIX 文件描述符，指向内核中的管道缓冲区，长度为2的整数数组，`pipefd[0]` 为读端，`pipefd[1]` 为写端
- 返回值: 成功返回 `0`，失败返回 `-1`

```c
// 管道相关实现

#include "types.h"
#include "riscv.h"
#include "defs.h"
#include "param.h"
#include "spinlock.h"
#include "proc.h"
#include "fs.h"
#include "sleeplock.h"
#include "file.h"

// 管道缓冲区的大小
#define PIPESIZE 512

// 管道结构体：内核中用于存储管道状态和数据的结构
struct pipe {
  struct spinlock lock;        // 自旋锁，保护 pipe 中的数据结构
  char data[PIPESIZE];         // 管道缓冲区（循环队列）
  uint nread;                  // 已读取的字节数（读指针）
  uint nwrite;                 // 已写入的字节数（写指针）
  int readopen;                // 读端是否打开（1: 打开，0: 关闭）
  int writeopen;               // 写端是否打开（1: 打开，0: 关闭）
};

// 创建一个管道，返回两个文件结构指针（读端和写端）
int pipealloc(struct file **f0, struct file **f1)
{
  struct pipe *pi;             // 新建的 pipe 结构体指针

  pi = 0;
  *f0 = *f1 = 0;              // 初始化返回的文件指针为 0

  // 分配两个 file 结构体（文件描述符抽象）
  if((*f0 = filealloc()) == 0 || (*f1 = filealloc()) == 0)
    goto bad;

  // 分配 pipe 内核结构体
  if((pi = (struct pipe*)kalloc()) == 0)
    goto bad;

  // 初始化 pipe 结构体
  pi->readopen = 1;           // 读端默认打开
  pi->writeopen = 1;          // 写端默认打开
  pi->nwrite = 0;             // 初始化读写指针
  pi->nread = 0;
  initlock(&pi->lock, "pipe");// 初始化自旋锁

  // 初始化 file 结构体 f0 为读端
  (*f0)->type = FD_PIPE;
  (*f0)->readable = 1;
  (*f0)->writable = 0;
  (*f0)->pipe = pi;

  // 初始化 file 结构体 f1 为写端
  (*f1)->type = FD_PIPE;
  (*f1)->readable = 0;
  (*f1)->writable = 1;
  (*f1)->pipe = pi;

  return 0; // 成功返回

bad:
  // 失败释放已分配资源
  if(pi)
    kfree((char*)pi);
  if(*f0)
    fileclose(*f0);
  if(*f1)
    fileclose(*f1);
  return -1;
}

// 关闭管道的一个端点（读端或写端）
// writable=1 表示关闭写端；writable=0 表示关闭读端
void pipeclose(struct pipe *pi, int writable)
{
  acquire(&pi->lock); // 加锁

  if(writable){
    pi->writeopen = 0;   // 关闭写端
    wakeup(&pi->nread);  // 唤醒可能等待读取数据的读端
  } else {
    pi->readopen = 0;    // 关闭读端
    wakeup(&pi->nwrite); // 唤醒可能等待缓冲区可用的写端
  }

  // 如果两端都关闭了，释放 pipe 内存
  if(pi->readopen == 0 && pi->writeopen == 0){
    release(&pi->lock);
    kfree((char*)pi);
  } else
    release(&pi->lock); // 只释放锁
}

// 写入管道函数
// addr: 用户空间的起始地址，n: 字节数
int pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int i = 0; // 写入的字节计数
  struct proc *pr = myproc(); // 获取当前进程

  acquire(&pi->lock); // 加锁

  while(i < n){ // 尝试写入 n 个字节
    if(pi->readopen == 0 || killed(pr)){ // 如果读端已关闭或进程被杀死
      release(&pi->lock);
      return -1;
    }

    // 如果缓冲区已满（写指针 - 读指针 == PIPESIZE），则等待
    if(pi->nwrite == pi->nread + PIPESIZE){
      wakeup(&pi->nread);                 // 唤醒可能阻塞的读端
      sleep(&pi->nwrite, &pi->lock);      // 自己挂起等待
    } else {
      char ch;
      if(copyin(pr->pagetable, &ch, addr + i, 1) == -1) // 从用户空间拷贝一个字节到内核
        break;
      pi->data[pi->nwrite++ % PIPESIZE] = ch; // 写入管道缓冲区
      i++;
    }
  }

  wakeup(&pi->nread); // 唤醒可能等待数据的读端
  release(&pi->lock); // 释放锁

  return i; // 返回写入的字节数
}

// 从管道读取数据
// addr: 用户空间缓冲区起始地址，n: 期望读取的字节数
int piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock); // 加锁

  // 如果缓冲区为空且写端未关闭，睡眠等待写入数据
  while(pi->nread == pi->nwrite && pi->writeopen){
    if(killed(pr)){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); // 等待新数据到来
  }

  // 尝试读取 n 字节
  for(i = 0; i < n; i++){
    if(pi->nread == pi->nwrite) // 没有更多数据可读
      break;
    ch = pi->data[pi->nread++ % PIPESIZE]; // 从缓冲区读出一个字节
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1) // 写入用户空间
      break;
  }

  wakeup(&pi->nwrite); // 唤醒可能等待写入的写端
  release(&pi->lock);  // 释放锁
  return i; // 返回实际读取的字节数
}
```