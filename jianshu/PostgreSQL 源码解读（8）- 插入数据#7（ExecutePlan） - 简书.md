本文简单介绍了PG插入数据部分的源码，主要内容包括ExecutePlan函数的实现逻辑，该函数位于execMain.c中。

### 一、基础信息

ExecutePlan函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、ScanDirection_

    
    
    //枚举变量，扫描的方向，向后/不需要移动/向前三种
     /*
      * ScanDirection was an int8 for no apparent reason. I kept the original
      * values because I'm not sure if I'll break anything otherwise.  -ay 2/95
      */
     typedef enum ScanDirection
     {
         BackwardScanDirection = -1,
         NoMovementScanDirection = 0,
         ForwardScanDirection = 1
     } ScanDirection;
    

_2、DestReceiver_

    
    
    //目标端接收器
    //包括启动/关闭/销毁/接收数据的函数以及接收器的命令目标类型
     /* ----------------      *      DestReceiver is a base type for destination-specific local state.
      *      In the simplest cases, there is no state info, just the function
      *      pointers that the executor must call.
      *
      * Note: the receiveSlot routine must be passed a slot containing a TupleDesc
      * identical to the one given to the rStartup routine.  It returns bool where
      * a "true" value means "continue processing" and a "false" value means
      * "stop early, just as if we'd reached the end of the scan".
      * ----------------      */
     typedef struct _DestReceiver DestReceiver;
     
     struct _DestReceiver
     {
         /* Called for each tuple to be output: */
         bool        (*receiveSlot) (TupleTableSlot *slot,
                                     DestReceiver *self);//接收Slot的函数指针
         /* Per-executor-run initialization and shutdown: */
         void        (*rStartup) (DestReceiver *self,
                                  int operation,
                                  TupleDesc typeinfo);//Startup函数指针
         void        (*rShutdown) (DestReceiver *self);//Shutdown函数指针
         /* Destroy the receiver object itself (if dynamically allocated) */
         void        (*rDestroy) (DestReceiver *self);//Destroy函数指针
         /* CommandDest code for this receiver */
         CommandDest mydest;//
         /* Private fields might appear beyond this point... */
     };
    
     /* ----------------      *      CommandDest is a simplistic means of identifying the desired
      *      destination.  Someday this will probably need to be improved.
      *
      * Note: only the values DestNone, DestDebug, DestRemote are legal for the
      * global variable whereToSendOutput.   The other values may be used
      * as the destination for individual commands.
      * ----------------      */
     typedef enum
     {
         DestNone,                   /* results are discarded */
         DestDebug,                  /* results go to debugging output */
         DestRemote,                 /* results sent to frontend process */
         DestRemoteExecute,          /* sent to frontend, in Execute command */
         DestRemoteSimple,           /* sent to frontend, w/no catalog access */
         DestSPI,                    /* results sent to SPI manager */
         DestTuplestore,             /* results sent to Tuplestore */
         DestIntoRel,                /* results sent to relation (SELECT INTO) */
         DestCopyOut,                /* results sent to COPY TO code */
         DestSQLFunction,            /* results sent to SQL-language func mgr */
         DestTransientRel,           /* results sent to transient relation */
         DestTupleQueue              /* results sent to tuple queue */
     } CommandDest;
    

_3、ResetPerTupleExprContext_

    
    
    //重置per-output-tuple exprcontext
     /* Reset an EState's per-output-tuple exprcontext, if one's been created */
     #define ResetPerTupleExprContext(estate) \
         do { \
             if ((estate)->es_per_tuple_exprcontext) \
                 ResetExprContext((estate)->es_per_tuple_exprcontext); \
         } while (0)
    

**依赖的函数**  
_1、EnterParallelMode_

    
    
    //进入并行模式函数以及其相关的数据结构/子函数等
     /*
      *  EnterParallelMode
      */
     void
     EnterParallelMode(void)
     {
         TransactionState s = CurrentTransactionState;
     
         Assert(s->parallelModeLevel >= 0);
     
         ++s->parallelModeLevel;
     }
    
     /*
      *  transaction state structure
      */
     typedef struct TransactionStateData
     {
         TransactionId transactionId;    /* my XID, or Invalid if none */
         SubTransactionId subTransactionId;  /* my subxact ID */
         char       *name;           /* savepoint name, if any */
         int         savepointLevel; /* savepoint level */
         TransState  state;          /* low-level state */
         TBlockState blockState;     /* high-level state */
         int         nestingLevel;   /* transaction nesting depth */
         int         gucNestLevel;   /* GUC context nesting depth */
         MemoryContext curTransactionContext;    /* my xact-lifetime context */
         ResourceOwner curTransactionOwner;  /* my query resources */
         TransactionId *childXids;   /* subcommitted child XIDs, in XID order */
         int         nChildXids;     /* # of subcommitted child XIDs */
         int         maxChildXids;   /* allocated size of childXids[] */
         Oid         prevUser;       /* previous CurrentUserId setting */
         int         prevSecContext; /* previous SecurityRestrictionContext */
         bool        prevXactReadOnly;   /* entry-time xact r/o state */
         bool        startedInRecovery;  /* did we start in recovery? */
         bool        didLogXid;      /* has xid been included in WAL record? */
         int         parallelModeLevel;  /* Enter/ExitParallelMode counter */
         struct TransactionStateData *parent;    /* back link to parent */
     } TransactionStateData;
     
     typedef TransactionStateData *TransactionState;
    
     /*
      *  transaction states - transaction state from server perspective
      */
     typedef enum TransState
     {
         TRANS_DEFAULT,              /* idle */
         TRANS_START,                /* transaction starting */
         TRANS_INPROGRESS,           /* inside a valid transaction */
         TRANS_COMMIT,               /* commit in progress */
         TRANS_ABORT,                /* abort in progress */
         TRANS_PREPARE               /* prepare in progress */
     } TransState;
     
    /*
      *  transaction block states - transaction state of client queries
      *
      * Note: the subtransaction states are used only for non-topmost
      * transactions; the others appear only in the topmost transaction.
      */
     typedef enum TBlockState
     {
         /* not-in-transaction-block states */
         TBLOCK_DEFAULT,             /* idle */
         TBLOCK_STARTED,             /* running single-query transaction */
     
         /* transaction block states */
         TBLOCK_BEGIN,               /* starting transaction block */
         TBLOCK_INPROGRESS,          /* live transaction */
         TBLOCK_IMPLICIT_INPROGRESS, /* live transaction after implicit BEGIN */
         TBLOCK_PARALLEL_INPROGRESS, /* live transaction inside parallel worker */
         TBLOCK_END,                 /* COMMIT received */
         TBLOCK_ABORT,               /* failed xact, awaiting ROLLBACK */
         TBLOCK_ABORT_END,           /* failed xact, ROLLBACK received */
         TBLOCK_ABORT_PENDING,       /* live xact, ROLLBACK received */
         TBLOCK_PREPARE,             /* live xact, PREPARE received */
     
         /* subtransaction states */
         TBLOCK_SUBBEGIN,            /* starting a subtransaction */
         TBLOCK_SUBINPROGRESS,       /* live subtransaction */
         TBLOCK_SUBRELEASE,          /* RELEASE received */
         TBLOCK_SUBCOMMIT,           /* COMMIT received while TBLOCK_SUBINPROGRESS */
         TBLOCK_SUBABORT,            /* failed subxact, awaiting ROLLBACK */
         TBLOCK_SUBABORT_END,        /* failed subxact, ROLLBACK received */
         TBLOCK_SUBABORT_PENDING,    /* live subxact, ROLLBACK received */
         TBLOCK_SUBRESTART,          /* live subxact, ROLLBACK TO received */
         TBLOCK_SUBABORT_RESTART     /* failed subxact, ROLLBACK TO received */
     } TBlockState;
    
     typedef struct MemoryContextMethods
     {
         void       *(*alloc) (MemoryContext context, Size size);
         /* call this free_p in case someone #define's free() */
         void        (*free_p) (MemoryContext context, void *pointer);
         void       *(*realloc) (MemoryContext context, void *pointer, Size size);
         void        (*reset) (MemoryContext context);
         void        (*delete_context) (MemoryContext context);
         Size        (*get_chunk_space) (MemoryContext context, void *pointer);
         bool        (*is_empty) (MemoryContext context);
         void        (*stats) (MemoryContext context,
                               MemoryStatsPrintFunc printfunc, void *passthru,
                               MemoryContextCounters *totals);
     #ifdef MEMORY_CONTEXT_CHECKING
         void        (*check) (MemoryContext context);
     #endif
     } MemoryContextMethods;
    
     /*
      * A memory context can have callback functions registered on it.  Any such
      * function will be called once just before the context is next reset or
      * deleted.  The MemoryContextCallback struct describing such a callback
      * typically would be allocated within the context itself, thereby avoiding
      * any need to manage it explicitly (the reset/delete action will free it).
      */
     typedef void (*MemoryContextCallbackFunction) (void *arg);
     
     typedef struct MemoryContextCallback
     {
         MemoryContextCallbackFunction func; /* function to call */
         void       *arg;            /* argument to pass it */
         struct MemoryContextCallback *next; /* next in list of callbacks */
     } MemoryContextCallback;
     
    
     typedef struct MemoryContextData
     {
         NodeTag     type;           /* identifies exact kind of context */
         /* these two fields are placed here to minimize alignment wastage: */
         bool        isReset;        /* T = no space alloced since last reset */
         bool        allowInCritSection; /* allow palloc in critical section */
         const MemoryContextMethods *methods;    /* virtual function table */
         MemoryContext parent;       /* NULL if no parent (toplevel context) */
         MemoryContext firstchild;   /* head of linked list of children */
         MemoryContext prevchild;    /* previous child of same parent */
         MemoryContext nextchild;    /* next child of same parent */
         const char *name;           /* context name (just for debugging) */
         const char *ident;          /* context ID if any (just for debugging) */
         MemoryContextCallback *reset_cbs;   /* list of reset/delete callbacks */
     } MemoryContextData;
     
     /* utils/palloc.h contains typedef struct MemoryContextData *MemoryContext */
    
    /*
      * Type MemoryContextData is declared in nodes/memnodes.h.  Most users
      * of memory allocation should just treat it as an abstract type, so we
      * do not provide the struct contents here.
      */
     typedef struct MemoryContextData *MemoryContext;
    
    
     /*
      * ResourceOwner objects look like this
      */
     typedef struct ResourceOwnerData
     {
         ResourceOwner parent;       /* NULL if no parent (toplevel owner) */
         ResourceOwner firstchild;   /* head of linked list of children */
         ResourceOwner nextchild;    /* next child of same parent */
         const char *name;           /* name (just for debugging) */
     
         /* We have built-in support for remembering: */
         ResourceArray bufferarr;    /* owned buffers */
         ResourceArray catrefarr;    /* catcache references */
         ResourceArray catlistrefarr;    /* catcache-list pins */
         ResourceArray relrefarr;    /* relcache references */
         ResourceArray planrefarr;   /* plancache references */
         ResourceArray tupdescarr;   /* tupdesc references */
         ResourceArray snapshotarr;  /* snapshot references */
         ResourceArray filearr;      /* open temporary files */
         ResourceArray dsmarr;       /* dynamic shmem segments */
         ResourceArray jitarr;       /* JIT contexts */
     
         /* We can remember up to MAX_RESOWNER_LOCKS references to local locks. */
         int         nlocks;         /* number of owned locks */
         LOCALLOCK  *locks[MAX_RESOWNER_LOCKS];  /* list of owned locks */
     }           ResourceOwnerData;
     
     
    /*
      * ResourceOwner objects are an opaque data structure known only within
      * resowner.c.
      */
     typedef struct ResourceOwnerData *ResourceOwner;
     
     typedef uint32 TransactionId;
    

_2、ExecShutdownNode_  
//关闭资源

    
    
     /*
      * ExecShutdownNode
      *
      * Give execution nodes a chance to stop asynchronous resource consumption
      * and release any resources still held.  Currently, this is only used for
      * parallel query, but we might want to extend it to other cases also (e.g.
      * FDW).  We might also want to call it sooner, as soon as it's evident that
      * no more rows will be needed (e.g. when a Limit is filled) rather than only
      * at the end of ExecutorRun.
      */
     bool
     ExecShutdownNode(PlanState *node)
     {
         if (node == NULL)
             return false;
     
         check_stack_depth();
     
         planstate_tree_walker(node, ExecShutdownNode, NULL);
     
         /*
          * Treat the node as running while we shut it down, but only if it's run
          * at least once already.  We don't expect much CPU consumption during
          * node shutdown, but in the case of Gather or Gather Merge, we may shut
          * down workers at this stage.  If so, their buffer usage will get
          * propagated into pgBufferUsage at this point, and we want to make sure
          * that it gets associated with the Gather node.  We skip this if the node
          * has never been executed, so as to avoid incorrectly making it appear
          * that it has.
          */
         if (node->instrument && node->instrument->running)
             InstrStartNode(node->instrument);
     
         switch (nodeTag(node))
         {
             case T_GatherState:
                 ExecShutdownGather((GatherState *) node);
                 break;
             case T_ForeignScanState:
                 ExecShutdownForeignScan((ForeignScanState *) node);
                 break;
             case T_CustomScanState:
                 ExecShutdownCustomScan((CustomScanState *) node);
                 break;
             case T_GatherMergeState:
                 ExecShutdownGatherMerge((GatherMergeState *) node);
                 break;
             case T_HashState:
                 ExecShutdownHash((HashState *) node);
                 break;
             case T_HashJoinState:
                 ExecShutdownHashJoin((HashJoinState *) node);
                 break;
             default:
                 break;
         }
     
         /* Stop the node if we started it above, reporting 0 tuples. */
         if (node->instrument && node->instrument->running)
             InstrStopNode(node->instrument, 0);
     
         return false;
     }
    

_3、ExitParallelMode_

    
    
    //退出并行模式
     /*
      *  ExitParallelMode
      */
     void
     ExitParallelMode(void)
     {
         TransactionState s = CurrentTransactionState;
     
         Assert(s->parallelModeLevel > 0);
         Assert(s->parallelModeLevel > 1 || !ParallelContextActive());
     
         --s->parallelModeLevel;
     }
    

### 二、源码解读

    
    
    /* ----------------------------------------------------------------     *      ExecutePlan
     *
     *      Processes the query plan until we have retrieved 'numberTuples' tuples,
     *      moving in the specified direction.
     *
     *      Runs to completion if numberTuples is 0
     *
     * Note: the ctid attribute is a 'junk' attribute that is removed before the
     * user can see it
     * ----------------------------------------------------------------     */
    /*
    输入：
        estate-Executor状态信息
        planstate-执行计划状态信息
        user_parallel_mode-是否使用并行模式
        operation-操作类型
        sendTuples-是否需要传输Tuples
        numberTuples-需要返回的Tuples数
        direction-扫描方向
        dest-目标接收器指针
        execute_once-是否只执行一次？
    输出：
        
    */
    static void
    ExecutePlan(EState *estate,
                PlanState *planstate,
                bool use_parallel_mode,
                CmdType operation,
                bool sendTuples,
                uint64 numberTuples,
                ScanDirection direction,
                DestReceiver *dest,
                bool execute_once)
    {
        TupleTableSlot *slot;//存储Tuple的Slot
        uint64      current_tuple_count;//Tuple计数器
    
        /*
         * initialize local variables
         */
        current_tuple_count = 0;//初始化
    
        /*
         * Set the direction.
         */
        estate->es_direction = direction;//扫描方向
    
        /*
         * If the plan might potentially be executed multiple times, we must force
         * it to run without parallelism, because we might exit early.
         */
        if (!execute_once)
            use_parallel_mode = false;//不使用并行模式
    
        estate->es_use_parallel_mode = use_parallel_mode;
        if (use_parallel_mode)
            EnterParallelMode();//如果并行执行，进入并行模式
    
        /*
         * Loop until we've processed the proper number of tuples from the plan.
         */
        for (;;)
        {
            /* Reset the per-output-tuple exprcontext */
            ResetPerTupleExprContext(estate);//重置 per-output-tuple exprcontext
    
            /*
             * Execute the plan and obtain a tuple
             */
            slot = ExecProcNode(planstate);//执行处理单元
    
            /*
             * if the tuple is null, then we assume there is nothing more to
             * process so we just end the loop...
             */
            if (TupIsNull(slot))
            {
                /* Allow nodes to release or shut down resources. */
                (void) ExecShutdownNode(planstate);//如果slot返回为空，已完成处理，可以释放资源了
                break;
            }
    
            /*
             * If we have a junk filter, then project a new tuple with the junk
             * removed.
             *
             * Store this new "clean" tuple in the junkfilter's resultSlot.
             * (Formerly, we stored it back over the "dirty" tuple, which is WRONG
             * because that tuple slot has the wrong descriptor.)
             */
            if (estate->es_junkFilter != NULL)
                slot = ExecFilterJunk(estate->es_junkFilter, slot);//去掉“无价值”的属性
    
            /*
             * If we are supposed to send the tuple somewhere, do so. (In
             * practice, this is probably always the case at this point.)
             */
            if (sendTuples)//如果需要传输结果集
            {
                /*
                 * If we are not able to send the tuple, we assume the destination
                 * has closed and no more tuples can be sent. If that's the case,
                 * end the loop.
                 */
                if (!dest->receiveSlot(slot, dest))//传输，如果接收完成，则退出
                    break;
            }
    
            /*
             * Count tuples processed, if this is a SELECT.  (For other operation
             * types, the ModifyTable plan node must count the appropriate
             * events.)
             */
            if (operation == CMD_SELECT)
                (estate->es_processed)++;//统计计数
    
            /*
             * check our tuple count.. if we've processed the proper number then
             * quit, else loop again and process more tuples.  Zero numberTuples
             * means no limit.
             */
            current_tuple_count++;//统计返回的Tuple数
            if (numberTuples && numberTuples == current_tuple_count)
            {
                /* Allow nodes to release or shut down resources. */
                (void) ExecShutdownNode(planstate);//释放资源后退出循环
                break;
            }
        }
    
        if (use_parallel_mode)
            ExitParallelMode();//退出并行模式
    }
    
    
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- #7 ExecutePlan
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               3294
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(15,'ExecutePlan','ExecutePlan','ExecutePlan');
    (挂起)
    

启动gdb，跟踪调试：

    
    
    [root@localhost ~]# gdb -p 3294
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b ExecutePlan
    Breakpoint 1 at 0x692df1: file execMain.c, line 1697.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecutePlan (estate=0x2cca440, planstate=0x2cca790, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, dest=0x2cd0888, 
        execute_once=true) at execMain.c:1697
    1697        current_tuple_count = 0;
    (gdb) 
    #查看输入参数
    #1、Executor状态 estate
    (gdb) p *estate
    $1 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x2c3f920, es_crosscheck_snapshot = 0x0, es_range_table = 0x2cd0768, es_plannedstmt = 0x2c1d478, 
      es_sourceText = 0x2c1bef0 "insert into t_insert values(15,'ExecutePlan','ExecutePlan','ExecutePlan');", es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x2cca680, 
      es_num_result_relations = 1, es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x2ccb750, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x2cca650, es_queryEnv = 0x0, es_query_cxt = 0x2cca330, 
      es_tupleTable = 0x2ccb000, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x2ccac50, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0}
    #执行期间构造的snapshot
    (gdb) p *(estate->es_snapshot)
    $2 = {satisfies = 0x9f73fc <HeapTupleSatisfiesMVCC>, xmin = 1612872, xmax = 1612872, xip = 0x0, xcnt = 0, subxip = 0x0, subxcnt = 0, suboverflowed = false, takenDuringRecovery = false, copied = true, 
      curcid = 0, speculativeToken = 0, active_count = 1, regd_count = 2, ph_node = {first_child = 0xe7bac0 <CatalogSnapshotData+64>, next_sibling = 0x0, prev_or_parent = 0x0}, whenTaken = 0, lsn = 0}
    (gdb) 
    #执行计划对应的Statement
    #commandType = CMD_INSERT，插入操作
    #hasReturning =false，没有返回值
    (gdb) p *(estate->es_range_table)
    $3 = {type = T_List, length = 1, head = 0x2cd0748, tail = 0x2cd0748}
    (gdb) p *(estate->es_plannedstmt)
    $4 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x2c1cff8, rtable = 0x2cd0768, resultRelations = 0x2cd0808, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x2cd07b8, invalItems = 0x0, paramExecTypes = 0x2c43698, utilityStmt = 0x0, stmt_location = 0, stmt_len = 73}
    (gdb) 
    #SQL语句
    #es_sourceText = 0x2c1bef0 "insert into t_insert values(15,'ExecutePlan','ExecutePlan','ExecutePlan');"
    #junkFilter为NULL(0x0)
    #结果Relation相关信息
    (gdb) p *(estate->es_result_relations)
    $5 = {type = T_ResultRelInfo, ri_RangeTableIndex = 1, ri_RelationDesc = 0x7f3c13f49b78, ri_NumIndices = 1, ri_IndexRelationDescs = 0x2ccafd0, ri_IndexRelationInfo = 0x2ccafe8, ri_TrigDesc = 0x0, 
      ri_TrigFunctions = 0x0, ri_TrigWhenExprs = 0x0, ri_TrigInstrument = 0x0, ri_FdwRoutine = 0x0, ri_FdwState = 0x0, ri_usesFdwDirectModify = false, ri_WithCheckOptions = 0x0, 
      ri_WithCheckOptionExprs = 0x0, ri_ConstraintExprs = 0x0, ri_junkFilter = 0x0, ri_returningList = 0x0, ri_projectReturning = 0x0, ri_onConflictArbiterIndexes = 0x0, ri_onConflict = 0x0, 
      ri_PartitionCheck = 0x0, ri_PartitionCheckExpr = 0x0, ri_PartitionRoot = 0x0, ri_PartitionReadyForRouting = false}
    #es_trig_tuple_slot
    gdb) p *(estate->es_trig_tuple_slot)
    $6 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x0, tts_mcxt = 0x2cca330, 
      tts_buffer = 0, tts_nvalid = 0, tts_values = 0x0, tts_isnull = 0x0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, 
        t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = false}
    #estate中的其他变量
    (gdb) p *(estate->es_param_exec_vals)
    $7 = {execPlan = 0x0, value = 0, isnull = false}
    (gdb) p *(estate->es_query_cxt)
    $8 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c3f3d0, firstchild = 0x2ccc340, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb1a840 "ExecutorState", ident = 0x0, reset_cbs = 0x0}
    (gdb) p *(estate->es_tupleTable)
    $9 = {type = T_List, length = 3, head = 0x2ccb240, tail = 0x2ccb7e0}
    (gdb) p *(estate->es_exprcontexts)
    $10 = {type = T_List, length = 1, head = 0x2ccafb0, tail = 0x2ccafb0}
    #2、planstate
    (gdb) p *planstate
    $14 = {type = T_ModifyTableState, plan = 0x2c1cff8, state = 0x2cca440, ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
      worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2ccb6a0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    #plan
    (gdb) p *(planstate->plan)
    $15 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    #PlanState中的EState
    (gdb) p *(planstate->state)
    $16 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x2c3f920, es_crosscheck_snapshot = 0x0, es_range_table = 0x2cd0768, es_plannedstmt = 0x2c1d478, 
      es_sourceText = 0x2c1bef0 "insert into t_insert values(15,'ExecutePlan','ExecutePlan','ExecutePlan');", es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x2cca680, 
      es_num_result_relations = 1, es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x2ccb750, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x2cca650, es_queryEnv = 0x0, es_query_cxt = 0x2cca330, 
      es_tupleTable = 0x2ccb000, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x2ccac50, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0}
    #ExecProcNode=ExecProcNodeFirst
    #ExecProcNodeReal =ExecModifyTable
    #3、use_parallel_mode
    (gdb) p use_parallel_mode
    $17 = false #非并行模式
    #4、operation
    (gdb) p operation
    $18 = CMD_INSERT #插入操作
    #5、sendTuples
    (gdb) p sendTuples
    $19 = false
    #6、numberTuples
    (gdb) p numberTuples
    $20 = 0
    #7、direction
    (gdb) p direction
    $21 = ForwardScanDirection
    #8、dest
    (gdb) p *dest
    $22 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    #9、execute_once
    (gdb) p execute_once
    $23 = true
    #----------------------------------------------------    #继续执行
    (gdb) next
    1702        estate->es_direction = direction;
    (gdb) 
    1708        if (!execute_once)
    (gdb) 
    1711        estate->es_use_parallel_mode = use_parallel_mode;
    (gdb) 
    1712        if (use_parallel_mode)
    (gdb) 
    1721            ResetPerTupleExprContext(estate);
    (gdb) 
    1726            slot = ExecProcNode(planstate);
    (gdb) 
    1732            if (TupIsNull(slot))
    (gdb) p *slot
    Cannot access memory at address 0x0
    #返回的slot为NULL
    (gdb) next
    1735                (void) ExecShutdownNode(planstate);
    (gdb) 
    1736                break;
    (gdb) 
    1787        if (use_parallel_mode)
    (gdb) 
    1789    }
    (gdb) 
    standard_ExecutorRun (queryDesc=0x2c3f4e0, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:377
    377     if (sendTuples)
    #DONE!
    

### 四、小结

1、统一的处理模型：先前也提过，INSERT/UPDATE/DELETE/SELECT等，均采用统一的处理模式进行处理；  
2、重要的数据结构：PlanState和EState，在整个执行过程中，非常重要的状态信息；  
3、并行模式：并行模式的处理，看起来只是开启&打开，后续值得探究一番。

