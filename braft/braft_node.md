<font face="Monaco">

# braft node

## 0x00 node init

Node节点的实际实现是NodeImpl，位于src/braft/node.h中，初始化init函数非常的长，这里仅仅截取部分：

```cpp
int NodeImpl::init(const NodeOptions& options) {
    // Create _fsm_caller first as log_manager needs it to report error
    _fsm_caller = new FSMCaller();
    // log storage and log manager init
    if (init_log_storage() != 0) {
        return -1;
    }

    if (init_fsm_caller(LogId(0, 0)) != 0) {
        return -1;
    }
}
```

分别是进行Log模块的初始化，还有fsm_caller的初始化：

```cpp
int NodeImpl::init_fsm_caller(const LogId& bootstrap_id) {
    CHECK(_fsm_caller);
    _closure_queue = new ClosureQueue(_options.usercode_in_pthread);
    // fsm caller init, node AddRef in init
    FSMCallerOptions fsm_caller_options;
    fsm_caller_options.usercode_in_pthread = _options.usercode_in_pthread;
    this->AddRef();
    fsm_caller_options.after_shutdown =
        brpc::NewCallback<NodeImpl*>(after_shutdown, this);
    fsm_caller_options.log_manager = _log_manager;
    fsm_caller_options.fsm = _options.fsm;
    fsm_caller_options.closure_queue = _closure_queue;
    fsm_caller_options.node = this;
    fsm_caller_options.bootstrap_id = bootstrap_id;
    const int ret = _fsm_caller->init(fsm_caller_options);
    if (ret != 0) {
        delete fsm_caller_options.after_shutdown;
        this->Release();
    }
    return ret;
}
```

其中我们比较关注的是传入到init的内容，比如`_log_manager`、`fsm`、`_closure_queue`、`this，也就是NodeImpl`。

## 0x01 FSMCaller

这个名字听起来就像是对StateMachine控制的一个类，它的内部属性有这些：

```cpp
class BAIDU_CACHELINE_ALIGNMENT FSMCaller {
private:
	bthread::ExecutionQueueId<ApplyTask> _queue_id;
    LogManager *_log_manager;
    StateMachine *_fsm;
    ClosureQueue* _closure_queue;
    butil::atomic<int64_t> _last_applied_index;
    int64_t _last_applied_term;
    google::protobuf::Closure* _after_shutdown;
    NodeImpl* _node;
    TaskType _cur_task;
    butil::atomic<int64_t> _applying_index;
    Error _error;
    bool _queue_started;
}
```

它的init函数会做一些初始化赋值，通过FSMCallerOptions来进行初始化：

```cpp
int FSMCaller::init(const FSMCallerOptions &options) {
    if (options.log_manager == NULL || options.fsm == NULL 
            || options.closure_queue == NULL) {
        return EINVAL;
    }
    _log_manager = options.log_manager;
    _fsm = options.fsm;
    _closure_queue = options.closure_queue;
    _after_shutdown = options.after_shutdown;
    _node = options.node;
    _last_applied_index.store(options.bootstrap_id.index,
                              butil::memory_order_relaxed);
    _last_applied_term = options.bootstrap_id.term;
    if (_node) {
        _node->AddRef();
    }
    
    bthread::ExecutionQueueOptions execq_opt;
    execq_opt.bthread_attr = options.usercode_in_pthread 
                             ? BTHREAD_ATTR_PTHREAD
                             : BTHREAD_ATTR_NORMAL;
    if (bthread::execution_queue_start(&_queue_id,
                                   &execq_opt,
                                   FSMCaller::run,
                                   this) != 0) {
        LOG(ERROR) << "fsm fail to start execution_queue";
        return -1;
    }
    _queue_started = true;
    return 0;
}
```

同时也会启动一个执行队列，跑在另外的一个“bthread”，为execution_queue，它所执行的，是FSMCaller::run，这些代码就是提交线程了，用来将已经commited的数据提交到状态机，这个状态机最终由我们实现，FSMCaller做的就是帮我们调用这些状态机的各种接口，当然，其API也是被定义在了StateMachine中了。

```cpp
// 这是一个static函数，所以meta传入FSMCaller*是必要的
int FSMCaller::run(void* meta, bthread::TaskIterator<ApplyTask>& iter) {
    FSMCaller* caller = (FSMCaller*)meta;
    if (iter.is_queue_stopped()) {
        caller->do_shutdown();
        return 0;
    }
    int64_t max_committed_index = -1;
    int64_t counter = 0;
    size_t  batch_size = FLAGS_raft_fsm_caller_commit_batch;
    for (; iter; ++iter) {
        if (iter->type == COMMITTED && counter < batch_size) {
            if (iter->committed_index > max_committed_index) {
                max_committed_index = iter->committed_index;
                counter++;
            }
        } else {
            if (max_committed_index >= 0) {
                caller->_cur_task = COMMITTED;
                caller->do_committed(max_committed_index);
                max_committed_index = -1;
                g_commit_tasks_batch_counter << counter;
                counter = 0;
                batch_size = FLAGS_raft_fsm_caller_commit_batch;
            }
            switch (iter->type) {
            case COMMITTED:
                if (iter->committed_index > max_committed_index) {
                    max_committed_index = iter->committed_index;
                    counter++;
                }
                break;
            case SNAPSHOT_SAVE:
                caller->_cur_task = SNAPSHOT_SAVE;
                if (caller->pass_by_status(iter->done)) {
                    caller->do_snapshot_save((SaveSnapshotClosure*)iter->done);
                }
                break;
            case SNAPSHOT_LOAD:
                caller->_cur_task = SNAPSHOT_LOAD;
                // TODO: do we need to allow the snapshot loading to recover the
                // StateMachine if possible?
                if (caller->pass_by_status(iter->done)) {
                    caller->do_snapshot_load((LoadSnapshotClosure*)iter->done);
                }
                break;
            case LEADER_STOP:
                caller->_cur_task = LEADER_STOP;
                caller->do_leader_stop(*(iter->status));
                delete iter->status;
                break;
            case LEADER_START:
                caller->do_leader_start(*(iter->leader_start_context));
                delete iter->leader_start_context;
                break;
            case START_FOLLOWING:
                caller->_cur_task = START_FOLLOWING;
                caller->do_start_following(*(iter->leader_change_context));
                delete iter->leader_change_context;
                break;
            case STOP_FOLLOWING:
                caller->_cur_task = STOP_FOLLOWING;
                caller->do_stop_following(*(iter->leader_change_context));
                delete iter->leader_change_context;
                break;
            case ERROR:
                caller->_cur_task = ERROR;
                caller->do_on_error((OnErrorClousre*)iter->done);
                break;
            case IDLE:
                CHECK(false) << "Can't reach here";
                break;
            };
        }
    }
    if (max_committed_index >= 0) {
        caller->_cur_task = COMMITTED;
        caller->do_committed(max_committed_index);
        g_commit_tasks_batch_counter << counter;
        counter = 0;
    }
    caller->_cur_task = IDLE;
    return 0;
}

```

### 1. type == COMMITED

提交内容的情况下，每次最多提交batch_size个，默认是512，然后调用do_commited：

```cpp
void FSMCaller::do_committed(int64_t committed_index) {
    if (!_error.status().ok()) {
        return;
    }
    int64_t last_applied_index = _last_applied_index.load(
                                        butil::memory_order_relaxed);
	// 回滚肯定是不允许的
    // We can tolerate the disorder of committed_index
    if (last_applied_index >= committed_index) {
        return;
    }
    std::vector<Closure*> closure;
    int64_t first_closure_index = 0;
    CHECK_EQ(0, _closure_queue->pop_closure_until(committed_index, &closure,
                                                  &first_closure_index));
	// 获得这些回调的数据，然后一个个的遍历它们
    IteratorImpl iter_impl(_fsm, _log_manager, &closure, first_closure_index,
                 last_applied_index, committed_index, &_applying_index);
    for (; iter_impl.is_good();) {
        if (iter_impl.entry()->type != ENTRY_TYPE_DATA) {
            if (iter_impl.entry()->type == ENTRY_TYPE_CONFIGURATION) {
                if (iter_impl.entry()->old_peers == NULL) {
                    // Joint stage is not supposed to be noticeable by end users.
                    _fsm->on_configuration_committed(
                            Configuration(*iter_impl.entry()->peers),
                            iter_impl.entry()->id.index);
                }
            }
            // For other entries, we have nothing to do besides flush the
            // pending tasks and run this closure to notify the caller that the
            // entries before this one were successfully committed and applied.
            if (iter_impl.done()) {
                iter_impl.done()->Run();
            }
            iter_impl.next();
            continue;
        }
        Iterator iter(&iter_impl);
        // 这里的on_apply就是真正的用户实现的函数
        _fsm->on_apply(iter);
        // Try move to next in case that we pass the same log twice.
        iter.next();
    }
    if (iter_impl.has_error()) {
        set_error(iter_impl.error());
        iter_impl.run_the_rest_closure_with_error();
    }
    const int64_t last_index = iter_impl.index() - 1;
    const int64_t last_term = _log_manager->get_term(last_index);
    LogId last_applied_id(last_index, last_term);
    _last_applied_index.store(committed_index, butil::memory_order_release);
    _last_applied_term = last_term;
    _log_manager->set_applied_id(last_applied_id);
}
```

















</font>