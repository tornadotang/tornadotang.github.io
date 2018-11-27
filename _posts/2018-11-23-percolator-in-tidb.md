---
title: [tidb] percolator in tidb
catagory: tidb
draft: true
---


(**本文由 hoveywang 整理，转载在此博客**)

# background

Percolator要解决的问题很明确，如何高效地对一个既有数据库进行增量更新，其直接动机是提高google的网页库索引更新时效性。

Percolator 提供两个主要的功能：

>一个是提供跨行，跨表，跨机器的分布式事务处理能力
>另外一个是 notification框架，类似触发器，通知机制是为了帮助组织一个增量的计算，而不是帮助维护数据一致性。

我们这里主要关注分布式事务的实现。

Percolator分布式事务依赖底层数据库提供行级事务能力。

# 事务

来看下Percolator进行分布式事物处理的整体流程

![](/asserts/per1.jpg)

- 1 初始状态，Bob 账户有10美金, Joe 有2美金，write 列中的 6:data @ 5 表示当前的数据的version为5，可以看成这两条数据存于不同机器上


![](/asserts/per2.jpg)

- 2 Bob开始往Joe账户转账7美金，Bob的账户变成3美金。lock列加锁，指明自己是primary，每个分布式事务中, 只有一个primary, 正是这个primary的存在, 使得我们能够用行级原子性来实现分布式事务


![](/asserts/per3.jpg)

- 3 然后给Joe加上7美金, 所以Joe是9美元了，lock列加锁，注意 Joe 这一行的lock指向 primary。如果因为crash死锁，其他事物可以由此同步地对primary进行解锁。

![](/asserts/per4.jpg)

- 4 现在事务到达提交点（commit point）：移除primary的lock列的内容，在write列写入最新数据的version，现在3美金对外可见。

![](/asserts/per5.jpg)

- 5 然后提交其它部分。提交方式也是移除lock, 同时在write列写入新数据的version

# pseudocode

```c++
class Transaction {
  struct Write { Row row; Column col; string value; }; 
  vector<Write> writes ;
  int start ts ;

Transaction() : start ts (oracle.GetTimestamp()) {} 
void Set(Write w) { writes .push back(w); } 
bool Get(Row row, Column c, string* value) {
  while (true) {
    bigtable::Txn T = bigtable::StartRowTransaction(row); 
    // Check for locks that signal concurrent writes.
    if (T.Read(row, c+"lock", [0, start ts ])) {
      // There is a pending lock; try to clean it and wait
      BackoffAndMaybeCleanupLock(row, c);
      continue; 
    }

    // Find the latest write below our start timestamp.
    latest write = T.Read(row, c+"write", [0, start ts ]); 
    if (!latest write.found()) return false; // no data
    int data ts = latest write.start timestamp();
    *value = T.Read(row, c+"data", [data ts, data ts]); 
    return true;
  }
}

// Prewrite tries to lock cell w, returning false in case of conflict.
bool Prewrite(Write w, Write primary) {
  Column c = w.col;
  bigtable::Txn T = bigtable::StartRowTransaction(w.row);

  // Abort on writes after our start timestamp ...
  if (T.Read(w.row, c+"write", [start ts , ∞])) return false; 
  // ... or locks at any timestamp.
  if (T.Read(w.row, c+"lock", [0, ∞])) return false;

  T.Write(w.row, c+"data", start ts , w.value); 
  T.Write(w.row, c+"lock", start ts ,
      {primary.row, primary.col});  // The primary’s location.
  return T.Commit();
}

bool Commit() {
  Write primary = writes [0];
  vector<Write> secondaries(writes .begin()+1, writes .end()); 
  if (!Prewrite(primary, primary)) return false;
  for (Write w : secondaries)
    if (!Prewrite(w, primary)) return false; 

    int commit ts = oracle .GetTimestamp();

    // Commit primary first.
    Write p = primary;
    bigtable::Txn T = bigtable::StartRowTransaction(p.row); 
    if (!T.Read(p.row, p.col+"lock", [start ts , start ts ]))
      return false; // aborted while working 
    T.Write(p.row, p.col+"write", commit ts,
        start ts ); // Pointer to data written at start ts . 
    T.Erase(p.row, p.col+"lock", commit ts);
    if (!T.Commit()) return false; // commit point

    // Second phase: write out write records for secondary cells.
    for (Write w : secondaries) {
      bigtable::Write(w.row, w.col+"write", commit ts, start ts ); 
      bigtable::Erase(w.row, w.col+"lock", commit ts);
    }
    return true; 
  }
} // class Transaction
```

# corner case

假如当前事务是T，T整个生命包含3个步骤：

- step1（抢到所有的锁！）。对于每个cell，通过执行prewrite来进行抢锁。
- step2（让首要的先走！）。 primary，首要的，是所有cell里随机选出来的一个cell，它本身并不特殊（之所以要选primary是为了处理“机器故障”导致的事务异常，后续会讨论）。一个cell的提交，就是做两件事情，1是填入写记录（write 列），2是删除锁（lock列）。需要注意的是在此step，这两个动作是在一个单行事务下提交的，保证原子性（后续会解释原因）。
- step3（提交其他！）。首要的cell提交了，step3就是提交剩下的cell，也是做和step2一样的两件事情，但是这里没有用单行事务。
（注意，step1对应“2PC”的阶段一，step2和3对应“2PC”的阶段二，这里细化是为了便于分析某些场景）

## 事务冲突分析

对于事务T，可能存在的事务冲突可以按时窗视角分为（由于step2在无机器故障时和step3无异，可一视同仁，所以这里统称为step23）：

1. 发生在T之前，当T刚刚开始（拿到开始时间戳）时，它的step23已经结束
2. 发生在T之前，当T刚刚开始（拿到开始时间戳）时，它的step1已经结束，正在进行step23
3. 与T差不多时间，当T刚刚开始时它也刚刚开始
4. 发生在T之后，当T step1提交后它才刚开始
5. 发生在T之后，当T step23提交后它才刚开始

其中1、5两种情况可以先排除，因为时窗上无交集，不会相互影响。假设另一个冲突事物为G。
- 情况2中，G的step1结束，说明已经将lock写入各cell的lock列，按常理，T是肯定会看到锁而取消的。但是又由于step23正在进行中（挨个删除曾经写入的锁），所以十分可能T十分“倒霉”的没有看到任何的锁。没看到锁是小事，真正悲剧的是G还在不断的提交写记录，相当于写入新值，而这些T都蒙在鼓里。所以才要补充对写记录的检查，一旦G瞒着T（瞒着的意思就是把锁删了没让T看到）提交了写记录，G写记录的时间戳必定晚于T的开始时间戳，于是T可以以此为线索，检查各cell锁的同时也检查一下它的写记录。
- 情况3中，那就是狭路相逢，谁能抢到所有cell的行锁，就能提交step1中的prewrite动作（写入真实data和lock列），谁就继续，否则就取消。那会不会出现交集里有5个cell，T抢到3个，G抢到2个？不会，只要没抢到一次，那就直接退出。

- 情况4，与2相同，只是角色互换。

**上面应该包括了所有可能发生的情况，每种情况都可以保证T的ACID，T要么一路过关斩将直到全部提交成功，要么退出，没有第三种可能。**

## 机器故障分析

一个事务就3个step，所以机器可能在5个不同阶段挂掉
1. step1开始前
2. step1期间
3. step2期间
4. step3期间
5. step3之后

同样的，1、5可以不用考虑。

**如果在第2阶段挂掉：**

step1做的事情是提交各个cell的真实值和锁，弄到一半，机器突然挂了，怎么知道提交了哪些？没提交哪些？简直是死无对证，毁尸灭迹——Percolator中没有任何一个组件知道这里死了个事务，更不知道更改了哪些表的哪些列。

Percolator必须清理这些锁，否则他们将导致将来的事务被非预期的挂起。Percolator用一个懒惰的途径来实现清理：当一个事务A遭遇一个被事务B遗弃的锁，A可以确定B遭遇故障，并清除它的锁。然而希望A很准确的判断出B失败是十分困难的；可能发生这样的情况，A准备清理B的事务，而事实上B并未故障还在尝试提交事务，我们必须想办法避免。现在就要详细介绍一下上面已经提到过的“primary”概念。Percolator在每个事务中会对任意的提交或者清理操作指定一个cell作为同步点。这个cell的锁被称之为“primary锁”。A和B在哪个锁是primary上达成一致（primary锁的位置被写入所有cell的锁中）。执行一个清理或提交操作都需要修改primary锁；这个修改操作会在一个Bigtable行事务之下执行，所以只有一个操作可以成功。特别的，在B提交之前，它必须检查它依然拥有primary锁，提交时会将它替换为一个写记录。在A删除B的锁之前，A也必须检查primary锁来保证B没有提交；如果primary锁依然存在它就能安全的删除B的锁。

如果一个客户端在第二阶段提交时崩溃，一个事务将错过提交点（它已经写过至少一个写记录），而且出现未解决的锁。我们必须对这种事务执行roll-forward。当其他事务遭遇了这个因为故障而被遗弃的锁时，它可以通过检查primary锁来区分这两种情况：如果primary锁已被替换为一个写记录，写入此锁的事务则必须提交，此锁必须被roll forward；否则它应该被回滚（因为我们总是先提交primary，所以如果primary没有提交我们能肯定回滚是安全的）。执行roll forward时，执行清理的事务也是将搁浅的锁替换为一个写记录。

清理操作在primary锁上是同步的，所以清理活跃客户端持有的锁是安全的；然而回滚会强迫事务取消，这会严重影响性能。所以，一个事务将不会清理一个锁除非它猜测这个锁属于一个僵死的worker。Percolator使用简单的机制来确定另一个事务的活跃度。运行中的worker会写一个token到Chubby锁服务来指示他们属于本系统，token会被其他worker视为一个代表活跃度的信号（当处理退出时token会被自动删除）。有些worker是活跃的，但不在运行中，为了处理这种情况，我们附加的写入一个wall time到锁中；一个锁的wall time如果太老，即使token有效也会被清理。有些操作运行很长时间才会提交，针对这种情况，在整个提交过程中worker会周期的更新wall time。

**如果在第3阶段挂掉：**

检查primary锁是否还存在，如果存在，则同情况2，如果不存在则同情况4。

**如果在第4阶段挂掉：**

每个尝试清理的Get()操作，都会先做一个判断，检查Primary锁是否存在。如果锁仍存在，则断定依然属于情况2，可以放心清理；但是如果锁消失了，则必须进入情况4的前滚模式roll forward。前滚也很简单，当Get()进入情况4模式后，它通过该锁，找到时间戳，以此时间戳往write列里提交个写记录即可（然后再删除锁）。

这就是为什么，Percolator一直强调它的分布式事务不是靠中央总控实现的，而是一种懒惰的，等着后来的目击者Get()去一个个的发现和自我修正的客户端协调控制型分布式事务。

## 为什么secondaries提交不用事务？

从伪代码中可以看出，secondaries提交时，陆陆续续的把每个cell的值开放出去了（写记录添加了，锁也删了），这样陆续开放，会不会导致中间一个瞬间，可能5个cell有3个被外界查询了新值，另2个被查询了旧值？答案是否定的，另2个没来得及开放出去的锁也没删掉，查询会停住，等待它释放锁。

# percolator in tidb

TiDB官网中，给了一个事务使用的示例：

![](/asserts/tidb-per1.jpg)

1 从bnechkv的代码中一步步跟进，可以看到store.Begin()会新起一个事务，txn.Set是告知事务要执行的具体操作，txn.Commit是提交事务

![](/asserts/tidb-per2.png)

2 store由tikv.Driver Open产生

![](/asserts/tidb-per22.png)

3 driver.Open里生成一个newTikvStore

![](/asserts/tidb-per3.png)

4 newTikvStore里初始化一个新的timestamp发生器（序列生成器）oracles，然后返回一个初始化好的tikvStore

![](/asserts/tidb-per4.png)

5 store.Begin()新起一个事务newTiKVTxn

![](/asserts/tidb-per5.png)

6 newTiKVTxn初始化事务的起始时间戳startTS，snapshot，NewUnionStore

![](/asserts/tidb-per6.png)

7 NewUnionStore初始化一个NewBufferStore，unionStore is an in-memory Store which contains a buffer for write and a snapshot for read

![](/asserts/tidb-per7.png)

8 NewBufferStore初始化lazyMemBuffer，lazyMemBuffer封装了MemBuffer，并实现的Set接口，最后对外表现的就是txn.Set()

![](/asserts/tidb-per8.png)
![](/asserts/tidb-per81.png)

9 txn.Commit()创建一个两阶段提交器committer，committer来执行实际的提交

![](/asserts/tidb-per9.png)

10 committer执行execute操作

![](/asserts/tidb-per10.png)

11 首先prewriteKeys

![](/asserts/tidb-per11.png)
![](/asserts/tidb-per112.png)
![](/asserts/tidb-per113.png)

12 appendBatchBySize将所有需要操作的值缓存到数组，如Percolator所述

![](/asserts/tidb-per12.png)

13 prewrite

![](/asserts/tidb-per13.png)

14 doActionOnBatches逻辑如下

![](/asserts/tidb-per14.png)

15 prewriteSingleBatch向TiKV发送一个PrewriteRequest

![](/asserts/tidb-per15.png)

16 然后获取commitTS

![](/asserts/tidb-per16.png)

17 然后commitKeys

![](/asserts/tidb-per17.png)
![](/asserts/tidb-per172.png)

18 commitKeys类似prewrite调用了doActionOnBatches，但传递的action参数不同

![](/asserts/tidb-per18.png)

19 doActionOnBatches先提交primary，然后提交secondaries

![](/asserts/tidb-per19.png)

20 commitSingleBatch向TiKV发送CommitRequest

![](/asserts/tidb-per20.png)

21 如果commit失败，则clean up all written keys

![](/asserts/tidb-per21.png)

# reference

[1] [Daniel Peng, Frank Dabek Large-scale Incremental Processing Using Distributed Transactions and Notifications Proceedings of the 9th USENIX Symposium on Operating Systems Design and Implementation, USENIX (2010)](/asserts/Percolator.pdf)
