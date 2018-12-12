---
title: (raft) etcd message tranport
category: raft
draft: true
---

```go
func NewServer(cfg ServerConfig) (srv *EtcdServer, err error) {
    ...
    tr := &rafthttp.Transport{
        ...
    }
    for _, m := range cl.Members() {
		if m.ID != id {
			tr.AddPeer(m.ID, m.PeerURLs)
		}
	}
}

func (t *Transport) AddPeer(id types.ID, us []string) {
    ...
    t.peers[id] = startPeer(t, urls, id, fs)
}


func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats) *peer {
    ...
	p.msgAppV2Reader = &streamReader{
		lg:     t.Logger,
		peerID: peerID,
		typ:    streamTypeMsgAppV2,
		tr:     t,
		picker: picker,
		status: status,
		recvc:  p.recvc,
		propc:  p.propc,
		rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
	}
	p.msgAppReader = &streamReader{
		lg:     t.Logger,
		peerID: peerID,
		typ:    streamTypeMessage,
		tr:     t,
		picker: picker,
		status: status,
		recvc:  p.recvc,
		propc:  p.propc,
		rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
	}

	p.msgAppV2Reader.start()
	p.msgAppReader.start()
    ...
}

func (cr *streamReader) start() {
	cr.done = make(chan struct{})
	if cr.errorc == nil {
		cr.errorc = cr.tr.ErrorC
	}
	if cr.ctx == nil {
		cr.ctx, cr.cancel = context.WithCancel(context.Background())
	}
	go cr.run()
}

func (cr *streamReader) run() {
    ...
    err = cr.decodeLoop(rc, t)
}

func (cr *streamReader) dial(t streamType) (io.ReadCloser, error) {
    ...
    resp, err := cr.tr.streamRt.RoundTrip(req)
}

func (cr *streamReader) decodeLoop(rc io.ReadCloser, t streamType) error {
    ...
    recvc := cr.recvc
		if m.Type == raftpb.MsgProp {
			recvc = cr.propc
		}

		select {
		case recvc <- m:
    ...
}
```

然后看一下 cr.recvc，即上面 recvc:  p.recvc, 就是每一个 Peer 初始化的 recvc

```go
func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats) *peer {
    ...
	go func() {
		for {
			select {
			case mm := <-p.recvc:
				if err := r.Process(ctx, mm); err != nil {
```

也就是说 Channel recvc 有事件了，那么会调用 r.Process 函数，进行 raft 的 Step 了 

当server需要向其他server发送数据时，只需要找到其他server对应的peer，然后向peer的streamWriter的msgc通道发送数据即可，streamWriter会监听msgc通道的数据并发送到对端server；

```go
func (r *raft) send(m pb.Message) {
   ...
   r.msgs = append(r.msgs, m)
}

func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
		Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
	}
   ...
}

func (r *raftNode) start(rh *raftReadyHandler) {
	...
	r.transport.Send(r.processMessages(rd.Messages))
	...
}

func (t *Transport) Send(msgs []raftpb.Message) {
	...
	p.send(m)
	...
}

func (p *peer) send(m raftpb.Message) {
	...
	writec, name := p.pick(m)
	...
}

func (p *peer) pick(m raftpb.Message) (writec chan<- raftpb.Message, picked string) {
	...
	} else if writec, ok = p.msgAppV2Writer.writec(); ok && isMsgApp(m) {
	...
}


func (cw *streamWriter) run() {
	...
	case conn := <-cw.connc:
	...
	heartbeatc, msgc = tickc.C, cw.msgc
	...
}
```

# 一个 PUT 的请求的执行过程

```go
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PutResponse), nil
}

func (s *EtcdServer) raftRequest(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	for {
		resp, err := s.raftRequestOnce(ctx, r)
		if err != auth.ErrAuthOldRevision {
			return resp, err
		}
	}
}

func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	result, err := s.processInternalRaftRequestOnce(ctx, r)
	if err != nil {
		return nil, err
	}
	if result.err != nil {
		return nil, result.err
	}
	return result.resp, nil
}

func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error) {
	...
	err = s.r.Propose(cctx, data)
	...
}

func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}
```

这里的 s.r.Propose 调用的是 node 的 Propose

使用的 golang 的 [embedded field 和 protomed](https://golang.org/ref/spec#Struct_types)

```go
type EtcdServer struct {
	r            raftNode        // uses 64-bit atomics; keep 64-bit aligned.
}

type raftNode struct {
	lg *zap.Logger

	tickMu *sync.Mutex
	raftNodeConfig
}

type raftNodeConfig struct {
	lg *zap.Logger

	// to check if msg receiver is removed from cluster
	isIDRemoved func(id uint64) bool
	raft.Node
}
```

这里 raftNode 用了 embedded field raftNodeConfig，然后 raftNodeConfig 里面使用了 embedded field raft.Node

所以调用 s.r.Propose, 看起来是用 EtcdServer 的 raftNode 的 Propose，而实际上是用了其 promoted 的 node.Propose

而调用 Propose 之后：

- 如果是 follower，则转给 leader 执行

```go
func stepFollower(r *raft, m pb.Message) error {
	switch m.Type {
	case pb.MsgProp:
		if r.lead == None {
			r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
			return ErrProposalDropped
		} else if r.disableProposalForwarding {
			r.logger.Infof("%x not forwarding to leader %x at term %d; dropping proposal", r.id, r.lead, r.Term)
			return ErrProposalDropped
		}
		m.To = r.lead
		r.send(m)
```

- 如果是 leader，执行 appendEntry，然后 broadcast 给 follower

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	...
	case pb.MsgProp:
		...
		if !r.appendEntry(m.Entries...) {
			return ErrProposalDropped
		}
		r.bcastAppend()
		return nil
```
