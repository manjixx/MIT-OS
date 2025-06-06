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

## primes(moderate/hard)

使用多进程和管道，每个进程作为一个筛子，输出当前最小的素数，并筛掉该素数的所有倍数，然后将剩下的数集传递给下一个进程。最后形成一个子进程链，由于每个进程都调用了`wait(0)`，等待其子进程，所以会在最后一个进程完成后，依次向上退出各个进程。

```text
主进程：生成 n ∈ [2,35] -> 子进程1：筛掉所有 2 的倍数 -> 子进程2：筛掉所有 3 的倍数 -> 子进程3：筛掉所有 5 的倍数 -> .....
```

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define RD 0  // pipe read end (for reading)
#define WR 1  // pipe write end (for writing)

/**
 * 筛子实现
 * 
 * 一次 sieve 调用实现一个筛选子进程，从left pipe 获取并输出一个素数 p，筛除 p 的所有倍数，
 * 然后创建 一个筛选子进程 和 相应的输入输出管道，将剩下的数传递至下一个筛选子进程
 * 
 * @param left_pipe:来自该进程左端进程的输入管道
 */
void sieve(int left_pipe[2]){
    int p;
    read(left_pipe[RD], &p, sizeof(p));
    // 如果是结束哨兵 -1，则代表所有数字处理完毕，退出程序
    if(p == -1){
        exit(0);
    }

    printf("prime %d\n", p);

    int right_pipe[2];
    pipe(right_pipe);
    // 创建下一个筛选子进程
    int pid = fork();
    if(pid == 0){
        close(right_pipe[WR]);  // 子进程只需要对输入管道 right_pipe 进行读，而不需要写，所以关掉子进程的输入管道写文件描述符，降低进程打开的文件描述符数量
        close(left_pipe[RD]);   // 这里的 left_pipe 是 *父进程* 的输入管道，子进程用不到，关掉
        sieve(right_pipe);
    } else {
        close(right_pipe[RD]); // 父进程只需要对子进程的输入管道进行写，不需要读，关闭
        // close(left_pipe[WR]);  父进程只需要对其输入管道进行读，不需要写，关掉,但在创建子进程时已经关掉，所以不用重复关闭

        int buf;
        while (read(left_pipe[RD], &buf, sizeof(buf)) && buf != -1)
        {
            if(buf % p != 0) { // 筛掉能被该进程筛选掉的数字，传递到下一进程
                write(right_pipe[WR], &buf,sizeof(buf));   // 将剩余的数字写到右端进程
            }    
        }
        // 补写最后的结束标志位
        buf = -1;
        write(right_pipe[WR], &buf, sizeof(buf));
        wait(0);
        exit(0);
    }
}

int main(int argc, char **argv){
    int input_pipe[2];
    pipe(input_pipe);

    if (fork() == 0){
        close(input_pipe[WR]);
        sieve(input_pipe);
        exit(0);
    } else {
        close(input_pipe[RD]);
        int i;
        for(i = 2;i <= 35; i++){
            write(input_pipe[WR], &i, sizeof(i));
        }
        i = -1;
        write(input_pipe[WR], &i, sizeof(i));
    }
    wait(0);
    exit(0);
}

```

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define RD 0  // 管道读端
#define WR 1  // 管道写端
#define END -1 // 结束标志

// 声明函数不会返回
void sieve_process(int read_fd) __attribute__((noreturn));

/**
 * 素数筛子进程 - 此函数不会返回，总是通过exit()终止
 * 
 * @param read_fd: 输入管道的读端文件描述符
 */
void sieve_process(int read_fd) {
    int p;
    while (1) {
        // 读取当前素数
        if (read(read_fd, &p, sizeof(p)) <= 0 || p == END) {
            exit(0);
        }
        printf("prime %d\n", p);

        int next_pipe[2];
        if (pipe(next_pipe) < 0) {
            fprintf(2, "pipe error\n");
            exit(1);
        }
        
        int pid = fork();
        if (pid < 0) {
            fprintf(2, "fork error\n");
            exit(1);
        }

        if (pid == 0) {  // 子进程：处理下一级筛选
            close(read_fd);         // 关闭父级输入
            close(next_pipe[WR]);   // 关闭未使用的写端
            sieve_process(next_pipe[RD]); // 递归处理
        } else {         // 父进程：筛选并传递数字
            close(next_pipe[RD]);  // 关闭未使用的读端
            
            int num;
            while (read(read_fd, &num, sizeof(num)) > 0 && num != END) {
                if (num % p != 0) {  // 筛选非倍数
                    write(next_pipe[WR], &num, sizeof(num));
                }
            }
            // 传递结束标志
            int end_flag = END;
            write(next_pipe[WR], &end_flag, sizeof(end_flag));
            close(next_pipe[WR]);  // 关闭写端
            wait(0);              // 等待子进程
            exit(0);
        }
    }
}

int main(int argc, char **argv) {
    int init_pipe[2];
    if (pipe(init_pipe) < 0) {
        fprintf(2, "pipe error\n");
        exit(1);
    }

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    }

    if (pid == 0) {  // 筛分子进程
        close(init_pipe[WR]);  // 关闭未使用的写端
        sieve_process(init_pipe[RD]); // 开始筛选过程
    } else {            // 主进程：提供初始数据
        close(init_pipe[RD]);  // 关闭未使用的读端
        
        // 生成2-35的数字序列
        for (int i = 2; i <= 280; i++) {
            write(init_pipe[WR], &i, sizeof(i));
        }
        // 写入结束标志
        int end_flag = END;
        write(init_pipe[WR], &end_flag, sizeof(end_flag));
        close(init_pipe[WR]);
        
        wait(0); // 等待筛分子进程结束
    }
    exit(0);
}
```