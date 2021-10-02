# AT&T汇编

先看一个简单的例子

```asm
# first.s
.data
hello: .ascii "Hello World\n"
.text
.type main, @function
.globl  main
main:
        movl $4, %eax
        movl $1, %ebx
        movl $hello, %ecx
        movl $12, %edx
        int $0x80
```

使用gcc进行链接操作

```shell
gcc first.s
```

运行即可看到打印了Hello World，但是以上还是有些问题，即没有处理退出，这些后续再来细分。

除了使用gcc，我们还可以使用as和ld来进行汇编链接操作，这回我们就需要将main改为_start，此函数才是真正的入口。gcc的话会直接寻找main作为入口。

```shell
$ as -o first.o first.s
$ ld -o first first.o
$ ./first
$ Hello World
```

### AT&T的语法

首先例子中的`.`开头的并不是真正的代码，而是给汇编器的一些提示，比如`.data`表明此数据应该放到`data`段，`.text`表明应该放到`text`段。

* .golbl
    这是一个类似C语言中的声明作用，当汇编完成后给链接器看的标记，比如`.globl main`将使得链接器可以看到`main`这个函数。
    
* .type
    `.type main, @function`声明了main是一个函数，这是GNU汇编器中的函数定义，如果你想写出一个可以让C语言进行调用的汇编函数，就必须用此进行标记。
    
    
### 系统调用

```asm
movl $4, %eax
movl $1, %ebx
movl $hello, %ecx
movl $12, %edx
int $0x80
```

以上其实就是一个32位下x86的系统调用过程，首先， __对于32bit的系统调用采用int 0x80系统中断来进行，它通过eax寄存器来传递函数表值，之后的函数参数则采用ebx, ecx, edx, esi, edi ebp__ 来进行参数传递，并且返回结果将方法到eax中回传至用户态。对于32bit的Intel CPU，貌似限制只能传参6个，也有说当系统调用参数大于6个时，需要将参数放到一个连续的内存区域中，并统一由ebx指向该区域。

现在就可以解析关于上述汇编的系统调用过程了，`movl $4 %eax`，对应32位系统调用函数表中sys_write函数，也就是C语言中比较常用的write系统调用。

```C++
size_t wirte(int fd, char* buf, size_t len);
```

接着，`movl $1, %ebx`传递参数fd，`movl $hello, %ecx`传递参数buf，以及`movl $12, %edx`传递参数len。之后调用`int 0x80`进入系统调用。

对于64位操作系统，它的函数表和32位的不同，并且中断方式也不同，它采用了syscall的方式，函数表则一样由rax进行传递，它的参数传递为 __rdi, rsi, rdx, rcx, r8, r9__ 。

现在就可以解释一下为何以上代码使用`as ld`配合进行汇编的代码执行后会出现Segmentation fault了，这是由于没有调用sys_exit系统调用的原因，而在32位下的系统调用表中，1代表sys_exit，所以，你只需要加上一下代码，程序就能正常退出了。

```asm
movl $1, %eax # sys_exit
int 0x80
```

以上，我们讨论的是关于系统调用的传递参数方式，如果是用户态函数调用，64位下基本不变，而32位略有出入，如果使用__cdecl的话(C调用一般都是这个规定)，则使用栈传递。

### 编写C能够调用的汇编函数

首先我们写一个简单汇编代码，用来打印一些字符串

```asm
# show.s
.section .data
hello: .ascii "Hello World\n" # 12

.section .text
.type show @function
.globl show # make ld find this symbol

show:
    
    pushl %ebp
    movl %esp, %ebp

    movl $4, %eax # call table index
    movl $1, %ebx # fd 1
    movl $hello, %ecx
    movl $12, %edx # str len
    int $0x80
    
    movl %ebp, %esp
    popl %ebp
    ret
```

编写C语言调用对应的汇编函数

```C++
extern void show();
int main(){
    show();
    show();
}
```

以上调用完全正常，只不过这里留一个坑，也就将指令ret去掉后，居然不会段错误，调试发现当EIP寄存器到结尾的nop指令时，会跳到 __libc_csu_init这个函数中去，具体原因不得知。

