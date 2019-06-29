本文简单介绍了PG INSERT语句的执行全过程,包括使用gdb跟踪调试的全过程,重点的数据结构等。

### 一、调用栈

INSERT语句的函数调用栈:

    
    
    (gdb) bt
    #0  PageAddItemExtended (page=0x7feaaefac300 "\001", item=0x29859f8 "2\234\030", size=61, offsetNumber=0, flags=2) at bufpage.c:196
    #1  0x00000000004cf4f9 in RelationPutHeapTuple (relation=0x7feac6e2ccb8, buffer=141, tuple=0x29859e0, token=false) at hio.c:53
    #2  0x00000000004c34ec in heap_insert (relation=0x7feac6e2ccb8, tup=0x29859e0, cid=0, options=0, bistate=0x0) at heapam.c:2487
    #3  0x00000000006c076b in ExecInsert (mtstate=0x2984c10, slot=0x2985250, planSlot=0x2985250, estate=0x29848c0, canSetTag=true) at nodeModifyTable.c:529
    #4  0x00000000006c29f3 in ExecModifyTable (pstate=0x2984c10) at nodeModifyTable.c:2126
    #5  0x000000000069a7d8 in ExecProcNodeFirst (node=0x2984c10) at execProcnode.c:445
    #6  0x0000000000690994 in ExecProcNode (node=0x2984c10) at ../../../src/include/executor/executor.h:237
    #7  0x0000000000692e5e in ExecutePlan (estate=0x29848c0, planstate=0x2984c10, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, 
        dest=0x2990dc8, execute_once=true) at execMain.c:1726
    #8  0x0000000000690e58 in standard_ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
    #9  0x0000000000690cef in ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:306
    #10 0x0000000000851d84 in ProcessQuery (plan=0x2990c68, sourceText=0x28c5ef0 "insert into t_insert values(9,'11','12','13');", params=0x0, queryEnv=0x0, dest=0x2990dc8, completionTag=0x7ffdbc052d10 "")
        at pquery.c:161
    #11 0x00000000008534f4 in PortalRunMulti (portal=0x292b490, isTopLevel=true, setHoldSnapshot=false, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:1286
    #12 0x0000000000852b32 in PortalRun (portal=0x292b490, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:799
    #13 0x000000000084cebc in exec_simple_query (query_string=0x28c5ef0 "insert into t_insert values(9,'11','12','13');") at postgres.c:1122
    #14 0x0000000000850f3c in PostgresMain (argc=1, argv=0x28efaa8, dbname=0x28ef990 "testdb", username=0x28ef978 "xdb") at postgres.c:4153
    #15 0x00000000007c0168 in BackendRun (port=0x28e7970) at postmaster.c:4361
    #16 0x00000000007bf8fc in BackendStartup (port=0x28e7970) at postmaster.c:4033
    #17 0x00000000007bc139 in ServerLoop () at postmaster.c:1706
    #18 0x00000000007bb9f9 in PostmasterMain (argc=1, argv=0x28c0b60) at postmaster.c:1379
    #19 0x00000000006f19e8 in main (argc=1, argv=0x28c0b60) at main.c:228
    

### 二、跟踪分析

插入测试数据：

    
    
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1610
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(25,'insert','insert','insert');
    (挂起)
    

启动gdb，跟踪调试：

    
    
    [root@localhost ~]# gdb -p 3294
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    #设置断点
    (gdb) b PostgresMain 
    Breakpoint 1 at 0x8507bb: file postgres.c, line 3631.
    (gdb) b exec_simple_query 
    Breakpoint 2 at 0x84cad8: file postgres.c, line 893.
    (gdb) b PortalRunMulti 
    Breakpoint 3 at 0x8533df: file pquery.c, line 1210.
    …
    (gdb) c
    Continuing.
    
    Breakpoint 2, exec_simple_query (
        query_string=0x219cef0 "insert into t_insert values(25,'insert','insert','insert');") at postgres.c:893
    893     CommandDest dest = whereToSendOutput;
    #PostmasterMain 的调试需要使用fork process的方式进行,不是重点,暂时不作介绍
    #进入exec_simple_query
    #输入参数:query_string=0x219cef0 "insert into t_insert values(25,'insert','insert','insert');"
    #生成解析列表,解析列表parsetree_list中的元素类型为RawStmt
    #INSERT语句,RawStmt的NodeType=T_InsertStmt
    (gdb) 
    944     parsetree_list = pg_parse_query(query_string);
    (gdb) p *parsetree_list
    $11 = {type = T_List, length = 1, head = 0x219dce0, tail = 0x219dce0}
    (gdb) p *(RawStmt *)(parsetree_list->head->data.ptr_value)
    $12 = {type = T_RawStmt, stmt = 0x219dc60, stmt_location = 0, stmt_len = 58}
    (gdb) p *((RawStmt *)(parsetree_list->head->data.ptr_value))->stmt
    $13 = {type = T_InsertStmt}
    #生成查询列表querytree_list,其中的元素为Query
    (gdb) 
    1047            querytree_list = pg_analyze_and_rewrite(parsetree, query_string,
    (gdb) p *(Query *)(querytree_list->head->data.ptr_value)
    $14 = {type = T_Query, commandType = CMD_INSERT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 1, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x219e260, jointree = 0x21c2a50, 
      targetList = 0x21c2ad0, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 58}
    (gdb) p *querytree_list
    $15 = {type = T_List, length = 1, head = 0x21c2b80, tail = 0x21c2b80}
    #查看Query中的元素
    #rtable,List中的实际类型为RangeTblEntry,可以理解为实际的数据表
    (gdb) p *((Query *)(querytree_list->head->data.ptr_value))->rtable
    $16 = {type = T_List, length = 1, head = 0x219e240, tail = 0x219e240}
    (gdb) p *(RangeTblEntry *)((Query *)(querytree_list->head->data.ptr_value))->rtable->head->data.ptr_value
    $17 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26731, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x219e060, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 1, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x21c28e8, updatedCols = 0x0, 
      securityQuals = 0x0}
    #jointree为NULL
    (gdb)  p *((Query *)(querytree_list->head->data.ptr_value))->jointree
    $18 = {type = T_FromExpr, fromlist = 0x0, quals = 0x0}
    #targetList是输出的目标Entry(数据列)
    (gdb) p *(TargetEntry *)((Query *)(querytree_list->head->data.ptr_value))->targetList->head->data.ptr_value
    $20 = {xpr = {type = T_TargetEntry}, expr = 0x219e590, resno = 1, resname = 0x219e2e0 "id", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    (gdb) n
    1054            if (snapshot_set)
    #生成PlannedStmt列表
    (gdb) p *plantree_list
    $21 = {type = T_List, length = 1, head = 0x225e488, tail = 0x225e488}
    (gdb)  p *(PlannedStmt *)(plantree_list->head->data.ptr_value)
    $22 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, 
      hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x225dfe8, rtable = 0x225e2a8, 
      resultRelations = 0x225e348, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x225e2f8, invalItems = 0x0, 
      paramExecTypes = 0x21c4320, utilityStmt = 0x0, stmt_location = 0, stmt_len = 58}
    #查看planTree(执行计划树)
    (gdb)  p *((PlannedStmt *)(plantree_list->head->data.ptr_value))->planTree
    $23 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, 
      parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    ...
    #继续执行,进入PortalRunMulti
    (gdb) c
    Continuing.
    
    Breakpoint 3, PortalRunMulti (portal=0x2202490, isTopLevel=true, setHoldSnapshot=false, dest=0x225e4d8, 
        altdest=0x225e4d8, completionTag=0x7ffd6452f100 "") at pquery.c:1210
    1210        bool        active_snapshot_set = false;
    #portal参数,其中stmts是已生成的PlannedStmt
    (gdb) p *portal
    $25 = {name = 0x2205e98 "", prepStmtName = 0x0, portalContext = 0x2252460, resowner = 0x21cdd10, 
      cleanup = 0x62f15c <PortalCleanup>, createSubid = 1, activeSubid = 1, 
      sourceText = 0x219cef0 "insert into t_insert values(25,'insert','insert','insert');", 
      commandTag = 0xb50908 "INSERT", stmts = 0x225e4a8, cplan = 0x0, portalParams = 0x0, queryEnv = 0x0, 
      strategy = PORTAL_MULTI_QUERY, cursorOptions = 4, run_once = true, status = PORTAL_ACTIVE, 
      portalPinned = false, autoHeld = false, queryDesc = 0x0, tupDesc = 0x0, formats = 0x0, holdStore = 0x0, 
      holdContext = 0x0, holdSnapshot = 0x0, atStart = true, atEnd = true, portalPos = 0, 
      creation_time = 587211499048183, visible = false}
    (gdb) p *(portal->stmts)
    $26 = {type = T_List, length = 1, head = 0x225e488, tail = 0x225e488}
    (gdb) p *(PlannedStmt *)(portal->stmts->head.data->ptr_value)
    $27 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, 
      hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x225dfe8, rtable = 0x225e2a8, 
      resultRelations = 0x225e348, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x225e2f8, invalItems = 0x0, 
      paramExecTypes = 0x21c4320, utilityStmt = 0x0, stmt_location = 0, stmt_len = 58}
    #进入ProcessQuery
    (gdb) c
    Continuing.
    
    Breakpoint 4, ProcessQuery (plan=0x225e378, 
        sourceText=0x219cef0 "insert into t_insert values(26,'insert','insert','insert');", params=0x0, 
        queryEnv=0x0, dest=0x225e4d8, completionTag=0x7ffd6452f100 "") at pquery.c:149
    149     queryDesc = CreateQueryDesc(plan, sourceText,
    #输入参数,plan=PlannedStmt变量
    (gdb) p *plan
    $29 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, 
      hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x225dfe8, rtable = 0x225e2a8, 
      resultRelations = 0x225e348, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x225e2f8, invalItems = 0x0, 
      paramExecTypes = 0x21c4320, utilityStmt = 0x0, stmt_location = 0, stmt_len = 58}
    #QueryDesc查询描述符结构体,plannedstmt为先前生成的PlannedStmt
    (gdb) p *queryDesc
    $30 = {operation = CMD_INSERT, plannedstmt = 0x225e378, 
      sourceText = 0x219cef0 "insert into t_insert values(26,'insert','insert','insert');", 
      snapshot = 0x21c0920, crosscheck_snapshot = 0x0, dest = 0x225e4d8, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x0, estate = 0x0, planstate = 0x0, already_executed = false, 
      totaltime = 0x0}
    #进入standard_ExecutorRun,queryDesc为查询描述符
    (gdb) c
    Continuing.
    
    Breakpoint 5, standard_ExecutorRun (queryDesc=0x2252570, direction=ForwardScanDirection, count=0, 
        execute_once=true) at execMain.c:322
    322     estate = queryDesc->estate;
    #ProcessQuery函数构建了PlanState/EState(ExecutorState)/tupDesc,为Execute作准备
    (gdb) p *queryDesc
    $31 = {operation = CMD_INSERT, plannedstmt = 0x225e378, 
      sourceText = 0x219cef0 "insert into t_insert values(26,'insert','insert','insert');", 
      snapshot = 0x21c0920, crosscheck_snapshot = 0x0, dest = 0x225e4d8, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x2254d50, estate = 0x2253c80, planstate = 0x2253fd0, 
      already_executed = false, totaltime = 0x0}
    (gdb) p *(queryDesc->estate)
    $32 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x21c0920, 
      es_crosscheck_snapshot = 0x0, es_range_table = 0x225e2a8, es_plannedstmt = 0x225e378, 
      es_sourceText = 0x219cef0 "insert into t_insert values(26,'insert','insert','insert');", 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x2253ec0, es_num_result_relations = 1, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x2254e30, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, 
      es_param_exec_vals = 0x2253e90, es_queryEnv = 0x0, es_query_cxt = 0x2253b70, es_tupleTable = 0x2254880, 
      es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, 
      es_finished = false, es_exprcontexts = 0x2254230, es_subplanstates = 0x0, es_auxmodifytables = 0x0, 
      es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0}
    (gdb) p *(queryDesc->planstate)
    $33 = {type = T_ModifyTableState, plan = 0x225dfe8, state = 0x2253c80, 
      ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, 
      instrument = 0x0, worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, 
      subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2254d80, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    (gdb) p *(queryDesc->tupDesc)
    $34 = {natts = 0, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, 
      attrs = 0x2254d70}
    #进入ExecutePlan
    (gdb) c
    Continuing.
    
    Breakpoint 6, ExecutePlan (estate=0x2253c80, planstate=0x2253fd0, use_parallel_mode=false, 
        operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, dest=0x225e4d8, 
        execute_once=true) at execMain.c:1697
    1697        current_tuple_count = 0;
    #输入参数,EState/PlanState,准备执行SQL
    (gdb) p *estate
    $35 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x21c0920, 
      es_crosscheck_snapshot = 0x0, es_range_table = 0x225e2a8, es_plannedstmt = 0x225e378, 
      es_sourceText = 0x219cef0 "insert into t_insert values(26,'insert','insert','insert');", 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x2253ec0, es_num_result_relations = 1, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x2254e30, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, 
      es_param_exec_vals = 0x2253e90, es_queryEnv = 0x0, es_query_cxt = 0x2253b70, es_tupleTable = 0x2254880, 
      es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, 
      es_finished = false, es_exprcontexts = 0x2254230, es_subplanstates = 0x0, es_auxmodifytables = 0x0, 
      es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0}
    (gdb) p *planstate
    $36 = {type = T_ModifyTableState, plan = 0x225dfe8, state = 0x2253c80, 
      ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, 
      instrument = 0x0, worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, 
      subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2254d80, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    #在ExecutePlan函数中执行ExecProcNode,直至ExecProcNode函数返回的TupleTableSlot为空才退出循环
    1726            slot = ExecProcNode(planstate);
    (gdb) 
    #进入ExecProcNode
    #执行PlanState的ExecProcNode函数,即ExecProcNodeFirst
    Breakpoint 7, ExecProcNode (node=0x2253fd0) at ../../../src/include/executor/executor.h:234
    234     if (node->chgParam != NULL) /* something changed? */
    (gdb) 
    237     return node->ExecProcNode(node);
    (gdb) 
    
    #进入ExecProcNodeFirst
    Breakpoint 8, ExecProcNodeFirst (node=0x2253fd0) at execProcnode.c:433
    433     check_stack_depth();
    #执行node的ExecProcNode函数,即ExecModifyTable(注:PlanState中的ExecProcNodeReal)
    (gdb) p *node
    $37 = {type = T_ModifyTableState, plan = 0x225dfe8, state = 0x2253c80, 
      ExecProcNode = 0x6c2485 <ExecModifyTable>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, 
      instrument = 0x0, worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, 
      subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2254d80, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    ...
    #在for循环中执行subplanstate的ExecProcNode,同样的,如果返回的Slot为NULL则退出
    (gdb) p subplanstate
    $38 = (PlanState *) 0x22543a0
    (gdb) p *subplanstate #node->mt_plans[node->mt_whichplan];//执行计划的State
    $39 = {type = T_ResultState, plan = 0x21c44e0, state = 0x2253c80, 
      ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, 
      worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, 
      chgParam = 0x0, ps_ResultTupleSlot = 0x2254750, ps_ExprContext = 0x22544b0, ps_ProjInfo = 0x22548b0, 
      scandesc = 0x0}
    (gdb) p *node
    $40 = {ps = {type = T_ModifyTableState, plan = 0x225dfe8, state = 0x2253c80, 
        ExecProcNode = 0x6c2485 <ExecModifyTable>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, 
        instrument = 0x0, worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, 
        subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2254d80, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
        scandesc = 0x0}, operation = CMD_INSERT, canSetTag = true, mt_done = false, mt_plans = 0x22541e0, 
      mt_nplans = 1, mt_whichplan = 0, resultRelInfo = 0x2253ec0, rootResultRelInfo = 0x0, 
      mt_arowmarks = 0x22541f8, mt_epqstate = {estate = 0x0, planstate = 0x0, origslot = 0x0, plan = 0x21c44e0, 
        arowMarks = 0x0, epqParam = 0}, fireBSTriggers = false, mt_existing = 0x0, mt_excludedtlist = 0x0, 
      mt_conflproj = 0x0, mt_partition_tuple_routing = 0x0, mt_transition_capture = 0x0, 
      mt_oc_transition_capture = 0x0, mt_per_subplan_tupconv_maps = 0x0}
    (gdb) n
    1974        saved_resultRelInfo = estate->es_result_relation_info;
    (gdb) 
    1976        estate->es_result_relation_info = resultRelInfo;
    (gdb) 
    1990            ResetPerTupleExprContext(estate);
    (gdb) 
    1992            planSlot = ExecProcNode(subplanstate);
    (gdb) 
    
    Breakpoint 7, ExecProcNode (node=0x22543a0) at ../../../src/include/executor/executor.h:234
    234     if (node->chgParam != NULL) /* something changed? */
    (gdb) 
    237     return node->ExecProcNode(node);
    (gdb) 
    #再次进入ExecProcNodeFirst,Node变为T_ResultState
    Breakpoint 8, ExecProcNodeFirst (node=0x22543a0) at execProcnode.c:433
    433     check_stack_depth();
    (gdb) 
    440     if (node->instrument)
    (gdb) 
    443         node->ExecProcNode = node->ExecProcNodeReal;
    (gdb) 
    445     return node->ExecProcNode(node);
    (gdb) p *node
    $41 = {type = T_ResultState, plan = 0x21c44e0, state = 0x2253c80, ExecProcNode = 0x6c5094 <ExecResult>, 
      ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, worker_instrument = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2254750, ps_ExprContext = 0x22544b0, ps_ProjInfo = 0x22548b0, scandesc = 0x0}
    (gdb) 
    #进入ExecInsert
    2126                    slot = ExecInsert(node, slot, planSlot,
    (gdb) 
    
    Breakpoint 10, ExecInsert (mtstate=0x2253fd0, slot=0x2254750, planSlot=0x2254750, estate=0x2253c80, 
        canSetTag=true) at nodeModifyTable.c:273
    273     List       *recheckIndexes = NIL;
    #调用heap_insert等函数插入数据
    (gdb) c
    Continuing.
    
    Breakpoint 11, heap_insert (relation=0x7f1a5655b378, tup=0x2254ee0, cid=0, options=0, bistate=0x0)
        at heapam.c:2444
    2444        TransactionId xid = GetCurrentTransactionId();
    (gdb) c
    Continuing.
    
    Breakpoint 12, RelationPutHeapTuple (relation=0x7f1a5655b378, buffer=95, tuple=0x2254ee0, token=false)
        at hio.c:51
    51      pageHeader = BufferGetPage(buffer);
    (gdb) c
    Continuing.
    
    Breakpoint 13, PageAddItemExtended (page=0x7f1a3e6a3300 "\001", item=0x2254ef8 "g\234\030", size=49, 
        offsetNumber=0, flags=2) at bufpage.c:196
    196     PageHeader  phdr = (PageHeader) page;
    (gdb) finish
    Run till exit from #0  PageAddItemExtended (page=0x7f1a3e6a3300 "\001", item=0x2254ef8 "g\234\030", size=49, 
        offsetNumber=0, flags=2) at bufpage.c:196
    0x00000000004cf4f9 in RelationPutHeapTuple (relation=0x7f1a5655b378, buffer=95, tuple=0x2254ee0, token=false)
        at hio.c:53
    53      offnum = PageAddItem(pageHeader, (Item) tuple->t_data,
    Value returned is $42 = 41
    ...
    #再次执行ExecProcNode(实质为ExecResult),返回NULL,退出循环,结束
    (gdb) c
    Continuing.
    
    Breakpoint 7, ExecProcNode (node=0x22543a0) at ../../../src/include/executor/executor.h:234
    234     if (node->chgParam != NULL) /* something changed? */
    (gdb) p *node
    $48 = {type = T_ResultState, plan = 0x21c44e0, state = 0x2253c80, ExecProcNode = 0x6c5094 <ExecResult>, 
      ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, worker_instrument = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2254750, ps_ExprContext = 0x22544b0, ps_ProjInfo = 0x22548b0, scandesc = 0x0}
    (gdb) c
    Continuing.
    
    

### 三、小结

1.执行过程:使用gdb跟踪INSERT语句执行的全过程;  
2.数据结构:如PlannedStmt/PlanState/EState等  
3."多态":ExecProcNode函数

