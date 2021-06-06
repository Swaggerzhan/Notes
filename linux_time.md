# 时间

### 关于时间的结构体

* timeval

```C++
struct timeval{
    time_t          tv_sec;  // 秒
    suseconds_t     tv_usec; // 微秒
};
```

* tm

```C++
struct tm {
    int tm_sec; // 秒  0-60
    int tm_min; // 分  0-60
    int tm_hour; // 时 0-23
    int tm_mday; // 日 1-31
    int tm_mon;  // 月 0-11
    int tm_year; // 自1900起
    int tm_wday; // sunday = 0,之后累加
    int tm_yday; // 距离一年第一天的天数
    int tm_isdst;//
};
```

* tms
这是一个关于程序运行多长时间的结构体。

```C++
struct tms{
    clock_t tms_utime; // 用户态下运行时间
    clock_t tms_stime; // 内核态下运行时间
    clock_t tms_cutime; // 子进程用户态下运行时间
    clock_t tms_cstime; // 子进程内核态下运行时间
};
```


### 获取时间

关于获取时间的系统调用在Linux上基本都保存在`sys/time.h`头文件中

* gettimeofday

```C++
int gettimeofday(struct timeval *tv, struct timezone *tz);
// return 0 on success, -1 on error
```

第二个参数时区一般设定为nullptr，gettimeofday可以返回微秒级别的时间。

* time

```C++
time_t time(time_t *timep);
// return number of seconds since the Epoch or -1 on error
```
time函数直接返回一个基于秒数的时间戳，又或者通过参数timep返回，一般直接将形参timep设置为nullptr。

* times

```C++
clock_t times(struct tms* buf);
```
返回进程所使用的时间，它又两种返回方式，一个是返回值，一个是参数buf。

* clock

```C++
clock_t clock(void);
```
clock函数返回的是进程所使用的时间，它包括用户时间和内核时间，是一个总和时间。

### 打印时间

* ctime

```C++
char* ctime(const time_t* timep);
```

传入一个时间戳将返回其时间戳所对应的时间字符串。
例:
传入: (time_t)1622897275
返回字符串: (char*)Sat Jun  5 08:47:55 2021

注:  由于返回的是一个指向char的指针，而且这个地址可能会被后来的ctime又或者其他函数改写，所以需要长期保存字符串时间的话需要进行复制，将字符串复制到自己的缓冲区中。


### 进程时间

当我们使用time命令运行一个程序时，可以得到以下类似输出

```shell
real    0m0.003s
user    0m0.002s
sys     0m0.001s
```

real时间表示总时间。
user时间则表示当程序运行在用户态下的时间。是真正的程序获取到的CPU时间。
sys时间则表示当程序运行在内核态下的时间。如处理页中断等等。


