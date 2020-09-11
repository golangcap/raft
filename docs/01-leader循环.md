
## Leader loop

当节点成为leader后, 会运行`runLeader`
```go
func (r *Raft) runLeader() {
	r.logger.Printf("[INFO] raft: %v entering Leader state", r)
	metrics.IncrCounter([]string{"raft", "state", "leader"}, 1)

	// 主要用于测试, 外部可以从leaderCh知道此刻有leader选出了
	// Notify that we are the leader
	asyncNotifyBool(r.leaderCh, true)

	// Push to the notify channel if given
	if notify := r.conf.NotifyCh; notify != nil {
		select {
		case notify <- true:
		case <-r.shutdownCh:
		}
	}

	// Setup leader state
	r.leaderState.commitCh = make(chan struct{}, 1)
	r.leaderState.inflight = newInflight(r.leaderState.commitCh)
	r.leaderState.replState = make(map[string]*followerReplication)
	r.leaderState.notify = make(map[*verifyFuture]struct{})
	r.leaderState.stepDown = make(chan struct{}, 1)

	// Cleanup state on step down
	defer func() {
		// Since we were the leader previously, we update our
		// last contact time when we step down, so that we are not
		// reporting a last contact time from before we were the
		// leader. Otherwise, to a client it would seem our data
		// is extremely stale.
		r.setLastContact()

		// Stop replication
		for _, p := range r.leaderState.replState {
			close(p.stopCh)
		}

		// Cancel inflight requests
		r.leaderState.inflight.Cancel(ErrLeadershipLost)

		// Respond to any pending verify requests
		for future := range r.leaderState.notify {
			future.respond(ErrLeadershipLost)
		}

		// Clear all the state
		r.leaderState.commitCh = nil
		r.leaderState.inflight = nil
		r.leaderState.replState = nil
		r.leaderState.notify = nil
		r.leaderState.stepDown = nil

		// If we are stepping down for some reason, no known leader.
		// We may have stepped down due to an RPC call, which would
		// provide the leader, so we cannot always blank this out.
		r.leaderLock.Lock()
		if r.leader == r.localAddr {
			r.leader = ""
		}
		r.leaderLock.Unlock()

		// Notify that we are not the leader
		asyncNotifyBool(r.leaderCh, false)

		// Push to the notify channel if given
		if notify := r.conf.NotifyCh; notify != nil {
			select {
			case notify <- false:
			case <-r.shutdownCh:
				// On shutdown, make a best effort but do not block
				select {
				case notify <- false:
				default:
				}
			}
		}
	}()

	// Start a replication routine for each peer
	for _, peer := range r.peers {
		r.startReplication(peer)
	}

	// Dispatch a no-op log first. Instead of LogNoop,
	// we use a LogAddPeer with our peerset. This acts like
	// a no-op as well, but when doing an initial bootstrap, ensures
	// that all nodes share a common peerset.
	peerSet := append([]string{r.localAddr}, r.peers...)
	noop := &logFuture{
		log: Log{
			Type: LogAddPeer,
			Data: encodePeers(peerSet, r.trans),
		},
	}
	r.dispatchLogs([]*logFuture{noop})

	// Disable EnableSingleNode after we've been elected leader.
	// This is to prevent a split brain in the future, if we are removed
	// from the cluster and then elect ourself as leader.
	if r.conf.DisableBootstrapAfterElect && r.conf.EnableSingleNode {
		r.logger.Printf("[INFO] raft: Disabling EnableSingleNode (bootstrap)")
		r.conf.EnableSingleNode = false
	}

	// Sit in the leader loop until we step down
	r.leaderLoop()
}

```

首先leader会设置一些自身的状态
```go
// 通知提交的channel
r.leaderState.commitCh = make(chan struct{}, 1)
// 日志复制冲突优化
r.leaderState.inflight = newInflight(r.leaderState.commitCh)
// 管理复制的peer的state
r.leaderState.replState = make(map[string]*followerReplication)
r.leaderState.notify = make(map[*verifyFuture]struct{})
// leader 退位channel
r.leaderState.stepDown = make(chan struct{}, 1)
```
 
之后leader会启动日志复制的goroutine, 然后分发一个noop日志项, 最后运行 `r.leaderLoop()`




然后leader开始复制日志到所有的peer
```go
	for _, peer := range r.peers {
		r.startReplication(peer)
	}
```

真正执行日志复制的函数是 `startReplication`

```go

```