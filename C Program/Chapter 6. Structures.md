`# Structures

结构是**一个或多个变量的集合**，这些变量可能为不同的类型，为了处理的方便而将这些变量组织在一个名字之下
由于结构将一组相关的变量看作一个单元而不是各自独立的实体，因此结构有助于组织复杂的数据，特别是在大型的程序中

## 6.1 Basics of Structures

> **结构定义**

```c
struct point{
    int x;
    int y;
};
```

- 关键字`struct`引入结构声明，**后面的名字是可选的**，称为**结构标记**（这里是 `point`）。其用于为**结构命名**，在定义之后，**结构标记就代表花括号内的声明**，可以用它作为该声明的简写形式
- 结构声明由包含在花括号内的一系列声明组成
- 结构中定义的变量称为**成员**
- 结构成员、结构标记和普通变量（即非成员）可以采用相同的名字
- 不同结构中的成员可以使用相同的名字

> **结构声明**

`struct`声明了一种数据类型：

```c
struct {...} x,y,z;
```

声明都将 `x、y 与 z` 声明为指定类型的变量，并且为它们分配存储空间。

如果结构声明的后面不带变量表，则不需要为它分配存储空间，它仅仅描述了一个结构的模板或轮廓。

**当带有结构标记时，其声明如下：**

```c
struct point pt;
```

> **struct初始化**：

```c
struct point maxpt = {320,200};
```

> **引用某个特定结构中的成员**：

```c
结构名.成员

```

> **结构嵌套**

**结构可以嵌套**。我们可以用对角线上的两个点来定义矩形：

```c
struct rect {
    struct point pt1;
    struct point pt2;
};
```

结构 `rect` 包含两个 `point` 类型的成员。如果按照下列方式声明 `screen` 变量：

```c
struct rect screen;
```

则可以用下列语句，引用`screen`的成员`pt1`的坐标`x`.

```c
screen.pt1.x
```

***

## 6.2 Structures and Functions

> **结构体的合法操作**

- 作为一个整体复制和赋值,包括向函数传递参数以及从函数返回值.
- 通过&运算符取地址，访问其成员。

**结构之间不可以进行比较**。

> **3种可能的方法传递结构**

- 一是分别传递各个结构成员
- 二是传递整个结构
- 三是传递指向结构的指针。

> **通过结构的指针访问结构成员**：

```c
struct point *pp;
```

`pp`定义为指向`struct point`类型对象的指针。
如果 `pp` 指向一个 `point` 结构，那么`*pp` 即为该结构，而`(*pp).x` 和`(*pp).y` 则是结构成员。

因为结构指针的使用频度非常高，为了使用方便，C语言提供了一种简写方式，**对于指向结构的指针p，可以用如下形式引用结构成员**：`p->结构成员`

```c
struct point origin, *pp;
pp = &origin;
printf("origin is (%d,%d)\n",(*pp).x,(*pp).y);
// 
printf("origin is (%d,%d)\n",pp->x,pp->y);
```

其中，`(*pp).x` 中的圆括号是必需的，因为结构成员运算符`“.”`的优先级比`“*”`的优先级高。表达式`*pp.x` 的含义等价于`*(pp.x)`，因为 `x` 不是指针，所以该表达式是非法的。

> **说明**

运算符`.`和`->`都是从左至右结合的，所以，对于下面的声明：

```c
struct rect r, *rp = &r;
// 以下 4 个表达式是等价的：
r.pt1.x;
rp->pt1.x;
(r.pt1).x;
(rp->pt1).x;
```

在所有运算符中，下面 `4` 个运算符的**优先级最高**：结构运算符`“.”`和`“->”`、用于函数调用的`“()”`以及用于下标的`“[]”`，因此，它们同操作数之间的结合也最紧密。

```c
struct {
    int len;
    char *str;
} *p;
```

`++p->len`:将增加`len`的值，而不是增加`p`的值,因为其中隐含括号关系`++(p->len)`;
`*p->str++`先读取指针`str`指向的对象的值，然后再将 `str` 加 `1`（与`*s++`相同）；
`(*p->str）++`将指针 `str` 指向的对象的值加 1；
`*p++->str` 先读取指针 `str` 指向的对象的值，然后再将 `p` 加 `1`。

***

## 6.3 Arrays of Structures

> **结构数组的声明**

下述代码声明了一个结构类型 `key`，并定义了该类型的结构数组 `keytab`，同时为其分配存储空间。

```c
struct key {
    char *word;
    int count;
} keytab[NKEYS];

/**等价于 */

struct key{
    char *word;
    int count;
}

struct key keytab[NKEYS];
```

因为结构 `keytab` 包含一个固定的名字集合，所以，最好将它声明为外部变量，这样，只需要初始化一次，所有的地方都可以使用。这种结构的初始化方法同前面所述的初始化方法类似——在定义的后面通过一个用圆括号括起来的初值表进行初始化，如下所示：

```c
struct key {
char *word;
int count;
} keytab[] = {
    "auto", 0,
    "break", 0,
    "case", 0,
    "char", 0,
    "const", 0,
    "continue", 0,
    "default", 0,
    /* ... */
    "unsigned", 0,
    "void", 0,
    "volatile", 0,
    "while", 0
};

/*更精确的做法:通常情况下，如果初值存在并且方括号[ ]中没有数值，编译程序将计算数组 keytab 中的项数。 */

struct key {
char *word;
int count;
} keytab[] = {
    {"auto", 0},
    {"break", 0},
    {"case", 0},
    {"char", 0},
    {"const", 0},
    {"continue", 0},
    {"default", 0},
    /* ... */
    {"unsigned", 0},
    {"void", 0},
    {"volatile", 0},
    {"while", 0}
};
```

C 语言提供了一个编译时（`compile-time`）一元运算符 `sizeof`，它可用来计算任一对象的长度。

```c
sizeof 对象

sizeof (类型名)
```

将返回一个无符号整型值(`size_t`)，它等于**指定对象**或**类型**占用的**存储空间字节数**。

- **对象**可以是变量、数组或结构；
- **类型**可以是基本类型，如 int、double，也可以是派生类型，如结构类型或指针类型

***

## 6.4 Pointers to Structures

重新编写关键字统计程序，这次采用指针，而不使用数组下标。keytab 的外部声明不需要修改。

```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>
#define MAXWORD 100
int getword(char *, int);
struct key *binsearch(char *, struct key *, int);

/* count C keywords; pointer version */
main()
{
    char word[MAXWORD];
    struct key *p;
    while (getword(word, MAXWORD) != EOF)
        if (isalpha(word[0]))
            if ((p=binsearch(word, keytab, NKEYS)) != NULL)
                 p->count++;
    for (p = keytab; p < keytab + NKEYS; p++)
        if (p->count > 0)
            printf("%4d %s\n", p->count, p->word);
    return 0;
}

/* binsearch: find word in tab[0]...tab[n-1] */

struct key *binsearch(char *word, struck key *tab, int n)
{
    int cond;
    struct key *low = &tab[0];
    struct key *high = &tab[n];
    struct key *mid;
    while (low < high) {
        mid = low + (high-low) / 2;
        if ((cond = strcmp(word, mid->word)) < 0)
            high = mid;
        else if (cond > 0)
            low = mid + 1;
        else
            return mid;
    }
    return NULL;
}
```

**需要注意：**

- 首先，`binsearch` 函数在声明中必须表明：它返回的值类型是一个指向 `struct key` 类型的指针，而非整型，这在函数原型及 `binsearch` 函数中都要声明。
- 其次，`keytab` 的元素在这里是通过指针访问的。这就需要对 `binsearch` 做较大的修改。
  - `low` 和 `high` 的初值分别是指向表头元素的指针和指向表尾元素后面的一个元素的指针，而且指针不支持加法运算，但减法是合法的，因此计算中间元素的位置更改为`mid = low + (high-low) / 2`
- 结构的长度不等于各成员长度的和，因为不同的对象有不同的对齐要求。假设 char 类型占用一个字节，int 类型占用 4 个字节，则下列结构：`struct {char c;int i;};`可能需要 8 个字节的存储空间，不是 5 个字节。
- 说明一点程序的格式问题：当函数的返回值类型比较复杂时（如结构指针）很难看出函数名，也不太容易使用文本编辑器找到函数名。

```c
struct key *binsearch(char *word, struct key *tab, int n)
/*我们可以采用另一种格式书写上述语句：*/
struct key *
binsearch(char *word, struct key *tab, int n)
```

***

## 6.5 Self-referential Structures

> **自引用的结构**

自引用的结构定义： 一个包含其自身实例的结构是非法的，但是在结构体重声明指向其自身类型的指针是合法的。

```c
struct tnode {
    char *word;
    int count;
    struct tnode *left;
    struct tnode *right;
}
```

自引用结构的一种变体：两个结构相互引用。具体的使用方法如下：

```c
struct t {
    ...
    struct s *p; /* p points to an s */
};
struct s {
    ...
    struct t *q; /* q points to a t */
};
```

> **存储分配程序的问题**

尽管存储分配程序需要为不同的对象分配存储空间，但显然，程序中只会有一个存储分配程序。但是，假定用一个分配程序来处理多种类型的请求，比如指向 `char` 类型的指针和指向 `struct tnode` 类型的指针，**则会出现两个问题**。

- 第一，它如何在大多数实际机器上满足各种类型对象的对齐要求（例如，整型通常必须分配在偶数地址上）
- 第二，使用什么样的声明能处理分配程序必须能返回不同类型的指针的问题？

***

## 6.6 Table Lookup

为了**对结构的更多方面进行深入的讨论**，我们来**编写一个表查找程序包**的**核心部分代码**。这段代码很典型，可以在**宏处理器或编译器的符号表管理例程**中找到。

考虑`#define`语句。当遇到类似于`#define IN 1`之类的程序行时，就需要把名字 `IN` 和替换文本 `1` 存入到某个表中。此后，当名字 `IN` 出现在某些语句中时，如：s`tatet = IN;`就必须用 `1` 来替换 `IN`。

- `install(s, t)`函数将名字 `s` 和替换文本 `t`记录到某个表中，其中 `s` 和 `t` 仅仅是字符串。
- `lookup(s)`函数在表中查找 `s`，若找到，则返回指向该处的指针；若没找到，则返回 `NULL`。

该算法采用的是**散列查找方法**——将输入的名字转换为一个小的非负整数，该整数随后将作为一个指针数组的下标。数组的每个元素指向某个链表的表头，链表中的各个块用于描述具有该散列值的名字。如果没有名字散列到该值，则数组元素的值为 NULL。

**定义链表与指针数组：**

链表中的每个块都是一个结构，它包含一个**指向名字的指针**、一个**指向替换文本的指针**以及一个**指向该链表后继块的指针**。如果指向链表后继块的指针为 NULL，则表明链表结束。

```c
struct nlist { /* table entry: */
    struct nlist *next; /* next entry in chain */
    char *name; /* defined name */
    char *defn; /* replacement text */
};

/**指针数组 */
#define HASHSIZE 101
static struct nlist *hashtab[HASHSIZE]; /* pointer table */
```

**散列函数：**

散列函数 `hash` 在 `lookup` 和 `install` 函数中都被用到，它通过一个 `for` 循环进行计算，每次循环中，它将上一次循环中计算得到的结果值经过变换（即乘以 `31`）后得到的新值同字符串中当前字符的值相加(`*s + 31 * hashval`)，然后将该结果值同数组长度执行取模操作，其结果即是该函数的返回值。

```c
/* hash: form hash value for string s */
unsigned hash(char *s)
{
    unsigned hashval;
    for (hashval = 0; *s != '\0'; s++)
        hashval = *s + 31 * hashval;
    return hashval % HASHSIZE;
}
```

**lookup函数：**

散列过程生成了在数组 hashtab 中执行查找的起始下标。如果该字符串可以被查找到，则它一定位于该起始下标指向的链表的某个块中。

具体查找过程由 `lookup` 函数实现。如果`lookup` 函数发现表项已存在，则返回指向该表项的指针，否则返回 `NULL`

```c
/* lookup: look for s in hashtab */
struct nlist *lookup(char *s)
{
    struct nlist *np;
    for (np = hashtab[hash(s)]; np != NULL; np = np->next)
        if (strcmp(s, np->name) == 0)
            return np; /* found */
    return NULL; /* not found */
}
```

**install方法：**

`install` 函数借助 `lookup` 函数判断待加入的名字是否已经存在。

- 如果已存在，则用新的定义取而代之；
- 否则，创建一个新表项。如无足够空间创建新表项，则 `install` 函数返回`NULL`。

```c
struct nlist *lookup(char *);
char *strdup(char *);
/* install: put (name, defn) in hashtab */
struct nlist *install(char *name, char *defn)
{
    struct nlist *np;
    unsigned hashval;

    if ((np = lookup(name)) == NULL) { /* not found */
        np = (struct nlist *) malloc(sizeof(*np));
        if (np == NULL || (np->name = strdup(name)) == NULL)
                return NULL;
        hashval = hash(name);
        np->next = hashtab[hashval];
        hashtab[hashval] = np;
    } else /* already there */
        free((void *) np->defn); /*free previous defn */
    if ((np->defn = strdup(defn)) == NULL)
        return NULL;
    return np;
}
```

***

## 6.7 Typedef

C 语言提供了一个称为 `typedef` 的功能，它用来**建立新的数据类型名**

```c
typedef int Length;
```

类型 `Length` 可用于类型声明、类型转换等，它和类型 `int` 完全相同。注意，`typedef` 中声明的类型在变量名的位置出现，而不是紧接在关键字 `typedef` 之后。`typedef` 在语法上类似于存储类`extern、static` 等。

**举例：**

```c
typedef struct tnode *Treeptr;
typedef struct tnode { /* the tree node: */
    char *word; /* points to the text */
    int count; /* number of occurrences */
    struct tnode *left; /* left child */
    struct tnode *right; /* right child */
} Treenode;
```

上述类型定义创建了两个新类型关键字：`Treenode`（一个结构）和 `Treeptr`（一个指向该结构的指针）。

**注意：**

- `typedef` 声明并没有创建一个新类型，它只是为某个已存在的类型增加了一个新的名称而已。
- `typedef` 声明也没有增加任何新的语义：通过这种方式声明的变量与通过普通声明方式声明的变量具有完全相同的属性。
- 实际上，`typedef`类似于`#define` 语句，但由于 `typedef` 是由编译器解释的，因此它的文本替换功能要超过预处理器的能力。

```c
/*
语句定义了类型 PFI 是“一个指向函数的指针，
该函数具有两个 char *类型的参数，
返回值类型为 int 
*/
typedef int (*PFI)(char *, char *);
```

除了表达方式更简洁之外，使用 typedef 还有另外两个重要原因：

- 首先，它可以使程序参数化，以提高程序的可移植性。
- typedef 的第二个作用是为程序提供更好的说明性——Treeptr 类型显然比一个声明为指向复杂结构的指针更容易让人理解

***

## 6.8 Unions

联合：是可以（**在不同时刻**）**保存不同类型和长度**的对象的变量，编译器负责跟踪对象的长度和对其要求。**读取的类型必须是最近一次存入的类型**。

**联合的目的：**一个变量可以合法地保存多种数据类型中任何一种类型的对象。

联合是一个结构，它的所有成员相对基地址偏移量都为`0`，此结构空间要大到足够容纳最”宽“的成员，并且，其对齐方式要适合于联合中所有类型的成员。

```c
union u_tag {
    int ival;
    float fval;
    char *sval;
} u;
```

联合只能用其第一个成员类型的值进行初始化,因此，上述联合 u 只能用整数值进行初始化。

可以通过下列语法访问联合中的成员：

```c
联合名.成员
联合指针->成员
```

***

## 6.9 Bit-fileds

在存储空间很宝贵的情况下，有可能需要将多个对象保存在一个机器字中。一种常用的方法是，使用类似于编译器符号表的单个二进制位标志集合。外部强加的数据格式（如硬件设备接口）也经常需要从字的部分值中读取数据。

> **通常采用的方法**

定义一个与相关位的位置对应的“屏蔽码”集合：

```c
#define KEYWORD 01
#define EXTRENAL 02
#define STATIC 04

/*或*/
enum { KEYWORD = 01, EXTERNAL = 02, STATIC = 04 };
```

> **位字段**

位字段：直接定义和访问一个字中的位字段的能力，而不需要通过按位逻辑运算。位字段是字中相邻位的集合。“字”（word）是单个的存储单元，它同具体的实现有关。

```c
struct {
    unsigned int is_keyword : 1;
    unsigned int is_extern  : 1;
    unsigned int is_static  : 1;
}flags;
```

以上定义了一个变量flags，它包含3个一位的字段，冒号后的数字表示字段的宽度（二进制位数）。

单个字段的引用方式与其它结构成员相同，例如：`flags.is_keyword 、flags.is_extern` 等等。字段的作用与小整数相似。同其它整数一样，字段可出现在算术表达式中。

字段的所有属性几乎都同具体的实现有关。字段是否能覆盖字边界由具体的实现定义。字段可以不命名，无名字段（只有一个冒号和宽度）起填充作用。特殊宽度 0 可以用来强制在下一个字边界上对齐.

字段也可以仅仅声明为 int，为了方便移植，需要显式声明该 int 类型是 signed 还是 unsigned 类型。字段不是数组，并且没有地址，因此对它们不能使用&运算符。
