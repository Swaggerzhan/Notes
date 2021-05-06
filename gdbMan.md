# gdb使用经验

### 1.常用的命令

* break 下断点
    缩写为b，对代码下断点有许多种办法，可以从行下断点，也可以从函数名下断点

```C++
/* main.cpp */
1 #include <ctime>
2 #include <iostream>
3
4 int main(){
5    srand(time(nullptr));
6    int data* = new int[100];
7    for (int i=0; i<100; ++i)
8        data[i] = rand() % 100;
9       
10   delete [] data;
11 }
```

上述事例中可以使用`b 6` 直接在代码第6行处下断点，也可以使用`b main`直接在main函数处也就是第4行下断点。如果是多源码下断点可以使用`b "文件路径+文件名.cpp:行数"`进行下断点。

* continue 执行
    缩写为c，对于当前停下的断点，使用c可以直接运行到下一个断点的地方。

* next 下一步
    缩写为n，对于当前停下的断点位置，使用n可以单步执行一行代码，不会进入函数。
    
* step 进入函数
    缩写为s，对于当前停下的断点位置，如果是一个函数，使用s可以进入其函数内部，
    进入内部后也可使用n进行单步执行
    
* info 查看当前信息
    * 使用 `info b` 可以查看目前所有的断点。
    * 使用 `info watchpoint` 可以查看所有的观察点。

* shell 命令
    使用 `shell + command` 可以直接执行linux的命令。如`shell cat -n main.cpp`可以直接查看当前目录下的main.cpp的源码。
    
* set logging on 
    设置日志模式，将gdb输出的所有数据保存到当前目录下的gdb.txt中。
    
* print 打印
    缩写为p，使用 `p 变量`打印变量值 或者 `p &变量`打印变量地址。
    
* set follow-fork-mode [parent|child] 设置跟踪模式
    提前设置跟踪模式可以调试父进程或者子进程
    
* set disassembly-flavor intel 
* set disassembly-flavor att


### 2.关于线程调试

* info thread 查看线程信息

* thread ID 切换到某个线程
    如果只要调试某个线程，其余线程继续运行，则可以使用
    * set scheduler-locking off 任何线程运行到固定断点都会暂停。
    * set scheduler-locking on 只暂停当前调试线程。
* thread apply ID1 ID2... 命令
    让线程ID1和线程ID2执行指定的GDB命令，如果想要所有所有线程都执行对应命令则可以使用命令:
    * thread apply all 命令


