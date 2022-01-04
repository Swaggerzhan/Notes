# 进程间通讯

### 1. 管道

```C++
int pipe(int fd[2]);
// success return 0 fail return -1 and set errno
```

pipe为单向通讯，向pipe[1]写入可以从pipe[0]中读取到，这点和标准输入输出刚好相反，标准IO中，fd为0表示input，fd为1表示output。

pipe的局限性很明显，只能用于父子进程间通讯。并且默认为阻塞形，其中默认缓冲区大小为4k。


### 2. 共享内存

```C++
int shmget(key_t key, size_t size, int shmflag);
```

从参数中就可以大致明白作用，key为标识，size代表大小，shmflag则表示操作，shmget函数返回一个int数，表示为shmid。

下面解释一下key和shmid，对于内核来说，当多个进程共用一个内存的时候，需要有一个对应的索引来进行比对才能明白，key就是这样的一个数， __它就像一个数组的索引存在内核中，其数组的值就是对应的内存，所有进程只要拿着这个key给内核，内核就能找到对应的共享内存地址__ 。

而这个shmid和key作用完全一样，区别在于key是给内核看的，而且shmid是给自己看的，是本进程内的对应内存块，接下来就要介绍通过shmid获取共享地址。

一个key只能对应一个共享内存，也就是说，当你对一个已经存在的key再次申请的时候，如果size相同，则返回对应的shmid，如果不同，则返回-1并设置errno。

shmflag的参数一般为`IPC_CREAT | 0666`

这里的key是需要通过另外的函数申请的

```C++
key_t ftok(const char* pathname, int proj_id);
```

其中pathname一定要存在，建议直接设定为`.`即可，后面的proj_id计划代号则随意，需要申请到相同的key就使用相同的proj_id即可。


```C++
void* shmat(int shmid, const void* shmaddr, int shmflag);
```

第二个参数一般直接给nullptr即可，不必自己定义虚拟内存开始地址，我们直接通过函数返回值拿到共享内存地址即可(内核帮我们找合适的地址)。shmflag一般就设定一些读写权限之类的。第一个参数就是之前提到的获取共享内存的索引。

现在整理一下对应逻辑，进程A通过key为1创建了一个共享内存，进程B通过key为1获取了一个共享内存，进程B通过这个key=1得到的一个shmid去获取一个真正的共享内存地址，当B往里写入数据时，A就能看到了。

其本质原理是A和B共享了同一个`物理内存`，A和B上的共享内存地址并不相同。

shmflag一般为0，`SHM_R`，`SHM_AND`表示取整，
