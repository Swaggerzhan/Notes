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







