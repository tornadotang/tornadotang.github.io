---
title: (raft) committing entries from previous term
category: raft
draft: true
---


![](/asserts/raft1.png)

那么在 (c) ,S1 是否能提交 log 2 ？

假如提交了 log 2，那么 S1 的 log entry 是 12，然后在 (d) S5 提交了 log 3，那么 S5 的log entry 是 13，而 S1 是 123，这就违背了 log matching

Log Matching: if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index

所以 raft 的机制是：

Raft never commits log entries from previous terms by count- ing replicas. Only log entries from the leader’s current term are committed by counting replicas; once an entry from the current term has been committed in this way, then all prior entries are committed indirectly because of the Log Matching Property

下面看下代码：

![](/asserts/raft2.png)


而在 maybeCommit 函数里：

![](/asserts/raft3.png)

这里会判断，S1 在 (c) 获得主的 term 是 4，而之前 log 2 的 term 是 2，所以这次 StepLeader 里并不会提交 2，也就是论文里提到的 indirectly。



再额外说一下：

为什么 maybeCommit 之后还要 broadcostAppend，因为日志不一定全部复制完了，要直到日志收敛完成才停止。
