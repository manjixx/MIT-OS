# A Tutorial Introduction

## 1.1 Getting Started

Hello World

```c
#include <stdio.h>  // 本程序中包含标准输入／输出库

main(){
    printf("hello world\n");
}
```

一个 C 语言程序，无论其大小如何，都是由**函数和变量**组成:

- 函数中包含一些语句，以指定所要执行的计算操作；
- 变量则用于存储计算过程中使用的值。

每个程序都从 main 函数的起点开始执行，这意味着每个程序都必须在某个位置包含一个 main 函数。

***

## 1.2 Variables and Arithmetic Expressions

使用公式${℃=(5/9)(℉-32)}$打印下列华氏温度与摄氏温度对照表

```c
#include <stdio.h>

/* 当 fahr=0，20，… ，300 时，分别 打印华氏温度与摄氏温度对照表 */
main(){

    int fahr, celsius;
    int lower, upper, step;
    
    lower = 0; /* 温度表的下限 */
    upper = 300; /* 温度表的上限 */
    step = 20; /* 步长 */

    fahr = lower;

    while(fahr <= upper){
        celsius = 5 * (fahr - 32) / 9;
        printf("%d\t%d\n",fahr,celsius);
        fahr += step;
    }
}
```

- /* 注释 */
- 所有变量先声明后使用，声明用于说明变量的属性，它由一个类型名和一个变量表组成
- C语言中类型的取值范围取决于具体的机器。
- while 循环语句的执行方式是这样的：首先测试圆括号中的条件；如果条件为真(fahr<=upper)，则执行循环体，循环往复
- while循环包含一条循环语句时，可以不用花括弧包括，而用缩进度控制
- `printf` 函数的第一个参数中的各个%分别对应于第二个、第三个、……参数，它们在数
目和类型上都必须匹配，否则将出现错误的结果。
- 如果在 `printf` 语句的第一个参数的`%d` 中指明打印宽度，则打印的数字会在打印区域内右对齐。例如，可以用语句`printf(" %3d %6d\n", fahr, celsius);`

**float point version**

```c
#include <stdio.h>
/* print Fahrenheit-Celsius table
for fahr = 0, 20, ..., 300; floating-point version */
main()
{
    float fahr, celsius;
    float lower, upper, step;
    lower = 0; /* lower limit of temperatuire scale */
    upper = 300; /* upper limit */
    step = 20; /* step size */
    fahr = lower;
    while (fahr <= upper) {
        celsius = (5.0/9.0) * (fahr-32.0); /* different with int version */
        printf("%3.0f %6.1f\n", fahr, celsius);
        fahr = fahr + step;
    }
}
```

- 强制类型转换:如果某个算术运算符有一个浮点型操作数和一个整型操作数，则在开始运算之前整型操作数将会被转换为浮点型。
- `printf`函数跨度限制
  - `%d`：按照十进制整型数打印
  - `%6d`：按照十进制整型数打印,至少6个位宽
  - `%f`：按照浮点数打印
  - `%6f`：按照浮点数打印,至少6个位宽
  - `%6.2f`按照浮点数打印,至少6个位宽,小数点后保留两位小数
  - `%o`：八进制数
  - `%x`：十六进制数
  - `%c`：字符
  - `%s`：字符串
  - `%%`：百分号本身

***

## 1.3 The For Statement

```c
#include <stdio.h>
/*打印华氏温度—摄氏温度对照表*/main()
{
    int fahr;
    for (fahr = 0; fahr <= 300; fahr = fahr + 20)
        printf("%3d %6.1f\n", fahr, (5.0/9.0)*(fahr-32));
}
```

for循环语句包括三部分:

- 第一部分`fahr = 0`是初始化部分，仅在进入循环前执行一次。
- 第二部分`fahr <= 300`是控制循环的测试或条件部分，循环控制将对该条件求值，如果结果值为真（true），则执行循环体
- 此后将执行第三部分`fahr = fahr + 20`以将循环变量 fahr 增加一个步长，并再次对条件求值。

***

## 1.4 Symbolic Constants(符号常量)

`#define` 指令可以把符号名（或称为符号常量）定义为一个特定的字符串：

`#define name replacement text`

```c
#include <stdio.h>
#define LOWER 0 /* lower limit of table */
#define UPPER 300 /* upper limit */
#define STEP 20 /* step size */
/* print Fahrenheit-Celsius table */
main()
{
    int fahr;
    for (fahr = LOWER; fahr <= UPPER; fahr = fahr + STEP)
        printf("%3d %6.1f\n", fahr, (5.0/9.0)*(fahr-32));
}
```

- LOWER、UPPER 与 STEP 都是符号常量，而非变量，因此不需要出现在声明中。
- 符号常量名通常用大写字母拼写
- 注意，#define 指令行的末尾没有分号。

***

## 1.5 Character Input and Output

接下来我们看一组与**字符型数据**处理有关的程序。标准库提供的输入／输出模型非常简单。无论文本从何处输入，输出到何处，其输入／输出都是按照**字符流**的方式处理。

标准库提供了一次读／写一个字符的函数，其中最简单的是 `getchar` 和 `putchar` 两个函数。

- `getchar` 函数从文本流中读入下一个输入字符，并将其作为结果值返回。
- 调用 `putchar` 函数时将打印一个字符

### 1.5.1 文件复制

```c
#include <stdio.h>
/* copy input to output; 1st version */
main()
{
    int c;  /*声明为int是确保其既可以存储字符也可以存储EOF */
    c = getchar();
    while (c != EOF) { /* EOF 定义在头文件<stdio.h>中，是个整型数 */
        putchar(c);
        c = getchar();
    }
}
```

**解决如何区分文件中有效数据与输入结束符的问题。**
C 语言采取的解决方法是：**在没有输入时**，getchar 函数将返回一个特殊值，这个特殊值与任何实际字符都不同。这个值称为 EOF（end of file，文件结束）。

**优化简练**

```c
#include <stdio.h>

main(){
    int c;
    while((c = getchar()) != EOF){  
        putchar(c);
    }
}
```

### 1.5.2 字符计数

```c
#include <stdio.h>
/* count characters in input; 1st version */
main()
{
    long nc;
    nc = 0;
    while (getchar() != EOF)
        ++nc;
    printf("%ld\n", nc);
}
```

- `%ld`：printf 函数其对应的参数是 long 整型。

使用 double（双精度浮点数）类型可以处理更大的数字。

```c
#include <stdio.h>
/* count characters in input; 2nd version */
main()
{
    double nc;
    for (nc = 0; gechar() != EOF; ++nc)
        ;
    printf("%.0f\n", nc);
}
```

- C 语言的语法规则要求 for 循环语句必须有一个循环体，因此用单独的分号代替。单独的分号称为空语句，它正好能满足 for 语句的这一要求。
- whi1e 语句与 for 语句的优点之一就是在执行循环体之前就对条件进行测试，如果条件不满足，则不执行循环体，这就可能**出现循环体一次都不执行的情况**。


### 1.5.3 行计数

```c
#include <stdio.h>

/*count line in input */

main(){
    int c, nl;

    nl = 0;

    while((c = getchar()) != EOF)
        if(c == '\n')
            ++nl;
    
    printf("%d\n", nl);  
}
```

### 1.5.4 单词计数

实用程序用于统计行数、单词数与字符数。这里对单词的定义比较宽松，它是任何其中不包含空格、制表符或换行符的字符序列。

下面这段程序是 UNIX 系统中 wc 程序的骨干部分：

```c
#include <stdio.h>

#define IN 1 /*inside a word */
#define OUT 0 /*outside a word */

/* count lines, words, and characters in input */
main(){

    int c, nl, nw, nc, state;

    state = OUT;
    nl = nw = nc = 0;

    while((c == getchar()) != EOF){
        ++nc;
        if(c == '\n')
            ++nl;
        
        if(c == '\t' || c == '\n' || c == ' '){
            state = OUT;
        }else if(state == OUT){
            state = IN;
            ++nw;
        }
    }

    printf("%d %d %d \n",nl, nw, nc);
}
```

***

## 1.6 Arrays

```c
#include <stdio.h>
/* count digits, white space, others */
main()
{
    int c, i, nwhite, nother;
    int ndigit[10];
    nwhite = nother = 0;
    /* initial arrays */
    for (i = 0; i < 10; ++i)
        ndigit[i] = 0;
    while ((c = getchar()) != EOF)
        if (c >= '0' && c <= '9')
            ++ndigit[c-'0'];
        else if (c == ' ' || c == '\n' || c == '\t')
            ++nwhite;
        else
        ++nother;
    printf("digits =");
    for (i = 0; i < 10; ++i)
        printf(" %d", ndigit[i]);
    printf(", white space = %d, other = %d\n", nwhite, nother);
}
```

***

## 1.7 Functions

函数`power(m,n)`

```c
#include <stdio.h>

int power(int m, int n);

/* test power function */
main()
{
    int i;
    for (i = 0; i < 10; ++i)
    printf("%d %d %d\n", i, power(2,i), power(-3,i));
    return 0;
}
/* power: raise base to n-th power; n >= 0 */
int power(int base, int n)
{
    int i, p;
    p = 1;
    for (i = 1; i <= n; ++i)
        p = p * base;
    return p;
}
```

函数定义的一般形式:

```txt
返回值类型 函数名(0 个或多个参数声明){
    声明部分
    语句序列
}
```

函数原型与函数声明中参数名不要求相同。事实上，函数原型中的参数名是可选的，这样上面的函数原型也可以写成以下形式`int power(int, int);`

***

## 1.8 Arguments-Call by Value

在 C 语言中，**所有函数参数都是“通过值”传递的**。也就是说，传递给被调用函数的参数值存放在临时变量中，而不是存放在原来的变量中。

最主要的区别在于，在 C 语言中，**被调用函数不能直接修改主调函数中变量的值**，而**只能修改其私有的临时副本的值**。

传值调用的利大于弊。在被调用函数中，参数可以看作是便于初始化的局部变量，因此额外使用的变量更少。

```c
/* power: raise base to n-th power; n >= 0; version 2 */
int power(int base, int n)
{
    int p;
    for (p = 1; n > 0; --n)
        p = p * base;
    return p;
}
```

必要时，**也可以让函数能够修改主调函数中的变量**。这种情况下，**调用者**需要向被调用函数提供待设置值的变量的地址（从技术角度看，地址就是指向变量的指针），而**被调用函数**则需要将对应的参数声明为指针类型，并通过它间接访问变量。

***

## 1.9 Character Arrays

```c
#include <stdio.h>
#define MAXLINE 1000 /* maximum input line length */

int getline(char line[], int maxline);
void copy(char to[], char from[]);

/* print the longest input line */
main()
{
    int len; /* current line length */
    int max; /* maximum length seen so far */
    char line[MAXLINE]; /* current input line */
    char longest[MAXLINE]; /* longest line saved here */
    max = 0;
    while ((len = getline(line, MAXLINE)) > 0)
        if (len > max) {
            max = len;
            copy(longest, line);
        }
    
    if (max > 0) /* there was a line */
        printf("%s", longest);
    return 0;
}

/* getline: read a line into s, return length */
int getline(char s[],int lim)
{
    int c, i;
    for (i=0; i < lim-1 && (c=getchar())!=EOF && c!='\n'; ++i)
        s[i] = c;
    if (c == '\n') {
        s[i] = c;
        ++i;
    }
    s[i] = '\0';
    return i;
}

/* copy: copy 'from' into 'to'; assume to is big enough */
void copy(char to[], char from[])
{
    int i;
    i = 0;
    while ((to[i] = from[i]) != '\0')
        ++i;
}
```

- C 语言约定采用字符数组的形式存储字符串常量，数组中的各元素分别存储字符串的各个字符，并以`\0`标志字符串的结束
- `printf` 函数中的格式规范%s 规定，对应的参数必须是以上述形式表示的字符串。
- `copy` 函数的实现正是依赖于输入参数由'\0'结束这一事实，它将'\0'拷贝到输出参数中。
- 空字符'\0'不是普通文本的一部分

***

## 1.10 External Variable and Scope

**私有变量/局部变量/自动变量**

函数中声明的变量是函数的私有变量或局部变量，其他函数没有办法直接访问他们。

函数中的每个局部变量只在函数被调用时存在，在函数执行完毕退出时消失。

这也是其他语言通常把这类变量称为自动变最的原因。以后我们使用“自动变量”代表“局部变量”。

**全局变量/外部变量**

除自动变量外，还可以定义位于所有函数外部的变量，也就是说，在所有函数中都可以通过变量名访问这种类型的变量；

外部变量定义:`extern int 变量名`

在通常的做法中，所有外部变量的定义都放在源文件的开始处，这样就可以省略 extern 声明。
