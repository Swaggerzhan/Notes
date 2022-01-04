# Linux下的Hook技术

### 动态链接库的创建

Linux和Windows都有静态和动态链接库，Linux静态链接库为`.a`文件，动态链接库为`.so`文件，而在Windows下是`.lib`和`.dll`文件。

首先我们使用gcc来生成一个属于自己的动态链接库。[gcc语法]()

头文件

```C++
// libhello.h
void hello();
```
以及源文件

```C++
// libhello.cc
#include <iostream>
void hello(){
    std::cout << "this is my first lib" << std::endl;
}
```

使用命令gcc命令生成动态链接库

```shell
g++ -fpic -shared -o libhello.so libhello.cc
```

使用动态链接库

```C++
#include "libhello.h"
int main(){
    hello();
}
```

```shell
g++ main.cc -L. -lhello
```

不出所料的话会出现找不到hello这个库的问题，这个是由于操作系统在加载动态链接库的时候找不到对应的hello库，这是环境变量的问题。使用ldd即可查看到目标文件所需要的动态链库。

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/路径
```

之后即可正常运行。

### dlopen系列的系统调用

Linux提供一套关于动态链接库句柄的使用，头文件为`dlfcn.h`，其中提供了几个比较重要的系统调用。

##### dlopen

用于打开动态链接库，具体函数声明

```C++
void* dlopen(const char* pathname, int mode);
```

函数调用成功返回对应的句柄，失败返回nullptr。其中第一个参数`pathname`为对应需要的动态链接库的文件地址， __如果我们给定相对地址，则操作系统会从环境变量LD_LIBRARY_PATH中进行寻找，而如果我们直接给定绝对地址，那么不需要通过环境变量__ 。第二个参数`mode`则为打开的模式。

```
RTLD_LAZY   使用时才解析符号
RTLD_NOW    载入时立刻解析符号
RTLD_LOCAL
RTLD_GLOBAL 允许导出符号
RTLD_GROUP
RTLD_WROLD
```
dlopen声明于`dlfcn.h`函数实现于动态链接库`libdl.so`，所以我们需要在编译时加入`-ldl`。

有了句柄，接下来就需要使用了。

##### dlsym

使用dlsym我们可以找到具体函数的地址，声明如下

```C++
void* dlsym(void* handle, const char* symbol);
```

成功返回对应符号的函数地址，失败则返回nullptr。第一个参数`handle`为我们通过dlopen得到的动态库句柄，第二个参数`symbol`为我们需要符号。

##### dlclose

动态链接库加载在内存中，供进程共享，如果调用了dlclose，则将关闭这个动态链接库，但是其真正从内存中卸载取决于操作系统，当操作系统发现还有其他进程使用此动态库的时候，它就不会真正的被`卸载`，这是类似一个引用计数的方式。

```C++
int dlclose(void* handle);
```

传入句柄，关闭对应的动态链接库。

##### dlerror

当我们执行以上函数出错时，通过dlerror可以返回错误信息字符串。

```C++
const char* dlerror(void);
```

之前函数有发生错误，则返回对应字符串地址，如果成功则返回nullptr。

### dlopen使用

生成了动态链接库，我们就可以通过dlopen来使用它了。

```C++
#include <dlfcn.h>
#include <iostream>
typedef void (*hello_t)();
int main() {
    void* handle = dlopen("/root/share_lib/libhello.so", RTLD_LAZY);
    if ( handle == nullptr ){
        std::cout << "load lib error" << std::endl;
        return 1;
    }
    hello_t hello;
    hello = (hello_t)dlsym(handle, "hello"); // decode test symbol
    const char* er = nullptr;
    if ( (er = dlerror()) != nullptr ){
        std::cout << er << std::endl;
        return 2;
    }
    hello(); // 调用
    return 0;
}
```

由此生成对应可执行文件，我们即可运行，这种方法和一开始的方法有所不同，不用使用头文件也可以调用对应函数。接下来就可以进行一些hook相关的操作了。

### 劫持fgets函数

先看一下fgets函数

```C++
char* fgets(char* str, int n, FILE* stream);
```

写一个简单的读取输入的程序

```C++
// main.cc
#include <cstdio>
#include <cstdlib>

int main(){
    char buf[50];
    fgets(buf, 50, stdin);
    return 0;
}
```

这是一个很简单的程序，其调用`libc.so.6`动态链接库中的fgets函数，现在通过之前dlsym将其hack掉。

```C++
// hook.cc
#include <dlfcn.h>
#include <cstdlib>
#include <cstdio>
typedef char*(*FGETS)(char*, int, FILE*);

char* fgets(char* str, int n, FILE* stream){
    void* func = dlsym(RTLD_NEXT, "fgets");
    if ( func == nullptr ){
        return nullptr;
    }
    printf("you hooked!\n");
    return ((FGETS)(func))(str, n, stream);
}
```

通过hook.cc文件中，可以看到我们使用`dlsym`来寻找下一个动态链接库中的`fgets`函数，使用的是`RTLD_NEXT`来进行寻找，那么问题来了，如果我们直接使用命令行来生成对应的libhook.so文件就行了么？

```shell
g++ -o libhook.so -fpic -shared -ldl
```

运行发现根本没有跳转到我们想要的fgets函数，这是由于libhook.so并不是当前程序的首要链接库，既然我们要通过`RTLD_NEXT`来寻找真正的fgets，那就首先要将我们创建的动态链接库放到第一位，或者说优先加载才行。

##### LD_PRELOAD

使用环境变量`LD_PRELOAD`指定的动态链接库，可以优先的将其加载到程序中去，不过这里不建议直接这样做，会污染当前的环境变量的，建议写到一个`.sh`文件中另起一个进程来做比较好。

使用export命令加载优先的动态链接库后再运行程序就可发现fgets函数已被劫持了。

```shell
export LD_PRELOAD=/root/hook/libhook.so
./a.out
```



