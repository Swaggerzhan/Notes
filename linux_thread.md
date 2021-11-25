# 利用多核性能

在Linux中，我们可以使用其提供的POSIX标准的函数来创建线程，进程等，通过多核来提升程序的性能。

### 0x00 预备知识：线程的创建以及销毁

其调用的系统调用为:

```C++
#include <pthread.h>

int pthread_create(pthread_t *restrict tidp,
    const pthread_attr_t *restrict attr,
    void *(*star_rtn)(void *), void *restrict arg);
```

* tidp       为成功创建后写入的 __新创建的线程的ID__ 
* attr       参数用于定制各种不同的 __线程属性__
* start_rtn  参数为函数地址开始运行， __新线程将从这里开始运行__
* arg为      __函数start_rtn的参数__

还有一些其他的系统调用接口提供了一些功能，如`pthread_self()`返回其线程的线程ID，`pthread_join()`来等待线程的结束，并回收捕获线程的返回数据，也可以直接`pthread_detach()`不等待其线程返回。

```c++
void *write_thread(void *arg){
    //TODO
    void *data = new Data;
    return data;
}
void main(){
    pthread_t t1;
    pthread_create(&t1, nullptr, write_thread, nullptr);
    void *retData;
    pthread_join(t1, &retData); // 捕获返回值
    return;
}
```

其余的一些使用方法具体可以查询Linux系统调用手册。

### 0x01 同步之Futex

对于线程进程等程序，由于逻辑需要，基本都需要用到关于锁相关的操作，其中比较常用的是Linux提供的`pthread_mutex`系列函数，但其底层的实现，在Linux2.6后，就由`futex`接管了。

`futex`即 fast user-space mutex，拥有相比`mutex`更快的速度，通过用户态原子变量的比较，使得不用每次都陷入内核态，避免了上下文切换的开销。

通过man我们可以看到`There is no glibc wrapper for this system call; see NOTES.`。所以对于使用，我们一般直接使用系统调用来实现。

`futex`的函数原型为:

```c++
int futex(int *uaddr, int futex_op, int val,
                 const struct timespec *timeout,   /* or: uint32_t val2 */
                 int *uaddr2, int val3);
```

函数通过比较`*uaddr`和`val`，如果发现值相等，就会执行所给定的`futex_op`中的操作，这里我们拿`FUTEX_WAIT`和`FUTEX_WAKE`来举例。

给定操作数`FUTEX_WAIT`，如果发现`*uaddr`和`val`相同，则陷入内核态，执行休眠的操作。

给定操作数`FUTEX_WAKE`，如果发现`*uaddr`和`val`相同，则陷入内核态，执行唤醒操作，具体唤醒线程数量取决于`val`的值。

需要用到的头文件有:

```c++
#include <linux/futex.h>	// SYS_futex
#include <syscall.h>			// syscall
#include <sys/time.h>			// timespec
```

用`futex`来实现一个简易版的阻塞于唤醒:

```c++
// 睡眠阻塞
long futex_wait(int* addr, int expected, timesepc* timeout){
	return syscall(SYS_futex, addr, (FUTEX_WAIT | FUTEX_PRIVATE_FLAG),
                 expected, timeout, nullptr, 0
  );
}
// 唤醒
long futex_wait(int* addr, int num){
  return syscall(SYS_futex, addr, (FUTEX_WAKE | FUTEX_PRIVATE_FLAG),
  							 num, nullptr, nullptr, 0
  );
}
```







### __7. 线程同步__

* 互斥锁

```C++
int pthread_mutex_init(pthread_mutex_t* mutex, const pthread_mutexattr_t attr);
// attr为锁的属性值，默认可设为nullptr
/* 线程初始化也可以直接赋值 PTHREAD_MUTEX_INITIALIZER */
int pthread_mutex_destory(pthread_mutex_t *mutex);

int pthread_mutex_lock(pthread_mutex_t *mutex);/* 堵塞上锁 */
int pthread_mutex_trylock(pthread_mutex_t *mutex);/* 非堵塞上锁 */
int pthread_mutex_unlock(pthread_mutex_t *mutex);/* 解锁 */

int pthread_mutex_timelock(pthread_mutex_t *mutex, const struct timespec tsptr);
/* 在时间内尝试上锁，超时返回 */
//成功返回 0，失败返回错误编号
```

互斥锁在C++下可以使用RAII方式进行封装:

```C++
class MutexLock{ // 封装的互斥锁类
public:
    MutexLock()
    :   
    {pthread_mutex_init(&mutex_);}
    void lock(){ // 上锁
        pthread_mutex_lock(&mutex_);
    }
    void unlock(){ // 解锁
        pthread_mutex_unlock(&mutex_);
    }
    ~MutexLock(){ //构析
        pthread_mutex_destory(&mutex_);
    }
private:
    pthread_mutex_t mutex_;
};
class MutexLockGuard{ // 互斥锁门卫
public:
    MutexLockGurad(MutexLock &mutex)
    :   mutex_(mutex) // mutex初始化，它无权管理MutexLock的生命
    {
        mutex_.lock(); // 上锁
    }
    ~MutexLockGuard(){
        mutex_.unlock(); // 解锁
    }
private:
    MutexLock &mutex_;
};
//////////////// 使用 ////////////////
void thread_1(){
    MutexLock mutex_; // 互斥锁
    MutexLockGurad lock(mutex_); // 构造函数上锁
    // 临界区做的事情....
    return; // 结束，栈上对象将自动掉用构析函数
    // ~MutexLockGuard函数被调用
    // mutex_这里只是作为示范，它的生命周期应该由使用它的对象进行管理
}

```


* 条件变量

```C++
#include <pthread.h>


int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);
/* 或者赋值PTHREAD_COND_INITIALIZER初始化 */
int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timewait(pthread_cond_t *cond, pthread_mutex_t *mtext, const struct timespec tsptr);

int pthread_cond_signal(pthread_cond_t *cond);
```

条件变量可以配合互斥量进行协同配合。

```C++
pthread_cond_t q_ready = PTHREAD_COND_INITIALIZE; /* 工作队列状态 */
pthread_mutex_t lock = PHTREAD_MUTEX_INITIALIZE;

std::queue<Work*> q; /* 工作队列，其中是需要处理的 */

/* thread_1 */

pthread_mutex_lock(&lock);
/* 如果此时队列是空的，pthread_cond_wait使得线程1进入挂起状态 */
/* 并且期间互斥量lock解锁，期间如果收到q_ready改变信号，将重新上锁，并且返回继续执行 */
if (q.empty()) // 虚假唤醒，危险！
    pthread_cond_wait(&q_ready, &lock);
/* 取出队列并且处理 */    
Work* work = q->front();
q->pop();
work->process();
pthread_mutex_unlock(&lock);


/* thread_2 */

pthread_mutex_lock(&lock);
q.push(new_work);/* 往队列中加入新工作 */
pthread_mutex_unlock(&lock);
/* 新工作加入队列，通过q_ready叫醒为其等待的线程继续工作 */
pthread_cond_signal(&q_ready);

```

上述用法存在虚假唤醒问题:
"This means that when you wait on a condition variable, the wait may (occasionally) return when no thread specifically broadcast or signaled that condition variable. Spurious wakeups may sound strange, but on some multiprocessor systems, making condition wakeup completely predictable might substantially slow all condition variable operations. The race conditions that cause spurious wakeups should be considered rare."

指一个线程被莫名唤醒，直接通过if语句去执行下面的代码，而该条件还未满足，自然会发生不可预知的错误，所以在IF语句那里应该该用循环方式实现:

```C++
while (q.empty())
    pthread_cond_wait(&q_ready, lock);
    
```
当一个条件变量虚假唤醒时会再次进行判断，重新进入pthread_cond_wait函数。
使用C++封装条件变量

```C++
class Condition{
public:
    Condition(MutexLock &mutex)
    :   mutex_(mutex)
    { pthread_cond_init(&cond_, nullptr);}
    ~Condition(){ pthread_cond_destory(&cond_);}
    void wait(){ // 外层使用者应该在循环中调用，防止虚假唤醒
        pthread_cond_wait(&cond_, mutex_);
    }
    void notify(){ // 叫醒一个等待此条件的线程
        pthread_cond_signal(&cond_);
    }  
    void notifyAll(){ // 叫醒所有等待此条件的线程
        pthread_cond_broadcast(&cond_);
    }

private:
    MutexLock &mutex_;
    pthread_cond_t cond_;
};


////////// 使用 ////////// 
void thread_1(){
    MutexLock mutex_;
    Condition cond_(mutex_); // 条件变量初始化
    MutexLockGuard lock(mutex_); // 上锁
    while (条件不成立) // 防止虚假唤醒
        cond_.wait(); // 解锁并且随眠，等待唤醒
    // 临界区
    return; // MutexLockGuard自动解锁
}

```


* 自旋锁
    这种锁和互斥锁类似，不同的是互斥锁会进入休眠状态，而自旋锁会处于`空等状态`，适用于内核类似中又或者`短时间持有`以及线程`不希望在重新调度上花费太多成本`，用户态下作用并没有互斥锁来得有效。
    
* 屏障
    TODO







