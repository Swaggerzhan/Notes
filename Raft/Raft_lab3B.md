# 6.824 Lab3B Raft学习笔记

## 0x00 理论

在原论文中，关于Snapshot的实现讨论比较的少，所以我觉得3B算是这些lab里比较难的一个了，实现了3B，接下来的大山就是集群间维护的问题了。

## 0x01 设计思路

### Raft layer

Snapshot的操作涉及到kv层以及raft层，在原先的lab2C中，我们只持久化了3个字段，这回就需要增加2个新字段了：

```go
type Raft struct {
    log                 []LogEntry
    voteFor             int
    currentTerm         int
    lastIncludeIndex    int
    lastIncludeTerm     int
}
```

重点在后2个字段，`lastIncludeIndex`代表着snapshot中最后一个日志条目的Index，而`lastIncludeTerm`也是`lastIncludeIndex`的日志条目任期。

考虑一下这种情况：Raft层收到日志压缩的操作，并且凑巧的是所有的log都被drop掉了，这时就不能再使用`PrevLogIndex以及PrevLogTerm`了，需要该用`lastIncludeIndex和lastIncludeTerm`。

在2B中，Leader通过`AppendEntries`来强制同步Follower上的日志条目，不过现在Leader的同步手段不止是发送日志条目了，Leader还可以通过发送Snapshot来提高同步的速度，所以Follower需要再暴露一个RPC接口，这里设计为`InstallSnapshotRPC`，其参数和响应为：

```go
type InstallSnapshotArgs struct {
    Term                int
    LeaderID            int
    LastIncludeIndex    int
    LastIncludeTerm     int
    Offset              int
    Data                []byte
    Done                bool
}
```



## 0x02 代码实现

KVServer层时检测Raft层的log长度，当发现log长度已经很长了，那么我们将生成一个snapshot。

```go
func (kv *KVServer)snapshotLoop() {
    for !kv.killed() {
        var snapshot []byte
        var lastIncludeIndex int
        func () {
            // 检测一下是否日志太长了
            if kv.maxraftstate != -1 && kv.rf.ExceedLogSize(kv.maxraftstate){
                kv.mu.Lock()
                w := new(bytes.Buffer)
                e := labgob.NewEncoder(w)
                e.Encode(kv.kvData)
                e.Encode(kv.lastAppliedMap)
                snapshot := w.Bytes()
                lastIncludeIndex = kv.lastAppliedIndex
                kv.mu.Unlock()
            }
        }()
        
        if snapshot != nil { // send to Raft layer
            kv.rf.TakeSnapshot(snapshot, lastIncludeIndex)
        }
        time.Sleep(10 * time.Millisecond)
    }
}


func (rf *Raft) ExceedLogSize(logSize int) bool {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    
    if rf.persister.RaftStateSize() >= logSize {
        return true
    }
    return false
}

```

在Raft层：

```go
func (rf *Raft) TakeSnapshot(snapshot []byte, lastIncludeIndex int){
    rf.mu.Lock()
    defer rf.mu.Unlock()
    
    // 存在更高的snapshot
    if lastIncludeIndex <= rf.lastIncludeIndex {
        return
    }
    
    zipLogLen := lastIncludeIndex - rf.lastIncludeIndex
    
    rf.lastIncludeTerm = rf.log[rf.index2LogPos(lastIncludeIndex)].Term
    rf.lastIncludeIndex = lastIncludeIndex
    
    // 压缩日志
    afterLog := make([]logEntry, len(rf.log) - zipLogLen)
    copy(afterLog, rf.log[zipLogLen:])
    rf.log = afterLog
    
    // 持久化Raft和Snapshot
    rf.persister.SaveStateAndSnapshot(rf.getPersistData(), snapshot)
    
}
```

Raft层向上层发送Snapshot信息：

```go
func (kv *KVServer) applyLog() {
	for {
		select {
		case msg := <- kv.applyCh:
	       if !msg.CommandValid {
	       
	       }else {
	           
	       }
			
		}
	}
}
```

需要持久化的：

```go
func (rf *Raft) getPersistData() []byte {
    w := new(bytes.Buffer)
    e := labgob.NewEncoder(w)
    e.Encode(rf.currentTerm)
    e.Encode(rf.voteFor)
    e.Encode(rf.log)
    e.Encode(rf.lastIncludeIndex)
    e.Encode(rf.lastIncludeTerm)
    data := w.Bytes()
    return data
}

func (rf *Raft) persist() {
    data := rf.getPersisitData()
    rf.persister.SaveRaftState(data)
}

func (rf *Raft) readPersist() {
    r := bytes.NewBuffer(data)
    d := labgob.NewDecoder(r)
    
    var currentTerm int
    var voteFor int
    var log []LogEntry
    var lastIncludeIndex int
    var lastIncludeTerm int
    
    if d.Decode(&currentTerm) != nil || 
        d.Decode(&voteFor) != nil ||
        d.Decode(&log) != nil ||
        d.Decode(&lastIncludeIndex) ||
        d.Decode(&lastIncludeTerm) {
            // Error
        }else {
            rf.currentTerm = currentTerm
            rf.voteFor = voteFor
            rf.log = log
            rf.lastIncludeIndex =lastIncludeIndex
            rf.lastIncludeTerm = lastIcludeTerm
        }
}
```

### Follower接收Snapshot

在Raft层的`AppendEntriesRPC`中，Follower接受来自Leader的日志条目，与2B不同的是，此时Leader有了更快的同步手段Snapshot：

```go
func (rf *Raft)InstallSnapshotRPC(args *InstallSanpshotArgs, reply *InstallSnapshotReply){
    rf.mu.Lock()
    defer rf.mu.Unlock()
    
    // 对方任期太低了
    if rf.currentTerm > args.Term {
        return
    }
    
    rf.lastActiveTime = time.Now() // update
    
    // 有更高的任期，认识新的Leader
    if rf.currentTerm < args.Term {
        rf.currentTerm = args.Term
        rf.voteFor = -1 // 当前任期没投票
        rf.leaderID = args.LeaderID
        rf.role = Follower
        rf.persist() // 持久化
    }
    
    // Snapshot太久了，不用安装
    if args.LastIncludeIndex <= rf.lastIncludeIndex {
        return
    }
    
    defer rf.persist() // 再次持久化一次
    
    
    // 这里安装日志有2种情况，一种是本地日志是在是太短了，那么Log直接全部丢掉即可
    if rf.lastIndex() <= args.LastIncludeIndex {
        rf.log = make([]LogEntry, 0) // 全部扔掉
    }else { // 本地日志条目长度长于Snapshot
        if rf.log[rf.index2LogPos(args.LastIncludeIndex)].Term != args.LastIncludeTerm {
            rf.log = make([]LogEntry, 0) // 全部扔掉
        }else { // 保留部分
            leftLog := make([]LogEntry, rf.lastIndex() - args.LastIncludeIndex)
            copy(leftLog, rf.log[rf.index2LogPos(args.LastIncludeIndex)+1:])
            rf.log = leftLog
        }
    }
    
    rf.lastIncludeIndex = args.LastIncludeIndex
    rf.lastIncludeTerm = args.LastIncludeTerm
    
    // TODO
    rf.persister.SaveStateAndSnapshot(rf.raftStateForPersist(), args.Data)
    rf.installSnapshotToApplication()
}
```

Folloer接收普通的日志条目：

```go
func (rf *Raft) AppendEntriesRPC(args AppendEntriesArgs, reply AppendEntriesReply) {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    
    // 对方任期太低
    if rf.currentTerm > args.Term {
        reply.Term = rf.currentTerm
        reply.Success = false
        return
    }
    
    rf.lastActiveTime = time.Now()
    
    // 更高的任期
    if rf.currentTerm < args.Term {
        rf.currentTerm = args.Term
        rf.voteFor = -1
        rf.leaderID = args.LeaderID
        rf.role = Follower
        rf.persist()
    }
    
    // 开始同步日志
    // Follower上找不到PrevLogIndex
    if args.PrevLogIndex < rf.lastIncludeIndex {
        reply.Success = false
        reply.XIndex = -1 // -1 for need snapshot TODO
        return
    }else if args.PrevLogIndex == rf.lastIncludeIndex {
    // 正好是Snapshot截断的部分，如果出现不同，只能重新安装Snapshot
        if args.PrevLogTerm != rf.lastIncludeTerm {
            reply.Success = false
            reply.XIndex = -1
            return
        }
    }else {
        // 日志条目长度不够
        if rf.lastIndex() < args.PrevLogIndex {
            reply.Success = false
            reply.XTerm = -1
            reply.XLen = args.PrevLogIndex - rf.lastIndex()
            return
        }
        // 日志条目相同，但是Term不同
        if rf.log[rf.index2LogPos(args.PrevLogIndex)].Term != args.PrevLogTerm {
            reply.XLen = 0
            reply.XTerm = rf.lastTerm()
            for XIndex := args.PrevLogIndex; XIndex > 0; XIndex -- {
                if reply.XTerm != rf.log[rf.index2LogPos(XIndex)].Term {
                    break
                }
                reply.XLen += 1
            }
            reply.XIndex = XIndex
            reply.Success = false
            return
        }
    }
    // 接下来就都正常了
    
    for i, logEntry := range args.Entries {
        index := args.PrevLogIndex + 1 + i
        logPos := rf.index2LogPos(index)
        if index > lastIndex() {
            rf.log = append(rf.log, logEntry)
        }else {
            if rf.log[logPos].Term != logEntry.Term {
                rf.log = rf.log[:logPos]
                rf.log = append(rf.log, logEntry)
            }
        }
    }
    rf.persist()
    
    // update
    if args.LeaderCommit > rf.commitIndex {
        rf.commitIndex = args.LeaderCommit
        if rf.lastIndex() < rf.commitIndex {
            rf.commitIndex = rf.lastIndex()
        }
    }
    reply.Success = true
}
```


### Leader上发送AppendEntries和Snapshot

Leader发送日志或者Snapshot总是通过nextIndex来判断的：

```go
func (rf *Raft)AppendEntriesLoop() {
    for !rf.killed() {
        time.Sleep(10 * time.Millisecond)
        AppendEntries() 
    }
}

func (rf *Raft)AppendEntries() {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    
    if rf.role != Leader {
        return
    }
    
    now := time.Now()
    if now.Sub(rf.lastBroadcastTime) < rf.broadcastInterval {
        return
    }
    
    for index := 0; index < len(rf.peers); index ++ {
        if index == rf.me {
            continue
        }
        // 对方日志太短，直接同步Snapshot
        if rf.nextIndex[index] <= rf.lastIncludeIndex {
            rf.doInstallSnapshot(index)
        }else {
            rf.doAppendEntries(index)
        }
    }
    rf.lastBroadcastTime = time.Now()
    
}
```

普通的日志条目同步，和之前2B中的差距不大

```go
func (rf *Raft) doAppendEntries(index int) {
    var args AppendEntresArgs
    args.Term = rf.currentTerm
    args.LeaderID = rf.me
    args.LeaderCommit = rf.commitIndex
    args.Entres = make([]LogEntry, 0)
    args.PrevLogIndex = rf.nextIndex[index] - 1
    
    if args.PrevLogIndex == rf.lastIncludeIndex {
        args.PrevLogTerm = rf.lastIncludeTerm
    }else {
        args.PrevLogTerm = rf.log[rf.index2LogPos(args.PrevLogIndex)].Term
    }
    args.Entries = append(args.Entries, rf.log[rf.index2LogPos(args.PrevLogIndex+1):]...)
    
    go func() {
        reply := AppendEntriesReply{}
        ok := sendAppendEntries(index, &args, &reply)
        if !ok {
            return
        }
        rf.mu.Lock()
        defer rf.mu.Unlock()
        
        if rf.currentTerm != args.Term {
            return
        }
        
        if rf.currentTerm < reply.Term {
            rf.currentTerm = reply.Term
            rf.role = Follower
            rf.leaderID = -1
            rf.voteFor = -1
            rf.persist()
            return
        }
        
        if reply.Success {
            newNextIndex := args.PrevLogIndex + len(args.Entries) + 1
            if newNextIndex > rf.nextIndex[index]{
                rf.nextIndex[index] = newNextIndex
                rf.matchIndex[index] = rf.nextIndex[index] - 1
            }
            // TODO: updateCommitIndex
        }else { // quick backup
            if reply.XIndex = -1 { // Follower是需要Snapshot的
                nextIndex[index] = 1
                return
            }
            // XTerm = -1则表示Follower需要XLen长度的日志
            if reply.XTerm == -1 {
                rf.nextIndex[index] = args.PrevLogIndex - reply.XLen + 1
                return
            }
        
            // find conflict index
            for conflictIndex := reply.XIndex; conflictIndex < reply.XIndex + reply.XLen; conflictIndex ++ {
                if rf.log[index2LogPos(conflictIndex)].Term != reply.XTerm {
                    rf.nextIndex[index] = conflictIndex // 冲突位置
                    rf.matchIndex[index] = rf.nextIndex[index] - 1
                    break
                }
            }
        
        }
    }()
}
```

安装Snapshot操作：

```go
func (rf *Raft)doInstallSnapshot(index int){
    args := InstallSnapshotArgs{}
    args.Term = rf.currentTerm
    args.LeaderID = rf.me
    args.LastIncludeIndex = rf.lastIncludeIndex
    args.LastIncludeTerm = rf.lastIncludeTerm
    args.Offset = 0
    args.Data = rf.persister.ReadSnapshot() // TODO
    args.Done = true // ?
    
    reply := InstallSnapshotReply{}
    go func() {
        ok := rf.sendInstallSnapshot(index, &args, &reply)
        if !ok {
            return
        }
        
        rf.mu.Lock()
        defer rf.mu.Unlock()
        
        if rf.currentTerm != args.Term {
            return
        }
        
        if rf.currentTerm < reply.Term {
            rf.currentTerm = reply.Term
            rf.voteFor = -1
            rf.leaderID = -1
            rf.role = Follower
            rf.persist()
            return
        }
        rf.nextIndex[index] = args.LastIncludeIndex + 1
        rf.matchIndex[index] = args.LastIncludeIndex
        rf.updateCommitIndex() // update
    }()
    
}
```

一些经常用到的操作封装的函数，日志条目Index从1开始。

```go
// 将Index直接转化为日志的索引
func (rf *Raft) index2LogPos(index int) int {
    return index - rf.lastIncludeIndex - 1
}

// 获得当前节点的最后一条日志条目索引
func (rf *Raft) lastIndex () int {
    return rf.lastIncludeIndex + len(rf.log)
}

// 当前节点的最后一条日志条目的任期
func (rf *Raft) lastTerm () int {
    if len(rf.log) == 0 {
        return rf.lastIncludeTerm
    }
    return rf.log[len(rf.log)-1].Term
}
```
