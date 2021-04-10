# debug_error

### 1. 类地址不同

```C++
class TimerLink {
public:
    TimerNode* head;
    TimerNode* tail;
public:
    TimerLink();
    /**
     * 心跳，每次调用tick都检查链表中超时的时间节点，并调用其回调函数
     * 之后将其从链表中删除出去
     */
    void tick();
    /**
     * 添加时间节点
     * @param target
     * @return
     */
    bool addTimer(TimerNode* target);
    /**
     * 删除时间节点
     * @param target
     */
    void delTimer(TimerNode* target);
};
```


类调用tick或者addTimer函数访问head，其中发现head指针不同，主要区别是在主线程中调用和信号量处理函数问题。有一个奇怪的问题是普通的static int类型全局变量是可以在信号处理函数中调用到的，但是gdb调试一下就可以发现其实地址不是相同的。