



## 选举计时器

```go
heartbeatTimer := randomTimeout(r.conf.HeartbeatTimeout)

// randomTimeout returns a value that is between the minVal and 2x minVal.
func randomTimeout(minVal time.Duration) <-chan time.Time {
	if minVal == 0 {
		return nil
	}
	extra := (time.Duration(rand.Int63()) % minVal)
	return time.After(minVal + extra)
}
```



## Leader选举

**初始状态**

Raft节点开始的State为follower

```go
// Initialize as a follower
r.setState(Follower)
```

**触发选举**

通过follower的心跳超时触发选举, 这样无论是初始化需要选举出一个Leader, 还是
节点心跳超时发起一轮新的Leader选举可以视为一种情况一并处理.

> Raft论文关于这部分的描述: Raft uses a heartbeat mechanism to trigger leader election. 
When servers start up, they begin as followers

`NewRaft` 创建完成后, 会启用一个goroutine处理Raft状态机

```go
r.goFunc(r.run)
```


`r.run`定义了Raft状态机转换及处理过程

```go
// run is a long running goroutine that runs the Raft FSM.
func (r *Raft) run() {
	for {
		// Check if we are doing a shutdown
		select {
		case <-r.shutdownCh:
			// Clear the leader to prevent forwarding
			r.setLeader("")
			return
		default:
		}

		// Enter into a sub-FSM
		switch r.getState() {
		case Follower:
			r.runFollower()
		case Candidate:
			r.runCandidate()
		case Leader:
			r.runLeader()
		}
	}
}
```

`r.getState` 方法获取当前Raft节点的FSM状态，不同的状态对应不同的处理函数，其状态转变在Paper中如下图所示

![](raft-fsm-switch.jpg)

## Follower FSM

```go
// runFollower runs the FSM for a follower.
func (r *Raft) runFollower() {
	didWarn := false
	r.logger.Printf("[INFO] raft: %v entering Follower state (Leader: %q)", r, r.Leader())
	metrics.IncrCounter([]string{"raft", "state", "follower"}, 1)
	heartbeatTimer := randomTimeout(r.conf.HeartbeatTimeout)
	for {
		select {
		case rpc := <-r.rpcCh:
			r.processRPC(rpc)

		case a := <-r.applyCh:
			// Reject any operations since we are not the leader
			a.respond(ErrNotLeader)

		case v := <-r.verifyCh:
			// Reject any operations since we are not the leader
			v.respond(ErrNotLeader)

		case p := <-r.peerCh:
			// Set the peers
			r.peers = ExcludePeer(p.peers, r.localAddr)
			p.respond(r.peerStore.SetPeers(p.peers))

		case <-heartbeatTimer:
			// Restart the heartbeat timer
			heartbeatTimer = randomTimeout(r.conf.HeartbeatTimeout)

			// Check if we have had a successful contact
			lastContact := r.LastContact()
			if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
				continue
			}

			// Heartbeat failed! Transition to the candidate state
			lastLeader := r.Leader()
			r.setLeader("")
			if len(r.peers) == 0 && !r.conf.EnableSingleNode {
				if !didWarn {
					r.logger.Printf("[WARN] raft: EnableSingleNode disabled, and no known peers. Aborting election.")
					didWarn = true
				}
			} else {
				r.logger.Printf(`[WARN] raft: Heartbeat timeout from %q reached, starting election`, lastLeader)

				metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1)
				r.setState(Candidate)
				return
			}

		case <-r.shutdownCh:
			return
		}
	}
}
```



`runFollower` 会执行Follower的职责，主要分为

- `<-r.rpcCh`, 统一处理其他节点的RPC请求
- `<-r.applyCh`,  TODO
- `<-r.verifyCh`, TODO
- `<-r.peerCh`, 处理Raft节点加入或退出Raft集群
- `<-heartbeatTimer`, 心跳超时计时器，当事件到来会发起Leader选举
- `<-r.shutdownCh`, TODO

### Follower Heartbeat Timeout

`<-heartbeatTimer` 到期后, Follower转变为Candidate发起选举
```go
// Restart the heartbeat timer
heartbeatTimer = randomTimeout(r.conf.HeartbeatTimeout)

// Check if we have had a successful contact
lastContact := r.LastContact()
if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
	continue
}

// Heartbeat failed! Transition to the candidate state
lastLeader := r.Leader()
r.setLeader("")
if len(r.peers) == 0 && !r.conf.EnableSingleNode {
	if !didWarn {
		r.logger.Printf("[WARN] raft: EnableSingleNode disabled, and no known peers. Aborting election.")
		didWarn = true
	}
} else {
	r.logger.Printf(`[WARN] raft: Heartbeat timeout from %q reached, starting election`, lastLeader)

	metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1)
	r.setState(Candidate)
	return
}
```

1. 重置选举计时器
2. 如果最近一次的通信时间否小于配置的`HeartbeatTimeout` 则不发起选举, 目的是防止有通信成功的情况下, follower在一个心跳超时周期内发起过多选举导致leader一直选不出来的情况
3. 清空leader状态
4. 如果当前集群是单节点, 同时配置了不允许单节点节点, 则不发起选举
5. 设置节点状态为`Candidate`


### Convert Candidate

follower heartbeat timeout 后, 状态转为leader, 在 FSM 循环中处于`Candidate`会执行 `runCandidate` 方法:

```go
// runCandidate runs the FSM for a candidate.
func (r *Raft) runCandidate() {
	r.logger.Printf("[INFO] raft: %v entering Candidate state", r)
	metrics.IncrCounter([]string{"raft", "state", "candidate"}, 1)

	// Start vote for us, and set a timeout
	voteCh := r.electSelf()
	electionTimer := randomTimeout(r.conf.ElectionTimeout)

	// Tally the votes, need a simple majority
	grantedVotes := 0
	votesNeeded := r.quorumSize()
	r.logger.Printf("[DEBUG] raft: Votes needed: %d", votesNeeded)

	for r.getState() == Candidate {
		select {
		case rpc := <-r.rpcCh:
			r.processRPC(rpc)

		case vote := <-voteCh:
			// Check if the term is greater than ours, bail
			if vote.Term > r.getCurrentTerm() {
				r.logger.Printf("[DEBUG] raft: Newer term discovered, fallback to follower")
				r.setState(Follower)
				r.setCurrentTerm(vote.Term)
				return
			}

			// Check if the vote is granted
			if vote.Granted {
				grantedVotes++
				r.logger.Printf("[DEBUG] raft: Vote granted from %s. Tally: %d", vote.voter, grantedVotes)
			}

			// Check if we've become the leader
			if grantedVotes >= votesNeeded {
				r.logger.Printf("[INFO] raft: Election won. Tally: %d", grantedVotes)
				r.setState(Leader)
				r.setLeader(r.localAddr)
				return
			}

		case a := <-r.applyCh:
			// Reject any operations since we are not the leader
			a.respond(ErrNotLeader)

		case v := <-r.verifyCh:
			// Reject any operations since we are not the leader
			v.respond(ErrNotLeader)

		case p := <-r.peerCh:
			// Set the peers
			r.peers = ExcludePeer(p.peers, r.localAddr)
			p.respond(r.peerStore.SetPeers(p.peers))
			// Become a follower again
			r.setState(Follower)
			return

		case <-electionTimer:
			// Election failed! Restart the election. We simply return,
			// which will kick us back into runCandidate
			r.logger.Printf("[WARN] raft: Election timeout reached, restarting election")
			return

		case <-r.shutdownCh:
			return
		}
	}
}
```

`Candidate` 会按照以下步骤运行
1. 执行选举, 通过 `rf.electSelf` 
2. 设置选举超时计时器
3. 收集选举结果


#### Candidate状态循环
`runCandidate`内, 如果节点是Candidate状态, 则会一直运行状态循环, 直到发送以下变更退出
1. 状态转变, 比如收到了比自身term转变为follower, 或者选举成功变为leader, 或者在其他的并行gouroutine循环中转变了状态, 无论如何, 只要身份发生改变, 就立即退出状态循环
2. 选举计时器超时`<-electionTimer`
3. 收到退出信号 `<-r.shutdownCh`


#### electSelf

调用`rf.electSelf` 会返回一个 `voteCh`, 类型为 `<-chan *voteResult` 的缓冲 channel, 其大小为Raft集群节点数量 + 1, 后续可以从这个channel中读取选举结果

```go
// electSelf is used to send a RequestVote RPC to all peers,
// and vote for ourself. This has the side affecting of incrementing
// the current term. The response channel returned is used to wait
// for all the responses (including a vote for ourself).
func (r *Raft) electSelf() <-chan *voteResult {
	// Create a response channel
	respCh := make(chan *voteResult, len(r.peers)+1)

	// Increment the term
	r.setCurrentTerm(r.getCurrentTerm() + 1)

	// Construct the request
	lastIdx, lastTerm := r.getLastEntry()
	req := &RequestVoteRequest{
		Term:         r.getCurrentTerm(),
		Candidate:    r.trans.EncodePeer(r.localAddr),
		LastLogIndex: lastIdx,
		LastLogTerm:  lastTerm,
	}

	// Construct a function to ask for a vote
	askPeer := func(peer string) {
		r.goFunc(func() {
			defer metrics.MeasureSince([]string{"raft", "candidate", "electSelf"}, time.Now())
			resp := &voteResult{voter: peer}
			err := r.trans.RequestVote(peer, req, &resp.RequestVoteResponse)
			if err != nil {
				r.logger.Printf("[ERR] raft: Failed to make RequestVote RPC to %v: %v", peer, err)
				resp.Term = req.Term
				resp.Granted = false
			}

			// If we are not a peer, we could have been removed but failed
			// to receive the log message. OR it could mean an improperly configured
			// cluster. Either way, we should warn
			if err == nil {
				peerSet := decodePeers(resp.Peers, r.trans)
				if !PeerContained(peerSet, r.localAddr) {
					r.logger.Printf("[WARN] raft: Remote peer %v does not have local node %v as a peer",
						peer, r.localAddr)
				}
			}

			respCh <- resp
		})
	}

	// For each peer, request a vote
	for _, peer := range r.peers {
		askPeer(peer)
	}

	// Persist a vote for ourselves
	if err := r.persistVote(req.Term, req.Candidate); err != nil {
		r.logger.Printf("[ERR] raft: Failed to persist vote : %v", err)
		return nil
	}

	// Include our own vote
	respCh <- &voteResult{
		RequestVoteResponse: RequestVoteResponse{
			Term:    req.Term,
			Granted: true,
		},
		voter: r.localAddr,
	}
	return respCh
}
```

首先`respCh := make(chan *voteResult, len(r.peers)+1)` 创建channel, 用于返回RPC请求响应

然后`r.setCurrentTerm(r.getCurrentTerm() + 1)`, Candidate首先递增自己的term

`lastIdx, lastTerm := r.getLastEntry()`, 获取当前节点的最后一条Log Entry的 index和term, 配合RequestVote实现论文中所说的**Leader Elector Restriction**, 然后创建RPC 请求参数
```go
req := &RequestVoteRequest{
	Term:         r.getCurrentTerm(),
	Candidate:    r.trans.EncodePeer(r.localAddr),
	LastLogIndex: lastIdx,
	LastLogTerm:  lastTerm,
}
```

`electSelf` 内创建了一个函数`askPeer`, 这个函数用于并行发起RequestVote RPC请求 
```go
// Construct a function to ask for a vote
askPeer := func(peer string) {
	r.goFunc(func() {
		defer metrics.MeasureSince([]string{"raft", "candidate", "electSelf"}, time.Now())
		resp := &voteResult{voter: peer}
		err := r.trans.RequestVote(peer, req, &resp.RequestVoteResponse)
		if err != nil {
			r.logger.Printf("[ERR] raft: Failed to make RequestVote RPC to %v: %v", peer, err)
			resp.Term = req.Term
			resp.Granted = false
		}

		// 集群发生变动
		if err == nil {
			peerSet := decodePeers(resp.Peers, r.trans)
			if !PeerContained(peerSet, r.localAddr) {
				r.logger.Printf("[WARN] raft: Remote peer %v does not have local node %v as a peer",
					peer, r.localAddr)
			}
		}

		respCh <- resp
	})
}
```
1. `askPeer` 函数首先创建 `resp := &voteResult{voter: peer}` 响应参数, `vote` 记录了发送地址的一些信息
2. `r.trans.RequestVote(peer, req, &resp.RequestVoteResponse)` 发送RequestVote RPC
3. 请求失败, 设置term不变 `resp.Term = req.Term` , 投票失败 `resp.Granted = false` 
4. 请求成功, `respCh <- resp` 将RPC请求结果写入到 `respCh`
5. 并行发送请求
   ```go
	for _, peer := range r.peers {
		askPeer(peer)
	}
	```

RequestVote 通过goroutine并行发送, 需要等待结果
```go
for _, peer := range r.peers {
	askPeer(peer)
}
```

最后, 由于`Candidate` 自己也会为自己投票, 因此将自己的赞成票写入channel
```go
respCh <- &voteResult{
	RequestVoteResponse: RequestVoteResponse{
		Term:    req.Term,
		Granted: true,
	},
	voter: r.localAddr,
}
```
#### 收集选举结果

RequstVote 是并行发出的, 因此选票结果收集也是基于channel来做的, 类似于事件, 每当请求完成后会写入到channel, Candidate只需要监听channel即可

```go
// Check if the term is greater than ours, bail
if vote.Term > r.getCurrentTerm() {
	r.logger.Printf("[DEBUG] raft: Newer term discovered, fallback to follower")
	r.setState(Follower)
	r.setCurrentTerm(vote.Term)
	return
}

// Check if the vote is granted
if vote.Granted {
	grantedVotes++
	r.logger.Printf("[DEBUG] raft: Vote granted from %s. Tally: %d", vote.voter, grantedVotes)
}

// Check if we've become the leader
if grantedVotes >= votesNeeded {
	r.logger.Printf("[INFO] raft: Election won. Tally: %d", grantedVotes)
	r.setState(Leader)
	r.setLeader(r.localAddr)
	return
}
```
1. 检查term, 如果投票者的term比自己大`vote.Term > r.getCurrentTerm()`, 则转变为follower `r.setState(Follower)`, 同时设置自己的term为投票者的term `r.setCurrentTerm(vote.Term)`

2. 如果term检查通过, 并且投票者投了赞成票`vote.Granted == true`, 则累计选票 `grantedVotes++`

3. 如果超过半数节点投了赞成票 `grantedVotes >= votesNeeded`, 则Candidate转变为leader `r.setState(Leader)`, `r.setLeader(r.localAddr)`

#### 选举超时
前面的Candidate发起选举, 并行发起RequestVote, 同时等待并收集选票来完成选举或选举失败. 

由于RequestVote通过网络发送, 因此会存在延迟或消息发送失败的情况, `Candidate` 不能无休止的等待. 因此在论文中, 如果一个Candidate在term内没有选举完成, 即选举超时, 就需要重新发起选举. 

这里通过开始设置的timeout触发, 当接收到 `<-electionTimer` 的超时通知时, 就需要退出此次选举

#### 处理RequestVote


### Become Leader