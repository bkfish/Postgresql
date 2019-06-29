本文简单介绍了PG插入数据部分的源码，主要内容包括ExecutorRun函数和standard_ExecutorRun函数的实现逻辑，这两个函数均位于execMain.c文件中。  
**_值得一提的是：  
1、解读方式：采用自底向上的方式，也就是从调用栈（调用栈请参加第一篇文章）的底层往上逐层解读，建议按此顺序阅读；  
2、问题处理：上面几篇解读并不深入，或者说只是浮于表面，但随着调用栈的逐步解读，信息会慢慢浮现，需要耐心和坚持_**

### 一、基础信息

ExecutorRun、standard_ExecutorRun函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、QueryDesc_

    
    
    //查询结构体
    //结构体中包含了执行查询所需要的所有信息
     /* ----------------      *      query descriptor:
      *
      *  a QueryDesc encapsulates everything that the executor
      *  needs to execute the query.
      *
      *  For the convenience of SQL-language functions, we also support QueryDescs
      *  containing utility statements; these must not be passed to the executor
      *  however.
      * ---------------------      */
     typedef struct QueryDesc
     {
         /* These fields are provided by CreateQueryDesc */
         CmdType     operation;      /* CMD_SELECT, CMD_UPDATE, etc. */
         PlannedStmt *plannedstmt;   /* planner's output (could be utility, too) */
         const char *sourceText;     /* source text of the query */
         Snapshot    snapshot;       /* snapshot to use for query */
         Snapshot    crosscheck_snapshot;    /* crosscheck for RI update/delete */
         DestReceiver *dest;         /* the destination for tuple output */
         ParamListInfo params;       /* param values being passed in */
         QueryEnvironment *queryEnv; /* query environment passed in */
         int         instrument_options; /* OR of InstrumentOption flags */
     
         /* These fields are set by ExecutorStart */
         TupleDesc   tupDesc;        /* descriptor for result tuples */
         EState     *estate;         /* executor's query-wide state */
         PlanState  *planstate;      /* tree of per-plan-node state */
     
         /* This field is set by ExecutorRun */
         bool        already_executed;   /* true if previously executed */
     
         /* This is always set NULL by the core system, but plugins can change it */
         struct Instrumentation *totaltime;  /* total time spent in ExecutorRun */
     } QueryDesc;
     
    //快照指针
    typedef struct SnapshotData *Snapshot;
     
     #define InvalidSnapshot     ((Snapshot) NULL)
     
     /*
      * We use SnapshotData structures to represent both "regular" (MVCC)
      * snapshots and "special" snapshots that have non-MVCC semantics.
      * The specific semantics of a snapshot are encoded by the "satisfies"
      * function.
      */
     typedef bool (*SnapshotSatisfiesFunc) (HeapTuple htup,
                                            Snapshot snapshot, Buffer buffer);
     
     /*
      * Struct representing all kind of possible snapshots.
      *
      * There are several different kinds of snapshots:
      * * Normal MVCC snapshots
      * * MVCC snapshots taken during recovery (in Hot-Standby mode)
      * * Historic MVCC snapshots used during logical decoding
      * * snapshots passed to HeapTupleSatisfiesDirty()
      * * snapshots passed to HeapTupleSatisfiesNonVacuumable()
      * * snapshots used for SatisfiesAny, Toast, Self where no members are
      *   accessed.
      *
      * TODO: It's probably a good idea to split this struct using a NodeTag
      * similar to how parser and executor nodes are handled, with one type for
      * each different kind of snapshot to avoid overloading the meaning of
      * individual fields.
      */
     typedef struct SnapshotData
     {
         SnapshotSatisfiesFunc satisfies;    /* tuple test function */
     
         /*
          * The remaining fields are used only for MVCC snapshots, and are normally
          * just zeroes in special snapshots.  (But xmin and xmax are used
          * specially by HeapTupleSatisfiesDirty, and xmin is used specially by
          * HeapTupleSatisfiesNonVacuumable.)
          *
          * An MVCC snapshot can never see the effects of XIDs >= xmax. It can see
          * the effects of all older XIDs except those listed in the snapshot. xmin
          * is stored as an optimization to avoid needing to search the XID arrays
          * for most tuples.
          */
         TransactionId xmin;         /* all XID < xmin are visible to me */
         TransactionId xmax;         /* all XID >= xmax are invisible to me */
     
         /*
          * For normal MVCC snapshot this contains the all xact IDs that are in
          * progress, unless the snapshot was taken during recovery in which case
          * it's empty. For historic MVCC snapshots, the meaning is inverted, i.e.
          * it contains *committed* transactions between xmin and xmax.
          *
          * note: all ids in xip[] satisfy xmin <= xip[i] < xmax
          */
         TransactionId *xip;
         uint32      xcnt;           /* # of xact ids in xip[] */
     
         /*
          * For non-historic MVCC snapshots, this contains subxact IDs that are in
          * progress (and other transactions that are in progress if taken during
          * recovery). For historic snapshot it contains *all* xids assigned to the
          * replayed transaction, including the toplevel xid.
          *
          * note: all ids in subxip[] are >= xmin, but we don't bother filtering
          * out any that are >= xmax
          */
         TransactionId *subxip;
         int32       subxcnt;        /* # of xact ids in subxip[] */
         bool        suboverflowed;  /* has the subxip array overflowed? */
     
         bool        takenDuringRecovery;    /* recovery-shaped snapshot? */
         bool        copied;         /* false if it's a static snapshot */
     
         CommandId   curcid;         /* in my xact, CID < curcid are visible */
     
         /*
          * An extra return value for HeapTupleSatisfiesDirty, not used in MVCC
          * snapshots.
          */
         uint32      speculativeToken;
     
         /*
          * Book-keeping information, used by the snapshot manager
          */
         uint32      active_count;   /* refcount on ActiveSnapshot stack */
         uint32      regd_count;     /* refcount on RegisteredSnapshots */
         pairingheap_node ph_node;   /* link in the RegisteredSnapshots heap */
     
         TimestampTz whenTaken;      /* timestamp when snapshot was taken */
         XLogRecPtr  lsn;            /* position in the WAL stream when taken */
     } SnapshotData;//存储快照的数据结构
    
    
     /* ----------------      *      PlannedStmt node
      *
      * The output of the planner is a Plan tree headed by a PlannedStmt node.
      * PlannedStmt holds the "one time" information needed by the executor.
      *
      * For simplicity in APIs, we also wrap utility statements in PlannedStmt
      * nodes; in such cases, commandType == CMD_UTILITY, the statement itself
      * is in the utilityStmt field, and the rest of the struct is mostly dummy.
      * (We do use canSetTag, stmt_location, stmt_len, and possibly queryId.)
      * ----------------      */
    //已Planned的Statement
    //也就是说已生成了执行计划的语句
     typedef struct PlannedStmt
     {
         NodeTag     type;
     
         CmdType     commandType;    /* select|insert|update|delete|utility */
     
         uint64      queryId;        /* query identifier (copied from Query) */
     
         bool        hasReturning;   /* is it insert|update|delete RETURNING? */
     
         bool        hasModifyingCTE;    /* has insert|update|delete in WITH? */
     
         bool        canSetTag;      /* do I set the command result tag? */
     
         bool        transientPlan;  /* redo plan when TransactionXmin changes? */
     
         bool        dependsOnRole;  /* is plan specific to current role? */
     
         bool        parallelModeNeeded; /* parallel mode required to execute? */
     
         int         jitFlags;       /* which forms of JIT should be performed */
     
         struct Plan *planTree;      /* tree of Plan nodes */
     
         List       *rtable;         /* list of RangeTblEntry nodes */
     
         /* rtable indexes of target relations for INSERT/UPDATE/DELETE */
         List       *resultRelations;    /* integer list of RT indexes, or NIL */
     
         /*
          * rtable indexes of non-leaf target relations for UPDATE/DELETE on all
          * the partitioned tables mentioned in the query.
          */
         List       *nonleafResultRelations;
     
         /*
          * rtable indexes of root target relations for UPDATE/DELETE; this list
          * maintains a subset of the RT indexes in nonleafResultRelations,
          * indicating the roots of the respective partition hierarchies.
          */
         List       *rootResultRelations;
     
         List       *subplans;       /* Plan trees for SubPlan expressions; note
                                      * that some could be NULL */
     
         Bitmapset  *rewindPlanIDs;  /* indices of subplans that require REWIND */
     
         List       *rowMarks;       /* a list of PlanRowMark's */
     
         List       *relationOids;   /* OIDs of relations the plan depends on */
     
         List       *invalItems;     /* other dependencies, as PlanInvalItems */
     
         List       *paramExecTypes; /* type OIDs for PARAM_EXEC Params */
     
         Node       *utilityStmt;    /* non-null if this is utility stmt */
     
         /* statement location in source string (copied from Query) */
         int         stmt_location;  /* start location, or -1 if unknown */
         int         stmt_len;       /* length in bytes; 0 means "rest of string" */
     } PlannedStmt;
     
     
    //参数列表信息
     typedef struct ParamListInfoData
     {
         ParamFetchHook paramFetch;  /* parameter fetch hook */
         void       *paramFetchArg;
         ParamCompileHook paramCompile;  /* parameter compile hook */
         void       *paramCompileArg;
         ParserSetupHook parserSetup;    /* parser setup hook */
         void       *parserSetupArg;
         int         numParams;      /* nominal/maximum # of Params represented */
     
         /*
          * params[] may be of length zero if paramFetch is supplied; otherwise it
          * must be of length numParams.
          */
         ParamExternData params[FLEXIBLE_ARRAY_MEMBER];
     }           ParamListInfoData;
     
    typedef struct ParamListInfoData *ParamListInfo;
    
    //查询环境，使用List存储相关信息
    /*
      * Private state of a query environment.
      */
     struct QueryEnvironment
     {
         List       *namedRelList;
     };
     
     
    //TODO
     typedef struct Instrumentation
     {
         /* Parameters set at node creation: */
         bool        need_timer;     /* true if we need timer data */
         bool        need_bufusage;  /* true if we need buffer usage data */
         /* Info about current plan cycle: */
         bool        running;        /* true if we've completed first tuple */
         instr_time  starttime;      /* Start time of current iteration of node */
         instr_time  counter;        /* Accumulated runtime for this node */
         double      firsttuple;     /* Time for first tuple of this cycle */
         double      tuplecount;     /* Tuples emitted so far this cycle */
         BufferUsage bufusage_start; /* Buffer usage at start */
         /* Accumulated statistics across all completed cycles: */
         double      startup;        /* Total startup time (in seconds) */
         double      total;          /* Total total time (in seconds) */
         double      ntuples;        /* Total tuples produced */
         double      ntuples2;       /* Secondary node-specific tuple counter */
         double      nloops;         /* # of run cycles for this node */
         double      nfiltered1;     /* # tuples removed by scanqual or joinqual */
         double      nfiltered2;     /* # tuples removed by "other" quals */
         BufferUsage bufusage;       /* Total buffer usage */
     } Instrumentation;
     
    

**依赖的函数**  
_1、InstrStartNode_

    
    
     /* Entry to a plan node */
     void
     InstrStartNode(Instrumentation *instr)
     {
         if (instr->need_timer)
         {
             if (INSTR_TIME_IS_ZERO(instr->starttime))
                 INSTR_TIME_SET_CURRENT(instr->starttime);
             else
                 elog(ERROR, "InstrStartNode called twice in a row");
         }
     
         /* save buffer usage totals at node entry, if needed */
         if (instr->need_bufusage)
             instr->bufusage_start = pgBufferUsage;
     }
     
    

_2、ScanDirectionIsNoMovement_

    
    
    //简单判断
     /*
      * ScanDirectionIsNoMovement
      *      True iff scan direction indicates no movement.
      */
     #define ScanDirectionIsNoMovement(direction) \
         ((bool) ((direction) == NoMovementScanDirection))
    

_3、ExecutePlan_

    
    
    //上一节已解读
    

_4、InstrStopNode_  
//TODO Instrumentation 的理解

    
    
     /* Exit from a plan node */
     void
     InstrStopNode(Instrumentation *instr, double nTuples)
     {
         instr_time  endtime;
     
         /* count the returned tuples */
         instr->tuplecount += nTuples;
     
         /* let's update the time only if the timer was requested */
         if (instr->need_timer)
         {
             if (INSTR_TIME_IS_ZERO(instr->starttime))
                 elog(ERROR, "InstrStopNode called without start");
     
             INSTR_TIME_SET_CURRENT(endtime);
             INSTR_TIME_ACCUM_DIFF(instr->counter, endtime, instr->starttime);
     
             INSTR_TIME_SET_ZERO(instr->starttime);
         }
     
         /* Add delta of buffer usage since entry to node's totals */
         if (instr->need_bufusage)
             BufferUsageAccumDiff(&instr->bufusage,
                                  &pgBufferUsage, &instr->bufusage_start);
     
         /* Is this the first tuple of this cycle? */
         if (!instr->running)
         {
             instr->running = true;
             instr->firsttuple = INSTR_TIME_GET_DOUBLE(instr->counter);
         }
     }
    

_5、MemoryContextSwitchTo_

    
    
    /*
      * Although this header file is nominally backend-only, certain frontend
      * programs like pg_controldata include it via postgres.h.  For some compilers
      * it's necessary to hide the inline definition of MemoryContextSwitchTo in
      * this scenario; hence the #ifndef FRONTEND.
      */ 
     #ifndef FRONTEND
     static inline MemoryContext
     MemoryContextSwitchTo(MemoryContext context)
     {
         MemoryContext old = CurrentMemoryContext;
     
         CurrentMemoryContext = context;
         return old;
     }
     #endif                          /* FRONTEND */
    

### 二、源码解读

    
    
    /* ----------------------------------------------------------------     *      ExecutorRun
     *
     *      This is the main routine of the executor module. It accepts
     *      the query descriptor from the traffic cop and executes the
     *      query plan.
     *
     *      ExecutorStart must have been called already.
     *
     *      If direction is NoMovementScanDirection then nothing is done
     *      except to start up/shut down the destination.  Otherwise,
     *      we retrieve up to 'count' tuples in the specified direction.
     *
     *      Note: count = 0 is interpreted as no portal limit, i.e., run to
     *      completion.  Also note that the count limit is only applied to
     *      retrieved tuples, not for instance to those inserted/updated/deleted
     *      by a ModifyTable plan node.
     *
     *      There is no return value, but output tuples (if any) are sent to
     *      the destination receiver specified in the QueryDesc; and the number
     *      of tuples processed at the top level can be found in
     *      estate->es_processed.
     *
     *      We provide a function hook variable that lets loadable plugins
     *      get control when ExecutorRun is called.  Such a plugin would
     *      normally call standard_ExecutorRun().
     *
     * ----------------------------------------------------------------     */
    /*
    输入：
        queryDesc-查询描述符，实际是需要执行的SQL语句的相关信息
        direction-扫描方向
        count-计数器
        execute_once-执行一次？
    输出：
    */
    void
    ExecutorRun(QueryDesc *queryDesc,
                ScanDirection direction, uint64 count,
                bool execute_once)
    {
        if (ExecutorRun_hook)//如果有钩子函数，则执行钩子函数
            (*ExecutorRun_hook) (queryDesc, direction, count, execute_once);
        else//否则执行标准函数
            standard_ExecutorRun(queryDesc, direction, count, execute_once);
    }
    
    //标准函数
    /*
    输入&输出：参见ExecutorRun
    */
    void
    standard_ExecutorRun(QueryDesc *queryDesc,
                         ScanDirection direction, uint64 count, bool execute_once)
    {
        EState     *estate;//执行器状态信息
        CmdType     operation;//命令类型，这里是INSERT
        DestReceiver *dest;//目标接收器
        bool        sendTuples;//是否需要传输Tuples
        MemoryContext oldcontext;//原内存上下文（PG自己的内存管理器）
    
        /* sanity checks */
        Assert(queryDesc != NULL);
    
        estate = queryDesc->estate;//获取执行器状态
    
        Assert(estate != NULL);
        Assert(!(estate->es_top_eflags & EXEC_FLAG_EXPLAIN_ONLY));
    
        /*
         * Switch into per-query memory context
         */
        oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);//切换至当前查询上下文，切换前保存原上下文
    
        /* Allow instrumentation of Executor overall runtime */
        if (queryDesc->totaltime)//需要计时？如Oracle在sqlplus中设置set timing on的计时
            InstrStartNode(queryDesc->totaltime);//
    
        /*
         * extract information from the query descriptor and the query feature.
         */
        operation = queryDesc->operation;//操作类型
        dest = queryDesc->dest;//目标端
    
        /*
         * startup tuple receiver, if we will be emitting tuples
         */
        estate->es_processed = 0;//进度
        estate->es_lastoid = InvalidOid;//最后一个Oid
    
        sendTuples = (operation == CMD_SELECT ||
                      queryDesc->plannedstmt->hasReturning);//查询语句或者需要返回值的才需要传输Tuples
    
        if (sendTuples)
            dest->rStartup(dest, operation, queryDesc->tupDesc);//启动目标端的接收器
    
        /*
         * run plan
         */
        if (!ScanDirectionIsNoMovement(direction))//需要扫描
        {
            if (execute_once && queryDesc->already_executed)
                elog(ERROR, "can't re-execute query flagged for single execution");
            queryDesc->already_executed = true;
    
            ExecutePlan(estate,
                        queryDesc->planstate,
                        queryDesc->plannedstmt->parallelModeNeeded,
                        operation,
                        sendTuples,
                        count,
                        direction,
                        dest,
                        execute_once);//执行
        }
    
        /*
         * shutdown tuple receiver, if we started it
         */
        if (sendTuples)
            dest->rShutdown(dest);//关闭目标端的接收器
    
        if (queryDesc->totaltime)
            InstrStopNode(queryDesc->totaltime, estate->es_processed);//完成计时
    
        MemoryContextSwitchTo(oldcontext);//执行完毕，切换回原内存上下文
    }
    
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- #8 ExecutorRun&standard_ExecutorRun
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1529
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(16,'ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun');
    (挂起)
    

启动gdb，跟踪调试：

    
    
    [root@localhost ~]# gdb -p 3294
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b standard_ExecutorRun
    Breakpoint 1 at 0x690d09: file execMain.c, line 322.
    (gdb) c
    Continuing.
    
    Breakpoint 1, standard_ExecutorRun (queryDesc=0x2c2d4e0, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:322
    322     estate = queryDesc->estate;
    #查看参数
    #1、queryDesc
    (gdb) p *queryDesc
    $1 = {operation = CMD_INSERT, plannedstmt = 0x2cc1488, 
      sourceText = 0x2c09ef0 "insert into t_insert values(16,'ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun');", snapshot = 0x2c866e0, 
      crosscheck_snapshot = 0x0, dest = 0x2cc15e8, params = 0x0, queryEnv = 0x0, instrument_options = 0, tupDesc = 0x2c309d0, estate = 0x2c2f900, planstate = 0x2c2fc50, already_executed = false, 
      totaltime = 0x0}
    (gdb) p *(queryDesc->plannedstmt)
    $2 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x2cc10f8, rtable = 0x2cc13b8, resultRelations = 0x2cc1458, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x2cc1408, invalItems = 0x0, paramExecTypes = 0x2c2f590, utilityStmt = 0x0, stmt_location = 0, stmt_len = 136}
    (gdb) p *(queryDesc->snapshot)
    $3 = {satisfies = 0x9f73fc <HeapTupleSatisfiesMVCC>, xmin = 1612874, xmax = 1612874, xip = 0x0, xcnt = 0, subxip = 0x0, subxcnt = 0, suboverflowed = false, takenDuringRecovery = false, copied = true, 
      curcid = 0, speculativeToken = 0, active_count = 1, regd_count = 2, ph_node = {first_child = 0x0, next_sibling = 0x0, prev_or_parent = 0x0}, whenTaken = 0, lsn = 0}
    (gdb) p *(queryDesc->dest)
    $4 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    (gdb) p *(queryDesc->tupDesc)
    $5 = {natts = 0, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x2c309f0}
    (gdb) p *(queryDesc->estate)
    $6 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x2c866e0, es_crosscheck_snapshot = 0x0, es_range_table = 0x2cc13b8, es_plannedstmt = 0x2cc1488, 
      es_sourceText = 0x2c09ef0 "insert into t_insert values(16,'ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun','ExecutorRun/standard_ExecutorRun');", es_junkFilter = 0x0, 
      es_output_cid = 0, es_result_relations = 0x2c2fb40, es_num_result_relations = 1, es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x2c30ab0, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, 
      es_param_exec_vals = 0x2c2fb10, es_queryEnv = 0x0, es_query_cxt = 0x2c2f7f0, es_tupleTable = 0x2c30500, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, 
      es_finished = false, es_exprcontexts = 0x2c2feb0, es_subplanstates = 0x0, es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0}
    (gdb) p *(queryDesc->planstate)
    $7 = {type = T_ModifyTableState, plan = 0x2cc10f8, state = 0x2c2f900, ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
      worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2c30a00, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    #2、direction
    (gdb) p direction
    $8 = ForwardScanDirection
    #3、count
    (gdb) p count
    $9 = 0
    #4、execute_once
    (gdb) p execute_once
    $10 = true
    #单步调试执行
    (gdb) next
    330     oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
    (gdb) 
    333     if (queryDesc->totaltime)
    #MemoryContext是PG中很重要的内存管理数据结构，需深入理解
    (gdb) p *oldcontext
    $11 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c6f380, firstchild = 0x2c2f7f0, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb8d2f1 "PortalContext", ident = 0x2c72e98 "", reset_cbs = 0x0}
    (gdb) p *(estate->es_query_cxt)
    $12 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c2d3d0, firstchild = 0x2cbce60, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb1a840 "ExecutorState", ident = 0x0, reset_cbs = 0x0}
    (gdb) next
    339     operation = queryDesc->operation;
    (gdb) 
    340     dest = queryDesc->dest;
    (gdb) 
    345     estate->es_processed = 0;
    (gdb) 
    346     estate->es_lastoid = InvalidOid;
    (gdb) 
    348     sendTuples = (operation == CMD_SELECT ||
    (gdb) 
    349                   queryDesc->plannedstmt->hasReturning);
    (gdb) 
    348     sendTuples = (operation == CMD_SELECT ||
    (gdb) 
    351     if (sendTuples)
    (gdb) 
    357     if (!ScanDirectionIsNoMovement(direction))
    (gdb) 
    359         if (execute_once && queryDesc->already_executed)
    (gdb) 
    361         queryDesc->already_executed = true;
    (gdb) 
    363         ExecutePlan(estate,
    (gdb) 
    365                     queryDesc->plannedstmt->parallelModeNeeded,
    (gdb) 
    363         ExecutePlan(estate,
    (gdb) 
    377     if (sendTuples)
    (gdb) 
    380     if (queryDesc->totaltime)
    (gdb) 
    383     MemoryContextSwitchTo(oldcontext);
    (gdb) 
    384 }
    (gdb) 
    ExecutorRun (queryDesc=0x2c2d4e0, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:307
    307 }
    (gdb) 
    #DONE!
    

### 四、小结

1、PG的扩展性：PG提供了钩子函数，可以对ExecutorRun进行Hack；  
2、重要的数据结构：MemoryContext，内存上下文，需深入理解。

