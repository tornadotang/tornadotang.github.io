---
title: (raft) etcd linearibility
category: raft
draft: true
---

如果请求在 Leader 上：

```go
	case pb.MsgReadIndex:
		case ReadOnlySafe:
            r.readOnly.addRequest(r.raftLog.committed, m)
            r.bcastHeartbeatWithCtx(m.Entries[0].Data)
	case pb.MsgHeartbeatResp:
		for _, rs := range rss {
			req := rs.req
			if req.From == None || req.From == r.id { // from local member
				r.readStates = append(r.readStates, ReadState{Index: rs.index, RequestCtx: req.Entries[0].Data})
			} else {
				r.send(pb.Message{To: req.From, Type: pb.MsgReadIndexResp, Index: rs.index, Entries: req.Entries})
			}
		}
```



如果请求发在 Follower 上：

```go
	case pb.MsgReadIndex:
		if r.lead == None {
			r.logger.Infof("%x no leader at term %d; dropping index reading msg", r.id, r.Term)
			return nil
		}
		m.To = r.lead
		r.send(m)
	case pb.MsgReadIndexResp:
		if len(m.Entries) != 1 {
			r.logger.Errorf("%x invalid format of MsgReadIndexResp from %x, entries count: %d", r.id, m.From, len(m.Entries))
			return nil
		}
		r.readStates = append(r.readStates, ReadState{Index: m.Index, RequestCtx: m.Entries[0].Data})
	}
```

# Reference
[etcd-raft的线性一致读方法一：ReadInde](https://zhuanlan.zhihu.com/p/31050303)

