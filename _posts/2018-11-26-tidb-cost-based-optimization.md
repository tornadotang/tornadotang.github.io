---
title: (tidb) cost based optimization
category: tidb
draft: true
---

# CBO(cost based optimization) 流程

```go
func physicalOptimize(logic LogicalPlan) (PhysicalPlan, error) {
	if _, err := logic.deriveStats(); err != nil {
		return nil, errors.Trace(err)
	}

	logic.preparePossibleProperties()

	prop := &property.PhysicalProperty{
		TaskTp:      property.RootTaskType,
		ExpectedCnt: math.MaxFloat64,
	}

	t, err := logic.findBestTask(prop)
	if err != nil {
		return nil, errors.Trace(err)
	}
	if t.invalid() {
		return nil, ErrInternal.GenWithStackByArgs("Can't find a proper physical plan for this query")
	}

	t.plan().ResolveIndices()
	return t.plan(), nil
}


// findBestTask implements LogicalPlan interface.
func (p *baseLogicalPlan) findBestTask(prop *property.PhysicalProperty) (bestTask task, err error) {
    
    ...

	physicalPlans := p.self.exhaustPhysicalPlans(prop)
	prop.Cols = oldPropCols

	for _, pp := range physicalPlans {
		// find best child tasks firstly.
		childTasks = childTasks[:0]
		for i, child := range p.children {
			childTask, err := child.findBestTask(pp.getChildReqProps(i))
			if err != nil {
				return nil, errors.Trace(err)
			}
			if childTask != nil && childTask.invalid() {
				break
			}
			childTasks = append(childTasks, childTask)
		}

		// This check makes sure that there is no invalid child task.
		if len(childTasks) != len(p.children) {
			continue
		}

		// combine best child tasks with parent physical plan.
		curTask := pp.attach2Task(childTasks...)

		// enforce curTask property
		if prop.Enforced {
			curTask = enforceProperty(prop, curTask, p.basePlan.ctx)
		}

		// get the most efficient one.
		if curTask.cost() < bestTask.cost() {
			bestTask = curTask
		}
	}

	p.storeTask(prop, bestTask)
	return bestTask, nil
}
```

# 例子

```sql
select sum(s.a),count(t.b) from s join t on s.a = t.a and s.c < 100 and t.c > 10 group bys.a
```

![](/asserts/cbo1.png)
> - SortMerge Join（SMJ）
> - Hash Join（HJ）
> - Index Join（IdxJ）
> - IdxScan(IdxS)
> - TableScan(TS) 

## 物理算子简介
- StreamAgg: 按 group by 的 key 有序的，所以它可以取到一个 group 的数据后，剋直接返回对应数据
- HashAgg: 通过 group by 的 key 算 hash 值，将所有数据按 hash 值都存到内存，等数据全部读完，然后一个个输出
从代码可以很清晰的看到先构建诸多 plan，然后遍历 plans，从中找到一个最优的

# 代价评估
    Cost(p) = N(p)*FN+M(p)*FM+C(p)*FC
    
    > 其中 N 表示网络开销，M 表示内存开销，C 表示 CPU 开销，F 表示因子




# Reference
[TiDB 源码阅读系列文章（八）基于代价的优化](https://pingcap.com/blog-cn/tidb-source-code-reading-8/)
