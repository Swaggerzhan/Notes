<font face="Monaco">

# braft log 引擎

## 0x00 SegmentLogStroage

在braft中，位于src/braft/storage.h中定义了这个LogStorage的抽象类，也就是日志存储引擎的抽象基础了，在src/braft下，log.h/log.cpp中的SegmentLogStorage实现了这些接口，不过还需要先看它的一些属性字段：

```cpp
class SegmentLogStorage : public LogStorage {
public:
    typedef std::map<int64_t, scoped_refptr<Segment> > SegmentMap;
private:
    std::string _path;
    butil::atomic<int64_t> _first_log_index;
    butil::atomic<int64_t> _last_log_index;
    raft_mutex_t _mutex;
    SegmentMap _segments;
    scoped_refptr<Segment> _open_segment;
    int _checksum_type;
    bool _enable_sync;
};
```

这个_segments是一个map，指向一个Segment类，由此可见，这个Segment其实就是一段段的日志条目，__raft中我们对日志比较关注的点可能在这两字段上，[first_log_index, last_log_index]，这是一个前闭后闭的区间，代表的即是当前SegmentLogStroage存储的内容__。

### 启动

init函数是启动的入口，载入的开始点总是从元数据载入开始`load_meta()`，这仅载入了`first_log_index`，之后便是`list_segments()`扫描一个个Segment，在本引擎中，Segment分好几种，有`close`的，也有`open`的，`open`故名思议就是当前的Segment还没写满的，list过程中也会严格区分，因为`close`的Segment可以明确的知道`first_log_index`和`last_log_index`。

这些Segment的磁盘命名中都有一些元数据，比如`first_log_index`和`last_log_index`等。

list过程中可以很快的遍历`close`的Segment从而得出`last_log_index`，当然这不一定是最终的数据，还需计算是否存在`open`的Segment，不过可以肯定的是，`last_log_index < open_segment->first_log_index`。

```cpp
int SegmentLogStorage::init(ConfigurationManager* configuration_manager) {
    // 路径信息
    butil::FilePath dir_path(_path);
    // ...
    int ret = 0;
    bool is_empty = false;
    do {
        // 加载log_meta的信息
        ret = load_meta();
        if (ret != 0 && errno == ENOENT) {
            LOG(WARNING) << _path << " is empty";
            is_empty = true;
        } else if (ret != 0) {
            break;
        }
        // 建立索引信息
        ret = list_segments(is_empty);
        if (ret != 0) {
            break;
        }
        // 加载信息
        ret = load_segments(configuration_manager);
        if (ret != 0) {
            break;
        }
    } while (0);
    // 如果是空的，那么就从头开始建立索引信息
    if (is_empty) {
        _first_log_index.store(1);
        _last_log_index.store(0);
        ret = save_meta(1);
    }
    return ret;
}
```

将Segment的重要元数据载入到内容中后，调用的是`load_segments()`来进行segment的读取，一个个的读取内容，找到真正的`last_log_index`，同时，如果发现这个log是不完整的，那么就丢弃掉。

所以启动过程结束后，要么就是崩溃之前已经完整持久化下来的数据，或者就是空启动，即`[1, 0]`表示没有任何日志条目。

### SegmentMap

通过上文的一些源码分析，可以得到segment大致在磁盘中的分布图

![](./pic/segment_log_storage_mem_layout.png)

### 添加日志条目

所有添加到SegmentLogStroage的日志条目都是需要进行持久化的，所开放的入口自然是`append_entries`了：

```cpp
int SegmentLogStorage::append_entries(const std::vector<LogEntry*>& entries, IOMetric* metric) {
    if (entries.empty()) {
        return 0;
    }
    if (_last_log_index.load(butil::memory_order_relaxed) + 1
            != entries.front()->id.index) {
        return -1;
    }
    scoped_refptr<Segment> last_segment = NULL;
    for (size_t i = 0; i < entries.size(); i++) {
        LogEntry* entry = entries[i];
        scoped_refptr<Segment> segment = open_segment();
        if (NULL == segment) {
            return i;
        }
        int ret = segment->append(entry);
        if (0 != ret) {
            return i;
        }
        _last_log_index.fetch_add(1, butil::memory_order_release);
        last_segment = segment;
    }
    last_segment->sync(_enable_sync);
    return entries.size();
}
```

先是进行日志index的判断，确保不会出现日志空洞，随后就可遍历的将这些数据塞到`open_segment`中了，调用的是`segment->append()`，它会确保持久化后返回所以在后面对`last_log_index`的操作是安全的，同时，当一个Segment的满时，函数`open_segment()`会重命名当前的Segment文件，使之成为一个`close`的Segment，同时创建一个新的`open`的Segment并返回。

所以综上，这个函数是堵塞的，由于涉及到磁盘IO，因此这个函数会有一个特定的IO线程来调用。

### 获取日志条目

这里先从Segment讲起，Segment是每一个磁盘的抽象，它的内部维护的这几个我们比较关注：

```cpp
const int64_t _first_index;
butil::atomic<int64_t> _last_index;
std::vector<std::pair<int64_t, int64_t>> _offset_and_term;
```

`_offset_and_term`是这样用的，它从`index=0`开始对应`first_index`，进而可以得到这个对应的index在磁盘中的offset位置和term信息，这是为了寻找的时候更为迅速，有种meta信息存内存那种感觉。

现在我们可以看get函数，其中`_get_meta`就是通过以上的方法快速的算出offset信息的。

```cpp
LogEntry* Segment::get(const int64_t index) const {

    LogMeta meta;
    if (_get_meta(index, &meta) != 0) {
        return NULL;
    }

    bool ok = true;
    LogEntry* entry = NULL;
    do {
        ConfigurationPBMeta configuration_meta;
        EntryHeader header;
        butil::IOBuf data;
        if (_load_entry(meta.offset, &header, &data, 
                        meta.length) != 0) {
            ok = false;
            break;
        }
        CHECK_EQ(meta.term, header.term);
        entry = new LogEntry();
        entry->AddRef();
      	// 以下略...
    }while(0);
    return entry;
}
```

### 磁盘上的数据

这里与一个结构：

```cpp
const static size_t ENTRY_HEADER_SIZE = 24
struct Segment::EntryHeader {
    int64_t term;
    int type;
    int checksum_type;
    uint32_t data_len;
    uint32_t data_checksum;
};

// header的checksum
RawPacker packer(header_buf);
packer.pack64(entry->id.term)
  .pack32(meta_field)
  .pack32((uint32_t)data.length())
  .pack32(get_checksum(_checksum_type, data));
packer.pack32(get_checksum(
  _checksum_type, header_buf, ENTRY_HEADER_SIZE - 4))
```

其中保存的是log的“头”数据，后面紧跟真正的log数据，所以它的内存结构大概就是这种：

![](./pic/segment_mem_layout.png)

## 0x01 LogManager 

LogManager是用来管理LogStroage的，同样的从创建开始看这个类。

### 创建

在LogManager的初始化中，很多字段都是由上层的options传入的，这点是受NodeImpl所控制的，其中`wait_map`后面细说，先是初始化SegmentLogStroage，然后得到最新的`_first_log_index`和`_last_log_index`，这里也进行了`fsm_caller`的保存。

```cpp
int LogManager::init(const LogManagerOptions &options) {
    BAIDU_SCOPED_LOCK(_mutex);
    if (options.log_storage == NULL) {
        return EINVAL;
    }
    if (_wait_map.init(16) != 0) {
        PLOG(ERROR) << "Fail to init _wait_map";
        return ENOMEM;
    }
    _log_storage = options.log_storage;
    _config_manager = options.configuration_manager;
    int ret = _log_storage->init(_config_manager);
    if (ret != 0) {
        return ret;
    }
    _first_log_index = _log_storage->first_log_index();
    _last_log_index = _log_storage->last_log_index();
    _disk_id.index = _last_log_index;
    // Term will be 0 if the node has no logs, and we will correct the value
    // after snapshot load finish.
    _disk_id.term = _log_storage->get_term(_last_log_index);
    _fsm_caller = options.fsm_caller;
    return 0;
}
```

### append_entries

WaitMeta所存储的，是一个回调函数，它被一个map所管理，是`LogManager::_wait_map`，类型为`butil::FlatMap<int64_t, WaitMeta*>`，在LogManager初始化时顺带初始化的。

```cpp
struct WaitMeta {
  int (*on_new_log)(void *arg, int error_code);
  void* arg;
  int error_code;
};
```

这里我们先按下不表，先看一下如何进行append_entries：

```cpp
void LogManager::append_entries(
            std::vector<LogEntry*> *entries, StableClosure* done) {
    CHECK(done);
    if (_has_error.load(butil::memory_order_relaxed)) {
        // 释放资源，调回调..略
        return run_closure_in_bthread(done);
    }
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (!entries->empty() && check_and_resolve_conflict(entries, done) != 0) {
        lck.unlock();
        // 释放资源，略
        entries->clear();
        return;
    }
		// ...
}
```

 首先append_entries所在的节点上都能调用，而这些调用又分是Leader角色调用的还是Follower角色调用的，解决这些事务就由`check_and_resolve_conflict()`来解决了。

如果是Leader角色，那么就需要为他们生成index，自然，是从`last_log_index + 1`开始赋值。

如果是Follower角色，就需要判断是否存在日志空洞、是否小于那些已经提交的日志index，如果刚好`entries->front()->id.index == _last_log_index + 1`，那么就属于符合条件，可以进行合并，又或者另外一种情况就是日志发生了重叠。

在发生重叠的这种情况，将Follower身上的所有从冲突位置起的日志全部丢弃掉，然后采用Leader发送过来的日志 __(之所以这么做的原因是在NodeImpl::handle_append_entries_request已经处理prev_log_index下term不同的特殊情况)__。

```cpp
int LogManager::check_and_resolve_conflict(
            std::vector<LogEntry*> *entries, StableClosure* done) {
    AsyncClosureGuard done_guard(done);   
    if (entries->front()->id.index == 0) {
        for (size_t i = 0; i < entries->size(); ++i) {
            (*entries)[i]->id.index = ++_last_log_index;
        }
        done_guard.release();
        return 0;
    } else {
        if (entries->front()->id.index > _last_log_index + 1) {
            return -1;
        }
        const int64_t applied_index = _applied_id.index;
        if (entries->back()->id.index <= applied_index) {
            return 1;
        }
        if (entries->front()->id.index == _last_log_index + 1) {
            // Fast path
            _last_log_index = entries->back()->id.index;
        } else {
            size_t conflicting_index = 0;
            for (; conflicting_index < entries->size(); ++conflicting_index) {
                if (unsafe_get_term((*entries)[conflicting_index]->id.index)
                        != (*entries)[conflicting_index]->id.term) {
                    break;
                }
            }
            if (conflicting_index != entries->size()) {
                if ((*entries)[conflicting_index]->id.index <= _last_log_index) {
                    unsafe_truncate_suffix(
                            (*entries)[conflicting_index]->id.index - 1);
                }
                _last_log_index = entries->back()->id.index;
            }
            for (size_t i = 0; i < conflicting_index; ++i) {
                (*entries)[i]->Release();
            }
            entries->erase(entries->begin(), 
                           entries->begin() + conflicting_index);
        }
        done_guard.release();
        return 0;
    }
    return -1;
}
```

所以当`check_and_resolve_conflict()`返回的时候，不管是Leader亦或者Follower都准备好了所对应的需要持久化的日志条目了，这里所做的，是将其丢入到一个执行队列只，所需的信息都交给了done这个属于`StableClosure*`类型的东西，之后将看到，它有很多种实现，分别对应的是Leader的`LeaderStableClosure`，和Follower的`FollowerStableClosure`。

```cpp
// 接 void LogManager::append_entries(std::vector<LogEntry*> *entries, StableClosure* done);
for (size_t i = 0; i < entries->size(); ++i) {
  // Add ref for disk_thread
  (*entries)[i]->AddRef();
  if ((*entries)[i]->type == ENTRY_TYPE_CONFIGURATION) {
    ConfigurationEntry conf_entry(*((*entries)[i]));
    _config_manager->add(conf_entry);
  }
}

if (!entries->empty()) {
  done->_first_log_index = entries->front()->id.index;
  _logs_in_memory.insert(_logs_in_memory.end(), entries->begin(), entries->end());
}

done->_entries.swap(*entries);
int ret = bthread::execution_queue_execute(_disk_queue, done);
CHECK_EQ(0, ret) << "execq execute failed, ret: " << ret << " err: " << berror();
wakeup_all_waiter(lck);
```

### 持久化

从加入到持久化队列后，程序流就来到了`disk_thread`身上了，我们先省略一些其他Closure的处理，只看append操作，从之前的`append_entries`函数我们可以明确的得知数据都有done传递了，到`disk_thread`种，done就是一个个的迭代器，同for循环可以取出这些数据，如果这些数据中的entry确实有数据，就将其加入到`AppendBatcher`中。

这里是一种优化操作，在`AppendBatcher::append`函数里，除非整合的entry达到了一定数量，否则是不会调用`flush()`来进行磁盘写入的，而只有到本次循环结束，或者是done中确实没有信息了，才会被调用刷盘。

```cpp
int LogManager::disk_thread(void* meta,
                            bthread::TaskIterator<StableClosure*>& iter) {
    if (iter.is_queue_stopped()) {
        return 0;
    }
    LogManager* log_manager = static_cast<LogManager*>(meta);
    // FIXME(chenzhangyi01): it's buggy
    LogId last_id = log_manager->_disk_id;
    StableClosure* storage[256];
    AppendBatcher ab(storage, ARRAY_SIZE(storage), &last_id, log_manager);
    
    for (; iter; ++iter) {
        if (!done->_entries.empty()) {
            ab.append(done);
        } else {
          	// 省略一些其他信号的Closure的处理
            ab.flush();
            done->Run();
        }
    }
    ab.flush();
    log_manager->set_disk_id(last_id);
    return 0;
}
```

而当刷盘返回后，这些数据都以落地了，同时，不仅仅如此，在这数据落地之后，__它调用了Closure中的Run，在Leader中，它会投出一票，这个票是Quorum中属于Leader本身的票(当前的那个LogEntry)，在Follower中，它会响应Leader节点的RPC__。

```cpp
 void flush() {
        if (_size > 0) {
            _lm->append_to_storage(&_to_append, _last_id, &metric);
            for (size_t i = 0; i < _size; ++i) {
                _storage[i]->_entries.clear();
                _storage[i]->Run();
            }
            _to_append.clear();
        }
        _size = 0;
        _buffer_size = 0;
    }
```

__在Leader节点上，append_entries已经结束了吗？在结束之前，Leader叫醒了一些线程，这些线程所做的事情，是将这些落地的日志条目发送到其他Follower节点上__。

### Wait

在LogManager中存在着这么一个字段：

```cpp
butil::FlatMap<int64_t, WaitMeta*> _wait_map
```

之后我们就可以转而看一个函数了，wait是做什么呢，它接收一个回调函数(当然还有调用这个回调所必须的参数了)，随后就调用了`notify_on_new_log`：

```cpp
LogManager::WaitId LogManager::wait(
        int64_t expected_last_log_index, 
        int (*on_new_log)(void *arg, int error_code), void *arg) {
    WaitMeta* wm = butil::get_object<WaitMeta>();
    if (BAIDU_UNLIKELY(wm == NULL)) {
        PLOG(FATAL) << "Fail to new WaitMeta";
        abort();
        return -1;
    }
    wm->on_new_log = on_new_log;
    wm->arg = arg;
    wm->error_code = 0;
    return notify_on_new_log(expected_last_log_index, wm);
}
```

通过对`notify_on_new_log`的观察，不难发现如果`expected_last_log_index != _last_log_index`时会立刻调用回调，而如何相同则不会，这是因为一个“发送”的线程总是知道`next_index`是多少，因为`next_index`是下一个需要进行同步的日志条目index，所以当他和`last_log_index`相等时，就表明Leader没有继续添加新的日志条目，而一旦不同，就证明Leader收到了新的日志条目了。

当然，它不会立刻被叫醒，只是乖乖的停在这个map中。

```cpp
LogManager::WaitId LogManager::notify_on_new_log(
        int64_t expected_last_log_index, WaitMeta* wm) {
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (expected_last_log_index != _last_log_index || _stopped) {
        wm->error_code = _stopped ? ESTOP : 0;
        lck.unlock();
        bthread_t tid;
        if (bthread_start_urgent(&tid, NULL, run_on_new_log, wm) != 0) {
            PLOG(ERROR) << "Fail to start bthread";
            run_on_new_log(wm);
        }
        return 0;  // Not pushed into _wait_map
    }
    if (_next_wait_id == 0) {  // skip 0
        ++_next_wait_id;
    }
    const int wait_id = _next_wait_id++;
    _wait_map[wait_id] = wm;
    return wait_id;
}
```

直到有人调用了这个`wakeup_all_waiter`，他就会叫醒所有等待者，在Leader中，这里的等待着通常有一个回调函数，就是`void Replicator::_wait_more_entries()`注册的

```cpp
_options.log_manager->wait(
                _next_index - 1, _continue_sending, (void*)_id.value);
```

它将启动下一次日志条目同步的RPC操作。



TODO: 丢弃已持久化的日志。

</font>