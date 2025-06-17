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

注意：

- 补写完结束标志位后，需要及时关闭写端口`next_pipe[WR]`,否则会因为资源耗尽而无法建立最后几个进程
- 防御性编程，构建管道与子进程要检测是否成功

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define RD 0  // pipe read end (for reading)
#define WR 1  // pipe write end (for writing)


void sieve(int read_fd) __attribute__((noreturn));


/**
 * 筛子实现
 * 
 * 一次 sieve 调用实现一个筛选子进程，从 read_fd 获取并输出一个素数 p，筛除 p 的所有倍数，
 * 然后创建 一个筛选子进程 和 相应的输入输出管道，将剩下的数传递至下一个筛选子进程
 * 
 * @param read_fd:来自该进程左端进程的输入管道
 */
void sieve(int read_fd){
    int p;
    read(read_fd, &p, sizeof(p));
    // 如果是结束哨兵 -1，则代表所有数字处理完毕，退出程序
    if(p == -1){
        exit(0);
    }

    printf("prime %d\n", p);

    int next_pipe[2];

    if (pipe(next_pipe) < 0){
        fprintf(2, "pipe error\n");
        exit(1);
    }
    // 创建下一个筛选子进程
    int pid = fork();

    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    }

    if(pid == 0){
        close(next_pipe[WR]);  // 子进程只需要对输入管道 right_pipe 进行读，而不需要写，所以关掉子进程的输入管道写文件描述符，降低进程打开的文件描述符数量
        close(read_fd);   // 这里的 left_pipe 是 *父进程* 的输入管道，子进程用不到，关掉
        sieve(next_pipe[RD]);
    } else {
        close(next_pipe[RD]); // 父进程只需要对子进程的输入管道进行写，不需要读，关闭

        int buf;
        while (read(read_fd, &buf, sizeof(buf)) && buf != -1)
        {
            if(buf % p != 0) { // 筛掉能被该进程筛选掉的数字，传递到下一进程
                write(next_pipe[WR], &buf,sizeof(buf));   // 将剩余的数字写到右端进程
            }    
        }
        // 补写最后的结束标志位
        buf = -1;
        write(next_pipe[WR], &buf, sizeof(buf));
        close(next_pipe[WR]);
        wait(0);
        exit(0);
    }
}

int main(int argc, char **argv){
    int input_pipe[2];
    pipe(input_pipe);

    if (fork() == 0){
        close(input_pipe[WR]);
        sieve(input_pipe[RD]);
        exit(0);
    } else {
        close(input_pipe[RD]);
        int i;
        for(i = 2;i <= 280; i++){
            write(input_pipe[WR], &i, sizeof(i));
        }
        i = -1;
        write(input_pipe[WR], &i, sizeof(i));
        close(input_pipe[WR]);
    }
    wait(0);
    exit(0);
}

```

不关闭管道写端口报错如下：下边为关闭了main中父进程中的`next_pipe[WR]`，而未关闭子进程中的`next_pipe[WR]`报错结果；如果二者均为关闭，则从227就开始报错。
  
```bash
== Test primes == primes: FAIL (8.1s) 
    ...
         init: starting sh
         $ primes
    GOOD prime 2
    GOOD prime 3
    GOOD prime 5
    GOOD prime 7
    GOOD prime 11
    GOOD prime 13
    GOOD prime 17
    GOOD prime 19
    GOOD prime 23
    GOOD prime 29
    GOOD prime 31
    GOOD prime 37
    GOOD prime 41
    GOOD prime 43
    GOOD prime 47
    GOOD prime 53
    GOOD prime 59
    GOOD prime 61
    GOOD prime 67
    GOOD prime 71
    GOOD prime 73
    GOOD prime 79
    GOOD prime 83
    GOOD prime 89
    GOOD prime 97
    GOOD prime 101
    GOOD prime 103
    GOOD prime 107
    GOOD prime 109
    GOOD prime 113
    GOOD prime 127
    GOOD prime 131
    GOOD prime 137
    GOOD prime 139
    GOOD prime 149
    GOOD prime 151
    GOOD prime 157
    GOOD prime 163
    GOOD prime 167
    GOOD prime 173
    GOOD prime 179
    GOOD prime 181
    GOOD prime 191
    GOOD prime 193
    GOOD prime 197
    GOOD prime 199
    GOOD prime 211
    GOOD prime 223
    GOOD prime 227
    GOOD prime 229
         pipe error
         $ echo OK
    GOOD OK
         $ qemu-system-riscv64: terminating on signal 15 from pid 81658 (<unknown process>)
    MISSING 'prime 233'
    MISSING 'prime 239'
    MISSING 'prime 241'
    MISSING 'prime 251'
    MISSING 'prime 257'
    MISSING 'prime 263'
    MISSING 'prime 269'
    QEMU output saved to xv6.out.primes
```

## find (moderate)

> **解题思路**

- **递归深度优先搜索**：采用深度优先遍历文件系统树形结构。如果检测到当前的路径是一个文件夹，那儿就 DFS 这个文件夹下的每一个文件/文件夹。如果检测到为一个文件，则直接进行比较。
- **双模式处理**：区分文件/目录两种类型进行不同处理
- **路径动态构建**：使用缓冲区动态拼接子路径
- **避免循环引用**：显式跳过`.`和`..`目录

> **关键数据结构**

- 使用`dirent`遍历目录
- 通过`stat`获取文件类型

```c
struct dirent {  // 目录条目结构
  ushort inum;   // inode编号
  char name[DIRSIZ]; // 文件名(14字节)
};

struct stat {    // 文件元数据结构
  int dev;       // 设备号
  uint ino;      // inode号
  short type;    // 文件类型(T_FILE/T_DIR)
  short nlink; // Number of links to file
  uint64 size; // Size of file in bytes
};

```

```mermaid
graph TD
    A[main(argc, argv)] --> B{参数是否为3个}
    B -- 否 --> C[输出 usage 并退出]
    B -- 是 --> D[调用 find(argv[1], argv[2])]

    D --> E[open(path)]
    E --> F{open 失败？}
    F -- 是 --> G[打印错误 return]
    F -- 否 --> H[fstat(fd)]
    H --> I{是文件 T_FILE？}

    I -- 是 --> J{文件名是否匹配目标名？}
    J -- 是 --> K[输出匹配路径]
    J -- 否 --> L[返回]
    I -- 否 --> M{是目录 T_DIR？}

    M -- 否 --> L
    M -- 是 --> N{路径是否过长？}
    N -- 是 --> O[输出错误并返回]
    N -- 否 --> P[构造 buf = path + / + de.name]

    P --> Q[循环 read(fd, &de)]
    Q --> R{de.inum 是否为 0？}
    R -- 是 --> Q
    R -- 否 --> S[stat(buf)]

    S --> T{是否为 . 或 ..？}
    T -- 是 --> Q
    T -- 否 --> U[递归调用 find(buf, target)]
    U --> Q
```

> **程序实现**

程序结构分为两部分：

- 主函数 `main()`
  - 参数校验：检查参数个数是否为 3（程序名 + 路径 + 文件名）
  - 调用 `find(argv[1], argv[2])` 递归查找

- 递归函数 `find(char *path, const char *target)`此函数是核心，实现了目录递归遍历和文件匹配。具体步骤如下：
  - A. 打开路径并判断类型
    - 若打开失败或无法获取文件状态，打印错误并返回
    - 获取类型后，进入下一步分类处理
  - B. 处理文件（T_FILE）：利用字符串比较判断是否文件名尾部与目标文件名一致
  - C. 处理目录（T_DIR）
    - 遍历目录项
    - 拼接完整子路径
    - 排除`.`与`..`目录，避免递归死循环
    - 对于每个子项递归调用 `find()` 实现深度优先搜索

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

void find(char *path, const char *target){
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, O_RDONLY)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }
    
    switch (st.type){
    case T_FILE:
        // If the end of the filename matches `/target`, it is considered a match.
        if(strcmp(path + strlen(path) - strlen(target), target) == 0){
            printf("%s\n", path);
        }
        break;
    
    case T_DIR:
        // input command length limit
        if(strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)){
            printf("find: path too long\n");
            break;
        }
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/';
        while(read(fd, &de, sizeof(de)) == sizeof(de)){
            if (de.inum == 0)
                continue;
            memmove(p,de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            if(stat(buf, &st) < 0){
                printf("find: cannot stat %s\n",buf);
                continue;
            }
            // don't enter `.` and `..`
            if(strcmp(buf + strlen(buf) - 2, "/.") != 0 && strcmp(buf + strlen(buf) - 3, "/..") != 0){
                find(buf, target);
            }
        }
        break;
    }
    close(fd);

}

int main(int argc, char *argv[]){
    if (argc != 3){
        fprintf(2, "usage:find <directory> <filename>\n");
        exit(1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```

## xargs（难度：Moderate）

> **题目解读**

`xv6` 中的很多用户命令（如 `echo, grep, rm` 等）**不支持从标准输入中读取参数**。为了将标准输入中的内容作为参数传递给另一个命令，就需要引入一个“中转”机制，这就是 `xargs` 的作用。

**`xargs` 的作用：**

- 从标准输入中读取数据（通常是通过管道传入）；
- 将这些数据当作参数，传给另一个命令；
- 最终执行该命令，就像用户手动输入这些参数一样。

> **功能示例**

```bash
echo hello too | xargs echo bye
```

**执行逻辑：**

- `echo hello too` 输出字符串：`hello too`
- 管道 `|` 把这两个单词作为标准输入传给 `xargs`
- `xargs` 读取标准输入后，执行：`echo bye hello too`。即把 `bye` 和标准输入的内容拼接成完整命令参数。

> **实现思路**

- 读取标准输入
  - 使用 `read()` 从标准输入中获取内容；
  - 按空格和换行符分隔单词；
  - 存入一个字符指针数组，例如 `argbuf`。

- 构造命令参数数组

  - 目标是构建一个 `char *argv[]` 传给 `exec()`；
  - 内容应包括：

    - 第一个元素：要执行的命令名（来自 `argv[1]`）；
    - 后续元素：从命令行参数（argv\[2]...）中复制的已有参数；
    - 最后追加：标准输入解析出的参数 `std_args[]`。

  对于：`echo hello too | xargs echo bye`

  应构造出如下 argv 数组：

  ```c
  argv[0] = "echo";
  argv[1] = "bye";
  argv[2] = "hello";
  argv[3] = "too";
  argv[4] = 0;  // NULL 结尾
  ```

- 调用 `exec()`

  - 使用 `exec()` 或 `execvp()` 启动新的子进程执行目标命令；
  - 参数数组必须以 `0` 结尾；
  - 如果 `fork()` + `exec()`，还应注意回收子进程。

> **具体实现**

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void run(char * program, char **args){
    int pid = fork();
    if(pid == 0){
        exec(program, args);
        exit(0);
    } else if(pid > 0) {
        wait(0);
    }
    return;
}

int main(int argc, char *argv[]){
    char buf[2048];                 // 读入时使用的内存池
    char *p = buf, *last_p = buf;   // 当前参数的开始、结束指针
    char *argsbuf[128];             // 命令参数数组，字符串指针数组，包含 argv 传进来的参数和 stdin 读入的参数
    char **args = argsbuf;          // 指向 argsbuf 中第一个从 stdin 读入的参数

    // 将 argv 的指针放入 *argbuf？
    for(int i = 1; i < argc; i++){  // 第一个为指令，所以不读取？
        *args = argv[i];            // 指针指向 
        args++;
    }

    // 开始读入参数
    char **pa = args;
    while(read(0, p, 1) != 0){      // 从标准输入读取放入，读入时使用的内存池，每次仅读取一个字符串
        // 读入一个参数完成（以空格分隔，如 `echo hello world`，则 hello 和 world 各为一个参数）
        if(*p == ' ' || *p == '\n'){
            char c = *p;  // 关键修复：保存原始字符
            *p = '\0'; // 将空格替换为 \0 分割开各个参数，这样可以直接使用内存池中的字符串作为参数字符串, 而不用额外开辟空间
            *(pa++)=last_p;
            last_p = p+1;
            // 读入一行完成
            if(c == '\n') {  // 使用保存的字符检查换行
                *pa = 0;
                run(argv[1], argsbuf);
                pa = args;
            }
        }
        p++;
    }
    
    // 如果最后一行不是空行
    if(pa != args){
        // 收尾最后一个参数
        *p = '\0';
        *(pa++) = last_p;
        // 收尾最后一行，参数列表末尾用 null 标识
        *pa = 0;
        run(argv[1], argsbuf);
    }

    // 循环等待所有子进程完成，每一次 wait(0) 等待一个
    // while(wait(0) != -1){};
    exit(0);
}
```

> **代码解读**

**先理解例子 `echo hello too | xargs echo bye` 发生了什么**

- `echo hello too` 输出字符串 `"hello too"`（注意结尾有换行符 `\n`）。

- 管道 `|` 把 `"hello too\n"` 传给 `xargs` 的标准输入。

- `xargs echo bye` 会：
  - 读取输入 `"hello too\n"`
  - 把输入拆分成参数 `"hello"` 和 `"too"`
  - 拼接成新命令：`echo bye hello too`
  - 执行它，最终输出 `bye hello too`

**代码整体思路**

- **准备阶段**：接收`xargs`后初始命令（如 `echo bye`）。
- **读取输入**：从键盘或管道读取数据（如 `"hello too\n"`）。
- **拆分参数**：遇到空格或换行符就切割成一个新参数。
- **执行命令**：把新参数拼接到初始命令后，执行完整命令。

**关键变量解释（想象成几个盒子）**

- `char buf[2048]`：**大内存池**，存放从输入读取的原始数据（如 `"hello too\n"`）。
- `char *p`：**扫描指针**，在 `buf` 中逐字符移动。
- `char *last_p`：**参数起点指针**，标记当前参数的开始位置。
- `char *argsbuf[128]`：**参数盒子数组**，存放所有参数（如 `{"echo", "bye", "hello", "too"}`）。
- `char **args`：指向参数盒子的**第一个空位**，用于添加新参数。
- `char **pa`：**当前操作指针**，在 `argsbuf` 中移动填充参数。

**代码执行流程详解**

- 准备初始命令 (`echo bye`)

    ```c
    // 命令行参数: argv = ["xargs", "echo", "bye"]
    for(int i = 1; i < argc; i++){  // 跳过 "xargs"，从 "echo" 开始
        *args = argv[i];  // 把 "echo" 和 "bye" 存入 argsbuf
        args++;
    }
    ```

  - 执行后 `argsbuf` 内容：
    - `argsbuf[0] = "echo"` ← 来自 `argv[1]`
    - `argsbuf[1] = "bye"`  ← 来自 `argv[2]`
  - `args` 指针指向 `argsbuf[2]`（下一个空位，等待输入参数）。

- 步骤 2: 读取输入 (`"hello too\n"`)

    ```c
    // 通过管道读入 "hello too\n" 到 buf
    while(read(0, p, 1) != 0){  // 每次读 1 个字符
        // 处理字符...
        p++;  // 指针向后移动
    }
    ```

  - `buf` 内容逐步变成：`h e l l o   t o o \n`（`_` 代表空格）。

- 步骤 3: 拆分参数（关键！）

  - 代码逐字符扫描 `buf`，遇到 **空格** 或 **换行** 就切割参数：
  
  ```c
  if(*p == ' ' || *p == '\n'){
      char c = *p;   // 保存当前字符（空格或换行）
      *p = '\0';     // 把空格/换行替换为字符串结束符 \0
      *(pa++) = last_p; // 把 last_p 开始的字符串存入 argsbuf
      last_p = p + 1;   // 更新下一个参数的起点
      if(c == '\n') {   // 如果是换行，执行命令！
          *pa = 0;      // 参数结尾标记 NULL
          run(argv[1], argsbuf); // 执行命令
          pa = args;    // 重置指针，准备下一行
      }
  }
  ```

  - 具体拆分过程：
    1. **读到 `hello` 后的空格**：
       - `*p` 是空格 → 触发切割。
       - 在空格位置写 `\0` → `buf` 中 `"hello\0too\n"`。
       - 把 `last_p`（指向 `"hello"` 开头）存入 `argsbuf[2]`。
       - `pa++` 指向下一个空位 `argsbuf[3]`。
       - `last_p` 更新指向 `"too"` 的开头。

    2. **读到结尾的 `\n`**：
       - `*p` 是换行 → 触发切割和执行。
       - 在换行位置写 `\0` → `buf` 中 `"hello\0too\0"`。
       - 把 `last_p`（指向 `"too"` 开头）存入 `argsbuf[3]`。
       - 设置 `argsbuf[4] = NULL`（参数结束标志）。
       - 此时 `argsbuf` 完整内容：

        ```c
        argsbuf[0] = "echo"  // 初始命令
        argsbuf[1] = "bye"   // 初始参数
        argsbuf[2] = "hello" // 输入的第一个参数
        argsbuf[3] = "too"   // 输入的第二个参数
        argsbuf[4] = NULL    // 结束标志
        ```

- 步骤 4: 执行命令 (`echo bye hello too`)
  
  ```c
  void run(char *program, char **args) {
      int pid = fork();  // 创建子进程
      if(pid == 0) {
          exec(program, args); // 子进程执行：echo bye hello too
          exit(0);
      } else {
          wait(0); // 父进程等待子进程结束
      }
  }
  ```

  - 最终输出：`bye hello too`
