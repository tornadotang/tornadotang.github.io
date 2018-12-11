---
title: (tidb) secondary index
category: tidb
draft: true
---

# sql 到 kv 到映射

假设表中有 3 行数据：

```
1, “TiDB”, “SQL Layer”, 10
2, “TiKV”, “KV Engine”, 20
3, “PD”, “Manager”, 30
```

每一行有3列数据，其中第一列是 PK, 第 4 列是 index，而 TiDB 底层是 kv 存储，怎么把关系型数据映射为 kv 呢

对于 PK 的查询，很简单，以 PK 作为 key，value 就是 [col1, col2, col3, col4]

每一行数据按照下面到方式编码

```
Key： tablePrefix_rowPrefix_tableID_rowID
Value: [col1, col2, col3, col4]
```

对于 index 数据，其中对于 unique 

类似于 mysql 的非聚集索引，先根据 secondary index 查到 pk，然后再根据 pk 查询到 row

```
Key: tablePrefix_idxPrefix_tableID_indexID_indexColumnsValue
Value: rowID
```

而对于非 unique, 为了保证 key 的唯一性，则需要把 rowID 加到 key 里，那么对于 unique 的 kv 查询的处理后面的章节会讲到

```
Key: tablePrefix_idxPrefix_tableID_indexID_ColumnsValue_rowID
Value：null
```

## 一个实际的例子

```
t_r_10_1 --> ["TiDB", "SQL Layer", 10]
t_r_10_2 --> ["TiKV", "KV Engine", 20]
t_r_10_3 --> ["PD", "Manager", 30]
```

而对于索引数据

```
t_i_10_1_10_1 --> null
t_i_10_1_20_2 --> null
t_i_10_1_30_3 --> null
```

## TiKV 实际的查询过程

因为 PK 具有唯一性，所以我们可以用 t + Table ID + PK 来唯一表示一行数据，value 就是这行数据。对于 Unique 来说，也是具有唯一性的，所以我们用 i + Index ID + name 来表示，而 value 则是对应的 PK。如果两个 name 相同，就会破坏唯一性约束。当我们使用 Unique 来查询的时候，会先找到对应的 PK，然后再通过 PK 找到对应的数据。

对于普通的 Index 来说，不需要唯一性约束，所以我们使用 i + Index ID + age + PK，而 value 为空。因为 PK 一定是唯一的，所以两行数据即使 age 一样，也不会冲突。当我们使用 Index 来查询的时候，会先 seek 到第一个大于等于 i + Index ID + age 这个 key 的数据，然后看前缀是否匹配，如果匹配，则解码出对应的 PK，再从 PK 拿到实际的数据。


# Reference

[TiKV 是如何存取数据到](https://pingcap.com/blog-cn/how-tikv-store-get-data/)

[三篇文章了解 TiDB 技术内幕 - 说计算](https://www.pingcap.com/blog-cn/tidb-internal-2/)
