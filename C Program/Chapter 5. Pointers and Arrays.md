# Pointers and Arrays

**指针是一种保存变量地址的变量**。在 C 语言中，指针的使用非常广泛:

- 原因之一是，指针常常是表达某个计算的惟一途径
- 另一个原因是，同其它方法比较起来，使用指针通常可以生成更高效、更紧凑的代码。

ANSI C 使用类型 `void *`（指向 void 的指针）代替 `char *`作为通用指针的类型。

## 5.1 Pointers and Address

> **内存组织方式**

通常的机器都有一系列连续编号或编址的存储单元，过些存储单元可以单个进行操纵，也可以以连续成组的方式操纵。

通常情况下，机器的一个字节可以存放一个 char 类型的数据，两个相邻的字节存储单元可存储一个 short（短整型）类型的数据，而 4 个相邻的字节存储单元可存储一个 long（长整型）类型的数据。

指针是能够存放一个地址的一组存储单元（通常是两个或 4 个字节）。

> 运算符 & 和 运算符 *

- 一元运算符 `&`可用于取一个对象的地址。**只能应用于内存中的对象**，即变量与数组元素。**它不能作用于**表达式、常量或 register 类型的变量。
- 一元运算符`*`是**间接寻址**或**间接引用运算符**。当它作用于指针时，将访问指针所指向的对象。

🙋🌰说明

```c
int x = 1, y = 2, z[10];
int *ip;    /* ip is a pointer to int */

ip = &x;    /* ip now points to x */
y = *ip;    /* y is now 1 */
*ip = 0;    /* x is now 0 */
ip = &z[0]; /* ip now points to z[0] */
```

> **指针的声明**

注意：

- **指针只能指向某种特定类型的对象**，也就是说，每个指针都必须指向某种特定的数据类型。

- 一个例外情况是**指向 void 类型的指针可以存放指向任何类型的指针**，但它不能间接引用其自身

```c
int *ip;
/* 在表达式中，*dp 和 atof(s)的值都是 double 类型，且 atof 的参数是一个指向 char 类型的指针。 */
double *dp, atof(char *);
```

```c
/* *ip 的值加 10 */
*ip = *ip + 10;

/**
 points1: 一元运算符*和&的优先级比算术运算符的优先级高
 */

y = *ip + 1; /*将 *ip指向的对象的值取出来并加1，然后再赋值给y */

/* 下列三句均为将 *ip指向的对象取出来并加1 */
*ip += 1;
++ *ip;
(*ip) ++; /*注意该句中()是必须的，因为 * 与 ++ 一元运算符遵循从左到右的结合顺序 */

/**
    points2:由于指针也是变量，所以在程序中可以直接使用，而不必通过间接引用的方法使用。
 */

iq = ip; /*表示将ip中的值拷贝到iq中，iq也将指向iq指向的对象 */

```

***

## 5.2 Pointers and Function Arguments

由于 C 语言是以传值的方式将参数值传递给被调用函数。因此，**被调用函数不能直接修改主调函数中变最的值**。

> swap函数为例

```c
/**wrong
    由于参数传递采用传值方式，因此上述的 swap 函数不会影响到调用它的例程中的参数 a 和 b 的值。该函数仅仅交换了 a 和 b 的副本的值。
*/

void swap(int x, int y){
    int temp;

    temp = x;
    x = y;
    y = temp;
}
```

```c
/**
    right
    将指向所要交换的变量的指针传递给被调用函数
 */

swap(&a, &b);

void swap(int *px, int *py){
    int temp;

    temp = *px;
    *px = *py;
    *py = temp;
}
```

***

## 5.3 Pointers and Arrays

通过**数组下标所能完成的任何操作都可以通过指针来实现**。一般来说，用指针编写的程序比用数组下标编写的程序执行速度快，但另一方面，用指针实现的程序理解起来稍微困难一些。

> **数组的声明**

```c
int a[10]; // 定义了一个由 10 个对象组成的集合，这 10 个对象存储在相邻的内存区域中，名字分别为 a[0]、a[1]、…、a[9]
```

> **声明一个指向整型对象的指针并将其指向数组a的第0个元素**

```c
int *pa
pa = &a[0];

/*将数组元素a[0]中的内容复制到变量x中*/
x = *pa;
```

> **一个通过数组和下标实现的表达式可等价地通过指针和偏移量实现。**

- 根据指针运算的定义`pa + 1`将指向下一个元素，那么`*(pa+1)`引用的是数组元素`a[1]`的内容;

- `pa + i`将指向`pa`所指向数组元素之后的第`i`个元素,`*(pa+i)`引用的是数组元素 `a[i]`的内容

有上述内容可知`pa = &a[0]`，此时`pa`与`a`具有相同的值，因为**数组名代表的就是该数组最开始的一个元素的地址**，所以赋值语句`pa = &a[0]`也可以写为`pa = a`

- **对数组元素 `a[i]`的引用也可以写成`*(a+i)`这种形式**,实际上在计算数组元素 a[i]的值时，C 语言实际上先将其转换为`*(a+i)`的形式，然后再进行求值
  
- `&a[i]`和 `a+i`的含义也是相同的。`a+i` 是 `a`之后第 `i` 个元素的地址

- 相应地，如果 `pa` 是个指针，那么，在表达式中也可以在它的后面加下标。`pa[i]`与`*(pa+i)`是等价的。

> **注意**

数组名和指针之间有一个不同之处，**指针是一个变量**。因此

- 在 `C`语言中，语句 `pa=a` 和 `pa++`都是合法的。
- 但数组名不是变量，因此，类似于 `a=pa` 和 `a++`形式的语句是非法的。

> **数组作为函数参数**

当把数组名传递给一个函数时，实际上传递的是该数组第一个元索的地址。**下列两种形式参数是一样的，**我们通常**更习惯于使用后一种形式**，因为它比前者更直观地表明了该参数是一个指针。

```c
void fun(char[] s){}

void fun(char *s){}

void fun(int[] arr){}

void fun(int *arr){}

/* 将指向子数组起始位置的指针传递给函数，这样，就将数组的一部分传递给了函数。 
  下面两个函数调用，都将把起始于 a[2]的子数组的地址传递给函数 f
*/

f(&a[2]);
f(a + 2);

```

***

## 5.4 Address Arithmetic(地址算术运算)

C 语言中的地址算术运算方法是**一致且有规律的**，将**指针、数组和地址的算术运算集成在一起**是该语言的一大优点。

> **例子:不完善的存储分配程序**

之所以说这两个函数是 **“不完善的”**，是因为对 `afree` 函数的调用次序必须与调用`alloc` 函数的次序相反。换句话说，`alloc`与`afree` 以**栈的方式（即后进先出的列表）进行存储空间的管理**。

**实现细节：**

- 让 `alloc` 函数**对一个大字符数组 `allocbuf` 中的空间进行分配**。该数组是 `alloc` 和 `afree` 两个函数**私有的数组**。由于函数 `alloc` 和 `afree` 处理的对象是指针而不是数组下标，因此，其它函数无需知道该数组的名字，这样，可以在包含`alloc`和`afree`的源文件中将该数组声明为 `static` 类型，使得它对外不可见。实际实现时，该数组甚至可以没有名字，它可以通过调用`malloc` 函数或向操作系统申请一个指向无名存储块的指针获得。
- 需要掌握`allocbuf`使用情况，我们使用指针`allocp`指向`allocbuf`中的下一个空闲单元。
- 如果 `p` 在 `allocbuf` 的边界之内，则 `afree(p)`仅仅只是将`allocp` 的值设置为`p`。
- `C` 语言保证，`0` 永远不是有效的数据地址，因此，返回值 `0` 可用来表示发生了异常事件。在本例中，返回值`0` 表示没有足够的空闲空间可供分配。
  
```c
#define ALLOCSIZE 10000 /* size of available space */
static char allocbuf[ALLOCSIZE]; /* storage for alloc */
static char *allocp = allocbuf; /* next free position */

/*返回一个指向 n 个连续字符存储单元的指针 */
char *alloc(int n) 
{   /*检查allocbuf是否有足够的地址 */
    if (allocbuf + ALLOCSIZE - allocp >= n) { /*its fit */
        allocp += n; /* 将allocp指向下一个空闲区域*/
        return allocp - n; /* 返回allocp 当前值，old p */
    } else /* not enough room */
        return 0;
}

/* free storage pointed to by p */
void afree(char *p) 
{
    if (p >= allocbuf && p < allocbuf + ALLOCSIZE)
        allocp = p;
}
```

> **指针的初始化**

对指针有意义的初始化值只能是 `0` 或者是**表示地址的表达式**，对后者来说，表达式所代表的地址必须是在此前**已定义**的具有**适当类型的数据的地址**。

> **有效的指针运算**

- **相同类型指针之间的赋值运算**；
- **指针同整数之间**的加法或减法运算；
  - 在计算 `p+n` 时，`n` 将根据 `p` 指向的对象的长度按比例缩放，而 `p` 指向的对象的长度则取决于 `p` 的声明。例如，如果 `int` 类型占 `4` 个字节的存储空间，那么在 `int` 类型的计算中，对应的 `n` 将按 `4`的倍数来计算。
- 指向**相同数组中元素的两个指针间的减法或比较运算**；
  - 指针 `p` 和 `q` 指向同一个数组的成员，那么它们之间就可以进行类似于`==、!=、<、>=`的关系比较运算
  - 如果 `p` 和 `q` 指向相同数组中的元索，且 `p<q`，那么 `q-p+1`就是位于 `p` 和 `q` 指向之间的元素的数目
- 将指针赋值为 0 或指针与 0 之间的比较运算。程序中经常用符号常量 `NULL` 代替常量 `0`，这样便于更清晰地说明常量`0` 是指针的一个特殊值。

**指针的算术运算具有一致性**：如果处理的数据类型是比字符型占据更多存储空间的浮点类型，并且 p 是一个指向浮点类型的指针，那么在执行 p++后，p 将指向下一个浮点数的地址。

> **非法的指针运算举例**

- 两个指针间的加法、乘法、除法、移位或屏蔽运算；
- 指针同 float 或 double 类型之间的加法运算；
- 不经强制类型转换而直接将指向一种类型对象的指针赋值给指向另一种类型对象的指针的运算（两个指针之一是 void *类型的情况除外）。

***

## 5.5 Character Pointers and Functions

### 5.5.1 字符串常量

**字符串常量是一个字符数组**，例如：`"I am a string"` 在字符串的内部表示中，字符数组以空字符'\0'结尾，所以，程序可以通过检查空字符找到字符数组的结尾。字符串常量占据的存储单元数也因此比双引号内的字符数大 1。

### 5.5.2 字符串常量常见的用法

> **作为函数参数**

`printf("hello, world\n"};`

当类似于这样的一个字符串出现在程序中时，实际上是通过字符指针访问该字符串。在上述语句中，`printf` 接受的**是一个指向字符数组第一个字符的指针**。也就是说，字符串常量可通过一个指向其第一个元素的指针访问。

> **定义指向字符串的指针**

```c
char *pmessage;
/**将指向该字符数组的指针赋给指针pmessage */
pmessage = "now is the time"
```

- 注意下列两个定义间存在很大差别

```c
char amessage[] = "nw is the time"; /* 定义一个数组 */
char *pmessage = "now is the time"; /* 定义一个指针 */
```

- amessage
  - `amessage` 是一个仅仅足以存放初始化字符串以及空字符'\0'的一维数组。
  - 数组中的单个字符可以进行修改始终只能指向同一个存储位置

- pmessage
  - pmessage是一个指针，其初值指向一个字符串常量,之后`pmessage`可以被修改以指向其它地址
  - 但字符串的内容无法被修改，结果是没有定义的

> **strcp函数**

函数功能：把指针 `t` 指向的字符串复制到指针`s` 指向的位置。

第一个版本的`strcpy`函数：利用数组实现
  
```c
/**
    利用数组方法实现
 */

/* strcpy: copy t to s; array subscript version */
void strcpy(char *s, char *t){
    int i;
    i = 0;
    while(s[i] = t[i] != '\0')
        i++;
}
```

第二个版本的`strcpy`函数，利用指针实现

- pointer version:改为指针
- pointer version2: 精简版本，s 和 t 的自增运算放到了循环的测试部分中，后缀运算符`++`表示在读取该字符之后才改变 `t` 的值
- pointer version3: 进一步精简版本，表达式同`'\0'`的比较是多余的，因为只需要判断表达式的值是否为 0 即可。

```c
/**
 利用指针方法实现
 */ 
/* strcpy: copy t to s; array pointer version */
void strcpy(char *s, char *t){
    while((*s = *t) != '\0'){
        s++;
        t++;
    }
}

/* strcpy: copy t to s; array pointer version2 */
void strcpy(char *s, char *t){
    while((*s++ = *t++) != '\0');
        ;
}

/* strcpy: copy t to s; array pointer version3 */
void strcpy(char *s, char *t){
    while(*s++ = *t++);
        ;
}
```

第三个版本的函数初看起来不太容易理解，但这种表示方法是很有好处的，我们应该掌握这种方法，C 语言程序中经常会采用这种写法。

标准库（<string.h>）中提供的函数 strcpy 把目标字符串作为函数值返回

> **strcmp(s,t)**

**函数功能：**字符串比较函数 strcmp(s, t)。该函数比较字符串 s 和 t，并且根据 s 按照字典顺序小于、等于或大于 t 的结果分别返回负整数、0 或正整数。该返回值是 `s` 和 `t` 由前向后逐字符比较时遇到的第一个不相等字符处的字符的差值。

第一个版本的函数：利用数组实现

```c
/* strcmp: return <0 if s<t, 0 if s==t, >0 if s>t */
int strcmp(char *s, char *t)
{
    int i;
    for (i = 0; s[i] == t[i]; i++)
        if (s[i] == '\0')
            return 0;
        return s[i] - t[i];
}
```

第二个版本的函数:利用指针的方式实现

```c
/* strcmp: return <0 if s<t, 0 if s==t, >0 if s>t */
int strcmp(char *s, char *t)
{
    for ( ; *s == *t; s++, t++)
        if (*s == '\0')
            return 0;
    return *s - *t;
}
```

> **指针与自增自减运算符**

```c
*--p;   //  读取p指针指向的字符之前先对p执行自减运算
*p--    // 先读取p指向的内容，读取后对指针进行自增自建运算

/**进栈出栈的标准用法*/

*p++ = val; /* 将val压入栈 */
val = *--p; /*将栈顶元素弹出到val */
```

## 5.6 Pointer Arrays; Pointers to Pointers

由于指针本身也是变量，所以它们也可以像其它变量一样存储在数组中。

### 5.6.1 举例: UNIX 程序 sort 的一个简化版本

该程序**按字母顺序对由文本行组成的集合**进行排序

**设计思想**:如果待排序的文本行首尾相连地存储在一个长字符数组中，那么每个文本行可通过指向它的第一个字符的指针来访问。将指针本身存储在一个数组中。这样，将指向两个文本行的指针传递给函数strcmp 就可实现对这两个文本行的比较。当交换次序颠倒的两个文本行时，实际上交换的是指针数组中与这两个文本行相对应的指针，而不是这两个文本行本身。

**优点**：消除了因移动文本行本身所带来的复杂的存储管理和巨大的开销这两个孪生问题。

排序过程包括下列 3 个步骤：

- 读取所有输入行
- 对文本行进行排序
- 按次序打印文本行
  
> **数据结构与输入和输出函数**

- 输入函数：收集和保存每个文本行中的字符，并建立一个指向这些文本行的指针的数组。它同时还必须统计输入的行数，因为在排序和打印时要用到这一信息。由于输入函数只能处理有限数目的输入行，所以在输入行数过多而超过限定的最大行数时，该函数返回某个用于表示非法行数的数值，例如-1。

- 输出函数：输出函数只需要按照指针数组中的次序依次打印这些文本行即可。

```c
#include <stdio.h>
#include <string.h>

#define MAXLINES 5000 /* max #lines to be sorted */

char *lineptr[MAXLINES]; /* pointers to text lines */

int readlines(char *lineptr[], int nlines);
void writelines(char *lineptr[], int nlines);
void qsort(char *lineptr[], int left, int right);

/* sort input lines */
main()
{
    int nlines; /* number of input lines read */
    
    if ((nlines = readlines(lineptr, MAXLINES)) >= 0) {
        qsort(lineptr, 0, nlines-1);
        writelines(lineptr, nlines);
        return 0;
    } else {
        printf("error: input too big to sort\n");
        return 1;
    }
}

#define MAXLEN 1000 /* max length of any input line */

int getline(char *, int);
char *alloc(int);

/* readlines: read input lines */
int readlines(char *lineptr[], int maxlines)
{
    int len, nlines;
    char *p, line[MAXLEN];
    nlines = 0;
    while ((len = getline(line, MAXLEN)) > 0)
        if (nlines >= maxlines || p = alloc(len) == NULL)
            return -1;
        else {
            line[len-1] = '\0'; /* delete newline */
            strcpy(p, line);
            lineptr[nlines++] = p;
        }
    return nlines;
}
/* writelines: write output lines */
void writelines(char *lineptr[], int nlines)
{
    int i;

    for (i = 0; i < nlines; i++)
        printf("%s\n", lineptr[i]);
}
```

- 指数数组`lineptr`的声明`char *lineptr[MAXLINES];`
  - 表示 `lineptr` 是一个具有 `MAXLINES` 个元素的一维数组，其中数组的**每个元素是一个指向字符类型对象的指针**。
  - lineptr[i]是一个字符指针，而*lineptr[i]是该指针指向的第 i 个文本行的首字符

- 由于 lineptr 本身是一个数组名，因此，可按照前面例子中相同的方法将其作为指针使用

> **文本排序**

```c
/* qsort: sort v[left]...v[right] into increasing order */
void qsort(char *v[], int left, int right)
{
    int i, last;
    void swap(char *v[], int i, int j);
    if (left >= right) /* do nothing if array contains */
        return; /* fewer than two elements */
    swap(v, left, (left + right)/2);
    last = left;
    for (i = left+1; i <= right; i++)
        if (strcmp(v[i], v[left]) < 0)
            swap(v, ++last, i);
    swap(v, left, last);
    qsort(v, left, last-1);
    qsort(v, last+1, right);
}

/* swap: interchange v[i] and v[j] */
void swap(char *v[], int i, int j)
{
    char *temp;
    temp = v[i];
    v[i] = v[j];
    v[j] = temp;
}
```

因为 `v`（别名为 `lineptr`）的所有元素都是字符指针，并且 `temp` 也必须是字符指针，因此`temp` 与 `v` 的任意元素之间可以互相复制。

***

## 5.7 Multi-dimensional Arrays

C 语言提供了类似于矩阵的多维数组，但实际上它们并不像指针数组使用得那样广泛.

二维数组实际上是一种特殊的一维数组，它的每个元素也是一个一维数组。因此，数组下标应该写成

```c
daytab[i][j] /* [row][col] */

daytab[i,j] /* WRONG */
```

数组可以用花括号括起来的初值表进行**初始化**，二维数组的每一行由相应的子列表进行初始化。
一般来说，除数组的第一维（下标）可以不指定大小外，**其余各维都必须明确指定大小**。

```c
static char daytab[2][13] = {
    {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31},
    {0, 31, 29, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31}
};
```

如果**将二维数组作为参数传递给函数**，那么在函数的参数声明中**必须指明数组的列数**。

数组的行数没有太大关系，因为前面已经讲过，函数调用时传递的是一个指针，它指向由行向量构成的一维数组，其中每个行向量是具有 13 个整型元素的一维数组。因此，如果将数组 `daytab` 作为参数传递给函数 f，那么 f 的声明应该写成下列形式：

```c
f(int daytab[2][13]) { ... }
//也可以写成
f(int daytab[][13]) { ... }
//因为数组的行数无关紧要，所以，该声明还可以写成
/*表明形参是一个指针，它指向具有13个整型元素的一维数组*/
f(int (*daytab)[13]) { ... }
/*因为方括号[]的优先级高于*的优先级，所以上述声明中必须使用圆括号。 */
int *daytab[13];//这相当于声明了一个数组，该数组有 13 个元素，其中每个元素都是一个指向整型对象的指针。
```

***

## 5.8 Initialization of Pointers Arrays

指针数组的初始化语法和前面所讲的其它类型对象的初始化语法类似：

```c
/* month_name: return name of n-th month */
char *month_name(int n)
{
    static char *name[] = {
        "Illegal month",
        "January", "February", "March",
        "April", "May", "June",
        "July", "August", "September",
        "October", "November", "December"
    };
```

- `name`声明是一个一维数组，数组的元素为字符指针。
- `name`数组的**初始化**通过一个字符串列表实现，列表中的每个字符串赋值给数组相应位置的元素
- 第 i 个字符串的所有字符存储在存储器中的某个位置，指向**字符串的指针**存储在name[i]中。
- 由于上述声明中没有指明 `name` 的长度，因此，编译器编译时将对初值个数进行统计，并将这一准确数字填入数组的长度。

***

## 5.9 Pointers vs Multi-dimensional Arrays

对于 C 语言的初学者来说，很容易混淆**二维数组**与**指针数组**之间的区别

```c
int a[10][20];
int *b[10];
```

- 对于上述定义，从语法角度而言`a[3][4]`和 `b[3][4]`都是对一个`int` 对象的合法引用。

- `a` 是一个真正的二维数组，它分配了 `200` 个 `int` 类型长度的存储空间，并且通过常规的矩阵下标计算公式 `20×row+col`（其中，`row` 表示行，`col` 表示列）计算得到元素`a[row][col]`的位置。
- 对 `b` 来说，该定义仅仅分配了 `10` 个指针，并且没有对其初始化，它们的初始化必须以**显式的方式进行**，比如静态初始化或通过代码初始化。假定 `b` 的每个元素都指向一个具有 `20` 个元素的数组，那么编译器就要为它分配 `200` 个 `int` 类型长度的存储空间以及 `10` 个指针的存储空间。
- 指针数组的一个**重要优点在于**，数组的每一行长度可以不同，也就是说，b 的每个元素不必都指向一个具有 20 个元素的向量，某些元素可以指向具有 2 个元素的向量，某些元素可以指向具有 50 个元素的向量，而某些元素可以不指向任何向量。

***

## 5.10 Command-line Arguments

### 5.10.1 概述

在支持 C 语言的环境中，可以**在程序开始执行时将命令行参数传递给程序**。

调用主函数main 时，它带有两个参数。

- 第一个参数（习惯上称为 `argc`，用于参数计数）的值表示**运行程序时命令行中参数的数目**；
  
- 第二个参数（称为 `argv`，用于参数向量）是一个**指向字符串数组的指针**，其中**每个字符串对应一个参数**。我们通常用多级指针处理这些字符串。

- 按照 C 语言的约定,`argv[0]`的值是启动该程序的程序名，因此 `argc` 的值至少为 `1`。如果 `argc` 的值为 `1`，则说明程序名后面没有命令行参数。

- 第一个可选参数为 `argv[1]`，而最后一个可选参数为 `argv[argc-1]`。另外，ANSI 标准要求
`argv[argc]`的值必须为一空指针

### 5.10.2 echo

> **第一个版本:将 argv 看成是一个字符指针数组**

```c
#include <stdio.h>
/* echo command-line arguments; 1st version */
main(int argc, char *argv[])
{
    int i;
    for (i = 1; i < argc; i++)
    printf("%s%s", argv[i], (i < argc-1) ? " " : "");
    printf("\n");
    return 0;
}
```

> **第二个版本**

在对 `argv` 进行自增运算、对 `argc` 进行自减运算的基础上实现的，其中 `argv` 是一个指向 `char` 类型的指针的指针

```c
#include <stdio.h>
/* echo command-line arguments; 2nd version */
main(int argc, char *argv[])
{
    while (--argc > 0)
        printf("%s%s", *++argv, (argc > 1) ? " " : "");
        printf("\n");
    return 0;
}
```

- `argv` 是一个指向参数字符串数组起始位置的指针，自增运算（`++argv`）将使得它在最开始指向 `argv[1]`而非 `argv[0]`。`*argv` 就是指向那个参数的指针。

### 5.10.3 模式查找程序

> **第一个版本**

```c
#include <stdio.h>
#include <string.h>
#define MAXLINE 1000
int getline(char *line, int max);
/* find: print lines that match pattern from 1st arg */
main(int argc, char *argv[])
{
    char line[MAXLINE];
    int found = 0;
    if (argc != 2)
        printf("Usage: find pattern\n");
    else
        while (getline(line, MAXLINE) > 0)
        if (strstr(line, argv[1]) != NULL) {
            printf("%s", line);
            found++;
    }
    return found;
}
```

标准库函数 `strstr(s, t)`返回一个指针，该指针指向字符串 `t` 在字符串 `s` 中第一次出现的位置；如果字符串 `t` 没有在字符串 `s` 中出现，函数返回 `NULL`（空指针）。该函数声明在头文件`<string.h>`中。

> **改进版本**

假定允许程序带两个可选参数。

- 其中一个参数表示“打印除匹配模式之外的所有行”
- 另一个参数表示“每个打印的文本行前面加上相应的行号”

UNIX 系统中的 C 语言程序有一个**公共的约定**：**以负号开头的参数表示一个可选标志或参数**。

- 用`-x`（代表“除……之外”）表示打印所有与模式不匹配的文本行，
- 用`-n`（代表“行号”）表示打印行号

那么命令：`find -x -n 模式`将打印所有与模式不匹配的行，并在每个打印行的前面加上行号。

**可选参数应该允许以任意次序出现**，同时，程序的其余部分应该与命令行中参数的数目无关。此外，如果可选参数能够组合使用，将会给使用者带来更大的方便，比如：`find -nx 模式`

```c
#include <stdio.h>
#include <string.h>
#define MAXLINE 1000
int getline(char *line, int max);
/* find: print lines that match pattern from 1st arg */
main(int argc, char *argv[])
{
    char line[MAXLINE];
    long lineno = 0;
    int c, except = 0, number = 0, found = 0;
    while (--argc > 0 && (*++argv)[0] == '-')
        while (c = *++argv[0])
        switch (c) {
            case 'x':
                except = 1;
                break;
            case 'n':
                number = 1;
                break;
            default:
                printf("find: illegal option %c\n", c);
                argc = 0;
                found = -1;
                break;
            }
    if (argc != 1)
        printf("Usage: find -x -n pattern\n");
    else
        while (getline(line, MAXLINE) > 0) {
            lineno++;
            if ((strstr(line, *argv) != NULL) != except) {
                if (number)
                    printf("%ld:", lineno);
                printf("%s", line);
                found++;
        }
    }
    return found;
}
```

- `argc` 执行自减运算，`argv` 执行自增运算。循环语句结束时，如果没有错误，则 `argc` 的值表示还没有处理的参数数目，而 `argv` 则指向这些未处理参数中的第一个参数。
- 注意，`*++argv` 是一个指向参数字符串的指引，因此`(*++argv)[0]`是它的第一个字符（另一种有效形式是`**++argv`）。因为 **`[]`与操作数的结合优先级比`*`和`++`高**，所以在上述表达式中必须使用圆括号，否则编译器将会把该表达式当做`*++(argv[0])`。实际上，我们在内层循环中就使用了表达式`*++argv[0]`，其目的是遍历一个特定的参数串。在内层循环中，表达式`*++argv[0]`对指针 `argv[0]`进行了自增运算

***

## 5.11 Pointers to Functions

在 C 语言中，**函数本身不是变量**，但可以**定义指向函数的指针**。这种类型的指针可以被赋值、存放在数组中、传递给函数以及作为函数的返回值等等

> **以排序函数为例说明指向函数的指针的用法**

通过在排序算法中调用不同的比较和交换函数，便可以实
现按照不同的标准排序。

```c
#include <stdio.h>
#include <string.h>

#define MAXLINES 5000 /* max #lines to be sorted */
char *lineptr[MAXLINES]; /* pointers to text lines */

int readlines(char *lineptr[], int nlines);
void writelines(char *lineptr[], int nlines);

void qsort(void *lineptr[], int left, int right, int (*comp)(void *, void *));

int numcmp(char *, char *);

/* sort input lines */
main(int argc, char *argv[])
{
    int nlines; /* number of input lines read */
    int numeric = 0; /* 1 if numeric sort */
    if (argc > 1 && strcmp(argv[1], "-n") == 0)
        numeric = 1;
    if ((nlines = readlines(lineptr, MAXLINES)) >= 0) {
        qsort((void**) lineptr, 0, nlines-1, (int (*)(void*,void*))(numeric ? numcmp : strcmp));
        writelines(lineptr, nlines);
        return 0;
    } else {
        printf("input too big to sort\n");
        return 1;
    }
}
```

在调用函数 `qsort` 的语句中，`strcmp`和 `numcmp` 是函数的地址。因为它们是函数，所以前面不需要加上取地址运算符`&`，同样的原因，数组名前面也不需要&运算符。

**qsort函数：**

```c
/**
*@breif:sort v[left]...v[right] into increasing order
*@param:
*@param: int (*comp)(void *, void *),表明 comp 是一个指向函数的指针，该函数具有两个 void *类型的参数，其返回值类型为
int。
 */
void qsort(void *v[], int left, int right, int (*comp)(void *, void *))
{
    int i, last;
    void swap(void *v[], int, int);
    if (left >= right) /* do nothing if array contains */
        return; /* fewer than two elements */
    swap(v, left, (left + right)/2);
    last = left;
    for (i = left+1; i <= right; i++)
        if ((*comp)(v[i], v[left]) < 0)
            swap(v, ++last, i);
    swap(v, left, last);
    qsort(v, left, last-1, comp);
    qsort(v, last+1, right, comp);
}

```

> **指向函数的指针com的声明**
qsort 函数的第四个参数声明如下：

```c
int (*comp)(void *, void *)
```

它表明 `comp` 是一个指向函数的指针，该函数具有两个`void *`类型的参数，其返回值类型为`int`.

> **com的使用**

`comp` 是一个指向函数的指针，`*comp` 代表一个函数。下列语句是对该函数进行调用：

```c
(*comp)(v[i], v[left]);
```

其中的圆括号是必须的，这样才能够保证其中的各个部分正确结合。如果没有括号，例如写成下面的形式：

```c
/*则表明 comp 是一个函数，该函数返回一个指向 int 类型的指针 */
int *comp(void *, void *) /* WRONG */
```

***

## 5.12 Complicated Declarations

对于简单的情况，`C` 语言的做法是很有效的，但是，如果情况比较复杂，则容易让人混淆，原因在于，`C` 语言的声明不能从左至右阅读，而且使用了太多的圆括号。

> **举例**

*是一个前缀运算符，其优先级低于()，所以，声明中必须使用圆括号以保正确的结合顺序

```c
int *f(); /* 返回int指针类型的函数 */

int (*pf)(); /* 指向返回int类型的函数指针 */
```

> **如何创建复杂的声明**

- 方法一:使用 `typedef` 通过简单的步骤合成，这种方法我们将在 6.7 节中讨论;
- 方法二:编写程序
  - 一个程序用于将正确的 C 语言声明转换为文字描述
  - 另一个程序完成相反的转换。

> **程序dcl:将C语言声明转换为文字描述**


```c
char **argv
    argv: pointer to char
int (*daytab)[13]
    daytab: pointer to array[13] of int
int *daytab[13]
    daytab: array[13] of pointer to int
void *comp()
    comp: function returning pointer to void
void (*comp)()
    comp: pointer to function returning void
char (*(*x())[])()
    x: function returning pointer to array[] of
    pointer to function returning char
char (*(*x[3])())[5]
    x: array[3] of pointer to function returning
    pointer to array[5] of char
```

程序 `dcl` 是基于声明符的语法编写的,下述为简化的语法形式：

```c
dcl: optional *'s direct-dcl
direct-dcl name
    (dcl)
    direct-dcl()
    direct-dcl[optional size]
```

声明符 `dcl` 就是前面可能带有多个`*`的 `direct-dcl`。该语法可用来对 C 语言的声明进行分析.`direct-dcl` 可以是:

- `name`
- 由一对圆括号括起来的 `dcl`
- 后面跟有一对圆括号的 `direct-dcl`
- 后面跟有用方括号括起来的表示可选长度的 `direct-dcl`。

例如对声明符`(*pfa[])()`按照语法分析：

- `pfa` 将被识别为一个 `name`，从而被认为是一个 `direct-dcl`。
- 于是，`pfa[]`也是一个 `direct-dcl`。
- 接着，`*pfa[]`被识别为一个 `dcl`，因此，判定`(*pfa[])`是一个 `direct-dcl`。
- 再接着，`(*pfa[])()`被识别为一个 `direct-dcl`，因此也是一个 `dcl`。

![](https://s2.51cto.com/images/blog/202207/12145706_62cd1b42dd17367087.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

程序 dcl 的核心是两个函数：**dcl 与 dirdcl**，它们根据声明符的语法对声明进行分析。

因为语法是递归定义的，所以在识别一个声明的组成部分时，**这两个函数是相互递归调用的**。我们称该程序是一个**递归下降语法分析程序**

```c
/* dcl: parse a declarator */
void dcl(void)
{
    int ns;
    for (ns = 0; gettoken() == '*'; ) /* count *'s */
        ns++;
    dirdcl();
    while (ns-- > 0)
        strcat(out, " pointer to");
}
/* dirdcl: parse a direct declarator */
void dirdcl(void)
{
    int type;
    if (tokentype == '(') { /* ( dcl ) */
        dcl();
        if (tokentype != ')')
        printf("error: missing )\n");
    } else if (tokentype == NAME) /* variable name */
        strcpy(name, token);
    else
        printf("error: expected name or (dcl)\n");
    while ((type=gettoken()) == PARENS || type == BRACKETS)
        if (type == PARENS)
            strcat(out, " function returning");
        else {
            strcat(out, " array");
            strcat(out, token);
            strcat(out, " of");
    }
}
```

程序的全局变量和主程序

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#define MAXTOKEN 100
enum { NAME, PARENS, BRACKETS };
void dcl(void);
void dirdcl(void);
int gettoken(void);
int tokentype; /* type of last token */
char token[MAXTOKEN]; /* last token string */
char name[MAXTOKEN]; /* identifier name */
char datatype[MAXTOKEN]; /* data type = char, int, etc. */
char out[1000];
main() /* convert declaration to words */
{
    while (gettoken() != EOF) { /* 1st token on line */
        strcpy(datatype, token); /* is the datatype */
        out[0] = '\0';
        dcl(); /* parse rest of line */
        if (tokentype != '\n')
            printf("syntax error\n");
        printf("%s: %s %s\n", name, out, datatype);
    }
    return 0;
}
```

函数 `gettoken` 用来跳过空格与制表符，以查找输入中的下一个记号

“记号”（token）可以是:

- 一个名字
- 一对圆括号
- 可能包含一个数字的一对方括号
- 也可以是其它任何单个字符。

```c
int gettoken(void) /* return next token */
{
    int c, getch(void);
    void ungetch(int);
    char *p = token;
    while ((c = getch()) == ' ' || c == '\t')
        ;
    if (c == '(') {
        if ((c = getch()) == ')') {
            strcpy(token, "()");
            return tokentype = PARENS;
        } else {
            ungetch(c);
            return tokentype = '(';
        }
    } else if (c == '[') {
        for (*p++ = c; (*p++ = getch()) != ']'; )
            ;
        *p = '\0';
        return tokentype = BRACKETS;
    } else if (isalpha(c)) {
        for (*p++ = c; isalnum(c = getch()); )
            *p++ = c;
        *p = '\0';
        ungetch(c);
        return tokentype = NAME;
    } else
        return tokentype = c;
}
```

> **程序 undcl ：相反方向的转换**

- 为了简化程序的输入，将“x is a function returning a pointer to an array of pointers to functions returning char”（x 是一个函数，它返回一个指针，该指针指向一个一维数组，该一维数组的元素为指针，这些指针分别指向多个函数，这些函数的返回值为 char 类型）的描述用下列形式表示：`x () * [] * () char`
- 程序 undcl 将把该形式转换为：`char (*(*x())[])()`

```c
/* undcl: convert word descriptions to declarations */
main()
{
    int type;
    char temp[MAXTOKEN];
    while (gettoken() != EOF) {
        strcpy(out, token);
        while ((type = gettoken()) != '\n')
            if (type == PARENS || type == BRACKETS)
                strcat(out, token);
            else if (type == '*') {
                sprintf(temp, "(*%s)", out);
                strcpy(out, temp);
            } else if (type == NAME) {
                sprintf(temp, "%s %s", token, out);
                strcpy(out, temp);
            } else
                printf("invalid input at %s\n", token);
    }
    return 0;
}
```