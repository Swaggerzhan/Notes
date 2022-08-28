# C/C++ backtrace

在运行中打印函数的调用栈

```c
#include <execinfo.h>
int backtrace(void** array, int size);

char** backtrace_symbols(void* const* array, int size);
```

backtrace只能拿到函数的运行时指针，大致用法：

```c
void func() {
    const int cap = 128;
    void* func_pointer[cap];
    int length = backtrace(func_pointer, cap);
    
    for (int i=0; i<length; ++i) {
        printf("%p\n", func_pointer[i]);
    }
}
```

或者也可以通过backtrace_symbols来拿到一点符号信息，但信息也是非常有限：

```c
void func() {
    const int cap = 128;
    void* func_pointer[cap];
    int length = backtrace(func_pointer, cap);
    char** symbols = NULL;
    symbols = backtrace_symbols(symbols, length);
    
    for (int i=0; i<length; ++i) {
        printf("%p\n", func_pointer[i]);
        printf("%s\n", symbols[i]);
    }
    // TODO: clean symbols mem
}
```

一般可以拿到类似这种的信息：

```shell
./main(+0xf5599) [0x562e92025599]
./main(+0x14a615) [0x562e9207a615]
```

可以通过addr2line来转换成更细节的：

```shell
addr2line -a 0xf5599 -e ./main -f -i 
```

不过有时候这些offset并不是以main为基础的，那么使用addr2line就不能实现转化了。

但我们同样有一些比较好用的东西，通过objdump找到对应的函数的base地址，然后加上偏移量，这样就可以拿到真正的地址了。


