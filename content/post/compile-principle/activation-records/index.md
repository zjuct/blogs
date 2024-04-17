---
title: 'Compile Principle: Activation Records'
description: 虎书第6章——活动记录
date: 2024-04-17T16:36:23+08:00
tags:
    - 编译原理
categories:
    - 课程笔记
draft: false
math: true
---

> 从这一章开始，与其说龙书在介绍编译原理，不如说它在介绍Tiger编译器是如何实现的。整个章节原理讲的少，反倒是大书特书Tiger编译器各个模块的接口定义、功能，以及它们的实现细节。

## 6.1 栈帧

这一节是这一章中少数偏理论的部分，主要有两个要点

1. 逃逸(escape)变量的定义，如何为逃逸变量分配空间
2. 对于Tiger这种支持嵌套函数定义的语言，如何实现内层函数能访问外层函数作用域的变量。这里主要有三种方法：静态链(static links)，嵌套层次显示表(display)，以及$\lambda$提升(lambda lifting)


### 逃逸变量

在为程序中的变量分配空间时，可以将其分配到寄存器中，也可以将其分配到栈空间(栈帧)中。显然在不冲突的情况下，分配到寄存器中更好。但是对于某些变量，我们不得不为其在栈帧中分配空间，这些变量就是逃逸变量。当一个变量满足以下条件之一时，称其为逃逸的：

- 以传引用的形式作为函数调用的实参
- 程序中显式地使用取地址运算符
- 被嵌套函数访问

可以看到，逃逸变量必须有一个显式的内存地址，因此必须在栈帧中为其分配空间

### 静态链

Tiger语言允许嵌套函数定义，其中内层函数能够访问外层函数的变量。由于外层函数的变量定义在其栈帧或者寄存器中，内层函数需要获取其帧指针，然后利用帧指针+偏移来访问外层函数定义的变量。静态链就是帧指针链，在对函数定义进行语义分析时，将该函数的直接外层函数的帧指针以隐式形参的方式传入内层函数。此时，如果内层函数访问到一个外层函数中定义的变量，就可以通过 **帧指针+偏移(偏移量可以在符号表中获得)** 的形式访问这个变量。


```  let
function f(i: int, a: string) = 
    let function g() = 
        let 
        in 
            a := "abc"
        end
    in
        if i < 1
        then g()
        else f(i - 1, a)
    end
in f(2, "123")
end
```

这里函数`g`直接定义在函数`f`内，静态链在函数调用语句中以隐式参数的形式传入，其指向静态程序(而非动态进程)中直接包含`g`的函数(即`f`)的最近一次活动记录。这里要求“最近一次”的原因是，在一个调用链中，函数`f`可能会多次被调用，例如这个例子中`f(2, "123")`一共会调用3次`f`，即动态调用链是`f -> f -> f -> g`，这里就有三个函数`f`的活动记录，而`g`所需要的是第三个`f`的活动记录。

为了实现静态链机制，在中间代码生成的过程中需要做下面几件事

1. 对于函数定义语句，需要定义一个隐式形参`static_link`，并且这个形参必须是escape的（因为外层的静态链会被内层函数访问，由前面逃逸变量的第三个条件，`static_link`是逃逸变量）
2. 对于函数调用语句，需要生成对应的中间代码把活动记录参数作为实参传递给静态链参数
3. 对于变量访问语句，需要判断该变量是否在同一层栈帧中定义，如果不是，则需要沿着静态链向上查找
4. 在中间代码生成的过程中，需要记录当前函数所处的嵌套层数，同时要在符号表中记录每个变量是在哪一层栈帧中定义的，这样在访问变量时才能知道要向上找几层静态链

### 其他处理嵌套函数定义的方式

嵌套层次显示表维护一个全局数组`display`，`display[i]`是一个指针，指向最近一次进入的，静态嵌套深度是`i`的函数的栈帧

可以这样理解`display`的用法，想要调用一个静态嵌套深度为$n$的函数$f_n$，必须先调用静态包含$f_n$的$f_1, f_2, \dots, f_{n-1}$，因此在对$f_n$中语句进行中间代码生成时，`display[i]`就是最近一次调用$f_i$的栈帧。原先使用静态链时，访问一个定义于外层的变量需要沿着静态链向上查找，现在使用display可以直接一步找到该变量定义的那个栈帧(通过符号表可以得到它定义在第$i$层，那么`display[i]`就是想要的那个栈帧)

为了维护display数组的不变性，需要为函数体的transExp生成额外的中间代码。假设被调用的函数f位于第$i$层，那么需要以下几步

1. 将原先的`display[i]`保存到f的栈帧中
2. 将`display[i]`赋值为f的栈帧
3. 正常的`transExp`函数体
4. 恢复原先的`display[i]`

lambda lifting的逻辑更简单，当g调用f时，g中每个实际被f(或被嵌套在f中的任意函数)访问了的变量，都将作为额外的参数传递给f

## 6.2 Tiger编译器中Frames的实现

整体的架构如下所示
```
        semant.c
       translate.h
       translate.c
frame.h         temp.h
μframe.c        temp.c
```

### frame模块

> 首先解决一个问题，为什么语义分析模块需要frame模块，好像在实验中，直到目标代码生成阶段才开始考虑栈帧的问题。
>
> Tiger编译器中的semant模块做两件事，其一是类型检查，其二是中间代码生成。语义分析的输入是AST，如果AST中有一个变量访问结点，该如何将其转化成中间代码？变量可能定义在寄存器中或栈帧中，

frame模块提供以下接口
```C
/* frame.h */
typedef struct F_frame_ *F_frame;
typedef struct F_access_ *F_access;
typedef struct F_accessList_ *F_accessList;
struct F_accessList_ {F_access head; F_accessList tail;};
F_frame F_newFrame(Temp_label name, U_boolList formals);
Temp_label F_name(F_frame f);
F_accessList F_formals(F_frame f);
F_access F_allocLocal(F_frame f, bool escape);
```

其中，`F_newFrame`用于在`transDec`时定义新的栈帧，`F_formals`用于在函数体内部访问形参，`F_allocLocal`用于为局部变量分配栈空间或寄存器

由于具体的栈帧布局由体系结构决定，而语义分析模块要求与目标语言无关，因此frame模块定义为抽象接口，即上层（translate模块）只使用frame.h提供的接口，而不能直接访问栈帧的内容。具体栈帧的实现由mipsframe.c等实现，每个目标平台都有自己的栈帧实现。

虽然不同架构的栈帧格式不同，但F_frame都应当维护以下内容

1. 函数声明的所有形参的位置，这些形参可能位于寄存器上，也可能位于栈帧上
2. 实现"view shift"所需的指令，这些指令会在中间代码生成时被加到函数对应的代码中
3. 已经分配了多大的局部变量空间
4. 函数入口地址对应的符号

#### view shift

下面是一个RISC-V架构下的view shift例子

```C
void f(int a, int b, int c) {
    int* p = &a;
    g(b, c);
    printf("%d %d", b, c);
}
```

由RISC-V calling convention，`a, b, c`会分别通过`a0, a1, a2`寄存器传给f。但是由于a是逃逸变量，因此必须将其从寄存器移入栈帧中，假设其偏移量为K，那么在f函数的头部一定会有一条`sw a0, s0(K)`语句。这里就出现了caller和callee之间的view shift，在caller的角度，变量a位于寄存器a0中，但在callee的角度，变量a位于`MEM[fp + K]`中

另外，b和c作为g的参数，由于a1和a2都是caller-saved寄存器，为了减小开销，寄存器分配器可能会将其移动到其他寄存器中（例如s1, s2），从而避免对a1和a2寄存器进行保护。我们还没到目标代码生成环节，因此这里可以分配虚拟的寄存器，即假设机器拥有无限的寄存器，那么这里可能分配了两个寄存器t100, t101用于存放这两个参数。因此这里也发生了view shift，在caller的角度，b，c位于寄存器a1，a2中，但在callee的角度，它们位于虚拟寄存器t100和t101中

综上，`F_frame`中需要维护以下实现view shift所需的指令

```
sp = sp - K
M[sp + K] = a0
t100 = a1
t101 = a2
```

#### 如何表示变量的位置

`F_access`用于描述形参和局部变量的位置，这些变量可以放在栈中，也可以放在寄存器中。如前所述，语义分析阶段还不知道物理寄存器如何分配，因此这里指的是无限的“抽象寄存器”，即Temp_temp变量，在后面的寄存器分配阶段，这些抽象寄存器会被映射到对应的物理寄存器或内存单元

```C
struct F_access_ {
  enum {inFrame, inReg} kind;
  union {
    int offset; /* InFrame */
    Temp_temp reg; /* InReg */
  } u;
};
static F_access InFrame(int offset);
static F_access InReg(Temp_temp reg);
```

### 局部变量

在生成中间代码阶段，局部变量可以被分配到栈上或者抽象寄存器中，`F_allocLocal(F_frame f, bool escape)`接口用于在指定的栈帧上分配一个局部变量，其中escape参数用于指定该变量是逃逸变量，必须被分配到栈上。对于非逃逸变量，上层的translate模块并不关心它到底是被分配到栈上还是寄存器上，这由具体架构的算法实现

在中间代码生成阶段，每当遇到局部变量定义，都应调用F_allocLocal

### 计算逃逸变量

由于分配局部变量时需要知道它是否是逃逸变量，这通过遍历整个抽象语法树实现，在遍历的过程中，寻找每个变量的逃逸使用。由于中间代码生成和语义分析一同进行，因此逃逸变量分析阶段需要在语义分析之前

```C
/* escape.h */
void Esc_findEscape(A_exp exp);
/* escape.c */
static void traverseExp(S_table env, int depth, A_exp e);
static void traverseDec(S_table env, int depth, A_dec d);
static void traverseVar(S_table env, int depth, A_var v);
```

这里用一个跟符号表类似的table结构env，其维护`Symbol -> (escape, depth)`的映射，表示每个变量定义的静态层级，以及其是否是逃逸变量。当发现在比起定义层级更深的层级中访问某个变量时，说明这个变量是逃逸变量。如果像支持C的取地址运算，只需要在遍历AST时检查所有取地址运算结点即可

### 临时变量和标号

前面已经说了，这个阶段假设有无限的虚拟寄存器可以使用，这个虚拟寄存器就是临时变量`Temp_temp`。与此同时，每条代码最终会位于什么地址在这个阶段也没有得到决议，因此需要label来标记一个地址（跟汇编中的label类似）

```C
/* temp.h */
typedef struct Temp_temp_ *Temp_temp;
Temp_temp Temp_newtemp(void);

typedef S_symbol Temp_label;
Temp_label Temp_newlabel(void);
Temp_label Temp_namedlabel(string name);
string Temp_labelstring(Temp_label s);

typedef struct Temp_tempList_ *Temp_tempList;
struct Temp_tempList_ {Temp_temp head; Temp_tempList tail;}
Temp_tempList Temp_TempList(Temp_temp head, Temp_tempList tail);
typedef struct Temp_labelList_ *Temp_labelList;
struct Temp_labelList_{Temp_label head; Temp_labelList tail;}
Temp_labelList Temp_LabelList(Temp_label head, Temp_labelList tail);
```

`Temp_newtemp`返回一个新的临时变量，`Temp_newlable`返回一个新的标号，临时变量和标号都是无限的。后续的阶段会将临时变量映射到具体的物理寄存器或内存地址

### translate模块和静态链管理

translate模块向semant模块提供中间代码生成服务。由于frame模块已经与具体的目标语言相关，因此其不能再与源语言相关，否则会造成$M*N$的强耦合。因此，静态链管理只能实现在translate模块中（因为不是所有语言都支持嵌套函数定义）

```C
/* translate.h */
typedef struct Tr_access_ *Tr_access;
typedef ... Tr_accessList ...
Tr_accessList Tr_AccessList(Tr_access head, Tr_accessList tail);
Tr_level Tr_outermost(void);
Tr_level Tr_newLevel(Tr_level parent, Temp_label name,
U_boolList formals);
Tr_accessList Tr_formals(Tr_level level);
Tr_access Tr_allocLocal(Tr_level level, bool escape);
```

这里的`Tr_level`就是静态作用域层级

为了实现嵌套函数定义，需要对符号表做以下扩充。可以看到，每个变量和函数现在都维护了它们定义在哪一静态层的信息

```C
struct E_enventry_ {
  enum {E_varEntry, E_funEntry} kind;
  union {
    struct {Tr_access access; Ty_ty ty;} var;
    struct {Tr_level level; Temp_label label;
            Ty_tyList formals; Ty_ty result;} fun;
  } u;
};
```

在中间代码生成阶段，对于函数生成和变量声明，会进行以下处理

- 函数声明：`transDec`会调用`Tr_newLevel`创建一个新的“嵌套层”，`Tr_newLevel`则会调用`F_newFrame`创建一个新栈帧。每个函数所对应的`Tr_level`会被放入符号表中用于后续的查找
- 变量声明：`transDec`会调用`Tr_allocLocal(lev, esc)`创建局部变量，其中`lev`是当前的静态层级，`esc`即是否为逃逸变量。该函数会调用`F_allocLocal`来在当前栈帧中创建局部变量。函数的返回值是`Tr_access`，即变量的层次和位置信息，这些信息会被放入符号表中

显然，在中间代码生成的过程中，需要维护一个变量用于记录当前的静态层级，这可以通过给`transDec`函数加一个`Tr_level`形参实现。由于`transExp`和`transVar`也可能需要当前的静态层级信息，因此其也需要加一个level形参

回顾前面理论部分讲的如何实现静态链：

1. 对于函数定义语句，需要定义一个隐式形参`static_link`，并且这个形参必须是escape的（因为外层的静态链会被内层函数访问，由前面逃逸变量的第三个条件，`static_link`是逃逸变量）
2. 对于函数调用语句，需要生成对应的中间代码把活动记录参数作为实参传递给静态链参数

首先考虑第一点如何实现。函数的形参在`Tr_newLevel`中定义，例如对于函数`f(x, y)`，假设x和y都不是逃逸变量，那么对应的函数调用就是`Tr_newLevel(level, f, U_BoolList(FALSE, U_BoolList(FALSE, NULL)))`，所有的形参串成一个list。因此，要加一个隐式的`static_link`形参，只需要在这个list头部加一个TRUE即可（static_link是逃逸变量），即`F_newFrame(label, U_boolList(TRUE, list))`

再考虑第二点，假设在函数g中调用函数f，可以从符号表中得到g和f所在的静态层级`level_g`和`level_f`，因此只需要用`F_formals(frame)`获取到g的static_link，然后沿着静态链向上查找`level_g - level_f`层，得到的就是f的static_link。因此，只需要把这个逻辑对应的中间代码放到函数调用指令之前即可（这个是龙书7.2.14的内容）
