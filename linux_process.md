# linux process

### 1.int atexit( void (*func)(void) );

atexit函数的参数是一个函数地址，它没有返回值，也无需参数，通过atexit函数可以注册 __退出函数__ atexit函数将以栈的形式将其压入。当一个程序退出的时候会将栈中注册的函数一个个弹出并执行
```C++
#include <stdlib.c>

static void my_exit1(void);
static void my_exit2(void);

int main(){
    atexit(my_exit2);
    atexit(my_exit1);
    atexit(my_exit1);
    return 0;
}
//运行程序，程序退出后将执行2次 my_exit1()，再执行1次my_eixt2()。
```

### 2.pid_t fork( void );

fork 函数用来创建新进程，它返回两次，一次在父进程中，返回值为`子进程PID`，另一次在子进程中，`返回值为0`。进程间的数据互不干扰，采用写时复制的方式。