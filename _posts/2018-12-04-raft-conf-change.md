---
title: (raft) confChange
category: raft
draft: true
---

(**本文由 hoveywang 整理，转载在此博客**)

# Preface

在进行具体配置变更的阐述前，先进行一些宏观上的说明，以方便阅读。

利用etcd实现一个分布式集群，那么在集群启动时会依次进行如下动作：

![](/asserts/raft-confchange.png)

核心处理模块说明，每个模块实际上都有对应的routine处理事件：
- node:用于处理raft算法相关逻辑。
- raftNode:用于调度node产生的消息，node模块的执行是通过消息机制，将产生的消息交给raftNode进行调度。
- raftNode.transport:主要处理其他server的连接。
- kvstore:用于监听kvstore的配置添加和集群变更请求，会将请求转换为消息交给raftNode来调度处理。

# 模块间通信的channel

main函数里创建了4个channel，分别是：

*数据入口：*

- proposeC： 让用户代码可以向raft提交写请求(Propose)
- confChangeC： 让用户代码向raft提交配置变更请求(ProposeConfChange)

*数据出口：*

- commitC： 把已经committed entries从raft中暴露出来给用户代码来进行写State Machine
- errorC ： 让用户可以及时处理raft抛出的错误信息

配置变更需要关注的是confChangeC。

# 配置变更逻辑

1 main函数里在会起一个serveHttpKVAPI来监听Http变更请求

![](/asserts/raft-confchange1.png)

2 serveHttpKVAPI接收confChangeC channel作为参数，会将节点变更信息写入confChangeC

![](/asserts/raft-confchange2.png)

3 配置变更的pb定义了消息格式

![](/asserts/raft-confchange3.jpg)

![](/asserts/raft-confchange32.jpg)

4 raftNode的routine中对confChangeC的事件进行处理

![](/asserts/raft-confchange4.jpg)

5 调用了node的ProposeConfChange方法

![](/asserts/raft-confchange5.jpg)

6 接下来调用了node的Step->step->stepWithWaitOption，向ch通道写入消息，触发MsgProp事件：

![](/asserts/raft-confchange6.png)

7 可见对于集群变更的处理是将集群变更信息写入n.propc通道，node对于该通道数据的处理

从node的routine中可以看到，propc实际上被底层的raft的r.step方法取走了

![](/asserts/raft-confchange7.jpg)

8 r.step有三个不同的实现。stepLeader, stepCandidate, stepFollower。 具体用哪个就要根据当前raft节点的状态来决定

![](/asserts/raft-confchange8.png)

配置变更我们只看stepLeader，可以看到将该集群变更作为日志追加到本地r.appendEntry(m.Entries...)，然后向其他follower发送附加日志rpc：r.bcastAppend()

9 当该集群变更日志复制到过半数server后，raftNode提交日志的处理逻辑如下

![](/asserts/raft-confchange9.png)

10 首先是更新集群变更的状态信息，向n.confc通道写入集群变更消息

![](/asserts/raft-confchange10.jpg)

11 node的进行对应的add,remove等操作

![](/asserts/raft-confchange11.jpg)

12 node的进行对应的add,remove等操作

![](/asserts/raft-confchange12.jpg)

就是设置下该peer的发送日志进度信息。

13 最后回到raftNode中对于可提交的集群变更日志的处理

在更新完集群信息后执行了rc.transport.AddPeer(types.ID(cc.NodeID), []string{string(cc.Context)})，即把该节点加入到当前集群，建立与该节点的通信。

当该日志在其他follower也提交时，其他follower也会同样处理把这个新节点加入集群。

因此，只有集群变更日志在当前server被提交完成后，当前server才建立与新节点的通信，才知道集群的最新规模，在复制集群变更日志的过程中他们依然不知道集群的最新规模。

但对于新节点来说，在启动式会知道老集群的节点信息，因此新节点启动后就知道了集群的最新规模。


# Conclusion

可以看到etcd在配置变更的实际实现中，并没有采用过渡配置的方式，而是每次变更只能添加或删除一个节点，不能一次变更多个节点，因为每次变更一个节点能保证不会有多个leader同时产生。

以下图为例：

![](/asserts/raft-confchange14.jpg)

最初节点个数为3，即server1、server2、server3，最初leader为server3，如果有2个节点要加入到集群，那么在原来的3个节点收到集群变更请求前认为集群中有3个节点，确切的说是集群变更日志提交前，假如server3作为leader，把集群变更日志发送到了server4、server5，这样可以提交该集群变更日志了，因此server3、server4、server5的集群变更日志提交后他们知道当前集群有5个节点了。而server1和server2还没收到集群变更日志或者收到了集群变更日志但没有提交，那么server1和server2认为集群中还是有3个节点。假设此时server3因为网络原因重新发起选举，server1也同时发起选举，server1在收到server2的投票赞成响应而成为leader，而server3可以通过server4和server5也可以成为leader，这时出现了两个leader，是不安全且不允许的。

但如果每次只添加或减少1个节点，假设最初有3个节点，有1个节点要加入。最初leader为server1，在server1的集群变更日志提交前，server1、server2、server3认为集群中有3个节点，只有server4认为集群中有4个节点，leader必然在server1、server2、server3中产生。如果在server1是leader时该集群变更日志提交了，那么集群中至少有2个server有该集群变更日志了，假如server1和server2都有该集群变更日志了，server3和server4还没有，那么server3和server4不可能被选为leader，因为他们的日志不够新。对于server4来说需要3个server同时同意才能成为leader，而server1和server2的日志比他新，不会同意他成为leader。对于server3来说，在集群变更日志提交前他认为集群中只有3个server，因此只会把投票请求发送给server1和server2，而server1和server2因为日志比他新不会同意；如果server3的集群变更日志也提交了，那么他人为集群中有4个节点，这时与server4一样，需要3个server同时同意才能成为leader，如果server1通过server2成为leader了，那么server1和server2都不会参与投票了。

*因此每次一个节点的加入不管在集群变更日志应用前、应用过程中还是提应用后都不会出现两个leader的情况。*

应用前是因为原来的节点不知道这个新的节点，不会发送投票给他，也不会处理新节点的投票请求；

应用后是因为大家都知道集群的最新规模了，不会产生两个大多数的投票；

应用过程中是因为没有这条集群变更日志的server由于日志不够新也不能成为leader，比如最初集群规模是2n+1，现在有1个新节点加入，如果集群变更日志复制到了过半数server，因为之前的leader是老的集群的，因此过半数是n+1，假如这个n+1个server中产生了一个leader，那么对于新的节点来说，因为集群变更日志还没有应用到状态机所以只有这个新节点认为集群中有2n+2个server，因此需要n+2个server同意投票他才能成为leader，但这是不可能的，因为已经有n+1个节点已经投过票了，而对于老集群中的剩下的没有投票的n个节点中，他们任何一个server都需要n+1个server同意才能成为leader，而他们因为还没有把集群变更日志真正提交即应用到状态机，还不知道新节点的存在，也就不能收到n+1个server投票，最多只能收到n个节点的投票，因此也不能成为leader，保证了只能有一个leader被选出来。

-- Appendix

```
Raft Rules:
// 所有server的原则 Rules for Servers
// 1. 如果commitIndex > lastApplied:则递增lastApplied,应用 log[lastApplied] 到状态机之中
// 2. 如果Rpc请求或回复包括任期T > currentTerm: 设置currentTerm = T,转换成 follower, 并且设置 votedFor=-1，表示未投票
// rules for Followers
// 回复 candidates与leaders的RPC请求
// 如果选举超时时间达到,并且没有收到来自当前leader或者要求投票的候选者的 AppendEnties RPC调 :转换角色为candidate
// rules for Candidates
// 转换成candidate时,开始一个选举:
// 1. 递增currentTerm;投票给自己;
// 2. 重置election timer;
// 3. 向所有的服务器发送 RequestVote RPC请求
// 如果获取服务器中多数投票:转换成Leader
// 如果收到从新Leader发送的AppendEnties RPC请求:转换成follower
// 如果选举超时时间达到:开始一次新的选举
// rules for Leaders
// 给每个服务器发送初始空的AppendEntires RPCs(heartbeat);指定空闲时间之后重复该操作以防 election timeouts
// 如果收到来自客户端的命令:将条目插入到本地日志,在条目应用到状态机后回复给客户端
// 如果last log index >= nextIndex for a follower:发送包含开始于nextIndex的日志条目的AppendEnties RPC
// 如果成功:为follower更新nextIndex与matchIndex
// 如果失败是由于日志不一致:递减nextIndex然后重试
// 如果存在一个N满足 N>commitIndex,多数的matchIndex[i] >= N,并且 log[N].term == currentTerm:设置commitIndex = N
// AppendEntries RPC的实现：在回复给RPCs之前需要更新到持久化存储之上
// 有3类用途
// 1. candidate赢得选举的后，宣誓主权
// 2. 保持心跳
// 3. 让follower的日志和自己保持一致
// 接收者的处理逻辑：
// 1. 如果term < currentTerm 则返回false
// 2. 如果日志不包含一个在preLogIndex位置任期为prevLogTerm的条目,则返回 false
 该规则是需要保证follower已经包含了leader在PrevLogIndex之前所有的日志了
// 3. 如果一个已存在的条目与新条目冲突(同样的索引但是不同的任期),则删除现存的该条目与其后的所有条
// 4. 将不在log中的新条目添加到日志之中
// 5. 如果leaderCommit > commitIndex,那么设置 commitIndex = min(leaderCommit,index of last new entry)
// RequestVote RPC 的实现: 由候选者发起用于收集选票
// 1. 如果term < currentTerm 则返回false
// 2. 如果本地的voteFor为空或者为candidateId,并且候选者的日志至少与接受者的日志一样新,则投给其选票
// 怎么定义日志新
// 比较两份日志中最后一条日志条目的索引值和任期号定义谁的日志比较新
// 如果两份日志最后的条目的任期号不同，那么任期号大的日志更加新
// 如果两份日志最后的条目任期号相同，那么日志比较长的那个就更加新。
// 以上所有的规则保证的下面的几个点：
// 1. Election Safety 在一个特定的任期中最多只有一个Leader会被选举出来
// 2. Leader Append-Only Leader不会在他的日志中覆盖或删除条 ,他只执行添加新的条
// 3. Log Matching:如果两个日志包含了同样index和term的条 ,那么在该index之前的所有条目都是相同的
// 4. Leader Completeness:如果在一个特定的term上提交了一个日志条目,那么该条目将显示在编号较大的任期的Leader的日志里
// 5. State Machine Safety:如果一个服务器在一个给定的index下应用一个日志条目到他的状态机上,没有其他服务器会在相同index上应用不同的日志条目
```
