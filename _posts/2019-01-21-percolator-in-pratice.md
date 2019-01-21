---
title: Percolator in practice
category: tidb
draft: true
---

# 读写事务

1) 事务提交前，在客户端 buffer 所有的 update/delete 操作。 

2) Prewrite 阶段:

首先在所有行的写操作中选出一个作为 primary，其他的为 secondaries。

PrewritePrimary: 对 primaryRow 写入 L 列(上锁)，L 列中记录本次事务的开始时间戳。写入 L 列前会检查:

- 是否已经有别的客户端已经上锁 (Locking)。

- 是否在本次事务开始时间之后，检查 W 列，是否有更新 [startTs, +Inf) 的写操作已经提交 (Conflict)。

在这两种种情况下会返回事务冲突。否则，就成功上锁。将行的内容写入 row 中，时间戳设置为 startTs。

将 primaryRow 的锁上好了以后，进行 secondaries 的 prewrite 流程:

类似 primaryRow 的上锁流程，只不过锁的内容为事务开始时间及 primaryRow 的 Lock 的信息。

检查的事项同 primaryRow 的一致。

当锁成功写入后，写入 row，时间戳设置为 startTs。

3) 以上 Prewrite 流程任何一步发生错误，都会进行回滚：删除 Lock，删除版本为 startTs 的数据。

4) 当 Prewrite 完成以后，进入 Commit 阶段，当前时间戳为 commitTs，且 commitTs> startTs :

commit primary：写入 W 列新数据，时间戳为 commitTs，内容为 startTs，表明数据的最新版本是 startTs 对应的数据。

删除L列。如果 primary row 提交失败的话，全事务回滚，回滚逻辑同 prewrite。如果 commit primary 成功，则可以异步的 commit secondaries, 流程和 commit primary 一致， 失败了也无所谓。


# 事务中的读操作

检查该行是否有 L 列，时间戳为 [0, startTs]，如果有，表示目前有其他事务正占用此行，如果这个锁已经超时则尝试清除，否则等待超时或者其他事务主动解锁。注意此时不能直接返回老版本的数据，否则会发生幻读的问题。

读取至 startTs 时该行最新的数据，方法是：读取 W 列，时间戳为 [0, startTs], 获取这一列的值，转化成时间戳 t, 然后读取此列于 t 版本的数据内容。

由于锁是分两级的，primary 和 seconary，只要 primary 的行锁去掉，就表示该事务已经成功 提交，这样的好处是 secondary 的 commit 是可以异步进行的，只是在异步提交进行的过程中 ，如果此时有读请求，可能会需要做一下锁的清理工作。

# Code

```go
// UnionStore is a store that wraps a snapshot for read and a BufferStore for buffered write.
// Also, it provides some transaction related utilities.
type UnionStore interface {
	MemBuffer
	// CheckLazyConditionPairs loads all lazy values from store then checks if all values are matched.
	// Lazy condition pairs should be checked before transaction commit.
	CheckLazyConditionPairs() error
	// WalkBuffer iterates all buffered kv pairs.
	WalkBuffer(f func(k Key, v []byte) error) error
	// SetOption sets an option with a value, when val is nil, uses the default
	// value of this option.
	SetOption(opt Option, val interface{})
	// DelOption deletes an option.
	DelOption(opt Option)
	// GetOption gets an option.
	GetOption(opt Option) interface{}
	// GetMemBuffer return the MemBuffer binding to this UnionStore.
	GetMemBuffer() MemBuffer
}
```

in tikv, the interface is kv_get corresponding CmdGet in tidb

```go
func (s *tikvSnapshot) get(bo *Backoffer, k kv.Key) ([]byte, error) {
	sender := NewRegionRequestSender(s.store.regionCache, s.store.client)

	req := &tikvrpc.Request{
		Type: tikvrpc.CmdGet,
		Get: &pb.GetRequest{
			Key:     k,
			Version: s.version.Ver,
		},
		Context: pb.Context{
			Priority:     s.priority,
			NotFillCache: s.notFillCache,
		},
	}
	for {
		loc, err := s.store.regionCache.LocateKey(bo, k)
		if err != nil {
			return nil, errors.Trace(err)
		}
		resp, err := sender.SendReq(bo, req, loc.Region, readTimeoutShort)
		if err != nil {
			return nil, errors.Trace(err)
		}
		regionErr, err := resp.GetRegionError()
		if err != nil {
			return nil, errors.Trace(err)
		}
		if regionErr != nil {
			err = bo.Backoff(BoRegionMiss, errors.New(regionErr.String()))
			if err != nil {
				return nil, errors.Trace(err)
			}
			continue
		}
		cmdGetResp := resp.Get
		if cmdGetResp == nil {
			return nil, errors.Trace(ErrBodyMissing)
		}
		val := cmdGetResp.GetValue()
		if keyErr := cmdGetResp.GetError(); keyErr != nil {
			lock, err := extractLockFromKeyErr(keyErr)
			if err != nil {
				return nil, errors.Trace(err)
			}
			ok, err := s.store.lockResolver.ResolveLocks(bo, []*Lock{lock})
			if err != nil {
				return nil, errors.Trace(err)
			}
			if !ok {
				err = bo.Backoff(boTxnLockFast, errors.New(keyErr.String()))
				if err != nil {
					return nil, errors.Trace(err)
				}
			}
			continue
		}
		return val, nil
	}
}
```

而在 tikv 的读会检查 L 和 W 锁，如果返回错误，则需要 tidb maybeCleanupLock

同样在 prewrite 阶段也需要 store.lockResolver.ResolveLocks

```go
func (c *twoPhaseCommitter) prewriteSingleBatch(bo *Backoffer, batch batchKeys) error {
	mutations := make([]*pb.Mutation, len(batch.keys))
	for i, k := range batch.keys {
		mutations[i] = c.mutations[string(k)]
	}

	req := &tikvrpc.Request{
		Type: tikvrpc.CmdPrewrite,
		Prewrite: &pb.PrewriteRequest{
			Mutations:    mutations,
			PrimaryLock:  c.primary(),
			StartVersion: c.startTS,
			LockTtl:      c.lockTTL,
		},
		Context: pb.Context{
			Priority: c.priority,
			SyncLog:  c.syncLog,
		},
	}
	for {
		resp, err := c.store.SendReq(bo, req, batch.region, readTimeoutShort)
		if err != nil {
			return errors.Trace(err)
		}
		regionErr, err := resp.GetRegionError()
		if err != nil {
			return errors.Trace(err)
		}
		if regionErr != nil {
			err = bo.Backoff(BoRegionMiss, errors.New(regionErr.String()))
			if err != nil {
				return errors.Trace(err)
			}
			err = c.prewriteKeys(bo, batch.keys)
			return errors.Trace(err)
		}
		prewriteResp := resp.Prewrite
		if prewriteResp == nil {
			return errors.Trace(ErrBodyMissing)
		}
		keyErrs := prewriteResp.GetErrors()
		if len(keyErrs) == 0 {
			return nil
		}
		var locks []*Lock
		for _, keyErr := range keyErrs {
			lock, err1 := extractLockFromKeyErr(keyErr)
			if err1 != nil {
				return errors.Trace(err1)
			}
			log.Debugf("con:%d 2PC prewrite encounters lock: %v", c.connID, lock)
			locks = append(locks, lock)
		}
		ok, err := c.store.lockResolver.ResolveLocks(bo, locks)
		if err != nil {
			return errors.Trace(err)
		}
		if !ok {
			err = bo.Backoff(BoTxnLock, errors.Errorf("2PC prewrite lockedKeys: %d", len(locks)))
			if err != nil {
				return errors.Trace(err)
			}
		}
	}
}
```

# Reference

[Percolator 和 TiDB 事务算法](https://pingcap.com/blog-cn/percolator-and-txn/)
