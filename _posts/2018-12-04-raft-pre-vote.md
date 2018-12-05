---
title: (raft) pre vote phase
category: raft
draft: true
---

[//]: # (CAP theorem)
[//]: # (在CAP理论中，C代表一致性，A代表可用性（在一定时间内，用户的请求都会得到应答），P代表分区容错。这里分区容错到底是指数据上的多个备份还是说其它的 ？ 我感觉分布式系统中，CAP理论应该是C和A存在不可同时满足， 既要保证高可用，又要保证强一致性，因为多个节点之间存在数据复制，所以要么保证强一致性，就不一定能在指定的时间内返回客户的请求， 要么保证高可用，但是各个节点的数据不一定是一致的。 但是和P有什么关系呢 ？)

[//]: # (一个分布式系统里面，节点组成的网络本来应该是连通的。然而可能因为一些故障，使得有些节点之间不连通了，整个网络就分成了几块区域。数据就散布在了这些不连通的区域中。这就叫分区。当你一个数据项只在一个节点中保存，那么分区出现后，和这个节点不连通的部分就访问不到这个数据了。这时分区就是无法容忍的。提高分区容忍性的办法就是一个数据项复制到多个节点上，那么出现分区之后，这一数据项就可能分布到各个区里。容忍性就提高了。然而，要把数据复制到多个节点，就会带来一致性的问题，就是多个节点上面的数据可能是不一致的。要保证一致，每次写操作就都要等待全部节点写成功，而这等待又会带来可用性的问题。总的来说就是，数据存在的节点越多，分区容忍性越高，但要复制更新的数据就越多，一致性就越难保证。为了保证一致性，更新所有节点数据所需要的时间就越长，可用性就会降低。)

[//]: # (Partition tolerance: The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes)
[//]: # (CAP theorem)

[//]: # (raft partition)
[//]: # (Raft在网络分区时leader选举的一个疑问？假设有A,B,C,D,E五台机，A是leader，某个时刻A,B出现了分区，但是A,C,D,E以及B,C,D,E都可以互相通信。B在超时没有收到心跳后，把term+1，发起leader选举，如果这段时间C,D,E没有写入更新的日志，由于B的term更大，就会被选为leader，A在后面的RPC中因为自己的term较小也会被降为follower。问题是A成为follower之后又会按照上面B的方式发起选举成为leader，同理B也会再次发起选举，这样周而复始会造成很大的网络开销吧，请问我上面的分析有没有问题呢?)
[//]: # (上面描述的是非对称网络分区的情况，确实，如果不做特殊处理，会反复出现新的选举，打断正常的 IO，造成，可用性降低的问题，一般可以通过 pre-vote 解决，例如，每次发起选举之前，先发起 pre-vote 如果获得大多数 pre-vote 选票，再增大 term 发起选举 vote 投票。为了避免问题描述的情况，其他节点只需要在收到 pre-vote 请求时，判断一下 leader 是否还在，一般实现上，判断最近和 leader 是否正常通信，如果有，那么说明 leader 正常在线，直接拒绝 pre-vote 即可。)
[//]: # (eader通过不断增大term可以一定程度上解决这个问题, raft给的方案是prevote)

[//]: # (前两位解释过了，稍微补充一下，prevote最开始是用于Preventing disruptions when a server rejoins the cluster，跟题主的场景有略微的不同。)

[//]: # (raft partition)

如果一个节点网络不好，总是心跳超时，就是触发 tickElection, 这会引起目前的 leader 恢复到 follower 到状态，然后由于这个抽疯的节点网络不好，就会重复到发起 Vote RPCs，导致很差的可用性

那么 raft 引入了 Pre-Vote 的机制来 eliminating 这种 disruptions.

看下面的代码，一个 candidate 需要先看它是否有 up-to-date 的日志，然后才能 increment 它的 term。也就是说只有 preVote 成功了才能真正进行 Vote

其他结果收到 preVote 的时候如果它和 leader 之间通信正常，则直接拒绝 preVote (canVote 中要满足 r.lead == None)

```go
	case pb.MsgVote, pb.MsgPreVote:
		...

		// We can vote if this is a repeat of a vote we've already cast...
		canVote := r.Vote == m.From ||
			// ...we haven't voted and we don't think there's a leader yet in this term...
			(r.Vote == None && r.lead == None) ||
			// ...or this is a PreVote for a future term...
			(m.Type == pb.MsgPreVote && m.Term > r.Term)
		// ...and we believe the candidate is up to date.
		if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {

```

下面来详细看下整个过程

```go
// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```

当发现超时后，它就会发 MsgHup 的消息

```go
func (r *raft) Step(m pb.Message) error {
	...
	switch m.Type {
	case pb.MsgHup:
		if r.state != StateLeader {
			...
			if r.preVote {
				r.campaign(campaignPreElection)
			} else {
				r.campaign(campaignElection)
			}
```

```go
func (r *raft) campaign(t CampaignType) {
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		r.becomePreCandidate()
		voteMsg = pb.MsgPreVote
		// PreVote RPCs are sent for the next term before we've incremented r.Term.
		term = r.Term + 1
	} else {
		r.becomeCandidate()
		voteMsg = pb.MsgVote
		term = r.Term
	}
	if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)
		} else {
			r.becomeLeader()
		}
		return
	}
	for id := range r.prs {
		if id == r.id {
			continue
		}
		r.logger.Infof("%x [logterm: %d, index: %d] sent %s request to %x at term %d",
			r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), voteMsg, id, r.Term)

		var ctx []byte
		if t == campaignTransfer {
			ctx = []byte(t)
		}
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

它就会向它的 prs 发送 PreVote

```go
func stepCandidate(r *raft, m pb.Message) error {
	...
		case myVoteRespType:
		gr := r.poll(m.From, m.Type, !m.Reject)
		r.logger.Infof("%x [quorum:%d] has received %d %s votes and %d vote rejections", r.id, r.quorum(), gr, m.Type, len(r.votes)-gr)
		switch r.quorum() {
		case gr:
			if r.state == StatePreCandidate {
				r.campaign(campaignElection)
			} else {
				r.becomeLeader()
				r.bcastAppend()
			}
```

而当它收到 myVoteRespType，则会真正去 Vote


[//]: # (![](/asserts/raft-pre-vote.png)4.2.3 Disruptive servers)
[//]: # (Without additional mechanism, servers not in Cnew can disrupt the cluster. Once the cluster leader has created the Cnew entry, a server that is not in Cnew will no longer receive heartbeats, so it will time out and start new elections. Furthermore, it will not receive the Cnew entry or learn of that entry’s commitment, so it will not know that it has been removed from the cluster. The server will send RequestVote RPCs with new term numbers, and this will cause the current leader to revert to follower state. A new leader from Cnew will eventually be elected, but the disruptive server will time out again and the process will repeat, resulting in poor availability. If multiple servers have been removed from the cluster, the situation could degrade further.)
[//]: # (Our first idea for eliminating disruptions was that, if a server is going to start an election, it would first check that it wouldn’t be wasting everyone’s time—that it had a chance to win the election. This introduced a new phase to elections, called the Pre-Vote phase. A candidate would first ask other servers whether its log was up-to-date enough to get their vote. Only if the candidate believed it could get votes from a majority of the cluster would it increment its term and start a normal election.)
[//]: # (Unfortunately, the Pre-Vote phase does not solve the problem of disruptive servers: there are situations where the disruptive server’s log is sufficiently up-to-date, but starting an election would still be disruptive. Perhaps surprisingly, these can happen even before the configuration change completes. For example, Figure 4.7 shows a server that is being removed from a cluster. Once the leader creates the Cnew log entry, the server being removed could be disruptive. The Pre-Vote check does not help in this case, since the server being removed has a log that is more up-to-date than a majority of either cluster. (Though the Pre-Vote phase does not solve the problem of disruptive servers, it does turn out to be a useful idea for improving the robustness of leader election in general; see Chapter 9.))
[//]: # (Because of this scenario, we now believe that no solution based on comparing logs alone (such as the Pre-Vote check) will be sufficient to tell if an election will be disruptive. We cannot require a server to check the logs of every server in Cnew before starting an election, since Raft must always be able to tolerate faults. We also did not want to assume that a leader will reliably replicate entries fast enough to move past the scenario shown in Figure 4.7 quickly; that might have worked in practice, but it depends on stronger assumptions that we prefer to avoid about the performance of finding)
[//]: # (where logs diverge and the performance of replicating log entries.)
[//]: # (Raft’s solution uses heartbeats to determine when a valid leader exists. In Raft, a leader is)
[//]: # (considered active if it is able to maintain heartbeats to its followers (otherwise, another server will start an election). Thus, servers should not be able to disrupt a leader whose cluster is receiving heartbeats. We modify the RequestVote RPC to achieve this: if a server receives a RequestVote request within the minimum election timeout of hearing from a current leader, it does not update its term or grant its vote. It can either drop the request, reply with a vote denial, or delay the request; the result is essentially the same. This does not affect normal elections, where each server waits at least a minimum election timeout before starting an election. However, it helps avoid disruptions from servers not in Cnew: while a leader is able to get heartbeats to its cluster, it will not be deposed by larger term numbers.)
[//]: # (This change conflicts with the leadership transfer mechanism as described in Chapter 3, in which a server legitimately starts an election without waiting an election timeout. In that case, RequestVote messages should be processed by other servers even when they believe a current cluster leader exists. Those RequestVote requests can include a special flag to indicate this behavior (“I have permission to disrupt the leader—it told me to!”).)
