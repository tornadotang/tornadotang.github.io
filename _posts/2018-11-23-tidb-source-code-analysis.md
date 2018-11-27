---
title: [tidb] source code analysis
category: tidb
draft: true
---


# 整体流程

data, err := cc.readPacket()

if err = cc.dispatch(data); err != nil {

func (cc *clientConn) handleQuery(goCtx goctx.Context, sql string) (err error) {

rs, err := cc.ctx.Execute(goCtx, sql)

func (tc *TiDBContext) Execute(goCtx goctx.Context, sql string) (rs []ResultSet, err error) { rsList, err := tc.session.Execute(goCtx, sql)

err = cc.writeResultset(goCtx, rs[0], false, false)

# Execute 中

    func (s *session) execute(ctx context.Context, sql string) (recordSets []sqlexec.RecordSet, err error) {
    	s.PrepareTxnCtx(ctx)
    	connID := s.sessionVars.ConnectionID
    	err = s.loadCommonGlobalVariablesIfNeeded()
    	if err != nil {
    		return nil, errors.Trace(err)
    	}
    
    	charsetInfo, collation := s.sessionVars.GetCharsetInfo()
    
    	// Step1: Compile query string to abstract syntax trees(ASTs).
    	startTS := time.Now()
    	stmtNodes, err := s.ParseSQL(ctx, sql, charsetInfo, collation)
    	if err != nil {
    		s.rollbackOnError(ctx)
    		log.Warnf("con:%d parse error:\n%v\n%s", connID, err, sql)
    		return nil, errors.Trace(err)
    	}
    	label := s.getSQLLabel()
    	metrics.SessionExecuteParseDuration.WithLabelValues(label).Observe(time.Since(startTS).Seconds())
    
    	compiler := executor.Compiler{Ctx: s}
    	for _, stmtNode := range stmtNodes {
    		s.PrepareTxnCtx(ctx)
    
    		// Step2: Transform abstract syntax tree to a physical plan(stored in executor.ExecStmt).
    		startTS = time.Now()
    		// Some executions are done in compile stage, so we reset them before compile.
    		if err := executor.ResetContextOfStmt(s, stmtNode); err != nil {
    			return nil, errors.Trace(err)
    		}
    		stmt, err := compiler.Compile(ctx, stmtNode)
    		if err != nil {
    			s.rollbackOnError(ctx)
    			log.Warnf("con:%d compile error:\n%v\n%s", connID, err, sql)
    			return nil, errors.Trace(err)
    		}
    		metrics.SessionExecuteCompileDuration.WithLabelValues(label).Observe(time.Since(startTS).Seconds())
    
    		// Step3: Execute the physical plan.
    		if recordSets, err = s.executeStatement(ctx, connID, stmtNode, stmt, recordSets); err != nil {
    			return nil, errors.Trace(err)
    		}
    	}

会先 parse，然后调用 compiler.Compile，然后调用 s.executeStatement，

## compile 函数

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

会先进行逻辑优化，然后进行物理优化

### 物理优化

物理优化主要指的是基于规则的优化 [rule based optimization](https://pingcap.com/blog-cn/tidb-source-code-reading-7/)

逻辑优化只要指的是基于代价的优化 [cost based optimization](https://pingcap.com/blog-cn/tidb-source-code-reading-8/)

## Execute 函数

而 execute 是按照 volcano 模型进行的

# 总结

![from tidb](/asserts/sql-life.png)
