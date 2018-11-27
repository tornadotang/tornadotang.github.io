---
title: [tidb] rule based optimization
category: tidb
draft: true
---


# introduce operator 

                 ------------
                | Projection |
                 ------------
               
                      |
               
                 -------------
                | Selection   |
                 -------------
               
                      |
               
                  ------------
                 |  Join      |
                  ------------
   
               /              \
   
     ---------------     ---------------
    | DataSource #1 |   | DataSource #2 |
     ---------------     ---------------


```go
// DoOptimize optimizes a logical plan to a physical plan.
func DoOptimize(flag uint64, logic LogicalPlan) (PhysicalPlan, error) {
	logic, err := logicalOptimize(flag, logic)
	if err != nil {
		return nil, errors.Trace(err)
	}
	if !AllowCartesianProduct && existsCartesianProduct(logic) {
		return nil, errors.Trace(ErrCartesianProductUnsupported)
	}
	physical, err := physicalOptimize(logic)
	if err != nil {
		return nil, errors.Trace(err)
	}
	finalPlan := eliminatePhysicalProjection(physical)
	return finalPlan, nil
}
```


而在 logicalOptimize 中

```go
func logicalOptimize(flag uint64, logic LogicalPlan) (LogicalPlan, error) {
	var err error
	for i, rule := range optRuleList {
		// The order of flags is same as the order of optRule in the list.
		// We use a bitmask to record which opt rules should be used. If the i-th bit is 1, it means we should
		// apply i-th optimizing rule.
		if flag&(1<<uint(i)) == 0 {
			continue
		}
		logic, err = rule.optimize(logic)
		if err != nil {
			return nil, errors.Trace(err)
		}
	}
	return logic, errors.Trace(err)
}

var optRuleList = []logicalOptRule{
	&columnPruner{},
	&buildKeySolver{},
	&decorrelateSolver{},
	&aggregationEliminator{},
	&projectionEliminater{},
	&maxMinEliminator{},
	&ppdSolver{},
	&outerJoinEliminator{},
	&partitionProcessor{},
	&aggregationPushDownSolver{},
	&pushDownTopNOptimizer{},
}
```

那可以逻辑优化，即基于规则的优化主要指下面几个方面：

- 列裁剪
- 构建节点属性
- 去相关
- 消除聚合算子
- 消除投射
- 最大最小消除
- 谓词下推
- 消除 outer join
- 分区处理
- 聚合算子下推
- TopN 下推

## 谓词下推（perdicate push down）

首先会做一个简化，将左外连接和右外连接转化为内连接。

什么情况下外连接可以转内连接？

左向外连接的结果集包括左表的所有行，而不仅仅是连接列所匹配的行。如果左表的某行在右表中没有匹配的行，则在结果集右边补 NULL。做谓词下推时，如果我们知道接下来的的谓词条件一定会把包含 NULL 的行全部都过滤掉，那么做外连接就没意义了，可以直接改写成内连接。

什么情况会过滤掉 NULL 呢？
- 某个谓词的表达式用 NULL 计算后会得到 false；
- 或者多个谓词用 AND 条件连接，其中一个会过滤 NULL；
- 又或者用 OR 条件连接，其中每个都是过滤 NULL 的。

> 术语里面 OR 条件连接叫做析取范式 DNF (disjunctive normal form)。对应的还有合取范式 CNF (conjunctive normal form)。TiDB 的代码里面用到这种缩写。

能转成 inner join 的例子:

```sql
select * from t1 left outer join t2 on t1.id = t2.id where t2.id != null;
select * from t1 left outer join t2 on t1.id = t2.id where t2.id != null and t2.value > 3;
```

不能转成 inner join 的例子:
```sql
select * from t1 left outer join t2 on t1.id = t2.id where t2.id != null or t2.value > 3;
```

## 最大最小消除

最大最小消除，会对 Min/Max 语句进行改写。

```sql
select min(id) from t
```

我们用另一种写法，可以做到类似的效果：

```sql
select id from t order by id desc limit 1
```

这个写法有什么好处呢？前一个语句，生成的执行计划，是一个 TableScan 上面接一个 Aggregation，也就是说这是一个全表扫描的操作。后一个语句，生成执行计划是 TableScan + Sort + Limit。

在某些情况，比如 id 是主键或者是存在索引，数据本身有序， Sort 就可以消除，最终变成 TableScan 或者 IndexLookUp 接一个 Limit，这样子就不需要全表扫了，只需要读到第一条数据就得到结果！全表扫操作跟只查一条数据，性能上可是天壤之别。

最大最小消除，做的事情就是由 SQL 优化器“自动”地做这个变换。

```sql
select max(id) from t
```

生成的查询树会被转换成下面这种：

```sql
select max(id) from (select id from t order by id desc limit 1 where id is not null) t
```

这个变换复杂一些是要处理 NULL 的情况。经过这步变换之后，还会执行其它变换。所以中间生成的额外的算子，可能在其它变换里面被继续修改掉。

min 也是类似的语句替换。相应的代码是在 max_min_eliminate.go 文件里面。实现是一个纯粹的 AST 结构的修改。

# Reference
[TiDB 源码阅读系列文章（七）基于规则的优化](https://pingcap.com/blog-cn/tidb-source-code-reading-7/)
