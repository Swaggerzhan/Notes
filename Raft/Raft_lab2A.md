<font face="Monaco">

# 6.824 Lab2A Raft学习笔记

## 0x00 理论

### 复制状态机

Raft的本质，就是一个状态复制机：

![](./Raft_lab2A/2.png)

每一个节点服务器中，都保存有执行指令日志，指令日志中执行到的位置等等，并且通过一致性模块将其同步到各个节点上，当客户端发起请求后，只有多数服务器中都保存有对应的指令日志并响应Leader后，客户端才会收到回复。取决于实现，有些实现允许client向replica请求数据，有些则只允许向Leader请求数据。

### Election

Raft中有3种角色，分别是`Leader`，`Follower`，`Candidate`。

在Raft中，Leader只能存在一个，Leader每过一段时间，就要向Follower发送心跳包以确认其Follower状态，Follower也能通过心跳包了解到Leader在线，每个Follower本身维护一个定时器，当收到Leader的心跳后重制定时器，如果一直没有收到来自Leader的心跳，则定时器超时，Follower开始新一轮的Leader选举。

流程大致为：

![](./Raft_lab2A/1.png)

如果总节点为2n个，碰巧的是发生了2个节点同时发起选举的请求，且他们平摊了选票，则本轮作废，重新进入下一轮选举。

### LogEntry

LogEntry就是集群一致的手段，Leader通过发送LogEntry来强制同步其他机器上的日志，同时也做心跳包作用，一般而言，Leader上的AppendEntry的发送时间如果设定为n，那么Follower应当容忍2n以上的时间才会超时进入Candidate状态。

并且论文5.4.1中有说明了关于选举限制的东西，如果一个Node想要成为Leader，那么这个Node就必须拥有所有已经commit的日志，我们不允许commit后的日志被丢弃！具体详见[lab2B](./Raft_lab2B.md)

### Follower 需要做的：

在Lab2A中，Follower要做的东西非常的少：

* 维护一个定时器，超时后，Follower状态转为Candidate，开始选举。

超时后，应该立刻进行新的定时，而选举RPC应该由另外线程来处理，不应该等待RPC结束后才进行定时，RPC有时候是非常耗时的，这样将有损定时精度，__不过，由于我这样的设计方式，可能会出现上一个RequestVoteRPC发起后，由于网络问题迟迟没有结束，而第二个RequestVoteRPC开始后又结束了，并且早于第一个RequestVoteRPC结束，所以我们需要保证请求的幂等性，计算选票的时候，如果当前Node状态有任何改变，那么本次选票都将直接作废__ 。

* 收到RequestVoteRPC后，判断是否同意投票即可。

在Lab2A中，还没有必要进行日志选举限制，所以我们只需要记住一个Node在一个Term只能投一票即可。

* 收到AppendEntry，也就是心跳后，重置定时的超时时间。


### Leader 需要做的：

* 发送日志，也就是心跳，这是一个固定时间，比如100ms。

## 0x01 代码实现

### Raft节点

```go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	applyCh        chan ApplyMsg // 2B中使用
	notifyCh       chan int // 2B中使用
	appendEntryCh  chan int // AppendEntry控制管道
	electionCh     chan int // Election控制管道

	currentTerm int
	votedFor int
	log []LogEntry

	nextIndex 			[]int
	matchIndex 			[]int

	commitIndex			     int
	lastApplied			     int
	lastIncludeIndex       int
	lastIncludeTerm        int

	role						string
	leaderID				int

	appendEntriesInterval time.Duration // 100ms

	// for debug
	loop int

}
```

### 选举

选举开放的RPC参数以及RPC响应：

```go
type RequestVoteArgs struct {
	Term 					int
	LastLogIndex 	int           // 选举限制
	LastLogTerm 	int           // 选举限制
	CandidateID		int
}


type RequestVoteReply struct {
	Term           int
	Success 	   bool
}
```

选举协程：

```go
// 获取从value -> 3value之间的随机时间
func getRandElectionTime(value int32) time.Duration {
	return time.Duration(value+rand.Int31n(value*3)) * time.Millisecond
}

func (rf *Raft) electionLoop() {
	timer := time.NewTimer(getRandElectionTime(200))
	for {
		select {
		// 其他信号
		case signal := <- rf.electionCh: {
			if signal == ElectionReset {
				dura := getRandElectionTime(200)
				timer.Reset(dura)
			}
		}
		// 超时
		case <- timer.C: {
			if _, isLeader := rf.GetState(); isLeader {
				dura := getRandElectionTime(200)
				timer.Reset(dura)
			}else {
				dura := getRandElectionTime(200)
				timer.Reset(dura)
				go rf.requestForVote()
			}

		}
		}
	}
}

func (rf *Raft) requestForVote() {
	rf.mu.Lock()
	// Follower超时就开始选举操作
	rf.role = Candidate
	rf.currentTerm += 1
	rf.votedFor = rf.me
	rf.persist() // TODO: 状态改变，持久化

	args := RequestVoteArgs{
		Term: rf.currentTerm,
		LastLogIndex: rf.lastIndex(),
		LastLogTerm: rf.lastTerm(),
		CandidateID: rf.me,
	}
	rf.mu.Unlock()

	resultChan := make(chan *Result, len(rf.peers))

	for peerID := 0; peerID < len(rf.peers); peerID ++ {
		if peerID == rf.me { continue }

		go func (id int) {
			reply := RequestVoteReply{}
			if ok := rf.sendRequestVote(id, &args, &reply); ok {
				resultChan <- &Result{peerID: id, respond: &reply}
				return
			}
			resultChan <- &Result{peerID: id, respond: nil}
		}(peerID)
	}
	rf.countRequestVote(resultChan, args.Term)
}

// startTerm表示当前计算选票所在的Term
func (rf *Raft) countRequestVote(resultChan chan *Result, startTerm int) {

	voteCount := 1
	totalCount := 1
	maxTerm := 0
	for {
		select {
		case result := <- resultChan: {
			totalCount += 1
			if result.respond != nil {
				if result.respond.Success {
					voteCount += 1
				}
				if result.respond.Term > maxTerm {
					maxTerm = result.respond.Term
				}
			}
			if totalCount == len(rf.peers) || voteCount > len(rf.peers) / 2 {
				goto END
			}
		}
		}
	}
	END:

	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 状态改变，或者任期已经发生了改变，那么就丢弃所有结果
	if rf.role != Candidate || startTerm != rf.currentTerm {
		return
	}

	// 有更高任期的存在，丢弃本次选举
	if maxTerm > rf.currentTerm {
		rf.role = Follower
		rf.leaderID = -1
		rf.votedFor = -1
		rf.currentTerm = maxTerm
		rf.persist()
		return
	}
	// TODO: 成为Leader是否需要持久化？
	if voteCount > len(rf.peers) / 2 {
		rf.role = Leader
		rf.leaderID = rf.me
		rf.nextIndex = make([]int, len(rf.peers))
		rf.matchIndex = make([]int, len(rf.peers))
		for i := 0; i < len(rf.peers); i++ {
			rf.nextIndex[i] = rf.lastIndex() + 1
			rf.matchIndex[i] = 0
		}
		// 启动发送日志
		rf.appendEntryCh <- AppendEntryStartSignal
	}
}

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVoteRPCServer", args, reply)
	return ok
}
```

electionLoop是整个选举协程，并且开放一个管道electionCh用于监听是否需要进行Reset，再者，后续我们也可以添加新的其他信号例如停止整个Node等信号。

关于开放的投票RPC接口就比较容易了：

```go
func (rf *Raft) RequestVoteRPCServer(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.Success = false
	if rf.currentTerm > args.Term {
		reply.Term = rf.currentTerm
		reply.Success = false
		return
	}

	defer rf.persist()
	// TODO: persist?
	if rf.currentTerm < args.Term {
		rf.currentTerm = args.Term
		rf.role = Follower
		rf.leaderID = -1
		rf.votedFor = -1
	}

	if rf.votedFor == -1 {
		lastLogTerm := rf.lastTerm()
		// 选举限制，2A中没明确规定，所以删除也是不影响的
		if args.LastLogTerm > lastLogTerm || (args.LastLogTerm == lastLogTerm && args.LastLogIndex >= rf.lastIndex()) {
			rf.votedFor = args.CandidateID
			reply.Success = true
			// 成功投票给别人，重置一下定时器
			rf.electionCh <- ElectionReset
			return
		}
	}
}
```

### 心跳

心跳，也就是AppendEntry，其RPC参数和节点做以下设计：

```go
type AppendEntryArgs struct {
	Term             int
	LeaderID         int
	PrevLogIndex     int // 2B
	PrevLogTerm      int // 2B

	Entries          []LogEntry // 2B
	LeaderCommit     int        // 2B
}

type AppendEntryReply struct {
	Term              int
	Success           bool
	XTerm             int // 2B 冲突的任期
	XIndex            int // 2B 冲突任期Log开始位置
	XLen              int // 2B 冲突任期的Log长度
}
```

AppendEntry协程部分为：

```go
func (rf *Raft) appendEntriesLoop() {
	timer := time.NewTimer(1 * time.Millisecond)
	for {
		select {
		// TODO	: case shut down
		case signal := <- rf.appendEntryCh: {
			// 如果是紧急日志或者新Leader上任，那么开始立即开始日志同步
			if signal == AppendEntryStartSignal || signal == AppendEntryEmergencySignal{
				if _, isLeader := rf.GetState(); !isLeader {
					timer.Stop()
				}else {
					timer.Reset(1 * time.Millisecond)
				}
			}
		}
		case <- timer.C: {
			if _, isLeader := rf.GetState(); !isLeader {
				timer.Stop()
			}else {
				timer.Reset(rf.appendEntriesInterval) // 100ms
				rf.sendAppendEntry()
			}
		}
		}
	}
}

func (rf* Raft) sendAppendEntry() {
	rf.mu.Lock()
	rf.mu.Unlock()
	for peerID := 0; peerID < len(rf.peers); peerID ++ {
		if peerID == rf.me {
			continue
		}
		// 发送日志
		rf.appendEntriesWithCoroutine(peerID)
	}
}

//rf.appendEntriesWithCoroutine见Lab2B
```

同样的，由于精度问题，需要同时另起协程发送RPC，并且需要保证幂等性即可。

## 0x02 测试

6.824中给定的测试用例测试100次，均通过，但不代表没有bug，也有可能测试用例未能体现。

```shell
➜  raft git:(master) ✗ go test -run 2A
Test (2A): initial election ...
  ... Passed --   3.1  3 4712  518338    0
Test (2A): election after network failure ...
  ... Passed --   8.6  3 14126  915384    0
Test (2A): multiple elections ...
  ... Passed --   5.7  7 1255  104664    0
PASS
ok  	6.824/raft	17.824s
```
</font>