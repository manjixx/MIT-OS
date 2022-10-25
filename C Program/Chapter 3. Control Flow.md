# Control Flow

## 3.1 Statements ans Blocks

- 在 C 语言中，分号是语句结束符
- 用一对花括号“{”与“}”把一组声明和语句括在一起就构成了一个复合语句（也叫作程序块），**复合语句在语法上等价于单条语句。** 
- 右花括号用于结束程序块，其后不需要分号。

***

## 3.2 If-Else

```c
if {表达式}
    语句 1
else // 可选
    语句 2
```

***

## 3.3 Else-If

```c
if (表达式)
    语句
else if (表达式)
    语句
else if (表达式)
    语句
else if (表达式)
    语句
else
    语句
```

***

## 3.4 Switch

switch 语句是一种多路判定语句，它测试表达式是否与一些常量整数值中的某一个值匹配，并执行相应的分支动作。

```c
switch (表达式) {
case 常量表达式: 语句序列
case 常量表达式: 语句序列
default: 语句序列
```

- 如果某个分支与表达式的值匹配，则从该分支开始执行。各分支表达式必须互不相同。
- 如果没有哪一分支能匹配表达式，则执行标记为 default 的分支。default 分支是可选的。
- 如果没有 default 分支也没有其它分支与表达式的值匹配，则该 switch 语句不执行任何动作。
- 各分支及 default 分支的排列次序是任意的。
- 在 switch 语句中，case的作用只是一个标号，**因此，某个分支中的代码执行完后，程序将进入下一分支继续执行，除非在程序中显式地跳转.**
  
***

## 3.5 Loops-While and For

while 与 for 这两种循环在循环体执行前对终止条件进行测试。

```c
// for 循环语句;
for (表达式 1; 表达式 2; 表达式 3)
    语句
//它等价于下列 while 语句：
// 但当 while 或 for 循环语句中包含 continue 语句时，上述二者之间就不一定等价了。
表达式 1;
while (表达式 2) {
    语句
    表达式 3;
}
```

shell排序算法

```c
/* shellsort: sort v[0]...v[n-1] into increasing order */
void shellsort(int v[], int n)
{
    int gap, i, j, temp;
    for (gap = n/2; gap > 0; gap /= 2)
        for (i = gap; i < n; i++)
            for (j=i-gap; j>=0 && v[j]>v[j+gap]; j-=gap) {
                temp = v[j];
                v[j] = v[j+gap];
                v[j+gap] = temp;
            }
}
```

***

## 3.6 Loops-Do-while

do-while 循环则**在循环体执行后测试终止条件**，这样循环体至少被执行一次。
do-while 循环的语法形式如下：

```c
do
    语句
while (表达式);

在这一结构中，先执行循环体中的语句部分，然后再求表达式的值。如果表达式的值为真，则再次执行语句，

```

***

## 3.7 Break and Continue

break 语句可用于从for、while与do-while等循环中提前退出，就如同从switch语句中提前退出一样。

break语句能使程序从 switch 语句或最内层循环中立即跳出。

continue 语句用于使 for、while 或 do-while 语句开始下一次循环的执行。

- 在 while 与 do-while语句中，continue 语句的执行意味着立即执行测试部分；
- 在 for 循环中，则意味着使控制转移到递增循环变量部分。
- continue 语句只用于循环语句，不用于 switch 语句。某个循环包含的 switch 语句中的 continue 语句，将导致进入下一次循环。

***

## 3.8 Goto and Labels

C 语言提供了可随意滥用的 goto 语句以及标记跳转位置的标号。从理论上讲，goto 语
句是没有必要的，实践中不使用 goto 语句也可以很容易地写出代码。

goto 最常见的用法是终止程序在某些深度嵌套的结构中的处理过程，例如一次跳出两层或多层循环。

```c
for ( ... )
    for ( ... ) {
        ...
        if (disaster)
            goto error;
        }
    ...
error:
/* clean up the mess */
```

在该例子中，如果错误处理代码很重要，并且错误可能出现在多个地方，使用 goto 语句将会比较方便。

标号的命名同变量命名的形式相同，标号的后面要紧跟一个冒号。
标号可以位于对应的goto 语句所在函数的任何语句的前面。
标号的作用域是整个函数。
