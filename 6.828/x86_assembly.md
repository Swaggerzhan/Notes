
# x86 assembly syntax(AT&T)

xv6-public是x86架构的教学操作系统，学习前先记录下一些可能用到的x86汇编语法吧，主要是以AT&T语法为主。

## 数据定义

### 普通的变量 

```assembly
.data # 定义一个data段
var:
    .byte 64 # 定义一个8bit的内存空间用来保存64这个数值，地址可以使用var
    .byte 10 # 定义一个8bit的内存空间用来保存10这个数值，地址可以使用var+1
x:
    .short 42 # 16bit，其余同上
y:
    .long 30000 # 32bit，其余同上
```
### 数组和其他数值

汇编中并没有直接提供类似高级语言的数组，要实现类似的功能，汇编可以定义一片连续的地址空间用于保存相同的值。

```assembly
s:
    .long 1, 2, 3 # 定义3个32bit的数字，地址分别为s、s+1、s+2
barr:
    .zone 10 # 定义80bit(10bytes)的空间，使用0填充，地址为barr
str:
    .string "hello" # 定义6bytes的空间，保存字符串hello + \0
```

## 一些指令

### mov

AT&T的风格中，mov首先的参数是src，然后才是dst，跟Intel的风格是相反的，例如：

```assembly
#    src    dst
mov (%ebx), %eax # 将ebx寄存器中地址指向的内存处4bytes数据挪到eax寄存器中
mov %ebx, var(,1) # 将ebx寄存器数据挪到var的地址处
mov -4(%esi), %eax # 将esi+(-4)的地址指向的内存区域中4bytes挪到eax中
mov %cl, (%esi,%eax,1) # 挪动cl寄存器的数据到esi+eax指向的地址区域
mov (%esi,%ebx,4), %edx # 挪动esi+4*ebx地址处的内容到edx寄存器中
```
需要指出的是，()中的风格，计算方式一般为：

```
(基址, 变址, 比例因子) => 基址 + (变址*比例因子)
比如：
mov %cl, (%esi,%eax,1) => 挪动cl数据到内存中，地址为：esi + (eax*1)
mov %ebx, var(,1) => 挪动ebx数据到内存中，地址为：var 0 + (0*1) 
```

需要注意的是，比例因子在x86上只能是1、2、4、8，相对于bit，就是不变、左移动1bit、2bit、3bit。

mov指令也提供了其他变种，用于提供挪动不同的长度，默认情况下，mov指令如果有跟随寄存器，那么指令操作的byte数一般和寄存器一样，但是有些情况下就不好搞了，比如：

```assembly
mov $2, %ebx # 4 bytes
mov $2, (%ebx) # oops 异常
```

mov指令可以跟随一些操作后缀，用于指定mov所操作的长度

```
movb $2, (%ebx) # 挪动1 byte
movw $2, (%ebx) # 挪动2 byte
movl $2, (%ebx) # 挪动4 byte
movq $2, (%ebx) # 挪动8 byte(x64)
```

## 指令语法

在汇编中，指令操作数据的参数类型，总结起来大概就是下面几种(暂时就考虑32bit，64bit的CPU其实也差不多)，如果需要更详细的，可以查阅一下Intel手册

```
<reg32> 各种32bit的寄存器，如%eax、%ebx、%ecx、%edx等
<reg16> 各种16bit的寄存器，如%ax、%bx、%cx、%dx等
<reg8> 各种8bit的寄存器，如%ah、%al、%bh、%bl等
<reg> 各种寄存器
<mem> 内存数，比如(%eax)、4+var(,1)、(%eax,%ebx,1)
<con> 立即数
<con8> 立即数8bit
<con16> 立即数16bit
<con32> 立即数32bit
```
### 数据操作
#### mov
syntax:
```
mov <reg>, <reg>
mov <reg>, <mem>
mov <mem>, <reg>
mov <con>, <reg>
mov <con>, <mem>
```
example:
```assembly
mov %ebx, %eax
mov $5, var(,1)
```

#### push
syntax:
```
push <reg32>
push <mem>
push <con32>
```
example:
```assembly
push %eax
push var(,1)
```
注：push默认是和架构相关的，比如32bit的CPU，push默认做32bit数据操作，但是也能通过pushw来做16个bit的操作，x64也是同理，默认64bit，可以使用pushl来操作32bit、pushw来操作16bit。

#### pop
syntax:
```
pop <reg32>
pop <mem>
```
example:
```assembly
push %eax
push (%eax)
```
注：操作长度可以类似push。

#### lea
syntax:
```
lea <mem>, <reg32>
```
example:
```assembly
lea (%ebx, %esi, 8), %edi
lea val, %eax
```
注：lea和mov不同，lea表示load effective address，不会进行地址访问，例如下面的例子比较：

```assembly
lea (%ebx, %esi, 8), %edi # 内容是ebx + esi*8
mov (%ebx, %esi, 8), %edi # 内容是ebx + esi*8地址处的数据，访问内存了

lea val, %eax # 内容是val
mov val, %eax # 内容是val地址处的数据，访问内存了
```

### 逻辑运算相关的指令

TODO

### 流程控制相关的指令

流程相关的指令和EIP这个寄存器有很大关系，32bit CPU是叫EIP，64bit就是RIP，EIP/RIP不能直接修改，但是能通过其他指令间接修改。

汇编中可以在代码的任意代码、数据片段插入一个label，就比如：

```assembly
    mov 8(%ebp), %esi
begin:
    xor %ecx, %ecx
    mov (%esi), %eax
```
其中的begin就是label

#### jmp
syntax
```
jmp <label>
```
example:
```assembly
jmp begin
```
注：jmp是无条件跳转的

#### 条件跳转
条件跳转的语法和jmp一样，只不过名字不一样，条件也不一样，有以下几种条件：
```
je <label> 
jne <label>
jz <label>
jg <label>
jge <label>
jl <label>
jle <label>
```
TODO
#### cmp
TODO
#### call,ret
syntax
```assembly
call <label>
ret
```

call和ret可以看作类似于push+jmp和pop+jmp，其操作的bit数量固定是cpu的架构。

call指令调用时，当前指令的下一条指令地址被push到stack中，然后jmp到label处执行；ret指令调用时，pop stack中的地址到eip/rip中，然后我们自然而然的回到了机器运行call时的下一条指令。

##### 调用规则(C)

TODO


# ref
https://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html