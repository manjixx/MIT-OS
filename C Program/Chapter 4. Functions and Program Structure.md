# Fucntions and Program Structure

- ANSI 标准对 C 语言所做的最明显的修改是函数声明与函数定义这两方面。目前 C 语言已经允许在声明函数时声明参数的类型。为了使函数的声明与定义相适应，ANSI 标准对函数定义的语法也做了修改。
- ANSI 标准进一步明确了名字的作用域规则，特别要求每个外部对象只能有一个定义。初始化的适用范围也更加广泛了，自动数组与结构都可以进行初始化
- C 语言预处理的功能也得到了增强。新的预处理器包含一组更完整的条件编译指令（一种
通过宏参数创建带引号的字符串的方法），对宏扩展过程的控制更严格。

## 4.1 Basics of Functions

> **一个简单的例子**

首先我们来设计并编写一个程序，它将输入中包含特定“模式”或字符串的各行打印出来（这是 UNIX 程序 grep 的特例）

```c
#include <stdio.h>
#define MAXLINE 1000 /* maximum input line length */
int getline(char line[], int max)
int strindex(char source[], char searchfor[]);
char pattern[] = "ould"; /* pattern to search for */
/* find all lines matching pattern */
main()
{
    char line[MAXLINE];
    int found = 0;
    while (getline(line, MAXLINE) > 0)
        if (strindex(line, pattern) >= 0) {
            printf("%s", line);
            found++;
        }
    return found;
}
/* getline: get line into s, return length */
int getline(char s[], int lim)
{
    int c, i;
    i = 0;
    while (--lim > 0 && (c=getchar()) != EOF && c != '\n')
        s[i++] = c;
    if (c == '\n')
        s[i++] = c;
    s[i] = '\0';
    return i;
}

/* strindex: return index of t in s, -1 if none */
int strindex(char s[], char t[])
{
    int i, j, k;
    for (i = 0; s[i] != '\0'; i++) {
        for (j=i, k=0; t[k]!='\0' && s[j]==t[k]; j++, k++)
            ;
        if (k > 0 && t[k] == '\0')
            return i;
    }
    return -1;
}
```

> **函数的定义形式**

如果函数定义中省略了返回值类型，则默认为 int 类型。

```c
返回值类型 函数明(参数声明表)
{
    声明和语句
}
```

最简单的函数`dummy() {}`

该函数不执行任何操作也不返回任何值。这种不执行任何操作的函数有时很有用，它可以在程序开发期间用以保留位置（留待以后填充代码）。

**程序可以看成是变量定义和函数定义的集合。**

函数之间的通信可以通过参数、函数返回值以及外部变量进行。

**函数在源文件中出现的次序可以是任意的。** 只要保证每一个函数不被分离到多个文件中，源程序就可以分成多个文件。

> **函数的返回值**

- 被调用函数通过 return 语句向调用者返回值，return 语句的后面可以跟任何表达式：`return 表达式；`
- 表达式将被转换为函数的返回值类型。表达式两边通常加一对圆括号，此处的括
号是可选的
- 调用函数可以忽略返回值。return 语句的后面也不一定需要表达式。当 return 语句的后面没有表达式时，函数将不向调用者返回值。当被调用函数执行到最后的右花括号而结束执行时，控制同样也会返回给调用者（不返回值）。

> **C语言编译与加载机制**

在不同的系统中，保存在多个源义件中的 C 语言程序的编译与加载机制是不同的。

例如在Unix系统中，可以使用第 1 章中提到过的 `cc` 命令执行这一任务。

- 编译：假定有 3 个函数分别存放在名为 main.c、getline.c 与 strindex.c 的 3 个文件中，则可以使用命令 `cc main.c getline.c strindex.c`
- 生成目标代码:生成的目标代码分别存放在文件 `main.o、getline.o 与strindex.o` 中
- 再把这 3 个文件一起加载到可执行文件 `a.out` 中
- 如果源程序中存在错误比如文件 main.c 中存在错误），则可以通过命令`cc main.c getline.o strindex.o`对 main.c 文件重新编译，并将编译的结果与以前已编译过的目标文件 getline.o 和 strindex.o 一起加载到可执行文件中。cc 命令使用“.c”与“.o”这两种扩展名来区分源文件与目标文件

***

## 4.2 Functions Returning Non-Integers

> **`atof(s)`**

我们通过函数`atof(s)`来说明函数返回非整型值的方法。该函数把字符串 s 转换为相应的双精度浮点数。

**首先**，由于 atof 函数的返回值类型不是 int，因此该函数必须声明返回值的类型：

```c
#include <ctype.h>
/* atof: convert string s to double */
double atof(char s[])
{
    double val, power;
    int i, sign;
    for (i = 0; isspace(s[i]); i++) /* skip white space */
        ;
    sign = (s[i] == '-') ? -1 : 1;
    if (s[i] == '+' || s[i] == '-')
        i++;
    for (val = 0.0; isdigit(s[i]); i++)
        val = 10.0 * val + (s[i] - '0');
    if (s[i] == '.')
        i++;
    for (power = 1.0; isdigit(s[i]); i++) {
        val = 10.0 * val + (s[i] - '0');
        power *= 10;
    }
    return sign * val / power;
}
```

**其次**，调用函数必须知道 atof 函数返回的是非整型值，这一点也是很重要的

一种方法是在调用函数中显式声明 atof 函数。

```c
#include <stdio.h>
#define MAXLINE 100
/* rudimentary calculator */
main()
{
    // 声明语句，表明 sum 是一个 double 类型的变量，atof 函数带有个 char[]型的参数，且返回一个double 类型的值
    double sum, atof(char []);
    char line[MAXLINE];
    int getline(char line[], int max);
    sum = 0;
    while (getline(line, MAXLINE) > 0)
        printf("\t%g\n", sum += atof(line));
    return 0;
}
```

- 函数 atof 的声明与定义必须一致。
  - 如果 atof 函数与调用它的主函数 main 放在同一源文件中，并且类型不一致，编译器就会检测到该错误。
  - 如果 atof 函数是单独编译的（这种可能性更大），这种不匹配的错误就无法检测出来，**atof 函数将返回 double 类型的值，而 main 函数却将返回值按照 int 类型处理，最后的结果值毫无意义。**
  - 如果没有函数原型，则函数将在第一次出现的表达式中被隐式声明，例如：`sum += atof(line)`.如果先前没有声明过的一个名字出现在某个表达式中，并且其后紧跟一个左圆括号，那么上下文就会认为该名字是一个函数名字，该函数的返回值将被假定为 int 类型，但上下文并不对其参数作任何假设。
  - 不过，在新编写的程序中这么做是不提倡的。如果函数带有参数，则要声明它们；如果没有参数，则使用 void 进行声明

> **在正确进行声明的函数 atof 的基础上，我们可以利用它编写出函数 atoi（将字符串转换为 int 类型）**

```c
/* atoi: convert string s to integer using atof */
int atoi(char s[])
{
    double atof(char s[]);
    return (int) atof(s);
}
```

***

## 4.3 External Variables

C 语言程序可以看成由一系列的外部对象构成，这些外部对象可能是变量或函数。

- 外部变量:定义在函数之外，因此可以在许多函数中使用。
- 函数本身是外部的，由于 C 语言不允许在一个函数中定义其它函数

形容词 external 与 internal 相对的，internal 用于描述定义在函数内部的函数参数及变量。

**外部变量与函数具有下列性质：** 通过同一个名字对外部变量的所有引用（即使这种引用来自于单独编译的不同函数）实际上都是引用同一个对象（标准中把这一性质称为外部链接）。

> **外部变量**

- 因为外部变量可以在全局范围内访问，这就为函数之间的数据交换提供了一种可以代替函数参数与返回值的方式。任何函数都可以通过名字访问一个外部变量
- 如果函数之间需要其享大量的变量，使用外部变量要比使用一个很长的参数表更方便、有效。
- 外部变量的用途还表现在它们与内部变量相比具有更大的作用域和更长的生存期。
  - 自动变量只能在函数内部使用，从其所在的函数被调用时变量开始存在，在函数退出时变量也将消失。
  - 而外部变量是永久存在的，它们的值在一次函数调用到下一次函数调用之间保持不变

> **举例**

如果两个函数必须共享某些数据，而这两个函数互不调用对方，这种情况下最方便的方式便是把这些共享数据定义为外部变量，而不是作为函数参数传递。下面我们通过一个更复杂的例子来说明这一点。

- 逆波兰表示法

为了更容易实现，我们在计算器中使用逆波兰表示法代替普通的中辍表示法，在逆波兰表示法中，所有运算符都跟在操作数的后面。比如，下列中缀表达式：`(1 – 2) * (4 + 5)`采用逆波兰表示法表示为：`1 2 - 4 5 + *`
逆波兰表示法中不需要圆括号，只要知道每个运算符需要几个操作数就不会引起歧义。

- 计算器程序的实现很简单。每个操作数都被依次压入到栈中；当一个运算符到达时，从栈中弹出相应数目的操作数（对二元运算符来说是两个操作数），把该运算符作用于弹出的操作数，并把运算结果再压入到栈中。

```c
while (下一个运算符或操作数不是文件结束指示符)
    if (是数)
        将该数压入到栈中
    else if (是运算符)
        弹出所需数目的操作数
        执行运算
        将结果压入到栈中
    else if (是换行符)
        弹出并打印栈顶的值
    else
        出错
```

- **栈放在哪里？**

把栈放在哪儿？也就是说，哪些例程可以直接访问它？一种可能是把它放在主函数 main 中，把栈及其当前位置作为参数传递给对它执行压入或弹出操作的函数。但是，main 函数不需要了解控制栈的变量信息，它只进行压入与弹出操作。因此，**可以把栈及相关信息放在外部变量中，并只供 push 与 pop 函数访问，而不能被 main 函数访问。**

- **main函数**

```c
#include <stdio.h>
#include <stdlib.h> /* for atof() */
#define MAXOP 100 /* max size of operand or operator */
#define NUMBER '0' /* signal that a number was found */
int getop(char []);
void push(double);
double pop(void);
/* reverse Polish calculator */
main()
{
    int type;
    double op2;
    char s[MAXOP];
    while ((type = getop(s)) != EOF) {
        switch (type) {
            case NUMBER:
                push(atof(s));
                break;
            case '+':
                push(pop() + pop());
                break;
            case '*':
                push(pop() * pop());
                break;

            // 因为+与*两个运算符满足交换律，因此，操作数的弹出次序无关紧要。但是，-与/两个运算符的左右操作数必须加以区分。
            case '-':
                op2 = pop();
                push(pop() - op2);
                break;
            case '/':
                op2 = pop();
                if (op2 != 0.0)
                    push(pop() / op2);
                else
                    printf("error: zero divisor\n");
                break;
            case '\n':
                printf("\t%.8g\n", pop());
                break;
            default:
                printf("error: unknown command %s\n", s);
                break;
        }
    }
    return 0;
}
```

- **push and pop**

```c
#define MAXVAL 100 /* maximum depth of val stack */
/*如果变量定义在任何函数的外部，则是外部变量。因此，我们把 push 和 pop 函数必须
共享的栈和栈顶指针定义在这两个函数的外部。 */
int sp = 0; /* next free stack position */
double val[MAXVAL]; /* value stack */
/* push: push f onto value stack */
void push(double f)
{
    if (sp < MAXVAL)
        val[sp++] = f;
    else
        printf("error: stack full, can't push %g\n", f);
}
/* pop: pop and return top value from stack */
double pop(void)
{
    if (sp > 0)
        return val[--sp];
    else {
        printf("error: stack empty\n");
        return 0.0;
    }
}
```

- **getop**

该函数获取下一个运算符或操作数。

如果下一个字符不是数字或小数点，则返回；

否则，把这些数字字符串收集起来（其中可能包含小数点），并返回 NUMBER，以标识数已经收集起来了。

```c
#include <ctype.h>
int getch(void);
void ungetch(int);
/* getop: get next character or numeric operand */
int getop(char s[])
{
    int i, c;
    while ((s[0] = c = getch()) == ' ' || c == '\t')
        ;
    s[1] = '\0';
    if (!isdigit(c) && c != '.')
        return c; /* not a number */
    i = 0;
    if (isdigit(c)) /* collect integer part */
        while (isdigit(s[++i] = c = getch()))
            ;
    if (c == '.') /* collect fraction part */
        while (isdigit(s[++i] = c = getch()))
            ;
    s[i] = '\0';
    if (c != EOF)
        ungetch(c);
    return NUMBER;
}
```

- **getch 与 ungetch**

**用途:**程序不能确定它已经读入的输入是否足够，除非超前多读入一些输入。读入一些字符以合成一个数字的情况便是一例：在看到第一个非数字字符之前，已经读入的数的完整性是不能确定的。由于程序要超前读入一个字符，这样就导致最后有一个字符不属于当前所要读入的数。

**解决方案:** 如果能“反读”不需要的字符，该问题就可以得到解决。**每当程序多读入一个字符时，就把它压回到输入中，对代码其余部分而言就好像没有读入该字符一样。** **getch** 函数用于读入下一个待处理的字符，而 **ungetch** 函数则用于把字符放回到输入中，这样，此后在调用 getch 函数时，在读入新的输入之前先返回 ungetch 函数放回的那个字符。**这里还需要增加一个下标变量来记住缓冲区中当前字符的位置**

由于**缓冲区与下标变量是供 getch 与 ungetch 函数共享的**，且在两次调用之间必须保持值不变，因此它们必须是这**两个函数的外部变量**。可以按照下列方式编写 getch、ungetch 函数及其共享变量：

```c
#define BUFSIZE 100
char buf[BUFSIZE]; /* buffer for ungetch */
int bufp = 0; /* next free position in buf */

int getch(void) /* get a (possibly pushed-back) character */
{
    return (bufp > 0) ? buf[--bufp] : getchar();
}
void ungetch(int c) /* push character back on input */
{
    if (bufp >= BUFSIZE)
        printf("ungetch: too many characters\n");
    else
        buf[bufp++] = c;
}
```

***

## 4.4 Scope Rules

构成 C 语言程序的**函数与外部变量可以分开进行编译**。一个程序可以存放在几个文件中，**原先已编译过的函数可以从库中进行加载**。这里我们感兴趣的问题有：

- 如何进行声明才能确保变量在编译时被正确声明？
- 如何安排声明的位置才能确保程序在加载时各部分能正确连接？
- 如何组织程序中的声明才能确保只有一份副本？
- 如何初始化外部变量？

> **举例:重新组织计算机程序**

- **名字的作用域**指的是程序中可以使用该名字的部分。
  - 在函数开头声明的自动变量来说，其作用域是声明该变量名的函数。不同函数中声明的具有相同名字的各个局部变量之间没有任何关系。
  - 函数的参数也是这样的，实际上可以将它看作是局部变量。

- **外部变量或函数的作用域**从声明它的地方开始，到其所在的（待编译的）文件的末尾结束。
- 另一方面，**如果要在外部变量的定义之前使用该变量**，或者**外部变量的定义与变量的使用不在同一个源文件中**，则必须在相应的**变量声明中强制性地使用关键字 extern**。

- **变量的定义与声明**
  
  - 变量的声明:用于说明变量的属性（主要是变量的类型）
  - 变量的定义:变量定义除此以外还将引起存储器的分配
  - 在一个源程序的所有源文件中，**一个外部变量只能在某个文件中定义一次**，而**其它文件可以通过 extern 声明来访问它**（定义外部变量的源文件中也可以包含对该外部变量的extern 声明）。
  - 外部变量的定义中必须指定数组的长度，但 extern 声明则不一定要指定数组的长度。
  - 外部变量的初始化只能出现在定义中

```c
/**下列语句均在函数外 */

/* 定义外部变量并为之分配存储单元 */
int sp;
double val[MAXVAL];

/* 声明了一个 int 类型的外部变量 sp 以及一个 double 数组类型的外部变量 val 但并未分配存储空间  */
extern int sp;
extern double val[];
```

- 举例：假定函数 push 与 pop 定义在`file1`中，而变量 val 与 sp 在另一个文件中定义并被初始化在`file2`中（通常不大可能这样组织程序）

由于文件 file1 中的 extern 声明不仅放在函数定义的外面，而且还放在它们的前面，因此它们适用于该文件中的所有函数。
```c
/**文件file1 */
extern int sp;
extern double val[];
void push(double f) { ... }
double pop(void) { ... }
```

```c
/**file 2 */
int sp = 0;
double val[MAXVAL];
```

***

## 4.5 Header Files

下面我们来考虑把上述的计算器程序分割到若干个源文件中的情况。我们做如下分割:

- 将主函数 main 单独放在文件 main.c中；
- 将 push 与 pop 函数以及它们使用的外部变量放在第二个文件 stack.c 中；
- 将 getop 函数放在第三个文件 getop.c 中；
- 将 getch 与 ungetch 函数放在第四个文件 getch.c 中。
- 对于共享的变量，我们尽可能把共享的部分集中在一起，这样就只需要一个副本，改进程序时也容易保证程序的正确性。我们把这些公共部分放在头文件 calc.h 中，在需要使用该头文件时通过#include 指令将它包含进来

> **calc.h**
 
```c
#define NUMBER '0'
void push(double);
double pop(void);
int getop(cahr []);
int getch(void);
void ungetch(int);
```

> **main.c**

```c
#include <stdio.h>
#include <stdlib.h>
#include "calc.h"
# define MAXDP 100;

main(){
    ......
}
```

> **getop.c**

```c
#include <stdio.h>
#include <ctype.h>
#include "calc.h"

getop(){
    ......
}
```

> **stack.c**

```c
#include <stdio.h>
#include "calc.h"
#define MAXVAL 100

int sp = 0;
double val[MAXVAL];
void push(double){
    ......
}
double pop(void){
    ......
}
```

> **getch.c**

```c
#include <stdio.h>
#define BUFSIZE 100
char buf[BUFSIZE]
int bufp = 0;
int getch(void){
    ......
}
void ungetch(int){
    ......
}
```

***

## 4.6 Static Variables

- 用 static声明限定**外部变量**，将其后声明的**外部对象的作用域**限定为**被编译源文件的剩余部分**
- **函数声明为 static类型**，则该函数名除了对该函数声明所在的文件可见外，其它文件都无法访问。
- **static 类型的内部变量**是一种只能在某个特定函数中使用但**一直占据存储空间的变量。**

注意: static类型的内部变量与自动变量不同的是，不管其所在函数是否被调用，它一直存在，而不像自动变量那样，随着所在函数的被调用和退出而存在和消失。

***

## 4.7 Register Variables

`register` 声明告诉编译器，它所声明的变量在程序中使用频率较高。其思想是:**将register 变量放在机器的寄存器中**，这样可以使程序更小、执行速度更快。**但编译器可以忽略此选项。**

**register声明的形式:**

```c
register int x;
register char c;
```

**register声明的作用范围:**

**只适用于自动变量以及函数的形式参数。**

```c
f(register unsigned m, register long n){
    register int i;
    ...
}
```

**注意:**

- 底层硬件环境的实际情况对寄存器变量的使用会有一些限制。每个函数中只有很少的变量可以保存在寄存器中，且只允许某些类型的变量。
- 过量的寄存器声明并没有什么害处，这是因为编译器可以忽略过量的或不支持的寄存器变量声明
- 寄存器地址不可访问
- 在不同的机器中，对寄存器变量的数目和类型的具体限制也是不同的

***

## 4.8 Block Structure

C 语言不允许在函数中定义函数。但是，在函数中可以以程序块结构的形式定义变量。

变量的声明（包括初始化）除了可以紧跟在**函数开始的花括号之后**，还可以紧跟**在任何其它标识复合语句开始的左花括号之后**。以这种方式声明的变**量可以隐藏程序块外与之同名的变量**，它们之间没有任何关系，并在与左花括号匹配的右花括号出现之前一直存在。

***

## 4.9 Initialization

- **在不进行显式初始化的情况下**
  - 外部变量和静态变量都将被初始化为 0
  - 而自动变量和寄存器变量的初值则没有定义（即初值为无用的信息）。

- 定义标量变量时，可以在变量名后紧跟一个等号和一个表达式来初始化变量：

```c
int x = 1;
char squota = '\'';
long day = 1000L * 60L * 60L * 24L; /* milliseconds/day */
```

- **对于外部变量与静态变量**来说，初始化表达式必须是常量表达式，且只初始化一次（从概念上讲是在程序开始执行前进行初始化）。
- **对于自动变量与寄存器变量**，则在每次进入函数或程序块时都将被初始化。而且初始化表达式可以不是常量表达式：表达式中可以包含任意在此表达式之前已经定义的值，包括函数调用，实际上，自动变量的初始化等效于简写的赋值语句
- **数组的初始化**可以在声明的后面紧跟一个初始化表达式列表，初始化表达式列表用花括号括起来，各初始化表达式之间通过逗号分隔。
  - 当省略数组的长度时，编译器将把花括号中初始化表达式的个数作为数组的长度，在本例中数组的长度为 12
  - 如果初始化表达式的个数比数组元素数少，则对外部变量、静态变量和自动变量来说，没有初始化表达式的元素将被初始化为 0，如果初始化表达式的个数比数组元素数多，则是错误的。

```c
int days[] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
```

- 不能一次将一个初始化表达式指定给多个数组元素，也不能跳过前面的数组元素而直接初始化后面的数组元素。

- 字符数组的初始化比较特殊：可以用一个字符串来代替用花括号括起来并用逗号分隔的
初始化表达式序列。

```c
char pattern[] = "ould ";
// 它同下面的声明是等价的：
char pattern[] = { 'o', 'u', 'l', 'd'}; /*数组的长度是 5（4 个字符加上一个字符串结束符'\0'）*/
```

***

## 4.10 Recursion

C 语言中的函数可以递归调用，即函数可以直接或间接调用自身。

递归并不节省存储器的开销，因为递归调用过程中必须在某个地方维护一个存储处理值的栈。

递归的执行速度并不快，但递归代码比较紧凑，并且比相应的非递归代码更易于编写更易于理解。

在描述树等递归定义的数据结构时使用递归尤其方便。

> **举例一**

- **问题定义**: 我们考虑一下将**一个数作为字符串打印**的情况。前面讲过，数字是以反序生成的：低位数字先于高位数字生成，但它们必须以与此相反的次序打印
  
- **解决方案:**
  - 一种方法是将生成的各个数字依次存储到一个数组中，然后再以相反的次序打印它们，这种方式与 3.6 节中 itoa 函数的处理方式相似。
  - 另一种方法则是使用递归，函数 printd 首先调用它自身打印前面的（高位）数字，然后再打印后面的数字。

```c
/**该函数不能处理最大的负数 */
#include <stdio.h>
/* printd: print n in decimal */
void printd(int n)
{
    if (n < 0) {
    putchar('-');
    n = -n;
    }
    if (n / 10)
        printd(n / 10);
    putchar(n % 10 + '0');
}
```

> **快速排序**

```c
/* qsort: sort v[left]...v[right] into increasing order */
void qsort(int v[], int left, int right)
{
    int i, last;
    void swap(int v[], int i, int j);
    if (left >= right) /* do nothing if array contains */
        return; /* fewer than two elements */
    swap(v, left, (left + right)/2); /* move partition elem */
    last = left; /* to v[0] */
    for (i = left + 1; i <= right; i++) /* partition */
        if (v[i] < v[left])
            swap(v, ++last, i);
    swap(v, left, last); /* restore partition elem */
    qsort(v, left, last-1);
    qsort(v, last+1, right);
}

/* swap: interchange v[i] and v[j] */
void swap(int v[], int i, int j)
{
    int temp;
    temp = v[i];
    v[i] = v[j];
    v[j] = temp;
}
```

***

## 4.11 The C Proprecessor

C 语言通过预处理器提供了一些语言功能。

从概念上讲，**预处理器**是编译过程中单独执行的第一个步骤。

两个最常用的预处理器指令是：

- #include 指令（用于在编译期间把指定文件的内容包含进当前文件中）
- #define 指令（用任意字符序列替代一个标记）。

本节还将介绍预处理器的其它一些特性，如条件编译与带参数的宏。

### 4.11.1 文件包含

文件包含指令（即#include 指令）使得处理大量的#define 指令以及声明更加方便。

```c
#include "文件名"
#include <文件名>
```

- 上述指令所在的行都将被替换为由文件名指定的文件的内容。
- 如果文件名用引号引起来，则在源文件所在位置查找该文件；
- 如果在该位置没有找到文件，或者如果文件名是用尖括号<与>括起来的，则将根据相应的规则查找该文件，这个规则同具体的实现有关。
- 被包含的文件本身也可包含 `#include` 指令。
- 源文件的开始处通常都会有多个#include 指令，它们用以包含常见的#define 语句和extern 声明，或从头文件中访问库函数的函数原型声明，比如<stdio.h>。

- 在大的程序中，#include 指令是将所有声明捆绑在一起的较好的方法。它保证所有的
源文件都具有相同的定义与变量声明，这样可以避免出现一些不必要的错误

### 4.11.2 宏替换

**宏定义的形式如下**：

```c
/**后续所有出现名字记号的地方都将被替换为替换文本。 */
#define name replacetext
```

- #define 指令中的名字与变量名的命名方式相同，替换文本可以是任意字符串。
- #define 指令占一行，替换文本是#define 指令行尾部的所有剩余部分内容，但也可以把一个较长的宏定义分成若干行，这时需要在待续的行末尾加上一个反斜杠符\
- #define 指令定义的名字的作用域从其定义点开始，到被编译的源文件的末尾处结束。
- 宏定义中也可以使用前面出现的宏定义。
- 替换只对记号进行，对括在引号中的字符串不起作用。例如，如果 YES是一个通过#define 指令定义过的名字，则在 printf("YES")或 YESMAN 中将不执行替换。
- 替换文本可以是任意的，例如下列语句为无限循环定义了一个新名字 forever。
  
```c
#define forever for (;;) /* infinite loop */
```

- 宏定义也可以带参数，这样可以对不同的宏调用使用不同的替换文本。例如，下列宏定义定义了一个宏 max：`#define max(A, B) ((A) > (B) ? (A) : (B))`使用宏max 看起来很像是函数调用，但宏调用直接将替换文本插入到代码中。形式参数（在此为 A 或 B）的每次出现都将被替换成对应的实际参数。

```c
x = max(p+q, r+s);
// 将被替换为下列形式：
x = ((p+q) > (r+s) ? (p+q) : (r+s));
```

仔细考虑一下 max 的展开式，就会发现它存在一些缺陷。其中，作为参数的表达式要重复计算两次，如果表达式存在副作用（比如含有自增运算符或输入／输出），则会出现不正确
的情况。`max(i++, j++) /* WRONG */`

- 可以通过`#undef` 指令取消名字的宏定义，这样做可以保证后续的调用是函数调用，而不是宏调用：

```c
#undef getchar
int getchar(void) { ... }
```

- 形式参数不能用带引号的字符串替换。但是，如果在替换文本中，参数名以#作为前缀则结果将被扩展为由实际参数替换该参数的带引号的字符串。

```c
#define dprint(expr) printf(#expr " = %g\n", expr)
// 使用语句
dprint(x/y)
// 调用该宏时，该宏将被扩展为：
printf("x/y" " = &g\n", x/y);
// 其中的字符串被连接起来了，这样，该宏调用的效果等价于
printf("x/y = &g\n", x/y);
```

- 预处理器运算符##为宏扩展提供了一种连接实际参数的手段。如果替换文本中的参数与##相邻，则该参数将被实际参数替换，##与前后的空白符将被删除，并对替换后的结果重新
扫描。下面定义的宏 paste 用于连接两个参数

```c
#define paste(front, back) front ## back
// 因此，宏调用 paste(name, 1)的结果将建立记号 name1。
```

### 4.11.3 条件包含

还可以使用条件语句对预处理本身进行控制，这种条件语句的值是在预处理执行的过程中进行计算。

这种方式为在编译过程中根据计算所得的条件值选择性地包含不同代码提供了一种手段。

> **#if语句**

`#if` 语句对其中的常量整型表达式（其中不能包含 sizeof、类型转换运算符或 enum 常量）进行求值，若该表达式的值不等于 0，则包含其后的各行，直到遇到#endif、#elif 或#else 语句为止（预处理器语句#elif 类似于 else if）。

在`#if` 语句中可以使用表达式defined(名字)，该表达式的值遵循下列规则：
  
- 当名字已经定义时，其值为 1；
- 否则，其值为 0。

**例子**

```c
#if !defined(HDR)
#define HDR
/* hdr.h 文件的内容放在这里 */
#endif
```

第一次包含头文件 hdr.h 时，将定义名字 HDR；此后再次包含该头文件时，会发现该名字已经定义，这样将直接跳转到#endif 处。

类似的方式也可以用来**避免多次重复包含同一文件**。如果多个头文件能够一致地使用这种方式，那么，每个头文件都可以将它所依赖的任何头文件包含进来，用户不必考虑和处理头文件之间的各种依赖关系。

下面的这段预处理代码首先测试系统变量 SYSTEM，然后**根据该变量的值确定包含哪个版本的头文件**：

```c
#if SYSTEM == SYSV
    #define HDR "sysv.h"
#elif SYSTEM == BSD
    #define HDR "bsd.h"
#elif SYSTEM == MSDOS
    #define HDR "msdos.h"
#else
    #define HDR "default.h"
#endif

#include HDR
```

> **#ifdef 与#ifndef**

用来**测试某个名字是否已经定义**。

```c
#ifndef HDR
#define EDR
/* hdr.h 文件的内容放在这里 */
#endif
```