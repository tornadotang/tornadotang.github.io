---
title: (raft) pre vote phase
category: raft
draft: true
---

4.2.3 Disruptive servers
Without additional mechanism, servers not in Cnew can disrupt the cluster. Once the cluster leader has created the Cnew entry, a server that is not in Cnew will no longer receive heartbeats, so it will time out and start new elections. Furthermore, it will not receive the Cnew entry or learn of that entry’s commitment, so it will not know that it has been removed from the cluster. The server will send RequestVote RPCs with new term numbers, and this will cause the current leader to revert to follower state. A new leader from Cnew will eventually be elected, but the disruptive server will time out again and the process will repeat, resulting in poor availability. If multiple servers have been removed from the cluster, the situation could degrade further.

Our first idea for eliminating disruptions was that, if a server is going to start an election, it would first check that it wouldn’t be wasting everyone’s time—that it had a chance to win the election. This introduced a new phase to elections, called the Pre-Vote phase. A candidate would first ask other servers whether its log was up-to-date enough to get their vote. Only if the candidate

![](/asserts/raft-pre-vote.png)

Unfortunately, the Pre-Vote phase does not solve the problem of disruptive servers: there are situations where the disruptive server’s log is sufficiently up-to-date, but starting an election would still be disruptive. Perhaps surprisingly, these can happen even before the configuration change completes. For example, Figure 4.7 shows a server that is being removed from a cluster. Once the leader creates the Cnew log entry, the server being removed could be disruptive. The Pre-Vote check does not help in this case, since the server being removed has a log that is more up-to-date than a majority of either cluster. (Though the Pre-Vote phase does not solve the problem of disruptive servers, it does turn out to be a useful idea for improving the robustness of leader election in general; see Chapter 9.)

Because of this scenario, we now believe that no solution based on comparing logs alone (such as the Pre-Vote check) will be sufficient to tell if an election will be disruptive. We cannot require a server to check the logs of every server in Cnew before starting an election, since Raft must always be able to tolerate faults. We also did not want to assume that a leader will reliably replicate entries fast enough to move past the scenario shown in Figure 4.7 quickly; that might have worked in practice, but it depends on stronger assumptions that we prefer to avoid about the performance of finding

where logs diverge and the performance of replicating log entries.

Raft’s solution uses heartbeats to determine when a valid leader exists. In Raft, a leader is
considered active if it is able to maintain heartbeats to its followers (otherwise, another server will start an election). Thus, servers should not be able to disrupt a leader whose cluster is receiving heartbeats. We modify the RequestVote RPC to achieve this: if a server receives a RequestVote request within the minimum election timeout of hearing from a current leader, it does not update its term or grant its vote. It can either drop the request, reply with a vote denial, or delay the request; the result is essentially the same. This does not affect normal elections, where each server waits at least a minimum election timeout before starting an election. However, it helps avoid disruptions from servers not in Cnew: while a leader is able to get heartbeats to its cluster, it will not be deposed by larger term numbers.

This change conflicts with the leadership transfer mechanism as described in Chapter 3, in which a server legitimately starts an election without waiting an election timeout. In that case, RequestVote messages should be processed by other servers even when they believe a current cluster leader exists. Those RequestVote requests can include a special flag to indicate this behavior (“I have permission to disrupt the leader—it told me to!”).
