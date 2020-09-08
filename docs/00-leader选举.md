



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

### heartbeat timeout

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


### convert candidate