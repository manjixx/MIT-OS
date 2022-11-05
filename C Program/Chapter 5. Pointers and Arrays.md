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

***

## 5.3 Pointers and Arrays

***

## 5.4 Address Arithmetic

***

## 5.5 Character Pointers and Functions

***

## 5.6 Pointer Arrays; Pointers to Pointers

***

## 5.7 Multi-dimensional Arrays

***

## 5.8 Initialization of Pointers Arrays

***

## 5.9 Pointers vs Multi-dimensional Arrays

***

## 5.10 Command-line Arguments

***

## 5.11 Pointers to Functions

***

## 5.12 Complicated Declarations