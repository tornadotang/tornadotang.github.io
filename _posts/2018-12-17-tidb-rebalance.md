---
title: (tidb) automatic rebalancing
category: tidb
draft: true
---


(**本文由 erikchai 整理，转载在此博客**)

# rebalance

在pd中rebalance是由一个被称作**coordinator**的模块控制的。每次的**schedule**会返回一个op，并会交由coordinator发给**TiKV**中的region做进一步的处理。

```go
type coordinator struct {
	sync.RWMutex

	wg     sync.WaitGroup
	ctx    context.Context
	cancel context.CancelFunc

	cluster          *clusterInfo
	replicaChecker   *schedule.ReplicaChecker
	regionScatterer  *schedule.RegionScatterer
	namespaceChecker *schedule.NamespaceChecker
	mergeChecker     *schedule.MergeChecker
	schedulers       map[string]*scheduleController
	opController     *schedule.OperatorController
	classifier       namespace.Classifier
	hbStreams        *heartbeatStreams
}
```

schedule是在coordinator的run函数由配置文件被装入的。

```go
func (c *coordinator) run() {
	ticker := time.NewTicker(runSchedulerCheckInterval)
	defer ticker.Stop()
	log.Info("coordinator: Start collect cluster information")
	for {
		if c.shouldRun() {
			log.Info("coordinator: Cluster information is prepared")
			break
		}
		select {
		case <-ticker.C:
		case <-c.ctx.Done():
			return
		}
	}
	log.Info("coordinator: Run scheduler")

	k := 0
	scheduleCfg := c.cluster.opt.load().clone()
	for _, schedulerCfg := range scheduleCfg.Schedulers {
		if schedulerCfg.Disable {
			scheduleCfg.Schedulers[k] = schedulerCfg
			k++
			log.Info("skip create ", schedulerCfg.Type)
			continue
		}
		s, err := schedule.CreateScheduler(schedulerCfg.Type, c.opController, schedulerCfg.Args...)
		if err != nil {
			log.Errorf("can not create scheduler %s: %v", schedulerCfg.Type, err)
		} else {
			log.Infof("create scheduler %s", s.GetName())
			if err = c.addScheduler(s, schedulerCfg.Args...); err != nil {
				log.Errorf("can not add scheduler %s: %v", s.GetName(), err)
			}
		}

		// only record valid scheduler config
		if err == nil {
			scheduleCfg.Schedulers[k] = schedulerCfg
			k++
		}
	}

	// remove invalid scheduler config and persist
	scheduleCfg.Schedulers = scheduleCfg.Schedulers[:k]
	c.cluster.opt.store(scheduleCfg)
	if err := c.cluster.opt.persist(c.cluster.kv); err != nil {
		log.Errorf("can't persist schedule config: %v", err)
	}

	c.wg.Add(1)
	go c.patrolRegions()
}
```

在addScheduler函数中，coordinator调用runScheduler来启动调度。

```go
func (c *coordinator) addScheduler(scheduler schedule.Scheduler, args ...string) error {
	c.Lock()
	defer c.Unlock()

	if _, ok := c.schedulers[scheduler.GetName()]; ok {
		return errSchedulerExisted
	}

	s := newScheduleController(c, scheduler)
	if err := s.Prepare(c.cluster); err != nil {
		return err
	}

	c.wg.Add(1)
	go c.runScheduler(s)
	c.schedulers[s.GetName()] = s
	c.cluster.opt.AddSchedulerCfg(s.GetType(), args)

	return nil
}
```

runScheduler函数通过预设的时间间隔来触发调度。

```go
func (c *coordinator) runScheduler(s *scheduleController) {
	defer logutil.LogPanic()
	defer c.wg.Done()
	defer s.Cleanup(c.cluster)

	timer := time.NewTimer(s.GetInterval())
	defer timer.Stop()

	for {
		select {
		case <-timer.C:
			timer.Reset(s.GetInterval())
			if !s.AllowSchedule() {
				continue
			}
			if op := s.Schedule(); op != nil {
				c.opController.AddOperator(op...)
			}

		case <-s.Ctx().Done():
			log.Infof("%v stopped: %v", s.GetName(), s.Ctx().Err())
			return
		}
	}
}
```

在pd的代码中存在多种的平衡策略

- **adjacent_region**
- **balance_leader**
- **balance_region**
- **evict_leader**
- **grant_leader**
- **hot_region**
- **label**
- **random_merge**
- **scatter_range**
- **shuffle_leader**
- **shuffle_region**

## SelectTarget

在下面的balance函数中会遇到大量的SelectTarget，这个SelectTarget是一个类似c++的重载函数，根据不同balance策略，会有不同的实现。

对于BalanceSelector，SelectTarget的选择目标是根据过滤条件选择一个**resource sorce值最大**的store，配套还有一个SelectSource函数选择一个**resource sorce值最小**的函数。

```go
// SelectTarget selects the store that can pass all filters and has the maximal
// resource score.
func (s *BalanceSelector) SelectTarget(opt Options, stores []*core.StoreInfo, filters ...Filter) *core.StoreInfo {
	filters = append(filters, s.filters...)
	var result *core.StoreInfo
	for _, store := range stores {
		if FilterTarget(opt, store, filters) {
			continue
		}
		if result == nil ||
			result.ResourceScore(s.kind, opt.GetHighSpaceRatio(), opt.GetLowSpaceRatio(), 0) >
				store.ResourceScore(s.kind, opt.GetHighSpaceRatio(), opt.GetLowSpaceRatio(), 0) {
			result = store
		}
	}
	return result
}
```

ReplicaSelector选择目标是根据过滤条件选择一个**distinct sorce值最大**的store，配套还有一个SelectSource函数选择一个**distinct sorce**值最小的函数。

```go
// SelectTarget selects the store that can pass all filters and has the maximal
// distinct score.
func (s *ReplicaSelector) SelectTarget(opt Options, stores []*core.StoreInfo, filters ...Filter) *core.StoreInfo {
	var (
		best      *core.StoreInfo
		bestScore float64
	)
	for _, store := range stores {
		if FilterTarget(opt, store, filters) {
			continue
		}
		score := DistinctScore(s.labels, s.regionStores, store)
		if best == nil || compareStoreScore(opt, store, score, best, bestScore) > 0 {
			best, bestScore = store, score
		}
	}
	if best == nil || FilterTarget(opt, best, s.filters) {
		return nil
	}
	return best
}
```

对于RandomSelector，SelectTarget根据过滤条件随机选择一个store。

```go
// SelectTarget randomly selects a target store from those can pass all filters.
func (s *RandomSelector) SelectTarget(opt Options, stores []*core.StoreInfo, filters ...Filter) *core.StoreInfo {
	filters = append(filters, s.filters...)

	var candidates []*core.StoreInfo
	for _, store := range stores {
		if FilterTarget(opt, store, filters) {
			continue
		}
		candidates = append(candidates, store)
	}
	return s.randStore(candidates)
}
```

关于**resourse score**和**distinct score**注释解释很详细，下面就贴出代码

```go
// DistinctScore returns the score that the other is distinct from the stores.
// A higher score means the other store is more different from the existed stores.
func DistinctScore(labels []string, stores []*core.StoreInfo, other *core.StoreInfo) float64 {
	var score float64
	for _, s := range stores {
		if s.GetId() == other.GetId() {
			continue
		}
		if index := s.CompareLocation(other, labels); index != -1 {
			score += math.Pow(replicaBaseScore, float64(len(labels)-index-1))
		}
	}
	return score
}

// ResourceScore reutrns score of leader/region in the store.
func (s *StoreInfo) ResourceScore(kind ResourceKind, highSpaceRatio, lowSpaceRatio float64, delta int64) float64 {
	switch kind {
	case LeaderKind:
		return s.LeaderScore(delta)
	case RegionKind:
		return s.RegionScore(highSpaceRatio, lowSpaceRatio, delta)
	default:
		return 0
	}
}
```

### adjacent_region

adjacent_region主要是分散临近region的leader保证**<u>临近region</u>**的leader不在一个store中，所谓<u>**临近region**</u>是指key值连续的region。

```go
// balanceAdjacentRegionScheduler will disperse adjacent regions.
// we will scan a part regions order by key, then select the longest
// adjacent regions and disperse them. finally, we will guarantee
// 1. any two adjacent regions' leader will not in the same store
// 2. the two regions' leader will not in the public store of this two regions
type balanceAdjacentRegionScheduler struct {
	*baseScheduler
	selector             *schedule.RandomSelector
	leaderLimit          uint64
	peerLimit            uint64
	lastKey              []byte
	cacheRegions         *adjacentState		//缓存每次获得的最长临近region
	adjacentRegionsCount int
}
```

在Schedule函数中，首先会检查cacheRegions中是否存在前一次平衡中。如果存在则对前两个进行平衡，否则将最长的连续region放进cacheRegions中。

```go
func (l *balanceAdjacentRegionScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
  //......
  if l.cacheRegions.len() >= 2 {
      return l.process(cluster)
  }
  //......
}
```

**process**函数主要的作用是平衡**临近的**两个集群，在函数中首先调用**disperseLeader**来讲两个key值连续的region的leader分散开，具体做法是找到两个集群中不想交的region节点，所谓不相交就是两个集群的节点没有位于同一个store的节点。随后将集群leader的角色转变到新的节点上。若没有leader则调用**dispersePeer**重新打散集群中所有peer到新的store中。

```go
func (l *balanceAdjacentRegionScheduler) process(cluster schedule.Cluster) []*schedule.Operator {
	if l.cacheRegions.len() < 2 {
		return nil
	}
	head := l.cacheRegions.head
	r1 := l.cacheRegions.regions[head]
	r2 := l.cacheRegions.regions[head+1]

	defer func() {
		if l.cacheRegions.len() < 0 {
			log.Fatalf("[%s]the cache overflow should never happen", l.GetName())
		}
		l.cacheRegions.head = head + 1
		l.lastKey = r2.GetStartKey()
	}()
	// after the cluster is prepared, there is a gap that some regions heartbeats are not received.
	// Leader of those region is nil, and we should skip them.
	if r1.GetLeader() == nil || r2.GetLeader() == nil || l.unsafeToBalance(cluster, r1) {
		schedulerCounter.WithLabelValues(l.GetName(), "skip").Inc()
		return nil
	}
	op := l.disperseLeader(cluster, r1, r2)
	if op == nil {
		schedulerCounter.WithLabelValues(l.GetName(), "no_leader").Inc()
		op = l.dispersePeer(cluster, r1)
	}
	if op == nil {
		schedulerCounter.WithLabelValues(l.GetName(), "no_peer").Inc()
		l.cacheRegions.assignedStoreIds = l.cacheRegions.assignedStoreIds[:0]
		return nil
	}
	return []*schedule.Operator{op}
}
```

若缓存中没有待处理的region，则Schedule函数会扫描一遍所有的store找到最长的有连续key的region并放入缓存，最后再调用**process**来进行**rebalance**。

```go
func (l *balanceAdjacentRegionScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	if l.cacheRegions == nil {
		l.cacheRegions = &adjacentState{
			assignedStoreIds: make([]uint64, 0, len(cluster.GetStores())),
			regions:          make([]*core.RegionInfo, 0, scanLimit),
			head:             0,
		}
	}
	// we will process cache firstly
	if l.cacheRegions.len() >= 2 {
		return l.process(cluster)
	}

	l.cacheRegions.clear()
	regions := cluster.ScanRegions(l.lastKey, scanLimit)
	// scan to the end
	if len(regions) <= 1 {
		schedulerStatus.WithLabelValues(l.GetName(), "adjacent_count").Set(float64(l.adjacentRegionsCount))
		l.adjacentRegionsCount = 0
		l.lastKey = []byte("")
		return nil
	}

	// calculate max adjacentRegions and record to the cache
	adjacentRegions := make([]*core.RegionInfo, 0, scanLimit)
	adjacentRegions = append(adjacentRegions, regions[0])
	maxLen := 0
	for i, r := range regions[1:] {
		l.lastKey = r.GetStartKey()

		// append if the region are adjacent
		lastRegion := adjacentRegions[len(adjacentRegions)-1]
		if lastRegion.GetLeader().GetStoreId() == r.GetLeader().GetStoreId() && bytes.Equal(lastRegion.GetEndKey(), r.GetStartKey()) {
			adjacentRegions = append(adjacentRegions, r)
			if i != len(regions)-2 { // not the last element
				continue
			}
		}

		if len(adjacentRegions) == 1 {
			adjacentRegions[0] = r
		} else {
			// got an max length adjacent regions in this range
			if maxLen < len(adjacentRegions) {
				l.cacheRegions.clear()
				maxLen = len(adjacentRegions)
				l.cacheRegions.regions = append(l.cacheRegions.regions, adjacentRegions...)
				adjacentRegions = adjacentRegions[:0]
				adjacentRegions = append(adjacentRegions, r)
			}
		}
	}

	l.adjacentRegionsCount += maxLen
	return l.process(cluster)
}
```

## balance_leader

这个函数的作用就是将leader最多的store的leader换出，并换入到leader节点最少的store。在pd中用一个被称作**score**的分数值来评价一个storeleader节点的权重。

```go
func (l *balanceLeaderScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(l.GetName(), "schedule").Inc()

	stores := cluster.GetStores()

	// source/target is the store with highest/lowest leader score in the list that
	// can be selected as balance source/target.
	source := l.selector.SelectSource(cluster, stores)
	target := l.selector.SelectTarget(cluster, stores)

	// No store can be selected as source or target.
	if source == nil || target == nil {
		schedulerCounter.WithLabelValues(l.GetName(), "no_store").Inc()
		// When the cluster is balanced, all stores will be added to the cache once
		// all of them have been selected. This will cause the scheduler to not adapt
		// to sudden change of a store's leader. Here we clear the taint cache and
		// re-iterate.
		l.taintStores.Clear()
		return nil
	}

	log.Debugf("[%s] store%d has the max leader score, store%d has the min leader score", l.GetName(), source.GetId(), target.GetId())
	sourceStoreLabel := strconv.FormatUint(source.GetId(), 10)
	targetStoreLabel := strconv.FormatUint(target.GetId(), 10)
	balanceLeaderCounter.WithLabelValues("high_score", sourceStoreLabel).Inc()
	balanceLeaderCounter.WithLabelValues("low_score", targetStoreLabel).Inc()

	opInfluence := l.opController.GetOpInfluence(cluster)
	for i := 0; i < balanceLeaderRetryLimit; i++ {
		if op := l.transferLeaderOut(source, cluster, opInfluence); op != nil {
			balanceLeaderCounter.WithLabelValues("transfer_out", sourceStoreLabel).Inc()
			return op
		}
		if op := l.transferLeaderIn(target, cluster, opInfluence); op != nil {
			balanceLeaderCounter.WithLabelValues("transfer_in", targetStoreLabel).Inc()
			return op
		}
	}

	// If no operator can be created for the selected stores, ignore them for a while.
	log.Debugf("[%s] no operator created for selected store%d and store%d", l.GetName(), source.GetId(), target.GetId())
	balanceLeaderCounter.WithLabelValues("add_taint", strconv.FormatUint(source.GetId(), 10)).Inc()
	l.taintStores.Put(source.GetId())
	balanceLeaderCounter.WithLabelValues("add_taint", strconv.FormatUint(target.GetId(), 10)).Inc()
	l.taintStores.Put(target.GetId())
	return nil
}
```

## balance_region

首先这个函数通过上述region的score来选出**score最高的**一个store作为要调整的**sourse**。在一个重试次数内，也就是**balanceRegionRetryLimit**，随机选出一个region并找到它在上述store中的peer，将改peer迁移到其他peer中。在选择需要迁移region的过程中，函数还会限制不能将有**不正常副本数的region**和**热点region**作为迁移对象。

```go
func (s *balanceRegionScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()

	stores := cluster.GetStores()

	// source is the store with highest region score in the list that can be selected as balance source.
	source := s.selector.SelectSource(cluster, stores)
	if source == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_store").Inc()
		// Unlike the balanceLeaderScheduler, we don't need to clear the taintCache
		// here. Because normally region score won't change rapidly, and the region
		// balance requires lower sensitivity compare to leader balance.
		return nil
	}

	log.Debugf("[%s] store%d has the max region score", s.GetName(), source.GetId())
	sourceLabel := strconv.FormatUint(source.GetId(), 10)
	balanceRegionCounter.WithLabelValues("source_store", sourceLabel).Inc()

	opInfluence := s.opController.GetOpInfluence(cluster)
	var hasPotentialTarget bool
	for i := 0; i < balanceRegionRetryLimit; i++ {
		region := cluster.RandFollowerRegion(source.GetId(), core.HealthRegion())
		if region == nil {
			region = cluster.RandLeaderRegion(source.GetId(), core.HealthRegion())
		}
		if region == nil {
			schedulerCounter.WithLabelValues(s.GetName(), "no_region").Inc()
			continue
		}
		log.Debugf("[%s] select region%d", s.GetName(), region.GetID())

		// We don't schedule region with abnormal number of replicas.
		if len(region.GetPeers()) != cluster.GetMaxReplicas() {
			log.Debugf("[%s] region%d has abnormal replica count", s.GetName(), region.GetID())
			schedulerCounter.WithLabelValues(s.GetName(), "abnormal_replica").Inc()
			continue
		}

		// Skip hot regions.
		if cluster.IsRegionHot(region.GetID()) {
			log.Debugf("[%s] region%d is hot", s.GetName(), region.GetID())
			schedulerCounter.WithLabelValues(s.GetName(), "region_hot").Inc()
			continue
		}

		if !s.hasPotentialTarget(cluster, region, source, opInfluence) {
			continue
		}
		hasPotentialTarget = true

		oldPeer := region.GetStorePeer(source.GetId())
		if op := s.transferPeer(cluster, region, oldPeer, opInfluence); op != nil {
			schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
			return []*schedule.Operator{op}
		}
	}

	if !hasPotentialTarget {
		// If no potential target store can be found for the selected store, ignore it for a while.
		log.Debugf("[%s] no operator created for selected store%d", s.GetName(), source.GetId())
		balanceRegionCounter.WithLabelValues("add_taint", sourceLabel).Inc()
		s.taintStores.Put(source.GetId())
	}

	return nil
}
```

## evict_leader

这个函数在指定store上随机选择一个leader region，并将leader迁移至该集群分数最低的节点上。

```go
func (s *evictLeaderScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()
	region := cluster.RandLeaderRegion(s.storeID, core.HealthRegion())
	if region == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_leader").Inc()
		return nil
	}
	target := s.selector.SelectTarget(cluster, cluster.GetFollowerStores(region))
	if target == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_target_store").Inc()
		return nil
	}
	schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
	step := schedule.TransferLeader{FromStore: region.GetLeader().GetStoreId(), ToStore: target.GetId()}
	op := schedule.NewOperator("evict-leader", region.GetID(), region.GetRegionEpoch(), schedule.OpLeader, step)
	op.SetPriorityLevel(core.HighPriority)
	return []*schedule.Operator{op}
}
```

SelectTarget这个函数就是通过store的score来选择目标节点。

```go
func (s *BalanceSelector) SelectTarget(opt Options, stores []*core.StoreInfo, filters ...Filter) *core.StoreInfo {
	filters = append(filters, s.filters...)
	var result *core.StoreInfo
	for _, store := range stores {
		if FilterTarget(opt, store, filters) {
			continue
		}
		if result == nil ||
			result.ResourceScore(s.kind, opt.GetHighSpaceRatio(), opt.GetLowSpaceRatio(), 0) >
				store.ResourceScore(s.kind, opt.GetHighSpaceRatio(), opt.GetLowSpaceRatio(), 0) {
			result = store
		}
	}
	return result
}
```

## grant_leader

这个函数在指定store上随机选择一个region，将该集群中leader的角色转移到这个节点上。

```go
func (s *grantLeaderScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()
	region := cluster.RandFollowerRegion(s.storeID, core.HealthRegion())
	if region == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_follower").Inc()
		return nil
	}
	schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
	step := schedule.TransferLeader{FromStore: region.GetLeader().GetStoreId(), ToStore: s.storeID}
	op := schedule.NewOperator("grant-leader", region.GetID(), region.GetRegionEpoch(), schedule.OpLeader, step)
	op.SetPriorityLevel(core.HighPriority)
	return []*schedule.Operator{op}
}
```

## hot_region

这个函数依据两个关键的指标来判断是否要进行balance操作。

- hot region count
- read/write flow

判断的标准是，若存在maxium的hot region count则将这个region作为**source**，否则继续判这些region中read/write flow最大的那一个。**target**的标准则刚好相反，首先优先将minium的hot region count region作为target，否则继续判断read/write flow。所有的这些信息都是通过一些**stats**结构获取的，所以**hot_region的主要功能就是读取统计信息，并根据以上规则balance**。

在函数中，hot_region首先会根据一个随机值来决定平衡写热点还是读热点，然后会计算统计信息并进行调度。

```go
func (h *balanceHotRegionsScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(h.GetName(), "schedule").Inc()
	return h.dispatch(h.types[h.r.Int()%len(h.types)], cluster)
}

func (h *balanceHotRegionsScheduler) dispatch(typ BalanceType, cluster schedule.Cluster) []*schedule.Operator {
	h.Lock()
	defer h.Unlock()
	switch typ {
	case hotReadRegionBalance:
		h.stats.readStatAsLeader = h.calcScore(cluster.RegionReadStats(), cluster, core.LeaderKind)
		return h.balanceHotReadRegions(cluster)
	case hotWriteRegionBalance:
		h.stats.writeStatAsLeader = h.calcScore(cluster.RegionWriteStats(), cluster, core.LeaderKind)
		h.stats.writeStatAsPeer = h.calcScore(cluster.RegionWriteStats(), cluster, core.RegionKind)
		return h.balanceHotWriteRegions(cluster)
	}
	return nil
}
```

**balanceHotReadRegions**和**balanceHotWriteRegions**就是找到source和target并进行balance的过程。

在balanceHotReadRegions中，**balanceByLeader**用于转换一个**hot region**的leader到其他store中，balanceByPeer**hot region**中的非leader region移动到其他区域。

```go
func (h *balanceHotRegionsScheduler) balanceHotReadRegions(cluster schedule.Cluster) []*schedule.Operator {
	// balance by leader
	srcRegion, newLeader := h.balanceByLeader(cluster, h.stats.readStatAsLeader)
	if srcRegion != nil {
		schedulerCounter.WithLabelValues(h.GetName(), "move_leader").Inc()
		step := schedule.TransferLeader{FromStore: srcRegion.GetLeader().GetStoreId(), ToStore: newLeader.GetStoreId()}
		return []*schedule.Operator{schedule.NewOperator("transferHotReadLeader", srcRegion.GetID(), srcRegion.GetRegionEpoch(), schedule.OpHotRegion|schedule.OpLeader, step)}
	}

	// balance by peer
	srcRegion, srcPeer, destPeer := h.balanceByPeer(cluster, h.stats.readStatAsLeader)
	if srcRegion != nil {
		schedulerCounter.WithLabelValues(h.GetName(), "move_peer").Inc()
		return []*schedule.Operator{schedule.CreateMovePeerOperator("moveHotReadRegion", cluster, srcRegion, schedule.OpHotRegion, srcPeer.GetStoreId(), destPeer.GetStoreId(), destPeer.GetId())}
	}
	schedulerCounter.WithLabelValues(h.GetName(), "skip").Inc()
	return nil
}
```

## label

意义暂时不明

## random_merge

这个函数的作用是选择随机一个store中的leader region和随机的前一个或后一个合并。

```go
func (s *randomMergeScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()

	stores := cluster.GetStores()
	store := s.selector.SelectSource(cluster, stores)
	if store == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_store").Inc()
		return nil
	}
	region := cluster.RandLeaderRegion(store.GetId(), core.HealthRegion())
	if region == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_region").Inc()
		return nil
	}

	target, other := cluster.GetAdjacentRegions(region)
	if (rand.Int()%2 == 0 && other != nil) || target == nil {
		target = other
	}
	if target == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_adjacent").Inc()
		return nil
	}

	schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
	ops, err := schedule.CreateMergeRegionOperator("random-merge", cluster, region, target, schedule.OpAdmin)
	if err != nil {
		return nil
	}
	return ops
}
```

GetAdjacentRegions函数会返回临近树中的前一个和后一个。

```go
// GetAdjacentRegions returns region's info that is adjacent with specific region
func (r *RegionsInfo) GetAdjacentRegions(region *RegionInfo) (*RegionInfo, *RegionInfo) {
	metaPrev, metaNext := r.tree.getAdjacentRegions(region.meta)
	var prev, next *RegionInfo
	// check key to avoid key range hole
	if metaPrev != nil && bytes.Equal(metaPrev.region.EndKey, region.meta.StartKey) {
		prev = r.GetRegion(metaPrev.region.GetId())
	}
	if metaNext != nil && bytes.Equal(region.meta.EndKey, metaNext.region.StartKey) {
		next = r.GetRegion(metaNext.region.GetId())
	}
	return prev, next
}
```

## scatter_range

这个函数的作用是随机选择一个cluster，做上述的**balanceLeader**和**balanceRegion**操作。

```go
func (l *scatterRangeScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(l.GetName(), "schedule").Inc()
	c := schedule.GenRangeCluster(cluster, l.startKey, l.endKey)
	c.SetTolerantSizeRatio(2)
	ops := l.balanceLeader.Schedule(c)
	if len(ops) > 0 {
		ops[0].SetDesc(fmt.Sprintf("scatter-range-leader-%s", l.rangeName))
		ops[0].AttachKind(schedule.OpRange)
		schedulerCounter.WithLabelValues(l.GetName(), "new-leader-operator").Inc()
		return ops
	}
	ops = l.balanceRegion.Schedule(c)
	if len(ops) > 0 {
		ops[0].SetDesc(fmt.Sprintf("scatter-range-region-%s", l.rangeName))
		ops[0].AttachKind(schedule.OpRange)
		schedulerCounter.WithLabelValues(l.GetName(), "new-region-operator").Inc()
		return ops
	}
	schedulerCounter.WithLabelValues(l.GetName(), "no-need").Inc()
	return nil
}
```

## shuffle_leader

这个函数随机选择一个store，然后随机选择一个follower并将该集群的leader转移到这个store上。

```go
func (s *shuffleLeaderScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	// We shuffle leaders between stores by:
	// 1. random select a valid store.
	// 2. transfer a leader to the store.
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()
	stores := cluster.GetStores()
	targetStore := s.selector.SelectTarget(cluster, stores)
	if targetStore == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_target_store").Inc()
		return nil
	}
	region := cluster.RandFollowerRegion(targetStore.GetId(), core.HealthRegion())
	if region == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_follower").Inc()
		return nil
	}
	schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
	step := schedule.TransferLeader{FromStore: region.GetLeader().GetStoreId(), ToStore: targetStore.GetId()}
	op := schedule.NewOperator("shuffleLeader", region.GetID(), region.GetRegionEpoch(), schedule.OpAdmin|schedule.OpLeader, step)
	op.SetPriorityLevel(core.HighPriority)
	return []*schedule.Operator{op}
}
```

## shuffle_region

同上一个类似将随机一个store中的region移除该store。

```go
func (s *shuffleRegionScheduler) Schedule(cluster schedule.Cluster) []*schedule.Operator {
	schedulerCounter.WithLabelValues(s.GetName(), "schedule").Inc()
	region, oldPeer := s.scheduleRemovePeer(cluster)
	if region == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_region").Inc()
		return nil
	}

	excludedFilter := schedule.NewExcludedFilter(nil, region.GetStoreIds())
	newPeer := s.scheduleAddPeer(cluster, excludedFilter)
	if newPeer == nil {
		schedulerCounter.WithLabelValues(s.GetName(), "no_new_peer").Inc()
		return nil
	}

	schedulerCounter.WithLabelValues(s.GetName(), "new_operator").Inc()
	op := schedule.CreateMovePeerOperator("shuffle-region", cluster, region, schedule.OpAdmin, oldPeer.GetStoreId(), newPeer.GetStoreId(), newPeer.GetId())
	op.SetPriorityLevel(core.HighPriority)
	return []*schedule.Operator{op}
}
```


