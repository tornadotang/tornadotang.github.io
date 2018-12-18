---
title: (tidb) chunk - physical memory layout
category: tidb
draft: true
---


# usage

```go
	for {
		err := e.children[0].Next(ctx, chk)
		if err != nil {
			return errors.Trace(err)
		}

		if chk.NumRows() == 0 {
			break
		}

		for rowIdx := 0; rowIdx < chk.NumRows(); rowIdx++ {
			chunkRow := chk.GetRow(rowIdx)
			datumRow := chunkRow.GetDatumRow(fields)
			newRow, err1 := e.composeNewRow(globalRowIdx, datumRow, colsInfo)
			if err1 != nil {
				return errors.Trace(err1)
			}
			e.rows = append(e.rows, datumRow)
			e.newRowsData = append(e.newRowsData, newRow)
			globalRowIdx++
		}
		chk = chunk.Renew(chk, e.maxChunkSize)
	}
```

每次调用 children 的 Next 获取到 下一个 chk

然后先 GetRow

然后获取数据 GetDatumRow


# Reference

[Arrow: Physical memory layout](https://arrow.apache.org/docs/memory_layout.html)

[TiDB 源码阅读系列文章（十）Chunk 和执行框架简介](https://pingcap.com/blog-cn/tidb-source-code-reading-10/)
