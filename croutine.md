# routine

协程，是一种可以在用户态调度的一种轻量级线程，由于用户态间的调度快，所以比线程轻量，接下来将使用汇编来实现一些简单的用户态上下文切换，具体[汇编语法](./assemble.md) 。

### 寄存器

寄存器是上下文切换的时候必定要保存的，当切换回来的时候，回复寄存器中的值从而重新从切换前的位置再次开始运行。

其中有8个寄存器一定要存起来，用于后续恢复使用。除此之外，我们还需要保存每个协程的堆栈信息，这里我们使用的是申请堆上内存作为栈使用，默认为128k。

```C++
struct _Rroutine{
    void* reg_rbx;
    void* reg_rsp; // 栈顶寄存器
    void* reg_rbp; // 栈底寄存器
    void* reg_r12;
    void* reg_r13;
    void* reg_r14;
    void* reg_r15;
    void* reg_rip; // 指令寄存器
};
```

这样，我们就可以简单的写出一个上下文保存的汇编函数，在C语言中，我们将其定义为`void swap_context(void* out, void* in);`，如果调用成功，那么函数将不会返回，将直接跳转到in的上下文中去执行，并且会将当前的上下文保存至out中。

```asm
# swap.s
.section .text
.type swap_context @function
.globl swap_context

swap_context:
    mov 0x00(%rsp), %rdx # rip
    lea 0x08(%rsp), %rcx # rsp
    mov %rbx, 0x00(%rdi)
    mov %rcx, 0x08(%rdi)
    mov %rbp, 0x10(%rdi)
    mov %r12, 0x18(%rdi)
    mov %r13, 0x20(%rdi)
    mov %r14, 0x28(%rdi)
    mov %r15, 0x30(%rdi)
    mov %rdx, 0x38(%rdi)

    mov 0x00(%rsi), %rbx
    mov 0x08(%rsi), %rsp
    mov 0x10(%rsi), %rbp
    mov 0x18(%r12), %r12
    mov 0x20(%r13), %r13
    mov 0x28(%r14), %r14
    mov 0x30(%r15), %r15
    jmpq *0x38(%rsi)
```

```C++
#include <cstdlib>
#include <iostream>

extern "C" void swap_context(void*, void*);

typedef struct _Rroutine{
    void* reg_rbx;
    void* reg_rsp; // 栈顶寄存器
    void* reg_rbp; // 栈底寄存器
    void* reg_r12;
    void* reg_r13;
    void* reg_r14;
    void* reg_r15;
    void* reg_rip; // 指令寄存器
} Routine;

Routine main_ctx, child_ctx;

void func(){
    printf("Wait\n");
    swap_context(&child_ctx, &main_ctx);
    printf("Wait2\n");
    swap_context(&child_ctx, &main_ctx);

}

int main() {
    void* stack = malloc(1024 * 1024 * 10);
    // 栈的空间从高向下生长
    child_ctx.reg_rsp = (void*)((char*)stack + 1024 * 1024 * 10);
    child_ctx.reg_rip = (void*)func;
    swap_context(&main_ctx, &child_ctx);
    printf("Resume\n");
    swap_context(&main_ctx, &child_ctx);
    return 0;
}
```

使用命令行

```shell
$ as swap.s -o swap.o
$ g++ -g main.cc swap.o
$ ./a.out # 运行得到结果
Wait
Resume
Wait2
```

至此，一个比较简单的上下文保存切换就完成了。