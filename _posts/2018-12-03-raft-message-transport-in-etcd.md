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
