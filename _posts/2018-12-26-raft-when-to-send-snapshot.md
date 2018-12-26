---
title: (raft) when to send snapshot
category: raft
draft: true
---

    'MsgSnap' requests to install a snapshot message. When a node has just
	become a leader or the leader receives 'MsgProp' message, it calls
	'bcastAppend' method, which then calls 'sendAppend' method to each
	follower. In 'sendAppend', if a leader fails to get term or entries,
	the leader requests snapshot by sending 'MsgSnap' type message.

为什么会 get term 失败呢

```go
// Term implements the Storage interface.
func (ms *MemoryStorage) Term(i uint64) (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if i < offset {
		return 0, ErrCompacted
	}
	if int(i-offset) >= len(ms.ents) {
		return 0, ErrUnavailable
	}
	return ms.ents[i-offset].Term, nil
}
```

Although servers normally take snapshots indepen- dently, the leader must occasionally send snapshots to followers that lag behind. This happens when the leader has already discarded the next log entry that it needs to send to a follower. Fortunately, this situation is unlikely in normal operation: a follower that has kept up with the leader would already have this entry. However, an excep- tionally slow follower or a new server joining the cluster (Section 6) would not. The way to bring such a follower up-to-date is for the leader to send it a snapshot over the network.

也就是说做完 snapshot 之后，一些 term 被 compact 了，这时候只能通过发送 snapshot 给 fellow，让 fellow 追上 leader
