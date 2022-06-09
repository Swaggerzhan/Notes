<font face="Monaco">

# RCU 

## 0x00 pthread_key_t

在pthread中提供了一系列的接口，在TLS(thread local storage)方面，pthread提供了pthread_key_t系列的接口，它的主要作用就是仅通过一个全局的一个pthread_key_t，为不同的线程提供不同的变量值。

这个pthread_key_t就像是一个托管的人，不同线程通过相同的key可以得到不同的值，基础的API：

```c
int pthread_key_create(pthread_key_t* key, void(* destr_function)(void*));
int pthread_key_delete(pthread_key_t key);
int pthread_setspecific(pthread_key_t key);
void* pthread_getspecific(pthread_key_t key);
// return 0 for success
```

使用方法非常简单，通过pthread_key_create创建出一个全局的key，之后所有的线程都可以直接访问这个key，调用pthread_getspecific并通过传入这个key来获取属于线程自己的“变量值”，只不过如果当前线程事先没有set，那么这里的get会是nullptr。

## 0x01 brpc中的double buffer数据结构

这是一个对读多写少极其优化的数据结构，回想一下关于读写锁的实现，它真的可以提高性能，但这种提高也有限制。

### 读写锁

读写锁的简单实现可以这样做：

```c
struct lock {
    int n;
};
struct lock l;
l.n = 0;

void read_lock() {
    while (1) {
        int x = l.n;
        if ( x < 0 ) {
            continue;
        }
        if ( CAS(&l.n, x, x + 1) ) {
            break;
        }
    }
}

void read_unlock() {
    while (1) {
        int x = l.n;
        if ( CAS(&l.n, x, x - 1) ) {
            break;
        }
    }
}
```

写锁就不在多说了，这里想讨论的问题不是关于写线程饿死或者各种问题，更多的是在核心非常多的情况下，多读线程所照成的问题。

在读线程极多的情况下，每次“CAS”只能有一个线程通过，而其余的线程总要进入到下一轮循环中，而每一次循环都只有一个线程能通过，__究其本质，读写锁的读，其实是需要“写”的，写的即是lock.n，它表示了“当前获得读锁的线程”__。

回顾MESI协议就会发现，在多核CPU中的写入操作会使得频繁的发送Invalid缓存消息，使得读写锁即使在“只读”下，性能也会随着核心的提升而降低，没能最大限度的做到并行。

### double buffer

brpc的double buffer实现非常的有意思，它假设每个线程都有一个“TLS的mutex锁”，每个线程如果想要读，那么直接对着自己的“TLS mutex锁”做上锁操作即可，这显然效率是极高的，因为每个线程的锁都是不一样的，免去了Invalid缓存发送的问题。

那么它怎么做写入操作？double buffer，顾名思义，数据拥有2份，分为前台buffer和后台buffer，当写入线程想要写入的时候，它总是先写入到后台的buffer中，然后做一次swap操作，将前台buffer和后台buffer(最新)进行对调，__随后写线程需要进行一次所有线程局部锁的遍历，对它进行上锁，然后解锁，以此遍历，全部遍历完后再对“之前换下来的前台buffer”做相同的操作，那么本次写入操作就结束了__。

__之所以进行全部线程的遍历在返回是一种“保证”，保证当写入线程返回时，所有的读线程一定可以看到最新的数据，但这里没有做这种保证：在写线程进行到一半时(遍历锁过程中，此时前后台buffer已对调)，老旧的读线程还在看“旧数据”，这里的等待其实就是使得所有的老旧读线程结束在返回__。

ps: 所以其实还是有数据突然不一致的表现，体现在write线程正在遍历锁的过程中，新来的read线程和之前还未退出的read线程看到的数据其实是不一样的。

## 0x02 Linux kernel中的RCU

TODO..

</font>