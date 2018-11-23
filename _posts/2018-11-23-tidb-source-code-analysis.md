data, err := cc.readPacket()

if err = cc.dispatch(data); err != nil {

func (cc *clientConn) handleQuery(goCtx goctx.Context, sql string) (err error) {

rs, err := cc.ctx.Execute(goCtx, sql)

func (tc *TiDBContext) Execute(goCtx goctx.Context, sql string) (rs []ResultSet, err error) { rsList, err := tc.session.Execute(goCtx, sql)

err = cc.writeResultset(goCtx, rs[0], false, false)

return s.parser.Parse(sql, charset, collation)

executor/adpter.go 227: e, err := a.buildExecutor(ctx)
