<font face="Monaco">

# braft AppendEntries

## 0x00 AppendEntries

todo

我们从状态机的apply操作开始，一步步分析这些日志是如何从Leader的上层rpc service分发到各个Follower上的。

## 0x01 request

首先从NodeImpl::apply开始，这就是用户提交raft log的入口了

```cpp
void NodeImpl::apply(const Task& task) {
    LogEntry* entry = new LogEntry;
    entry->AddRef();
    entry->data.swap(*task.data);
    LogEntryAndClosure m;
    m.entry = entry;
    m.done = task.done;
    m.expected_term = task.expected_term;
    if (_apply_queue->execute(m, &bthread::TASK_OPTIONS_INPLACE, NULL) != 0) {
        task.done->status().set_error(EPERM, "Node is down");
        entry->Release();
        return run_closure_in_bthread(task.done);
    }
}
```

可以看到，这里只是简单的将任务推到对应的队列中而已，从队列的另外一头，可以看到是一个batch的实现，堆砌一段entry后在使用真正的apply将其提交：

```cpp
	int NodeImpl::execute_applying_tasks(
        void* meta, bthread::TaskIterator<LogEntryAndClosure>& iter) {
    if (iter.is_queue_stopped()) {
        return 0;
    }
    // TODO: the batch size should limited by both task size and the total log
    // size
    const size_t batch_size = FLAGS_raft_apply_batch;
    DEFINE_SMALL_ARRAY(LogEntryAndClosure, tasks, batch_size, 256);
    size_t cur_size = 0;
    NodeImpl* m = (NodeImpl*)meta;
    for (; iter; ++iter) {
        if (cur_size == batch_size) {
            m->apply(tasks, cur_size);
            cur_size = 0;
        }
        tasks[cur_size++] = *iter;
    }
    if (cur_size > 0) {
        m->apply(tasks, cur_size);
    }
    return 0;
}
```

就是这里：

```cpp
void NodeImpl::apply(LogEntryAndClosure tasks[], size_t size) {
    std::vector<LogEntry*> entries;
    entries.reserve(size);
    std::unique_lock<raft_mutex_t> lck(_mutex);
    bool reject_new_user_logs = (_node_readonly || _majority_nodes_readonly);
    if (_state != STATE_LEADER || reject_new_user_logs) {
        lck.unlock();
        for (size_t i = 0; i < size; ++i) {
            tasks[i].entry->Release();
            if (tasks[i].done) {
                tasks[i].done->status() = st;
                run_closure_in_bthread(tasks[i].done);
            }
        }
        return;
    }
    for (size_t i = 0; i < size; ++i) {
        if (tasks[i].expected_term != -1 && tasks[i].expected_term != _current_term) {
            if (tasks[i].done) {
                run_closure_in_bthread(tasks[i].done);
            }
            tasks[i].entry->Release();
            continue;
        }
        entries.push_back(tasks[i].entry);
        entries.back()->id.term = _current_term;
        entries.back()->type = ENTRY_TYPE_DATA;
        _ballot_box->append_pending_task(_conf.conf,
                                         _conf.stable() ? NULL : &_conf.old_conf,
                                         tasks[i].done);
    }
    _log_manager->append_entries(&entries,
                               new LeaderStableClosure(
                                        NodeId(_group_id, _server_id),
                                        entries.size(),
                                        _ballot_box));
    // update _conf.first
    _log_manager->check_and_set_configuration(&_conf);
}
```

这里做了一些check，具体可以去看看详细的代码，其中比较重要的有：



一开始是做状态的检测，如果不是Leader，就直接原地销毁这些task里的entry，并且调用done(由用户定义的，一般是本次rpc的响应)。

第二次检测是检测任期是否和预期的任期发生了改变，如果改变了(可能是收到了更高任期的Leader了)，那么同样销毁，并且调用done。

否则就添加到entries中，并且添加一个task到ballot_box中，然后就进行日志持久化了，调用的是`_log_manager->append_entries`。



在另外的一条线程上有一个wait：

```cpp
void Replicator::_wait_more_entries() {
    if (_wait_id == 0 && FLAGS_raft_max_entries_size > _flying_append_entries_size &&
        (size_t)FLAGS_raft_max_parallel_append_entries_rpc_num > _append_entries_in_fly.size()) {
        _wait_id = _options.log_manager->wait(
                _next_index - 1, _continue_sending, (void*)_id.value);
        _is_waiter_canceled = false;
    }
    if (_flying_append_entries_size == 0) {
        _st.st = IDLE;
    }
    CHECK_EQ(0, bthread_id_unlock(_id)) << "Fail to unlock " << _id;
}
```

`wait_more_entries`会等待发送日志，当被叫醒时会调用`_continue_sending`：

```cpp
int Replicator::_continue_sending(void* arg, int error_code) {
    Replicator* r = NULL;
    bthread_id_t id = { (uint64_t)arg };
    if (bthread_id_lock(id, (void**)&r) != 0) {
        return -1;
    }
    if (error_code == ETIMEDOUT) {
        // Replication is in progress when block timeout, no need to start again
        // this case can happen when 
        //     1. pipeline is enabled and 
        //     2. disable readonly mode triggers another replication
        if (r->_wait_id != 0) {
            return 0;
        }
        
        // Send empty entries after block timeout to check the correct
        // _next_index otherwise the replictor is likely waits in
        // _wait_more_entries and no further logs would be replicated even if the
        // last_index of this followers is less than |next_index - 1|
        r->_send_empty_entries(false);
    } else if (error_code != ESTOP && !r->_is_waiter_canceled) {
        // id is unlock in _send_entries
        r->_wait_id = 0;
        r->_send_entries();
    } else if (r->_is_waiter_canceled) {
        // The replicator is checking current next index by sending empty entries or
        // install snapshot now. Although the registered waiter will be canceled
        // before the operations, there is still a little chance that LogManger already
        // waked up the waiter, and _continue_sending is waiting to execute.
        bthread_id_unlock(id);
    } else {
        bthread_id_unlock(id);
    }
    return 0;
}
```

比如从`_send_entries`开始：

```cpp
void Replicator::_send_entries() {
    if (_flying_append_entries_size >= FLAGS_raft_max_entries_size ||
        _append_entries_in_fly.size() >= (size_t)FLAGS_raft_max_parallel_append_entries_rpc_num ||
        _st.st == BLOCKING) {
        CHECK_EQ(0, bthread_id_unlock(_id)) << "Fail to unlock " << _id;
        return;
    }

    std::unique_ptr<brpc::Controller> cntl(new brpc::Controller);
    std::unique_ptr<AppendEntriesRequest> request(new AppendEntriesRequest);
    std::unique_ptr<AppendEntriesResponse> response(new AppendEntriesResponse);
    if (_fill_common_fields(request.get(), _next_index - 1, false) != 0) {
        _reset_next_index();
        return _install_snapshot();
    }
    EntryMeta em;
    const int max_entries_size = FLAGS_raft_max_entries_size - _flying_append_entries_size;
    int prepare_entry_rc = 0;
    CHECK_GT(max_entries_size, 0);
    for (int i = 0; i < max_entries_size; ++i) {
        prepare_entry_rc = _prepare_entry(i, &em, &cntl->request_attachment());
        if (prepare_entry_rc != 0) {
            break;
        }
        request->add_entries()->Swap(&em);
    }
    if (request->entries_size() == 0) {
        // _id is unlock in _wait_more
        if (_next_index < _options.log_manager->first_log_index()) {
            _reset_next_index();
            return _install_snapshot();
        }
        // NOTICE: a follower's readonly mode does not prevent install_snapshot
        // as we need followers to commit conf log(like add_node) when 
        // leader reaches readonly as well 
        if (prepare_entry_rc == EREADONLY) {
            if (_flying_append_entries_size == 0) {
                _st.st = IDLE;
            }
            CHECK_EQ(0, bthread_id_unlock(_id)) << "Fail to unlock " << _id;
            return;
        }
        return _wait_more_entries();
    }

    _append_entries_in_fly.push_back(FlyingAppendEntriesRpc(_next_index,
                                     request->entries_size(), cntl->call_id()));
    _append_entries_counter++;
    _next_index += request->entries_size();
    _flying_append_entries_size += request->entries_size();
    
    _st.st = APPENDING_ENTRIES;
    _st.first_log_index = _min_flying_index();
    _st.last_log_index = _next_index - 1;
    google::protobuf::Closure* done = brpc::NewCallback(
                _on_rpc_returned, _id.value, cntl.get(), 
                request.get(), response.get(), butil::monotonic_time_ms());
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(), 
                        response.release(), done);
    _wait_more_entries();
}
```

然后进行request的打包，调用rpc将entry发送至follower上去，之后继续等待是否有新的请求到来。

这里我们再回到之前的那个apply的结尾中去：

```cpp
_log_manager->append_entries(&entries,
                             new LeaderStableClosure(
                               NodeId(_group_id, _server_id),
                               entries.size(),
                               _ballot_box));
// update _conf.first
_log_manager->check_and_set_configuration(&_conf);
```

append_entries除了会将数据交给disk_thread持久化外，还会将LeaderStableClosure调用起来，投出属于leader节点本身的一票，然后调用`wakeup_all_waiter(lck)`叫醒由`_wait_more_entries()`等待的线程。

## 0x02 handle

日志从Leader发送之后就到了Follower上，Follower的入口是`NodeImpl::handle_append_entries_request`，首先是很多的内容检测，比如任期检测、脑裂检测。

之后就是更新FollowerLease，如果正在安装快照，那么就先驳回日志同步的请求。

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    std::vector<LogEntry*> entries;
    entries.reserve(request->entries_size());
    brpc::ClosureGuard done_guard(done);
    std::unique_lock<raft_mutex_t> lck(_mutex);

    // pre set term, to avoid get term in lock
    response->set_term(_current_term);

    if (!is_active_state(_state)) {
        const int64_t saved_current_term = _current_term;
        const State saved_state = _state;
        lck.unlock();
        return;
    }

    PeerId server_id;
    if (0 != server_id.parse(request->server_id())) {
        lck.unlock();
        return;
    }

    // check stale term
    if (request->term() < _current_term) {
        const int64_t saved_current_term = _current_term;
        lck.unlock();
        response->set_success(false);
        response->set_term(saved_current_term);
        return;
    }

    // check term and state to step down
    check_step_down(request->term(), server_id);   
     
    if (server_id != _leader_id) {
        // Increase the term by 1 and make both leaders step down to minimize the
        // loss of split brain
        butil::Status status;
        status.set_error(ELEADERCONFLICT, "More than one leader in the same term."); 
        step_down(request->term() + 1, false, status);
        response->set_success(false);
        response->set_term(request->term() + 1);
        return;
    }

    if (!from_append_entries_cache) {
        // Requests from cache already updated timestamp
        _follower_lease.renew(_leader_id);
    }

    if (request->entries_size() > 0 &&
            (_snapshot_executor
                && _snapshot_executor->is_installing_snapshot())) {
        cntl->SetFailed(EBUSY, "Is installing snapshot");
        return;
    }

    const int64_t prev_log_index = request->prev_log_index();
    const int64_t prev_log_term = request->prev_log_term();
    const int64_t local_prev_log_term = _log_manager->get_term(prev_log_index);
    if (local_prev_log_term != prev_log_term) {
        int64_t last_index = _log_manager->last_log_index();
        int64_t saved_term = request->term();
        int     saved_entries_size = request->entries_size();
        std::string rpc_server_id = request->server_id();
        if (!from_append_entries_cache &&
            handle_out_of_order_append_entries(
                    cntl, request, response, done, last_index)) {
            // It's not safe to touch cntl/request/response/done after this point,
            // since the ownership is tranfered to the cache.
            lck.unlock();
            done_guard.release();
            return;
        }

        response->set_success(false);
        response->set_term(_current_term);
        response->set_last_log_index(last_index);
        lck.unlock();
        if (local_prev_log_term != 0) {
        }
        return;
    }

    if (request->entries_size() == 0) {
        response->set_success(true);
        response->set_term(_current_term);
        response->set_last_log_index(_log_manager->last_log_index());
        response->set_readonly(_node_readonly);
        lck.unlock();
        // see the comments at FollowerStableClosure::run()
        _ballot_box->set_last_committed_index(
                std::min(request->committed_index(),
                         prev_log_index));
        return;
    }

    // Parse request
    butil::IOBuf data_buf;
    data_buf.swap(cntl->request_attachment());
    int64_t index = prev_log_index;
    for (int i = 0; i < request->entries_size(); i++) {
        index++;
        const EntryMeta& entry = request->entries(i);
        if (entry.type() != ENTRY_TYPE_UNKNOWN) {
            LogEntry* log_entry = new LogEntry();
            log_entry->AddRef();
            log_entry->id.term = entry.term();
            log_entry->id.index = index;
            log_entry->type = (EntryType)entry.type();
            if (entry.peers_size() > 0) {
                log_entry->peers = new std::vector<PeerId>;
                for (int i = 0; i < entry.peers_size(); i++) {
                    log_entry->peers->push_back(entry.peers(i));
                }
                CHECK_EQ(log_entry->type, ENTRY_TYPE_CONFIGURATION);
                if (entry.old_peers_size() > 0) {
                    log_entry->old_peers = new std::vector<PeerId>;
                    for (int i = 0; i < entry.old_peers_size(); i++) {
                        log_entry->old_peers->push_back(entry.old_peers(i));
                    }
                }
            } else {
                CHECK_NE(entry.type(), ENTRY_TYPE_CONFIGURATION);
            }
            if (entry.has_data_len()) {
                int len = entry.data_len();
                data_buf.cutn(&log_entry->data, len);
            }
            entries.push_back(log_entry);
        }
    }

    // check out-of-order cache
    check_append_entries_cache(index);

    FollowerStableClosure* c = new FollowerStableClosure(
            cntl, request, response, done_guard.release(),
            this, _current_term);
    _log_manager->append_entries(&entries, c);

    // update configuration after _log_manager updated its memory status
    _log_manager->check_and_set_configuration(&_conf);
}
```

随后解析正确的日志条目，随后同样调用append_entries，不同的是，它传入的回调是FollowerStableClosure，它的Run是响应本次日志同步的rpc。

在这个回调的内部，同样也有一些检测操作：

```cpp
void run() {
  brpc::ClosureGuard done_guard(_done);
  if (!status().ok()) {
    _cntl->SetFailed(status().error_code(), "%s",
                     status().error_cstr());
    return;
  }
  std::unique_lock<raft_mutex_t> lck(_node->_mutex);
  if (_term != _node->_current_term) {
    // The change of term indicates that leader has been changed during
    // appending entries, so we can't respond ok to the old leader
    // because we are not sure if the appended logs would be truncated
    // by the new leader:
    //  - If they won't be truncated and we respond failure to the old
    //    leader, the new leader would know that they are stored in this
    //    peer and they will be eventually committed when the new leader
    //    found that quorum of the cluster have stored.
    //  - If they will be truncated and we responded success to the old
    //    leader, the old leader would possibly regard those entries as
    //    committed (very likely in a 3-nodes cluster) and respond
    //    success to the clients, which would break the rule that
    //    committed entries would never be truncated.
    // So we have to respond failure to the old leader and set the new
    // term to make it stepped down if it didn't.
    _response->set_success(false);
    _response->set_term(_node->_current_term);
    return;
  }
  // It's safe to release lck as we know everything is ok at this point.
  lck.unlock();

  // DON'T touch _node any more
  _response->set_success(true);
  _response->set_term(_term);

  const int64_t committed_index =
    std::min(_request->committed_index(),
             // ^^^ committed_index is likely less than the
             // last_log_index
             _request->prev_log_index() + _request->entries_size()
             // ^^^ The logs after the appended entries are
             // untrustable so we can't commit them even if their
             // indexes are less than request->committed_index()
            );
  //_ballot_box is thread safe and tolerates disorder.
  _node->_ballot_box->set_last_committed_index(committed_index);
  int64_t now = butil::cpuwide_time_us();
  if (FLAGS_raft_trace_append_entry_latency && now - metric.start_time_us > 
      (int64_t)FLAGS_raft_append_entry_high_lat_us) {
  }
}
```

比如发生了任期改变等，都需要响应错误，避免老leader作出错误的响应。

## 0x03 return

对rpc响应的操作在Repliactor中`Replicator::_on_rpc_returned`：

```cpp
void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                     AppendEntriesRequest* request, 
                     AppendEntriesResponse* response,
                     int64_t rpc_send_time) {
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    std::unique_ptr<AppendEntriesRequest>  req_guard(request);
    std::unique_ptr<AppendEntriesResponse> res_guard(response);
    Replicator *r = NULL;
    bthread_id_t dummy_id = { id };
    const long start_time_us = butil::gettimeofday_us();
    if (bthread_id_lock(dummy_id, (void**)&r) != 0) {
        return;
    }

    std::stringstream ss;
    ss << "node " << r->_options.group_id << ":" << r->_options.server_id 
       << " received AppendEntriesResponse from "
       << r->_options.peer_id << " prev_log_index " << request->prev_log_index()
       << " prev_log_term " << request->prev_log_term() << " count " << request->entries_size();

    bool valid_rpc = false;
    int64_t rpc_first_index = request->prev_log_index() + 1;
    int64_t min_flying_index = r->_min_flying_index();
    CHECK_GT(min_flying_index, 0);

    for (std::deque<FlyingAppendEntriesRpc>::iterator rpc_it = r->_append_entries_in_fly.begin();
        rpc_it != r->_append_entries_in_fly.end(); ++rpc_it) {
        if (rpc_it->log_index > rpc_first_index) {
            break;
        }
        if (rpc_it->call_id == cntl->call_id()) {
            valid_rpc = true;
        }
    }
    if (!valid_rpc) {
        ss << " ignore invalid rpc";
        BRAFT_VLOG << ss.str();
        CHECK_EQ(0, bthread_id_unlock(r->_id)) << "Fail to unlock " << r->_id;
        return;
    }

    if (cntl->Failed()) {
        ss << " fail, sleep.";
        BRAFT_VLOG << ss.str();

        // TODO: Should it be VLOG?
        LOG_IF(WARNING, (r->_consecutive_error_times++) % 10 == 0)
                        << "Group " << r->_options.group_id
                        << " fail to issue RPC to " << r->_options.peer_id
                        << " _consecutive_error_times=" << r->_consecutive_error_times
                        << ", " << cntl->ErrorText();
        // If the follower crashes, any RPC to the follower fails immediately,
        // so we need to block the follower for a while instead of looping until
        // it comes back or be removed
        // dummy_id is unlock in block
        r->_reset_next_index();
        return r->_block(start_time_us, cntl->ErrorCode());
    }
    r->_consecutive_error_times = 0;
    if (!response->success()) {
        if (response->term() > r->_options.term) {
            BRAFT_VLOG << " fail, greater term " << response->term()
                       << " expect term " << r->_options.term;
            r->_reset_next_index();

            NodeImpl *node_impl = r->_options.node;
            // Acquire a reference of Node here in case that Node is destroyed
            // after _notify_on_caught_up.
            node_impl->AddRef();
            r->_notify_on_caught_up(EPERM, true);
            butil::Status status;
            status.set_error(EHIGHERTERMRESPONSE, "Leader receives higher term "
                    "%s from peer:%s", response->GetTypeName().c_str(), r->_options.peer_id.to_string().c_str());
            r->_destroy();
            node_impl->increase_term_to(response->term(), status);
            node_impl->Release();
            return;
        }
        ss << " fail, find next_index remote last_log_index " << response->last_log_index()
           << " local next_index " << r->_next_index 
           << " rpc prev_log_index " << request->prev_log_index();
        BRAFT_VLOG << ss.str();
        r->_update_last_rpc_send_timestamp(rpc_send_time);
        // prev_log_index and prev_log_term doesn't match
        r->_reset_next_index();
        if (response->last_log_index() + 1 < r->_next_index) {
            BRAFT_VLOG << "Group " << r->_options.group_id
                       << " last_log_index at peer=" << r->_options.peer_id 
                       << " is " << response->last_log_index();
            // The peer contains less logs than leader
            r->_next_index = response->last_log_index() + 1;
        } else {  
            // The peer contains logs from old term which should be truncated,
            // decrease _last_log_at_peer by one to test the right index to keep
            if (BAIDU_LIKELY(r->_next_index > 1)) {
                BRAFT_VLOG << "Group " << r->_options.group_id 
                           << " log_index=" << r->_next_index << " mismatch";
                --r->_next_index;
            } else {
                LOG(ERROR) << "Group " << r->_options.group_id 
                           << " peer=" << r->_options.peer_id
                           << " declares that log at index=0 doesn't match,"
                              " which is not supposed to happen";
            }
        }
        // dummy_id is unlock in _send_heartbeat
        r->_send_empty_entries(false);
        return;
    }

    ss << " success";
    BRAFT_VLOG << ss.str();
    
    if (response->term() != r->_options.term) {
        LOG(ERROR) << "Group " << r->_options.group_id
                   << " fail, response term " << response->term()
                   << " mismatch, expect term " << r->_options.term;
        r->_reset_next_index();
        CHECK_EQ(0, bthread_id_unlock(r->_id)) << "Fail to unlock " << r->_id;
        return;
    }
    r->_update_last_rpc_send_timestamp(rpc_send_time);
    const int entries_size = request->entries_size();
    const int64_t rpc_last_log_index = request->prev_log_index() + entries_size;
    BRAFT_VLOG_IF(entries_size > 0) << "Group " << r->_options.group_id
                                    << " replicated logs in [" 
                                    << min_flying_index << ", " 
                                    << rpc_last_log_index
                                    << "] to peer " << r->_options.peer_id;
    if (entries_size > 0) {
        r->_options.ballot_box->commit_at(
                min_flying_index, rpc_last_log_index,
                r->_options.peer_id);
        int64_t rpc_latency_us = cntl->latency_us();
        if (FLAGS_raft_trace_append_entry_latency && 
            rpc_latency_us > FLAGS_raft_append_entry_high_lat_us) {
            LOG(WARNING) << "append entry rpc latency us " << rpc_latency_us
                         << " greater than " 
                         << FLAGS_raft_append_entry_high_lat_us
                         << " Group " << r->_options.group_id
                         << " to peer  " << r->_options.peer_id
                         << " request entry size " << entries_size
                         << " request data size " 
                         <<  cntl->request_attachment().size();
        }
        g_send_entries_latency << cntl->latency_us();
        if (cntl->request_attachment().size() > 0) {
            g_normalized_send_entries_latency << 
                cntl->latency_us() * 1024 / cntl->request_attachment().size();
        }
    }
    // A rpc is marked as success, means all request before it are success,
    // erase them sequentially.
    while (!r->_append_entries_in_fly.empty() &&
           r->_append_entries_in_fly.front().log_index <= rpc_first_index) {
        r->_flying_append_entries_size -= r->_append_entries_in_fly.front().entries_size;
        r->_append_entries_in_fly.pop_front();
    }
    r->_has_succeeded = true;
    r->_notify_on_caught_up(0, false);
    // dummy_id is unlock in _send_entries
    if (r->_timeout_now_index > 0 && r->_timeout_now_index < r->_min_flying_index()) {
        r->_send_timeout_now(false, false);
    }
    r->_send_entries();
    return;
}
```

首先是一系列的检查：如果rpc出现错误，则大概率是Follower挂掉了，那么先block一段时间在继续发送rpc，也有可能Follower驳回了日志同步的要求：

一种原因是Follower有更高的任期，那么它就需要退位，即清理Repliactor等，另外一种情况就是角色不变，是日志条目中prev_log_index的任期不同导致的，这种情况下需要更新last_active_timestamp(lease使用)，然后重新设定这个对应的节点的next_index，随后再次发送append_entries的操作。

__如果检查都通过了，那么就是成功的日志条目同步，在更新完last_active_timestamp后开始计算选票，通过对应的ballot_box中的队列来对某个日志的index进行投票操作(这通常是一个范围的投票，for循环对这个范围投票)，如果达成Quorum检查，那么就将日志条目提交__。

其中被commited的ballot会被移除出ballot_box的队列，一些比较晚到的rpc会直接判断从而直接响应：

```cpp
if (last_log_index < _pending_index) {
  return 0;
}
```











<font>