<font face="Monaco">

# brpc socket

## 0x00 Create

Socket不允许通过构造函数来进行创建，需要通过static的Create来创建：

```cpp
int Socket::Create(const SocketOptions& options, SocketId* id) {
    // 通过资源池来获取一个Socket对象
    butil::ResourceId<Socket> slot;
    Socket* const m = butil::get_resource(&slot, Forbidden());
    // 然后就是一些列的SocketOptions设置...
    // EPOLLIN触发时调用的回调函数
    m->_on_edge_triggered_events = options.on_edge_triggered_events;
    m->_this_id = MakeSocketId(
        VersionOfVRef(m->_versioned_ref.fetch_add(1, 
            butil::memory_order_release)), slot);
    // ...
    // 重置当前Socket的fd
    if ( m->ResetFileDescriptor(options.fd) != 0 ) {
        // Error handle ...
        return -1;
    }
    *id = m->_this_id;
    return 0;
}
```

其中SocketId是uint64_t类型，高32bit为版本，低32bit为slot的id，也就是用作资源池中的Id，在ResetFileDescriptor中：

```cpp
int Socket::ResetFileDescriptor(int fd) {
    // 一些重置....
    if (_on_edge_triggered_events) {
        if (GetGlobalEventDispatcher(fd).AddConsumer(id(), fd) != 0) {
            // Error handle...
            _fd.store(-1, butil::memory_order_release);
            return -1;
        }
    }
    return 0;
}
```

一旦重置fd过程中发现这个fd拥有边缘触发时的回调函数，就将其加入到Dispatcher中(epoll)，在EPOLLIN触发时将调用这个回调函数。

## 0x01 SocketId

可以看到，Socket::Create不返回其指针，而返回一个SocketId，通过Address可以解析并从资源池中获取到这个Socket的指针，这里得先了解一下，在Socket这个类中，存有一个版本信息(_versioned_ref)，而SocketId这个uint64_t类型的数值中也保存有版本信息。

__在初始的时刻，Socket的_versioned_ref总是为偶数(一开始就是0)，一旦SetFailed被调用，_versioned_ref版本就会被+1，当完成归还之后，_versioned_ref才会再被+1(通过SocketId中的版本号+2做到的)__。

过程：

> 1. Create获得一个Socket，其_versioned_ref中版本号为0，SocketId高32bit为0，版本匹配，使用正常。

> 2. 某一个时刻，有一个线程调用了SetFailed，使得_versioned_ref为SocketId高32bit + 1，版本不一致，Address失效。

> 3. 没人用了，回收，使得_versioned_ref设定为SocketId + 2。

```cpp
inline int Socket::Address(SocketId id, SocketUniquePtr* ptr) {
    const butil::ResourceId<Socket> slot = SlotOfSocketId(id);
    Socket* const m = address_resource(slot);
    if (__builtin_expect(m != NULL, 1)) {
        // _version_ref的高32bit代表当前Socket的版本，低32bit代表“引用数量”
        // 在获取其地址的时候，总是将其引用数量+1，所以直接fetch_add
        const uint64_t vref1 = m->_version_ref.fetch_add(1, butil::memroy_order_acquire);
        // 找到这个Socket地址中的版本号
        const uint32_t ver1 = VersionOfVRef(verf1);
        // 和SocketId中的版本号进行对比，如果一样，那就可以直接返回了
        if (ver1 == VersionOfSocketId(id)) {
            ptr->reset(m);
            return 0;
        }
    }
    // 走到这一步就证明有其他地方通过设定版本的形式使得当前的Socket失效了
    // 那么就开始尝试销毁当前的Socket，至于由谁来进行做，就需要看引用数量
    // 了，最后一个发现的需要做销毁，如果verf2是大于1的，那么肯定还有地方
    // 引用了，如果verf2小于1，那就是有地方已经率先析构当前Socket了，如果
    // 刚刚好为1，那么证明就是当前这个线程是最后一个“去除引用”的，需要做析构
    const uint64_t verf2 = m->_versioned_ref.fetch_sub(1, butil::memory_order_release);
    if (nref > 1){
        return -1;
    }else if (__builtin_expect(nref == 1, 1)) {
        const uint32_t ver2 = VersionOfVRef(verf2);
        // 判断是否是奇数
        if ((verf2 & 1)) {
            // TODO...
            if (ver1 == ver2 || ver1 + 1 == ver2) {
                uint64_t expected_vref = vref2 - 1;
                if (m->_versioned_ref.compare_exchange_strong(
                        expected_vref, MakeVRef(ver2 + 1, 0), 
                        butil::memory_order_acquire, 
                        butil::memory_order_relaxed)) {
                    m->OnRecycle();
                    return_resource(SlotOfSocketId(id));
                }
            } // ...略
        }
    }else {
        // ...
    }
    return -1;
}
```

之所以通过这么复杂的手段来返回SocketId而不直接返回指针，就是为了可以做到类似“shared_ptr和weak_ptr”的效果，但shared_ptr无法直接将计数器归零，当遇到源源不断的请求时，某个Socket就有可能面对无法归零的情况，而SetFailed可以将这种“引用计数”直接归零：

```cpp
int Socket::SetFailed(int error_code, const char* error_fmt, ...) {
    // ...
    const uint32_t id_ver = VersionOfSocketId(_this_id);
    uint64_t vref = _versioned_ref.load(butil::memory_order_relaxed);
    for (;;) {
        // 在读取id_ver和vref过程中，已经有其他线程率先完成了SetFailed
        // 所以直接返回即可
        if (VersionOfVRef(vref) != id_ver) {
            return -1;
        }
        // MakeVRef首先将当前的版本+1，SocketId中的版本，然后跟Socket中的
        // 版本进行对比_versioned_ref，如果成功，那么就更新为新的版本
        // 一旦Socket内存中版本发生了改变，那么之后通过Address传入SocketId
        // 和Socket内存版本一比对必然会出现错误从而返回null，从而达成直接将
        // “引用”归零
        // 如果失败了，证明在另外的地方有线程改变了版本信息，重来一次如果发现
        // 版本发生改变且和SocketId不匹配了自然会退出
        if (_versioned_ref.compare_exchange_strong(
                vref, MakeVRef(id_ver + 1, NRefOfVRef(vref)), 
                butil::memory_order_release, 
                butil::memroy_order_relaxed)) {
            // ...
            ReleaseAdditionalReference();
        }
    }
}
```

其中的ReleaseAdditionalReference调用了Dereference：

```cpp
inline int Socket::Derefenrece() {
    const SocketId id = _this_id;
    const uint64_t vref = _versioned_ref.fetch_sub(1, butil::memory_order_release);
    const int32_t nref = NRefOfVRef(vref);
    if (nref > 1) {
        // 在fetch_sub时如果大于1，就证明还有线程在使用，直接返回即可
        return 0;
    }else if (__builtin_expect(nref == 1, 1)) {
        // 这种情况下就证明刚刚好是没人使用了，需要进行回收
        const uint32_t ver = VersionOfVRef(vref);
        const uint32_t id_ver = VersionOfVRef(id);
        // 有两种回收情况：
        // 1. ver == id_ver，版本完全一致，没有调用过SetFailed，但是其引用
        // 已经下降至0，所以进行回收
        // 2. ver == id_ver + 1，版本不一致，有调用过SetFailed，且目前引用
        // 已经下降为0，回收
        if (__builtin_expect(ver == id_ver || ver == id_ver + 1, 1)) {
            // 回收过程，就是将_versioned_ref设定为id_ver + 2，所以每过一次
            // 循环使用Socket，它的版本就会+2
            uint64_t expected_vref = vref - 1;
            if (_versioned_ref.compare_exchange_strong(
                    expected_vref, MakeVRef(id_ver + 2, 0), 
                    butil::memory_order_acquire, 
                    butil::memory_order_relaxed)) {
                OnRecycle();
                return_resource(SlotOfSocketId(id));
                return 1;
            }
            return 0;
        }
        return -1;
    }
    return -1;
}
```



## 0x02 Write

brpc socket的Write操作是由一个个WriteRequest单向链表构成的：

```cpp
struct WriteRequest{
    static WriteRequest* const UNCONNECTED; // 标明链表还没被链上
    WriteRequest* next; // 构成单向链表
};
```

每个WriteRequest中的data就是需要写入的，从“Socket::Write”开始：

```cpp
// src/brpc/socket.cpp
int Socket::Write(butil::IOBuf* data, const WriteOptions* options_in) {
    // ...
    WriteRequest* req = butil::get_object<WriteRequest>();
    // ...
    req->data.swap(data);
    req->next = WriteRequest::UNCONNECTED;
    // ...
    return StartWrite(req, opt);
}
```

StartWrite分为第一次写入和后面的写入，第一次写入的线程需要去做真正的写入操作，而后面的线程只需要将其加入到单向链表之中即可，这个链表的加入操作是wait-free的。

```cpp
int Socket::StartWrite(WriteRequest* req, const WriteOptions& opt) {
    WriteRequest* const prev_head = 
        _write_head.exchange(req, butil::memory_order_release);
    if (prev_head != NULL) {
        // 如果prev_head不是null，那么证明必然已经有其他线程做写入操作了
        // 那么只需要将新的链表->next链到老旧的头部上就行了
        // 只不过这样将导致顺序变反，比如N代表一个老旧的，N+1代表新来的WriteRequest
        // 那么其链表的分布：
        // 新head                    正在写的head
        // (N+3) -> (N+2) -> (N+1) -> (N)
        req->next = prev_head;
        return 0;
    }
    // 不再是UNCONNECTED的，为null则标明当前就有了写入的权限了
    req->next = NULL;
    int ret = ConnectIfNot(opt.abstime, req); // 如果没有连接就建立连接
    // ...
    // 检查是否已经完全写入了，指的是链表中已经没有其他WriteRequest了
    if (IsWriteComplete(req, true, NULL)) {
        ReturnSuccessfulWriteRequest(req);
        return 0;
    }
    // ... 
}
```

当之前的Write线程完成写入操作后，就需要进行检查是否写入完成：

```cpp
bool Socket::IsWriteComplete(Socket::WriteRequest* old_head,
                             bool singular_node,
                             Socket::WriteRequest** new_tail) {
    WriteRequest* new_head = old_head;
    WriteRequest* desired = NULL;
    bool return_when_no_more = true;
    // 如果连当前的WriteRequest的数据都还没写完，那么肯定还是需要再写的
    if ( !old_head->data.empty() || !singular_node ) {
        desired = old_head;
        return_when_no_more = false;
    }
    if (_write_head.compare_exchange_strong(
            new_head, desired, butil::memory_order_acquire)) {
        if ( new_tail ) {
            *new_tail = old_head;
        }
        return return_when_no_more;
    }
    
    // 到了这步就证明`new_head | old_head`已经发生了改变，那么显然有其他
    // 线程做了写入的操作，使得_write_head不再是old_head了，所以需要做的
    // 就是将接替其他线程的事情，继续做写入操作，但有个问题，那就是这里的链表
    // 顺序是反的
    
    WriteRequest* tail = NULL;
    WriteRequest* p = new_head;
    // 在之前的compare_exchange_strong中，我们将new_head的值更新为了最新的
    
    // 这里其实就是进行了链表反转的操作，从new_head开始，至old_head进行反转，
    // 反转后的链表顺序，就是我们期待的WriteRequest顺序了
    do {
        while ( p->next == WriteRequest::UNCONNECTED) {
            // 如果当前的链表已经添加了，但还没来得及链上，那我们就等一会
            // 但其实这种可能比较小会发生的
            sched_yield();
        }
        WriteRequest* const saved_next = p->next;
        p->next = tail;
        tail = p;
        p = saved_next;
    } while ( p != old_head )
    
    old_head->next = tail;
    
}
```

// TODO...


</font>