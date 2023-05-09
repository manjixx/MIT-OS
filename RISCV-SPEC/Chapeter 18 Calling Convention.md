# Calling Convention

本章介绍了RV32和RV64程序的**C编译器标准**和**两种调用约定(Calling Covention)**

- RVG Calling Convention:基本ISA和标准通用扩展的约定（RV32G/RV64G）
- Soft-Float Calling Convention:针对缺乏浮点单元的实现的软浮点约定（RV32I/RV64I）

## 一、C语言数据类型与对齐方式(Alignment)

| C type      | Description              | Bytes in RV32 | Bytes in RV64 |
| ----------- | ------------------------ | ------------- | ------------- |
| char        | Character value/byte     | 1             | 1             |
| short       | Short integer            | 2             | 2             |
| int         | Integer                  | 4             | 4             |
| long        | Long Integer             | 4             | 8             |
| long long   | Long  long Integer       | 8             | 8             |
| void*       | pointer                  | 4             | 8             |
| float       | Single-precision float   | 4             | 4             |
| double      | Double-precision float   | 8             | 8             |
| long double | Extended-precision float | 16            | 16            |

- 在 RV32 和 RV64 C 编译器中，C 类型 `int` 都是 32 位宽。
- `long` 和`pointer`都与整数寄存器一样宽，因此在 RV32 中，它们都是 32 位宽，而在 RV64 中，它们都是 64 位宽。等效地，RV32 采用 ILP32 整数模型，而 RV64 是 LP64。
- 在RV32和RV64中，C类型`long long`是64位整数
- `float`是 32 位 IEEE 754-2008 浮点数
- `double`是 64 位 IEEE 754-2008 浮点数
- `long double` 是一个 128 位 IEEE 浮点数
- `char` 和 `unsigned char` 是 8 位无符号整数，存储在 RISC-V 整数寄存器中时**进行零扩展**。`signed char` 是一个 8 位有符号整数，当存储在 RISC-V 整数寄存器中时**进行符号扩展**，即位 `(XLEN-1)..7` 都相等。
- `unsigned short` 是一个 16 位无符号整数，存储在 RISC-V 整数寄存器中时进行零扩展。`short` 是一个 16 位有符号整数，存储在寄存器中时进行符号扩展。
- 在 `RV64` 中，32 位类型（例如 int）作为其 32 位值的适当符号扩展存储在整数寄存器中；也就是说，位 63..31 都相等。此限制甚至适用于无符号 32 位类型。

RV32 和 RV64 C 编译器和兼容软件使上述所有数据类型在存储在内存中时自然对齐。

## 二、RVG Calling Convention

`RISC-V` 调用约定**尽可能在寄存器中传递参数**。最多八个整数寄存器 `a0–a7` 和最多八个浮点寄存器 `fa0–fa7` 用于实现该目的。

如果**函数的参数**被概念化为 `C struct`，每个字段都具有指针对齐，则参数寄存器是该`struct`前八个指针字(`pointer-words`)的影子。

如果参数 `i < 8` 是**浮点型**，则传入浮点寄存器 `fai`；否则，它被传递到整数寄存器 `ai`。**如果浮点型参数** 作为`union`或结构的数组字段的一部分(`array fields of structures`)将在整数寄存器中传递。**可变参数函数的浮点参数**（参数列表中明确命名的那些除外）在整数寄存器中传递。

**小于指针字(`pointer-words`)的参数**在参数寄存器的最低有效位中传递。在堆栈上传递的子指针字(`sub-pointer-word`)参数出现在指针字(`pointer-words`)的较低地址中，因为 RISC-V 具有**小端存储系统**。

**当两倍于指针字(`pointer-words`)大小的原始参数(`primitive arguments`)**在堆栈上传递时，它们自然对齐。当它们在整数寄存器中传递时，它们位于对齐的奇偶寄存器对(`even-odd register pair`)中，**偶寄存器**保存最低有效位。例如，在 RV32 中，函数 `void foo(int, long long)` 在 `a0` 中传递第一个参数，在 `a2 和 a3` 中传递第二个参数。 `a1` 中没有传递任何内容。

**大于指针字(`pointer-words`)大小两倍的参数通过引用传递**。

未在参数寄存器中传递的概念结构体(`conceptual struct`)部分在堆栈上传递。堆栈指针 `sp` 指向未在寄存器中传递的第一个参数。

从**函数中返回的值**在整数寄存器`a0`和`a1`以及浮点寄存器`fa0`和`fa1`中。只有当`floating-point`是`primitives`或是仅有由一个或两个浮点值组成的`struct`的成员时，**浮点值**才会在浮点寄存器中返回。适合两个指针字(`pointer-words`)的**其他返回值**在 `a0 和 a1` 中返回。**较大的返回值**完全在内存中传递；调用者分配此内存区域并将指向它的指针作为隐式第一个参数传递给被调用者。

在标准的 RISC-V 调用约定中，堆栈向下增长，堆栈指针始终保持 16 字节对齐。

除了参数和返回值寄存器之外，**7 个整数寄存器 t0-t6** 和 **12 个浮点寄存器 ft0-ft11** 是临时寄存器，在调用过程中是易变的，如果以后使用，必须由调用者保存。

12 个整数寄存器 s0–s11 和 12 个浮点寄存器 fs0–fs11 在调用之间保留，如果使用，则必须由被调用者保存。

表 18.2 指出了每个整数和浮点寄存器在调用约定中的作用。

**Table 18.2: RISC-V calling convention register usage**

| Register | ABI Name | Description                      | Saver  |
| -------- | -------- | -------------------------------- | ------ |
| x0       | zero     | Hard-wired zero                  | —      |
| x1       | ra       | Return address                   | Caller |
| x2       | sp       | Stack pointer                    | Callee |
| x3       | gp       | Global pointer                   | —      |
| x4       | tp       | Thread pointer                   | —      |
| x5–7     | t0–2     | Temporaries                      | Caller |
| x8       | s0/fp    | Saved register/frame pointer     | Callee |
| x9       | s1       | Saved register                   | Callee |
| x10–11   | a0–1     | Function arguments/return values | Caller |
| x12–17   | a2–7     | Function arguments               | Caller |
| x18–27   | s2–11    | Saved registers                  | Callee |
| x28–31   | t3–6     | Temporaries                      | Caller |
| f0–7     | ft0–7    | FP temporaries                   | Caller |
| f8–9     | fs0–1    | FP saved registers               | Callee |
| f10–11   | fa0–1    | FP arguments/return values       | Caller |
| f12–17   | fa2–7    | FP arguments                     | Caller |
| f18–27   | fs2–11   | FP saved registers               | Callee |
| f28–31   | ft8–11   | FP temporaries                   | Caller |

## 三、Soft-Float Calling Convention

软浮点调用约定(`soft-float calling convention`)用于缺乏浮点硬件的RV32和RV64实现。它避免使用F、D和Q标准扩展中的所有指令，因此也避免使用f寄存器。

**整数参数**的传递和返回方式与RVG惯例相同，堆栈规则也相同。

**浮点参数**在整数寄存器中传递和返回，使用相同大小的整数参数的规则。

例如，在`RV32`中，函数`double foo(int, double, long double)`的第一个参数在`a0`中传递，第二个参数在`a2`和`a3`中传递，第三个参数通过`a4`引用；其结果在`a0和a1`中返回。在`RV64`中，参数在`a0、a1`和`a2-a3`对中传递，结果在`a0`中返回。

动态四舍五入模式(`dynamic rounding mode`)和应计异常标志(`accrued exception`)是通过`C99`头文件`fenv.h`提供。
