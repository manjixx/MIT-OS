# Input and Output

本章将讲述标准库，介绍一些输入／输出函数、字符串处理函数、存储管理函数与数学函数，以及其它一些 C 语言程序的功能。

## 7.1 Standard Input and Output

> **输入机制**

输入机制是使用 `getchar` 函数从标准输入中（一般为键盘）一次读取一个字符：

```c
int getchar(void)
```

- `getchar` 函数在每次被调用时返回下一个输入字符。若遇到文件结尾，则返回 `EOF`。
- 符号常量 `EOF` 在头文件`<stdio.h>`中定义，其值一般为`-1`，但程序中应该使用 `EOF` 来测试文件是否结束，这样才能保证程序同 `EOF` 的特定值无关。

- 重定向符号`<`:将把键盘输入替换为文件输入

```c
prog < infile;  // 如果程序 prog 中使用了函数 getchar，那么程序 prog 从输入文件 infile（而不是从键盘）中读取字符。
```

- 如果输入通过管道机制来自于另一个程序，那么这种输入切换也是不可见的。

```c
otherprog | prog; // 将运行两个程序 otherprog 和 prog，并将程序 otherprog 的标准输出通过管道重定向到程序 prog 的标准输入上。
```

> **输出函数**

```c
int putchar(int);
```

- `putchar(c)`将字符 `c` 送至标准输出上，在默认情况下，标准输出为屏幕显示。如果没有发生错误，则函数 `putchar` 将返同输出的字符；如果发生了错误，则返回 `EOF`。

- 重定向符`>`

```c
prog > 输出文件名; //如果程序 prog 调用了函数 putchar,将把程序 prog 的输出从标准输出设备重定向到文件中
```

- 如果系统支持管道，那么命令行

```c
prog | anotherprog; // 将把程序 prog 的输出从标准输出通过管道重定向到程序 anotherprog 的标准输入中。

```

使用输入／输出库函数的每个源程序文件必须在引用这些函数之前包含下列语句

```c
#include <stdio.h>
```

当文件名用一对尖括号`<>`括起来时，预处理器将在由具体实现定义的有关位置中查找指定的文件（例如，在 UNIX 系统中，文件一般放在目录`/usr/include` 中）。

***

## 7.2 Formatted Output-Printf

> **printf**

输出函数 `printf` 将内部数值转换为字符的形式

```c
/**
 *@brief:函数 printf 在输出格式 format 的控制下，将其参数进行转换与格式化，并在标准输出设
备上打印出来。
 *@return:打印的字符数。
 */
int printf(char *format, arg1, arg2, ...);
```

格式字符串`fromat`包含两种类型的对象：

- 普通字符:在输出时，普通字符将原样不动地复制到输出流中
- 转换说明:**并不直接输出到输出流中**，而是用于控制 `printf` 中参数的转换和打印，每个转换说明都由一个百分号字符（`%`）开始，并以一个转换字符结束。
  - 负号，用于指定被转换的参数按照**左对齐**的形式输出。
  - 数，用于**指定最小字段宽度**。转换后的参数将打印不小于最小字段宽度的字段。如果有必要，字段左边（如果使用左对齐的方式，则为右边）多余的字符位置用空格填充以保证最小字段宽。
  - 小数点，用于将**字段宽度和精度**分开。
  - 数，**用于指定精度**，即指定字符串中要打印的最大字符数、浮点数小数点后的位数、整型最少输出的数字数目。
  - 字母 `h` 或 `l`，字母 `h` 表不将整数作为 `short` 类型打印，字母 `l` 表示将整数作为 `long`类型打印

| 字符 | 参数类型                                                                                                     |
| ---- | ------------------------------------------------------------------------------------------------------------ |
| d, i | int 类型；十进制数                                                                                           |
| o    | int 类型；无符号八进制数（没有前导 0）                                                                       |
| x, X | int 类型；无符号十六进制数（没有前导 0x 或 0X），10～15 分别用 abcdef 或 ABCDEF 表示                         |
| u    | int 类型；无符号十进制数                                                                                     |
| c    | int 类型；单个字符                                                                                           |
| s    | char *类型；顺序打印字符串中的字符，直到遇到'\0'或已打印了由精度指定的字符数为止                             |
| f    | double 类型；十进制小数[-]m.dddddd，其中 d 的个数由精度指定（默认值为 6）                                    |
| e, E | double 类型；[-]m.dddddd e ±xx 或[-]m.dddddd E ±xx，其中 d 的个数由精度指定（默认值为 6）                    |
| g, G | double 类型；如果指数小于-4 或大于等于精度，则用%e 或%E 格式输出，否则用%f 格式输出。尾部的 0 和小数点不打印 |
| p    | void *类型；指针（取决于具体实现）                                                                           |
| %    | 不转换参数；打印一个百分号%                                                                                  |

在转换说明中，**宽度或精度**可以用星号`*`表示，这时，宽度或精度的值**通过转换下一参数**（必须为 `int` 类型）来计算。

```c
printf("%.*s", max, s);
```

```c
:%s: :hello, world:
:%10s: :hello, world:
:%.10s: :hello, wor:
:%-10s: :hello, world:
:%.15s: :hello, world:
:%-15s: :hello, world :
:%15.10s: : hello, wor:
:%-15.10s: :hello, wor :
```

注意：函数 `printf` 使用第一个参数判断后面参数的个数及类型。如果参数的个数不够或者类型错误，则将得到错误的结果。

> **sprintf**

```c
int sprintf(char *string, char *format, arg1, arg2, ...);
```

`sprintf` 函数和 `printf` 函数一样，按照 `format` 格式格式化参数序列 `arg1、arg2、…`，但它将输出结果存放到 `string` 中，而不是输出到标准输出中。当然，`string` 必须足够大以存放输出结果。

***

## 7.3 Variable-length Argument List

本节以实现函数 `printf` 的一个最简单版本为例，介绍如何以**可移植的方式编写可处理变长参数表的函数**。因为我们的**重点在于参数的处理**，所以，函数 `minprintf` 只处理格式字符串和参数，格式转换则通过调用函数 `printf` 实现。

函数`printf`的正确声明方式如下所示，其中省略号表示参数表中参数的数量和类型是可变的。省略号只能出现在参数表的尾部。

```c
int printf(char *fmt, ...)
```

因为 `minprintf` 函数不需要像 `printf` 函数一样返回实际输出的字符数，因此，我们将它声明为下列形式：

```c
void minprintf(char *fmt, ...)
```

编写函数`minprintf` 的关键在于**如何处理一个甚至连名字都没有的参数表**。标准头文件`<stdarg.h>`中包含一组宏定义，它们对**如何遍历参数表进行了定义。**该头文件的实现因不同的机器而不同，但提供的接口是一致的

```c
#include <stdarg.h>
/* minprintf: minimal printf with variable argument list */
void minprintf(char *fmt, ...)
{
    va_list ap; /* points to each unnamed arg in turn */
    char *p, *sval;
    int ival;
    double dval;
    va_start(ap, fmt); /* make ap point to 1st unnamed arg */
    for (p = fmt; *p; p++) {
        if (*p != '%') {
            putchar(*p);
            continue;
        }
        switch (*++p) {
        case 'd':
            ival = va_arg(ap, int);
            printf("%d", ival);
            break;
        case 'f':
            dval = va_arg(ap, double);
            printf("%f", dval);
            break;
        case 's':
            for (sval = va_arg(ap, char *); *sval; sval++)
            putchar(*sval);
            break;
        default:
            putchar(*p);
            break;
        }
    }
    va_end(ap); /* clean up when done */
}
```

***

## 7.4 Formatted Input-Scanf

> **scanf**

输入函数 `scanf` 对应于输出函数 `printf`，它在与后者相反的方向上提供同样的转换功能。具有变长参数表的函数 `scanf` 的声明形式如下：

```c
int scanf(char *format, ...)
```

`scanf` 函数从标准输入中读取字符序列，按照**格式参数** `format` 中的格式说明**对字符序列进行解释，并把结果保存到其余的参数中**。**其它所有参数都必须是指针**，用于指定经格式转换后的相应输入保存的位置。

- 当 `scanf` 函数扫描完其格式串，或者碰到某些输入无法与格式控制说明匹配的情况时，该函数将终止
- 成功匹配并赋值的输入项的个数将作为函数值返回，所以，该函数的返回值可以用来确定已匹配的输入项的个数。
- 如果到达文件的结尾，该函数将返回 `EOF`。注意，返回 `EOF` 与 `0` 是不同的，`0` 表示下一个输入字符与格式串中的第一个格式说明不匹配。
- 下一次调用 `scanf` 函数将从上一次转换的最后一个字符的下一个字符开始继续搜索

> **sscanf**

```c
int sscanf(char *string, char *format, arg1, arg2, ...)
```

按照格式参数 `format` 中规定的格式扫描字符串 `string`，并把结果分别保存到 `arg1、arg2、…`这些参数中。**这些参数必须是指针**。

> **格式串**

格式串通常都包含转换说明，用于控制输入的转换。格式串可能包含下列部分：

- 空格或制表符，在处理过程中将被忽略。
- 普通字符（不包括%），用于匹配输入流中下一个非空白符字符。
- 转换说明，依次由一个%、一个可选的赋值禁止字符*、一个可选的数值（指定最大字段宽度）、一个可选的 h、l 或 L 字符（指定目标对象的宽度）以及一个转换字符组成
  - 转换说明控制下一个输入字段的转换。
  - 一般而言，转换结果存放在相应的参数指向的变量中。如果转换说明中有赋值禁止字符*，则跳过该输入字段，不进行赋值。
  - 输入字段定义为**一个不包括空白符的字符串**，其边界定义为到下一个空白符或达到指定的字段宽度。这表明 `scanf` 函数将越过行边界读取输入，因为换行符也是空白符。（空白符包括空格符、横向制表符、换行符、回车符、纵向制表符以及换页符）

**转换字符**指定对输入字段的解释。对应的参数必须是指针，这也是 C 语言通过值调用语义所要求的。

| 字符    | 输入数据；参数类型                                                                                                             |
| ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| d       | 十进制整数；int *类型                                                                                                          |
| i       | 整数；int *类型，可以是八进制（以 0 开头）或十六进制（以 0x 或 0X 开头）                                                       |
| o       | 八进制整数（可以以 0 开头，也可以不以 0 开头）；int *类型                                                                      |
| u       | 无符号十进制整数；unsigned int *类型                                                                                           |
| x       | 十六进制整数（可以 0x 或 0X 开头，也可以不以 0x 或 0X 开头）；int *类型                                                        |
| c       | 字符；char *类型，将接下来的多个输入字符（默认为 1 个字符）存放到指定位置。该转换规范通常不跳过空白符。 如果需要读入下一个非空白符，可以使用%1s |
| s       | 字符串（不加引号）；char *类型，指向一个足以存放该字符串（还包括尾部的字符'\0'）的字符数组。字符串的末尾将被添加一个结束符'\0' |
| e, f, g | 浮点数，它可以包括正负号（可选）、小数点（可选）及指数部分（可选）；float *类型                                                |
| %       | 字符%；不进行任何赋值操作                                                                                                      |

- 转换说明 `d、i、o、u 及 x` 的前面可以加上字符 `h` 或 `l`。
- 前缀 h 表明参数表的相应参数是一个指向`short`类型而非`int`类型的指针，前缀`l` 表明参数表的相应参数是一个指向`long`类型的指针。
- 转换说明 `e、f 和 g`的前面也可以加上前缀 `l`，它表明参数表的相应参数是一个指向 `double` 类型而非 `float` 类型的指针。

**注意**，`scanf` 和 `sscanf` 函数的所有参数都必须是指针。最常见的错误是将输入语句写成下列形式：

```c
scanf("%d", n);

scanf("%d", &n);
```

***

## 7.5 File Access

我们编写一个**访问文件的程序**，且它所访问的文件还没有连接到该程序。

- 程序`cat`把**一批命名文件串联后输出到标准输出上**。`cat` 可用来在屏幕上打印文件，对于那些无法通过名字访问文件的程序来说。它还可以用作通用的输入收集器。

```bash
cat x.c y.c #cat x.c y.c
```

**问题在于**：如何将用户需要使用的文件的外部名同读取数据的语句关联起来。

**解决方法**：。在读写一个文件之前，必须通过库函数 `fopen` 打开该文件。`fopen` 用类似于 `x.c`或 `y.c` 这样的外部名与操作系统进行某些必要的连接和通信，并返回一个随后可以**用于文件读写操作的指针**。

该指针称为文件指针，它指向一个**包含文件信息的结构**，这些信息包括：

- 缓冲区的位置
- 缓冲区中当前字符的位置
- 文件的读或写状态、是否出错或是否已经到达文件结尾
- `<stdio.h>`中已经定义了一个包含这些信息的结构 `FILE`。在程序中只需按照下列方式声明一个文件指针即可：

```c
FILE *fp;
FILE *fopen(char *name, char *mode);
```

**调用fopen函数**

```c
fp = fopen(name, mode);
```

- 第一个参数：是一个字符串，它包含文件名
- 第二个参数：**访问模式**，也是一个字符串，用于指定文件的使用方式。允许的模式包括：读（`“r”`）、写（`“w”`）及追加（`“a”`）。某些系统还区分文本文件和二进制文件，对后者的访问需要在模式字符串中增加字符`“b”`。
- 如果**打开一个不存在的文件**用于写或追加，该文件将被创建（如果可能的话）。
- 当以写方式打开一个已存在的文件时，该文件原来的内容将被覆盖。
- 但是，如果以追加方式打开一个文件，则该文件原来的内容将保留不变。
- 读一个不存在的文件会导致错误，其它一些操作也可能导致错误，比如试图读取一个无读取权限的文件。如果发生错误，fopen 将返回 NULL。

实现`cat`程序：

```c
#include <stdio.h>
/* cat: concatenate files, version 1 */
main(int argc, char *argv[])
{
    FILE *fp;
    void filecopy(FILE *, FILE *)
    if (argc == 1) /* no args; copy standard input */
        filecopy(stdin, stdout);
    else
        while(--argc > 0)
            if ((fp = fopen(*++argv, "r")) == NULL) {
                printf("cat: can't open %s\n", *argv);
                return 1;
            }else {
                filecopy(fp, stdout);
                fclose(fp);
            }
        return 0;
}

/* filecopy: copy file ifp to file ofp */
void filecopy(FILE *ifp, FILE *ofp)
{
    int c;
    while ((c = getc(ifp)) != EOF)
        putc(c, ofp);
}
```

***

## 7.6 Error Handling-Stderr and Exit

cat 程序的错误处理功能并不完善。

问题在于，如果因为某种原因而造成其中的一个文件无法访问，相应的诊断信息要在该连接的输出的末尾才能打印出来。

- 当输出到屏幕时，这种处理方法尚可以接受
- 如果输出到一个文件或通过管道输出到另一个程序时，就无法接受了。

为了更好地处理这种情况，另一个输出流以与 `stdin` 和 `stdout` 相同的方式分派给程序，即 `stderr`。即使对标准输出进行了重定向，写到 `stderr` 中的输出通常也会显示在屏幕上。

下面我们改写 cat 程序，将**其出错信息写到标准错误文件上**。

```c
#include <stdio.h>
/* cat: concatenate files, version 2 */
main(int argc, char *argv[])
{
    FILE *fp;
    void filecopy(FILE *, FILE *);
    char *prog = argv[0]; /* program name for errors */
    if (argc == 1 ) /* no args; copy standard input */
        filecopy(stdin, stdout);
    else
        while (--argc > 0)
            if ((fp = fopen(*++argv, "r")) == NULL) {
                fprintf(stderr, "%s: can't open %s\n",prog, *argv);
                exit(1);
            } else {
                filecopy(fp, stdout);
                fclose(fp);
            }
    if (ferror(stdout)) {
        fprintf(stderr, "%s: error writing stdout\n", prog);
        exit(2);
    }
    exit(0);
}
```

该程序通过两种方式发出出错信息:

- 首先，将 fprintf 函数产生的诊断信息输出到stderr 上，因此诊断信息将会显示在屏幕上，而不是仅仅输出到管道或输出文件中。
- 其次，程序使用了标准库函数 exit，当该函数被调用时，它将终止调用程序的执行。任何调用该程序的进程都可以获取 exit 的参数值，因此，可通过另一个将该程序作为子进程的程序来测试该程序的执行是否成功。
  
***

## 7.7 Line Input and Output

> **fgets**

标准库提供了一个输入函数 fgets，它和前面几章中用到的函数 getline 类似。

```c
char *fgets(char *line, int maxline, FILE *fp)
```

- 功能：`fgets` 函数从 `fp` 指向的文件中读取下一个输入行（包括换行符），并将它存放在字符数组`line` 中，它最多可读取 `maxline-1` 个字符。
- 读取的行将以`'\0'`结尾保存到数组中。通常情况下，`fgets` 返回 `line`，但如果遇到了文件结尾或发生了错误，则返回 `NULL`（我们编写的getline 函数返回行的长度，这个值更有用，当它为 0 时意味着已经到达了文件的结尾）。

> **fputs**

输出函数 `fputs` 将一个字符串（不需要包含换行符）写入到一个文件中,如果发生错误，该函数将返回 EOF，否则返回一个非负值。

```c
int fputs(char *line, FILE *fp);
```

> **gets和puts**

库函数`gets`和`puts`的功能与`fgets`和`fputs`函数类似，但它们是对`stdin`和`stdout`进行操作。

**注意**：`gets`函数在读取字符串时将删除结尾的换行符（`'\n'`），而 `puts` 函数在写入字符串时将在结尾添加一个换行符。

```c
/* fgets: get at most n chars from iop */
char *fgets(char *s, int n, FILE *iop)
{
    register int c;
    register char *cs;
    cs = s;
    while (--n > 0 && (c = getc(iop)) != EOF)
        if ((*cs++ = c) == '\n')
            break;
    *cs = '\0';
    return (c == EOF && cs == s) ? NULL : s;
}

/* fputs: put string s on file iop */
int fputs(char *s, FILE *iop)
{
    int c;
    while (c = *s++)
        putc(c, iop);
    return ferror(iop) ? EOF : 0;
}
```

ANSI 标准规定，`ferror` 在发生错误时返回非 0 值，而 `fputs` 在发生错误时返回 `EOF`，其它情况返回一个非负值。

```c
/* getline: read a line, return length */
int getline(char *line, int max)
{
    if (fgets(line, max, stdin) == NULL)
        return 0;
    else
        return strlen(line);
}
```

***

## 7.8 Miscellaneous Functions

### 7.8.1 String Operations

在下面的各个函数中，s 与 t 为 char *类型，c 与 n 为 int 类型。

```c
strcat(s, t)        //将 t 指向的字符串连接到 s 指向的字符串的末尾
strncat(s, t, n)    //将 t 指向的字符串中前 n 个字符连接到 s 指向的字符串的末尾
strcmp(s, t)        // 根据 s 指向的字符串小于（s<t）、等于（s==t）或大于（s>t）t指向的字符串的不同情况，分别返回负整数、0 或正整数
strncmp(s, t, n)    // 同 strcmp 相同，但只在前 n 个字符中比较
strcpy(s, t)        // 将 t 指向的字符串复制到 s 指向的位置
strncpy(s, t, n)    // 将 t 指向的字符串中前 n 个字符复制到 s 指向的位置
strlen(s)           // 返回 s 指向的字符串的长度
strchr(s, c)        // 在 s 指向的字符串中查找 c，若找到，则返回指向它第一次出现的位置的指针，否则返回 NULL
strrchr(s, c)       // 在 s 指向的字符串中查找 c，若找到，则返回指向它最后一次出现的位置的指针，否则返回 NULL
```

***

### 7.8.2 Character Class Testing and Conversion

头文件<ctype.h>中定义了一些用于字符测试和转换的函数。在下面各个函数中，c 是一个可表示为 unsigned char 类型或 EOF 的 int 对象。该函数的返回值类型为 int。

```c
isalpha(c)  // 若 c 是字母，则返回一个非 0 值，否则返回 0
isupper(c)  // 若 c 是大写字母，则返回一个非 0 值，否则返回 0
islower(c)  // 若 c 是小写字母，则返回一个非 0 值，否则返回 0
isdigit(c)  // 若 c 是数字，则返回一个非 0 值，否则返回 0
isalnum(c)  // 若 isalpha(c)或 isdigit(c)，则返回一个非 0 值，否则返回 0
isspace(c)  // 若 c 是空格、横向制表符、换行符、回车符，换页符或纵向制表符，则返回一个非 0 值
toupper(c)  // 返回 c 的大写形式
tolower(c)  // 返回 c 的小写形式
```

***

### 7.8.3 Ungetc

```c
int ungetc(int c, FILE *fp)
```

- 该函数将字符 c 写回到文件 fp 中。如果执行成功，则返回 c，否则返回 EOF。
- 每个文件只能接收一个写回字符。

***

### 7.8.4 Command Execution

```c
system(char* s)
```

执行包含在字符串 s 中的命令，然后继续执行当前程序。s 的内容在很大程度上与所用的操作系统有关。

***

### 7.8.5 Storage Management

函数 `malloc` 和 `calloc` 用于动态地分配存储块。

函数 `malloc` 的声明如下：

```c
void *malloc(size_t n)
```

当分配成功时，它返回一个指针，设指针指向 n 字节长度的未初始化的存储空间，否则返回NULL。

函数 `calloc` 的声明为

```c
void *calloc(size_t n, size_t size)
```

当分配成功时，它返回一个指针，该指针指向的空闲空间足以容纳由 n 个指定长度的对象组成的数组，否则返回 NULL。该存储空间被初始化为 0。

```c
free(p)
```

函数释放 `p` 指向的存储空间，其中，`p` 是此前通过调用 `malloc` 或 `calloc` 函数得到的指针。存储空间的释放顺序没有什么限制，但是，如果释放一个不是通过调用 `malloc`或 `calloc` 函数得到的指针所指向的存储空间，将是一个很严重的错误。

***

### 7.8.6 Mathematical Functions

头文件`<math.h>`中声明了 20 多个数学函数。下面介绍一些常用的数学函数，每个函数带有一个或两个 `double` 类型的参数，并返回一个 `double` 类型的值。

```c
sin(x)          // x 的正弦函数，其中 x 用弧度表示
cos(x)          // x 的余弦函数，其中 x 用弧度表示
atan2(y, x)     // y/x 的反正切函数，其中，x 和 y 用弧度表示
exp(x)          // 指数函数 ex
log(x)          // x 的自然对数（以 e 为底），其中，x>0
log10(x)        // x 的常用对数（以 10 为底），其中，x>0
pow(x, y)       // 计算 xy的值
sqrt(x)         // x 的平方根（x≥0）
fabs(x)         // x 的绝对值
```

***

### 7.8.7 Random Number Generation

函数 `rand()` 生成介于 `0` 和 `RAND_MAX` 之间的伪随机整数序列。其中 `RAND_MAX`是在头文件`<stdlib.h>`中定义的符号常量。下面是一种生成大于等于 0 但小于 1 的随机浮点数的方法：

```c
#define frand() ((double) rand() / (RAND_MAX+1.0))
```

***
