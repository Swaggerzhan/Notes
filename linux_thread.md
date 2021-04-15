# linux_thread

### 进程的创建 thread_create()

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


### 进程的操作

#### __1. pthread_t thread_self(void);__

用来获取自身进程的进程ID

#### __2. int pthread_equal(pthread_t tid1, pthread_t tid2);__

用来比对进程ID

#### __3. int pthread_join(pthread_t thread, void** rval_ptr);__

用于捕获thread对应线程ID的返回值，如果线程thread尚未返回，则堵塞等待返回。join函数不能调用于已经detach的函数。

```C++

void *write_thread(void *arg){
    //TODO
    void *data = new Data;
    return data;
}

void main(){
    pthread_t t1;
    pthread_create(&t1, nullptr, write_thread, nullptr);
    void *retData;
    /* rval_ptr是个二级指针 */
    pthread_join(t1, &retData);
    return;
}

```


#### __4. int pthread_cancel(pthread_t tid);__

用于取消同一进程中的其他线程，pthread_cancel函数不等待终止，它仅仅提出要求。

#### __5. int pthread_detach(pthread_t tid);__

分离线程

#### __6. cleanup_push和cleanup_pop__

```C++
#include <pthread.h>

void pthread_cleanup_push(void (*rtn)(void*), void* arg);

void pthread_cleanup_pop(int execute);
```

cleanup函数系列使用的是栈来记录类型，所以push进的函数调用方式是反方向的。使用pop(0)可以取消栈顶的函数。

触发清理函数rtn:
    * 调用 __pthread_exit时__
    *  __响应取消请求时__
    * 使用 __非零execute参数__ 调用pthread_cleanup_pop时候

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
if (q.empty())
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

* 自旋锁
    这种锁和互斥锁类似，不同的是互斥锁会进入休眠状态，而自旋锁会处于`空等状态`，适用于内核类似中又或者`短时间持有`以及线程`不希望在重新调度上花费太多成本`，用户态下作用并没有互斥锁来得有效。
    
* 屏障
    TODO







