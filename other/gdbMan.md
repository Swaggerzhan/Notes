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


### 3.关于tui开启

可以在gdb中使用：
```gdb
tui disable # 关闭
tui enable # 开启
```
    
# Linux一些命令调试技巧

### 1. strace

strace命令可以获得一个进程的系统调用记录，可以直接在进程开始的时候进行跟踪。

```shell
strace ./a.out
```

也可以使用关键字`-p`跟上对应的进程`pid`来跟踪某个进程的系统调用。

```shell
strace -p 1000 # 跟踪pid为1000的进程
```

有时候，某些进程的系统调用非常的多，这将导致输出的数据非常的多，所以我们使用`-c`即count来做一个统计。

```shell
strace -cp 1000 # 统计pid为1000的进程各个系统调用的次数
```

又或者加上`-T`来查看每个系统调用所使用的时间，可以用于检测某些进程的性能瓶颈，但`-T`不可与`-c`同时使用。

### 2. top

top命令除了查看进程占用，系统负载等一些数据，也可以用来观察某个进程中的线程使用情况。

```shell
top -H -p 1000 # 查看pid为1000的进程中所有线程占用量
```

其中`-p`表示只查看某个进程，`-H`表示查看进程中线程的使用情况。

### 3. lsof

由于Linux是一切皆文件的方式，所以lsof其实也可以做到和netstat类似的操作，甚至做的更好。

通过某个协议或者端口甚至是ip地址等来查看相关的打开`fd`可以使用`-i`。

```shell
lsof -i TCP # 查看tcp相关的文件描述符 UDP 同理
lsof -i TCP:22 # tcp 22号端口相关的文件描述符 UDP同理
lsof -i TCP@127.0.0.1:22 # 通过ip和协议和端口来确定
```

或者，我们直接通过`-p`指定某个pid来确定这个进程所开启了多少个文件

```shell
lsof -p 458 # 查看458进程开了多少个fd
```

可以使用`-u`来查看某个user打开了多少个进程，当然这个用的比较的少。
lsof这个命令还是比较强大的，还可以通过grep和awk等等命令等配合抓出各个TCP连接的状态等。

### 4. netstat

netstat能做到的，lsof配合一下其他命令也基本能做到，并且netstat在unix和Linux上有所不同。

### 5. 磁盘IO查看

这个有许多命令，iotop是一个比较容易使用的，和top类似，不过iotop没有自带，需要安装

```shell
iotop # 查看磁盘读写情况
iotop -p pid # 查看某个pid的磁盘读写情况
```
 
sar命令，这个是Linux有自带的，比较好用

```shell
sar -b 刷新间隔 刷新次数
```
如果想要一直刷新下去，那就把刷新次数留空即可

```shell
sar -b 1
08:02:53 PM       tps      rtps      wtps   bread/s   bwrtn/s
08:02:54 PM      0.00      0.00      0.00      0.00      0.00
```

其中tps为rtps和wtps的总和，`rtps`即read tps，`wtps`就为write tps，那么`bread`为bytes read，`bwrtn`为bytes write。

### 6. vmstat

用于查看整体内存的使用情况，也可以看到CPU的一些情况，可以看到操作系统每秒的上下文切换次数。
使用`-t 刷新间隔 刷新次数`来持续查看，刷新次数可以放空即表示无限刷新下去。

```
vmstat -t 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----- -----timestamp-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st                 CST
 3  0      0 199780 175828 1074912    0    0     5     5    6    3  1  1 98  0  0 2021-09-27 20:14:13
 0  0      0 199764 175828 1074912    0    0     0     0  637 1501  0  1 99  0  0 2021-09-27 20:14:14
 0  0      0 199764 175828 1074912    0    0     0     0  631 1516  4  0 96  0  0 2021-09-27 20:14:15
```

procs下的r和b分别表示为等待运行的进程数和等待IO的进程数量，如果r大于了CPU核心数量，则考虑是否负载太高了，b太高则考虑是否IO瓶颈。

__在system下有2个指标，分别为in和cs，表示的是每秒中断次数和每秒上下文切换次数，特别是这个cs，当cs太大时，我们就要考虑是否时某些进程中的线程太多了，CPU都消耗在了上下文切换中了__ 。

swap下的si和so表示交换分区的换入和换出，如果有，可能是内存已经不够使用了，考虑内存泄漏的情况。

还有cpu下的wa，表示的是cpu的等待时间，太大(>20)则表明IO占用了过多时间，CPU很少运作。

