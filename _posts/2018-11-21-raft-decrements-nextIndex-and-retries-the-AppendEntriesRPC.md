---
title: [raft] decrement next indexes and retries the AppendEntriesRPC
category: raft
draft: true
---


Leader 先发送一个 MsgProp

![](/asserts/raft4.png)

在bcastAppend 中会发送给 follower MsgApp

![](/asserts/raft5.png)

然后 Follower 接收到 MsgApp 之后

![](/asserts/raft6.png)

stepFollower 如果 matchTerm 返回 false，则返回 Reject 给 Leader

而这里的 index 则是

![](/asserts/raft7.png)

所以比对的是 leader 准备 append 的 entry 的前一个，如果不相同，则 reject

但是如果 leader准备复制 index 9，而  follower 已经有日志 20，和 leader 不一样会怎么样？

![](/asserts/raft8.png)

然后 Leader 收到 Reject，则 调用 maybeDecrTo

而在 maybeDecrTo 中还区分状态机 Probe, Replicate, Snapshot
