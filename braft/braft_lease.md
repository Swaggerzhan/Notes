<font face="Monaco">

# braft lease

## 0x00 why?

raft组在复制的过程中需要进行日志同步的操作，这是一种串行化的操作，当raft需要对外提供一种线性一致性的服务时就会出现很多问题。

### write

在线性一致性的情况下，写入无可厚非，需要等到集群中大多数节点都同步好才会进行响应。

### read

线性一致性下读取就有很多种操作了，比较常见的是只能对Leader所属的节点进行读取操作，理论上这是完全符合线性一致性的，但网络分区的存在使得Leader有可能被孤立，如此情况下老Leader可能会响应客户端旧数据使得破坏一致性。

一般而言，对于这种情况下比较简单的解决办法就是read也走一遍日志同步，这样做的过程就是确立Leader有效的一种过程，但对于性能而已实在称不上多好。

__Lease是一种解决办法，因为read也走一遍日志就是为了确立leader的合法性，而raft中，每一个follower只要收到来自leader的一种rpc都会进行重置定时器(假设这些rpc逻辑上都是成功的)，所以在Leader端就有一种方法，记录每一轮成功响应的rpc时间`last_active_timestamp`，那么我们就可以假设这些Follower在某段时间内肯定不会抢主：`last_active_timestamp + election_timeout`，这些都是建立在所有节点上的时钟速度比较一致的情况下的__。

而事实上这套保证应该是：`last_active_timestamp + election_timeout / clock drift bound`。

并且Lease并不能绝对的保证线性一致性，它也存在一些问题，比如当Leader完成一次heart_beat后还是有一些漏网之鱼进行了新一轮的Leader选举，这就需要FollowerLease了。

## 0x01 Leader Lease

在braft中，当一个节点当选为Leader时，就会启动LeaderLease， 在`src/braft/node.cpp become_leader()`时就有体现出来，其节点会调用：

```cpp
_leader_lease.on_leader_start(_current_term);
```

来确立当前任期中的lease，随后的每次RPC操作中，都会在RPC得到响应的时候进行记录，主要代码在Replicator中：

```cpp
void _update_last_rpc_send_timestamp(int64_t new_timestamp) {
  if (new_timestamp > _last_rpc_send_timestamp()) {
    _options.replicator_status->last_rpc_send_timestamp
      .store(new_timestamp, butil::memory_order_relaxed);
  }
}
```

__当start时，Leader会组建属于它的复制组ReplicatorGroup，为每一个Follower节点创建一个Replicator来管理这些复制操作，同时Replicator的start会更新最后一个rpc的时间(我想应该是此时Leader节点刚刚赢得选举的缘由)，heart_beat_rpc返回，rpc返回时都会进行记录__。

LeaderLease在term为0时代表失效，last_active_timestamp为0则代表还未完全初始化，当然还有一种情况，就是Lease超时了(很久没有去查Replicator中的last_active_timestamp了)，这就需要重新查询。

### 使用

在Leader节点上，通过调用：

```cpp
bool is_leader_lease_valid();
void NodeImpl::get_leader_lease_status(LeaderLeaseStatus* lease_status);
```

而它会继续调用LeaderLease的函数：

```cpp
void LeaderLease::get_lease_info(LeaseInfo* lease_info);
```

如果得到SUSPEND，那么就会更深入的去ReplicatorGroup中查询对应的last_active_timestamp了：

```cpp
int64_t NodeImpl::last_leader_active_timestamp(const Configuration& conf);
```

### 过期

```cpp
void NodeImpl::get_leader_lease_status(LeaderLeaseStatus* lease_status);
```

中如果发现缓存的Lease过期，就会到ReplicatorGroup查，如果这里的Lease也已经过期，那么就会调用NodeImpl的step_down来退位。

## 0x02 Follower Lease

FollowerLease保证在`last_leader_timestamp + election_timeout / clock drift bound`的时间内，不会将选票投给其他节点。

在FollowerLease中比较关注的就是这个函数：

```cpp
int64_t FollowerLease::votable_time_from_now() {
    if (!FLAGS_raft_enable_leader_lease) {
        return 0;
    }

    int64_t now = butil::monotonic_time_ms();
    int64_t votable_timestamp = _last_leader_timestamp + _election_timeout_ms +
                                _max_clock_drift_ms;
    if (now >= votable_timestamp) {
        return 0;
    }
    return votable_timestamp - now;
}
```

通过FollowerLease，其他网络分区节点即使没有收到quorum check也没有办法获得到足够的选票。

FollowerLease在每次成功通过检查的rpc操作中(AppendEntries)都会进行重制。

</font>