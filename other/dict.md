<font face="Monaco">

# CSAPP 笔记

## 0x00 Register name && Some assembly

### Register name

R for Register, E for Extend

* ax : accumulate
* cx : counter
* dx : data
* bx : base
* si : source index
* di : destination index
* sp : stack pointer
* bp : base pointer
* ip : instruction pointer

在x64中加入的：

* r8 - r15

flag标记位

* cf : carry flag (for unsigned)
* sf : sign flag (for signed)
* zf : zero flag 
* of : overflow flag

flag位被置位的情况例如：

```C++
int t = a + b;
// t 超出最大位(溢出)，这时可能会出现t小于0(有符号)，sf置为1
// t 为0， zf置为1
// 
```

lea指令不会修改flag位。

### Assembly

汇编中的操作大小一般体现在指令后缀中：

```
1. char   -> b
2. short  -> w : word
4. int    -> l 
8. long   -> q : quad word
```

#### 1. mov

mov中有3种类型的操作，分别为立即数，内存地址，寄存器，仅能有以下操作方式：

![](./dict_pic/register.png)

mov操作数组：

![](./dict_pic/register2.png)

#### 2. lea

load effect address，类似C/C++中的&。

和mov类似，lea同样可以配合一些()来做一些取地址的操作，比如：

```assembly
leaq (%rdi, %rsi), %rax # rdi地址处和rsi地址处值相加，并且赋值到rax中
```

这种语法为：t(x, y, z) -> x + (y * z) + t

有时候，我们会通过这种小技巧来计算一些简单的算术运算，比如我们不在寄存器中存入地址，而是存入一个“值”，然后通过lea来计算地址的方式来计算一个“地址”(但其实是我们放入的一个值)，进而可以减少指令的数量。

比如这样的代码：

```C++
int add(int x, int y){
    return x + y;
}
```
gcc在Og编译下会得到这样的汇编：

```assembly
leal (%rdi, %rsi), %eax
ret

# 或者，我们可以这样写：
movq %rdi, %rax
addq %rsi, %rax
ret
```

编译器巧妙的利用了原本应用于计算地址的lea指令来做了加法的操作，进而减少了指令的数量。


#### 3. 其他算术指令

![](./dict_pic/operand.png)
![](./dict_pic/operand2.png)
#### 4. call && ret && stack frame

call指令有2个隐式的操作，分别是：

* push返回地址
* 设定rip为调用函数的起始地址

ret则和call是一个相反的操作：

* pop返回地址到rip中

在x64中，函数的前6个参数由寄存器来进行传入(fastcall仅有4个)，优先级分别是：

* %rdi
* %rsi
* %rdx
* %rcx
* %r8
* %r9

超过6个参数的部分将由stack来保存，并且由Arg n开始推入，然后是Arg n-1 ... 直到Arg 7，函数的返回值将由%rax来进行保存。

stack frame的方式则是为记录每个function所使用的stack区域，一般而已，调用的过程为：

* caller准备函数需要的参数
* call指令运行，ret地址被push到stack中
* rbp被push到stack中
* rbp被修改为当前的rsp，从这里开始，为callee的stack

而一些寄存器的值在经过一个函数调用后就将有可能改变，如果需要保存这些值，就需要进行保存，分别有：

* caller保存

在call指令准备之前，提前将需要保存的寄存器push入stack，即保存在caller stack frame中，后续ret返回后再重新从stack中pop出原先保存的值到对应的寄存器中。



* callee保存

在进入call指令后，立刻将需要保存寄存器push入stack，即保存在callee stack frame中，在要ret返回之前，从stack中pop出原先保存的值到对应的寄存器中。

需要由callee的有：

> %rbx, %r12, %r13, %r14, %rbp

</font>