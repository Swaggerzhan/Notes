# 并发编程知识整理

## 0x00 CPU缓存-False Sharing

前置知识：MESI协议。

CPU中存在多级缓存结构，越靠近CPU的缓存读取速度越快，但空间也就越小，如何高效的利用CPU缓存也是多线程中比较重要的点，我遇到的坑点是假共享(False Sharing)，直接上一个例子：

```C++
int arr[100];
int len = 500000000;

void thread1() {
  long start = TimeStamp::Now();
  for (int i=0; i<len; i++) {
    arr[0] = i;
  }
  long end = TimeStamp::Now();
  TimeStamp::diff(start, end); // 计算时间差
}

void thread2() {
  long start = TimeStamp::Now();
  for (int i=0; i<len; i++) {
    arr[1] = i;
  }
  long end = TimeStamp::Now();
  TimeStamp::diff(start, end);
}

void thread3() {
  long start = TimeStamp::Now();
  for (int i=0; i<len; i++) {
    arr[50] = i;
  }
  long end = TimeStamp::Now();
  TimeStamp::diff(start, end);
}
```

如果我同时启动`Thread1`和`Thread2`，可以看到，他们没有任何临界区，可以随意读写，在我机器上执行后平均时间大致在4s-5s之间。而如何同时启动`Thread1`和`Thread3`则在1s-1.4s之间，差了好几倍，做的确是相同的事情。

CPU中，每个核心有独立L1缓，并且CPU缓存Cache Line一般是在64Bytes，以上面的例子，Thread1在读取arr[0]后，通常也会将arr[1]以及之后的部分数据也读入缓存，看起来是没问题，但如果Thread1修改了arr[0]这个变量将导致整个缓存行改变，而此时另外一个核心在读写arr[1]，不同核心的高缓(L1)由于缓存不同，需要通过MESI来重新同步，这就增加了耗时。而`Thread3`中做修改的arr[50]和arr[0]不在同一个缓存行中，所以两个不同核心间不用频繁做同步，速度也就快了。

通过上面的做法姑且算是以空间换时间，如果过多变量都这样处理可能导致缓存被浪费，具体的话可以考虑网上比较常见的“数组”读写的案例。

__注：单核CPU永远无法体现以上的差距__ 。


## 0x01 LockFree-无锁编程

前置知识：无锁队列，MESI协议，原子变量(内存结构)。

无锁编程在一定程度上比有锁编程要快非常的多，很多人应该都听过Kfifo的大名了，这几天学到了Disruptor，用C++自己写一版记录一下：

Disruptor实现和kfifo很像，都是围绕一个ringBuffer来实现的，一些小优化比如False Sharing亦或者加快mod的操作这里就不再赘述了。

### 生产者

首先是生产者，Disruptor中维护一个`writeCursor_`和`commitWriteCursor_`的原子变量，前者表示可以写入的点(还没写入)，后者则表示 __已经写入__ 的点(commited，生产者可读了)。有可能看起来混淆，解释一下生产者的步骤就能更了解了：

1. 将`writeCursor_`自增1。
2. 成功，获取自增前的seq，失败则进入某种策略，比如让出CPU。
3. 获取的seq即为写入点，当生产者完成写入后，需要进行更新，等待`seq-1=commitWriteCursor_`。
4. 更新lastWriteCursor = seq 表示写入成功。

注： __生产者的commit过程是有严格顺序的，多位生产者也许可以同时写入数据，但更新只能是数组索引顺序commit__ 。


### 消费者

消费者同样维护2个原子变量，为`readCursor_`和`commitRead_`，`readCursor_`指向最后派发给消费者的索引(已经派发出去了)，`commitRead_`则表示已经被消费者读完的索引。

1. 获取readCursor_ + 1的值为seq。
2. 读取数组索引为seq的值。
3. 读取成功后，消费者需要提交已经读取完成的操作，需要等待seq-1=commitRead_。
4. 消费者更新读取进度，commitRead_=seq

注：同生产者。

### 边界处理及实现

如果生产者速度过快，而消费者来不及处理的话，需要保证`writeCursor_`和`commitRead_`之差不能超过`ringBuffer`的长度。

相反，如果消费者过快，那么也要保证`readCursor_`不超过`commitWriteCursor_`。

可写范围一直在`[commitRead_, seq]`之间，也即`[commitRead_, writeCursor_)`。

我这边简单写了一个Disruptor的[C++实现](https://github.com/Swaggerzhan/Raiden/blob/master/base/Disruptor.h)，是一个其中策略非常的简单(spin)，排除False Sharing后速度确实快了非常多，但对CPU的压力也不小，具体需要取决于业务需求。

并且，采用Spin的策略在核心较少的处理器上可能会适得其反，在单核CPU中主动让出资源一直是比较好的选择。

## 0x02 volatile以及原子变量

C++中的volatile不同于JVM中的概念，它代表某个变量是易变的，volatile作用于编译器，多线程情况下不能使用volatile来做线程同步，这个是肯定的。至于volatile如何作用于编译器，我们可以看个小例子：

```C++
int a = 0;
int do_something() {
    while ( a > 1 );
    return 1;
}
```

在x86-64 gcc 10.1版本下编译，得到以下汇编：

```assembly
a:
        .zero   4
do_something():
        push    rbp
        mov     rbp, rsp
.L3:
        mov     eax, DWORD PTR a[rip]
        cmp     eax, 1
        jle     .L2
        jmp     .L3
.L2:
        mov     eax, 1
        pop     rbp
        ret
```

可以看到，编译器“老老实实”的做了我们写的代码，事实上我们也确实希望编译器这样做， __但有些时候，我们想使用一些编译器的优化选项来提高代码的运行效率，并且降低执行文件的大小，当我们使用一些优化选项时，也就赋予了编译器一些“不老实”的权限，当通过`-Ofast`的优化选项后，相同的代码将得到以下汇编__ ：

```assembly
do_something():
        mov     eax, 1
        ret
a:
        .zero   4
```

可以看到输出的汇编少了很多，并且 __函数do_something()直接摆烂，这是因为编译器开始自作聪明了，它认为，既然`a=0`，那么while语句中的`a>1`就永远不会成立，那么也就永远不用运行，所以可以直接删除，即刻返回1，乍一看也没问题(单线程)，但代码一放到多线程情况下运行，那么a就有可能被其他线程改变进而执行while语句__ 。

当我们使用volatile对a变量进行修饰后，同样开启-Ofast优化，得到汇编码：

```assembly
do_something():
.L2:
        mov     eax, DWORD PTR a[rip]
        cmp     eax, 1
        jg      .L2
        mov     eax, 1
        ret
a:
        .zero   4
```

可以看到，编译器对`a`这个变量又老实起来了。

__注：volatile作用于编译器，对CPU来说是透明的，volatile变量不能作多线程下的同步__ ! CPU的结构导致多线程的原子变量同步需要考虑到内存序。
