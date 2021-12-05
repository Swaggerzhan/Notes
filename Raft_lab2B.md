# 6.824 Lab2B Raft学习笔记

## 0x00 理论

### Raft节点

2B相比2A多了许多必要的信息，每个节点都需要保存的数据包括3种，分别是需要进行持久化(2C)的：

```
currentTerm           服务器最后一次知道的任期
votedFor              获得本服务器选票的候选人
log[]                 日志
```

所有服务器中经常改变的：

```
commitIndex           已经被提交的日志索引
lastApplied           被应用到状态机中的日志索引
```

Leader节点中经常改变的，在选举后重新初始化：

```
nextIndex[]           对于每一个节点，需要发送给他的下一个日志条目索引
matchIndex[]          对于每一个节点，已经复制给他的日志的最高索引值
```


### AppendEntries
2B相比2A增加了日志同步的操作，并且根据论文figure5中所说，AppendEntries操作同时也用作HeartBeat操作，所以2B中的结构体将由2A中的HeartBeat修改而来。

Follower要做的：

1 : 比对Leader发送过来的term，发现小于当前任期，则直接返回false。

2：检查prevLogIndex下的prevLogTerm是否相同，不同返回false。

3：附加参数中的entries到本地日志中

4：如果发现 leaderCommit > commitIndex；则令commitIndex等于LeaderCommit或最新的日志索引中的比较小的那个。

例子：

假定有日志为：

```
// 数字代表日志添加的term
index：   10 11 12 13
Server1:  3
Server2:  3  3  4
Server3:  3  3  5
```

如何生成这种日志：起初，client发送指令，所有节点均在index10下完成同步，Term为3，之后client又发送了一个指令，这条指令由Server2和Server3完成响应并返回，写入到日志index 11中，在Raft中，只要多数票通过，则Leader就会响应客户端，所以index 11的指令会被Commit掉，到此没有任何分歧。

index 12之前，Leader为Server3，而后，Leader节点宕机了，Server2当选为Leader，他执行了来自客户端的一次操作，并且写入日志，term为4，但此时还未来得及发送LogEntries，立马宕机了，随后，节点们又选出Server3作为Leader，并且执行了来自客户端的操作，写入日志，此时Term为5。

#### 强制同步日志

我们假定有以下日志，其中Server3为Leader：

```
index：  10 11 12 13
Server1: 3
Server2: 3  3  4
Server3: 3  3  5  6
```

Server3通过LogEntries(心跳)来同步其他节点的日志，通过之前的理论，可以知道Leader将发出

```
prevLogIndex = 12
prevLogTerm = 5
entries[] 中是 index=13，term = 6的日志

// Leader节点中的数据：
nextIndex[s1] = 13
nextIndex[s2] = 13
```

Server1和Server2中的日志和Leader发送的LogEntries显然不一样，所以会返回False， __Leader会将prevLogIndex向前移动一位__ ，修改nextIndex，继续发送LogEnties，这回将发送：

```
prevLogIndex = 11
prevLogTerm = 3
entries[] 中有(index=12, term=5),(index=13, term=6)

// Leader节点中的数据
nextIndex[s1] = 12
nextIndex[s2] = 12
```

此时Server2中发现prevLogIndex和pervLogTerm匹配，接受了来自Leader的日志，同步日志和Leader一样(此处先不讨论commitIndex)。而Server1在pervLogIndex= 11处没有数据，同样返回false，Leader收到后继续向前移动prevLogIndex为10，发送：

```
prevLogIndex = 10
prevLogTerm = 3
entries[] 中有(index=11, term=3),(index=12, term=5),(index=13, term=6)

// Leader节点中的数据
nextIndex[s1] = 11
nextIndex[s2] = 14
```

这回Server1比对通过，接受日志，完成所有的日志同步操作。

__这种同步一次将nextIndex向后移动一位的速度在follower落后过多条日志时，将使得同步时间过长，所以可以考虑自己引入其他方法来提高日志同步的速度__ 。

注：最初在Server2中，log index = 12的term4操作会被直接抛弃，因为当时的Raft没有得到多数票通过此条操作，所以Raft压根就没回应客户端，所以不必担心。

## 0x01 设计思路

### Raft 

节点结构体：

```go
type Raft struct {
  mu                sync.Mutex
  peers             []*labrpc.ClientEnd
  persister         *Persister // 2A中没有用到
  me                int
  dead              int32

  currentTerm       int // 当前任期
  voteFor           int // 当前任期获取本节点选票的节点
  log               []LogEntry // 日志，2A中没有用到
  
  nextIndex         []int // 每个Follower同步起点
  matchIndex        []int // 每个Follower已经同步的日志索引
  
  // 所有节点中，属于容失状态
  commitIndex       int // 已知的最大提交索引
  lastApplied       int // 提交到状态机到索引 

  role              string
  leaderID          int

  lastBroadcastTime         time.Time // Leader发送心跳的时间
  broadcastInterval         time.Duration // 100ms
  lastActiveTime            time.Time 
  lastActiveTimeInterval    time.Duration // 200ms - 400ms
  
  applyChan                 chan ApplyMsg // 提交给应用层状态机的管道

}
```

### AppendEntries

```go
type AppendEntriesArgs struct {
  term          int // 当前任期
  leaderID      int 
  prevLogIndex  int // 当前提交日志index的前一个index
  prevLogTerm   int // prevLogIndex下的任期
  
  entries       []LogEntry // 需要同步的日志
  leaderCommit  int
}

type AppendEntriesReply struct {
  term      int
  success   bool
}
```

### RequestVoteArgs

投票代码也需要修改，2B中，不仅仅只判断任期更高就能投票，还需要通过log长度等等条件来判断

```go
type RequestVoteArgs struct {
  Term          int
  Candidate     int
  LastLogIndex  int
  LastLogTerm   int
}

type RequestVoteReply struct {
  Term      int
  Success   int
}
```

## 0x02 代码实现

Leader节点中，Leader不再向Follower发送心跳，转而发送AppendEntries来进行强制同步日志的操作。

每个节点通过`AppendEntriesRPC`来暴露调用接口给Leader，Leader则通过`AppendEntiesLoop`来定时发送`AppendEntries`，这里则设定为100ms。

具体的`AppendEntriesLoop`和`AppendEntries`:

```go
func (rf *Raft) AppendEntriesLoop() {
	for !rf.killed() {
		time.Sleep(10 * time.Millisecond)
		rf.AppendEntries()
	}
}

func (rf *Raft) AppendEntries() {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// Leader 才需要广播
	if rf.role != Leader {
		return
	}

	now := time.Now()
	// 还未超时
	if now.Sub(rf.lastBroadcastTime) < rf.broadcastInterval {
		return
	}

	// 向所有Follower广播
	for index := 0; index < len(rf.peers); index ++ {
		if index == rf.me {
			continue
		}

		args := AppendEntriesArgs{
			Term : rf.currentTerm,
			LeaderID : rf.me,
			LeaderCommit: rf.commitIndex,
			Entries: make([]LogEntry, 0),
			PrevLogIndex: rf.nextIndex[index] - 1,
		}
		if args.PrevLogIndex > 0 {
			args.PrevLogTerm = rf.log[args.PrevLogIndex - 1].Term
		}
		// 填充log
		args.Entries = append(args.Entries, rf.log[rf.nextIndex[index] - 1:]...)
		// 启动协程发送
		go rf.coroutineAppendEntries(index, &args)
	}
	// 更新超时时间
	rf.lastBroadcastTime = time.Now()
}
```

其中，`AppendEntries`启动了协程并发发送RPC请求：

```go
func (rf *Raft)coroutineAppendEntries(index int, args* AppendEntriesArgs) {
	reply := AppendEntriesReply{}
	if ok := rf.sendAppendEntriesRPC(index, args, &reply); !ok {
		// Error!
		return
	}
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 必须？
	if rf.currentTerm != args.Term {
		return
	}

	// 有更高任期节点存在，成为当前任期Follower，等待Leader
	if rf.currentTerm < args.Term {
		rf.currentTerm = args.Term
		rf.role = Follower
		rf.voteFor = -1
		rf.leaderID = -1
		return
	}

	// 日志同步完成
	if reply.Success {
		// TODO
		rf.nextIndex[index] += len(args.Entries)
		rf.matchIndex[index] = rf.nextIndex[index] - 1

		sortedMatchIndex := make([]int, 0)
		sortedMatchIndex = append(sortedMatchIndex, len(rf.log))
		for i := 0; i < len(rf.peers); i ++ {
			if i == rf.me {
				continue
			}
			sortedMatchIndex = append(sortedMatchIndex, rf.matchIndex[i])
		}
		sort.Ints(sortedMatchIndex)
		newCommitIndex := sortedMatchIndex[len(rf.peers) / 2]
		if newCommitIndex > rf.commitIndex && rf.log[newCommitIndex - 1].Term == rf.currentTerm {
			rf.commitIndex = newCommitIndex
		}
		// TODO
	}else {
		rf.nextIndex[index] -= 1
		if rf.nextIndex[index] < 1 {
			rf.nextIndex[index] = 1
		}
	}
}
```

每个Follower暴露的RPC接口`AppendEntriesRPC`，论文figure5.1中提出了5点要求，分别为：
1. args.Term < rf.currentTerm 同步日志失败。
2. args.PrevLogIndex处点日志条目任期和args.PrevLogTerm不同，同步日志失败。
3. 在2通过的情况下，可以强制覆盖“Follower中的已存在日志”。
4. 附加Follower中不存在的日志
5. 令commitIndex = min(args.LeaderCommit，rf.commitIndex，新日志条目索引值)。

```go
func (rf *Raft) sendAppendEntriesRPC(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntriesRPC", args, reply)
	return ok
}

func (rf *Raft) AppendEntriesRPC(args* AppendEntriesArgs, reply* AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 对方任期更小，直接返回错误
	if args.Term < rf.currentTerm {
		reply.Success = false
		reply.Term = rf.currentTerm
		return
	}

	// 更大任期的Leader发来AppendLog
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.role = Follower
		rf.voteFor = -1
		rf.leaderID = args.LeaderID
	}

	// 更新
	rf.lastActiveTime = time.Now()

	// prevLogIndex 空
	if len(rf.log) < args.PrevLogIndex {
		reply.Success = false
		reply.Term = rf.currentTerm
		return
	}

	if args.PrevLogIndex > 0 && rf.log[args.PrevLogIndex - 1].Term != args.PrevLogTerm {
		reply.Success = false
		reply.Term = rf.currentTerm
		return
	}

	// 检查通过，没有问题
	for i, log := range args.Entries {
		index := args.PrevLogIndex + 1 + i
		if index > len(rf.log) {
			rf.log = append(rf.log, log)
		}else {
			if rf.log[index - 1].Term != log.Term {
				rf.log = rf.log[:index - 1] // 删除冲突以及之后的日志，强制同步为和Leader发送的日志
				rf.log = append(rf.log, log)
			}
		}
	}

	// 更新commitIndex
	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = args.LeaderCommit
		if len(rf.log) < rf.commitIndex {
			rf.commitIndex = len(rf.log)
		}
	}
	reply.Term = rf.currentTerm
	reply.Success = true

}

```

#### 选举

选举和2A中也有所不同，加入了Log长度以及Log执行任期的判断，同样每个Follower暴露`RequestVoteRPC`接口，每个节点通过`ElectionLoop`来定期发送选举请求，当收到Leader发送来的`AppendEntries`时，重置定时。

```go
func (rf *Raft) ElectionLoop () {
	for !rf.killed() {
		time.Sleep(1 * time.Millisecond)
		rf.Election()
	}
}

// 超时被调用，开始新一轮选举
func (rf* Raft) Election() {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.role == Leader {
		return
	}
	// 未超时
	now := time.Now()
	if now.Sub(rf.lastActiveTime) < rf.lastActiveTimeInterval {
		return
	}
	// 超时，状态转为Candidate，开始新一轮选举
	rf.role = Candidate
	rf.currentTerm += 1
	rf.voteFor = rf.me

	// 在投票给自己的同时，向其他节点发起RPC获取选票
	args := RequestVoteArgs{
		Term : rf.currentTerm,
		CandidateID : rf.me,
	}

	type Result struct {
		peerID int
		respond* RequestVoteReply
	}
	voteCount := 1 // 自己一票
	totalCount := 1 // 所有票数，包括没有给自己投票的节点
	resultChan := make(chan *Result, len(rf.peers)) // 管道
	// 启协程获取选票，解锁
	// 当结束后，由于解锁的请求，解锁期间，可能有任期变动，所以需要重新判断！
	rf.mu.Unlock()
	for index := 0; index < len(rf.peers); index ++ {
		if index == rf.me {
			continue
		}
		go func (id int){
			reply := RequestVoteReply{}
			if ok := rf.sendRequestVote(id, &args, &reply); ok {
				resultChan <- &Result{peerID : id, respond: &reply }
				return
			}
			resultChan <- &Result {peerID : id, respond : nil}
		}(index)
	}
	maxTerm := 0 // 获取最高任期，以确定当前的任期是正确的
	// 当发现有更高任期的节点存在，本次投票无意义
	for {
		select {
		case result := <- resultChan:
			totalCount += 1
			if result.respond != nil {
				if result.respond.Success {
					voteCount += 1
				}
				if result.respond.Term > maxTerm{
					maxTerm = result.respond.Term
				}
			}
			// 投票结束
			if totalCount == len(rf.peers) || voteCount > len(rf.peers) / 2 {
				goto END
			}
		}
	}
END:
	// 重新上锁，并且检测状态
	rf.mu.Lock()
	if rf.role != Candidate { // 不是Candidate状态，则抛弃一切投票结果
		return
	}
	// 有更高任期存在，转而成为Follower，等待Leader的心跳
	if maxTerm > rf.currentTerm {
		rf.role = Follower
		rf.leaderID = -1
		rf.voteFor = -1
		rf.currentTerm = maxTerm
		return
	}
	// 计算选票
	if voteCount > len(rf.peers) / 2 {
		rf.role = Leader
		rf.leaderID = rf.me
		rf.lastBroadcastTime = time.Now()
		rf.broadcastInterval = time.Duration(100) * time.Millisecond
		// 设定心跳发送间隔100ms
		// 初始化nextIndex和matchIndex
		rf.nextIndex = make([]int, len(rf.peers))
		rf.matchIndex = make([]int, len(rf.peers))
		for i := 0; i<len(rf.peers); i++ {
			rf.nextIndex[i]	= rf.lastIndex() + 1
			rf.matchIndex[i] = 0
		}
	}
	// 重新设定时器
	rf.lastActiveTime = time.Now()
	rf.lastActiveTimeInterval = time.Duration(200 + rand.Int31n(200)) * time.Millisecond
}
```

`RequestVoteRPC`中关于Log长度以及Log执行任期的判断，论文figure5.2中说明了：
1. args.Term < rf.currentTerm, 拒绝投票。(肯定的)
2. 当前节点还未投票，且候选人日志至少和自己一样的新，则投票给他。

这里出现了一个比较难的点，在论文figure5.4.1的选举限制中这样说道： __Raft使用投票的方式来阻止一个候选人赢得选票，除非这个候选人包含了全部的已经提交的日志条目__ 。[具体解释](#选举限制)

```go
func (rf *Raft) RequestVoteRPC(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.Success = false
	// 被选举者任期比当前任期小，拒绝
	//fmt.Printf("current node Term at %d\n", rf.currentTerm)
	//fmt.Printf("[%d] request for vote, server[%d].Term: %d\n", args.CandidateID, args.CandidateID, args.Term)
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		return
	}
	// 成为当前任期中的Follower
	// 这里的leaderID还未确定！
	// 2B 中不能直接投票，需要判断日志长度等等一系列操作
	if args.Term > rf.currentTerm {
		//fmt.Printf("accept: [%d] vote -> [%d] \n", rf.me, args.CandidateID)
		rf.currentTerm = args.Term
		rf.role = Follower
		rf.leaderID = -1
		rf.voteFor = -1 // 当前任期没有投票过
	}
	if rf.voteFor == -1 { // 还没投票
		lastLogTerm := rf.lastTerm()
		// 被选举者日志最后一条中的任期要么是最高的
		// 又或者是相同任期，但其日志是最长的
		if args.LastLogTerm > lastLogTerm || (args.LastLogIndex == lastLogTerm && args.LastLogIndex >= rf.lastIndex()) {
			rf.voteFor = args.CandidateID
			rf.lastActiveTime = time.Now()
			reply.Success = true
		}
	}
}

```

## 0x03 测试用例



## 0x04 一些具体解释

### 选举限制

论文figure5.4.1中明确指出了如果一个候选人想要赢得选举，就必须拥有所有已经提交的日志条目。