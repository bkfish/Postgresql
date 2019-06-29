本文简单介绍了PG插入数据部分的源码，主要内容包括PortalRunMulti函数和PortalRun函数的实现逻辑，PortalRunMulti函数和PortalRun均位于pquery.c文件中。

### 一、基础信息

PortalRunMulti函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、Portal_

    
    
     typedef struct PortalData *Portal;
     
     typedef struct PortalData
     {
         /* Bookkeeping data */
         const char *name;           /* portal's name */
         const char *prepStmtName;   /* source prepared statement (NULL if none) */
         MemoryContext portalContext;    /* subsidiary memory for portal */
         ResourceOwner resowner;     /* resources owned by portal */
         void        (*cleanup) (Portal portal); /* cleanup hook */
     
         /*
          * State data for remembering which subtransaction(s) the portal was
          * created or used in.  If the portal is held over from a previous
          * transaction, both subxids are InvalidSubTransactionId.  Otherwise,
          * createSubid is the creating subxact and activeSubid is the last subxact
          * in which we ran the portal.
          */
         SubTransactionId createSubid;   /* the creating subxact */
         SubTransactionId activeSubid;   /* the last subxact with activity */
     
         /* The query or queries the portal will execute */
         const char *sourceText;     /* text of query (as of 8.4, never NULL) */
         const char *commandTag;     /* command tag for original query */
         List       *stmts;          /* list of PlannedStmts */
         CachedPlan *cplan;          /* CachedPlan, if stmts are from one */
     
         ParamListInfo portalParams; /* params to pass to query */
         QueryEnvironment *queryEnv; /* environment for query */
     
         /* Features/options */
         PortalStrategy strategy;    /* see above */
         int         cursorOptions;  /* DECLARE CURSOR option bits */
         bool        run_once;       /* portal will only be run once */
     
         /* Status data */
         PortalStatus status;        /* see above */
         bool        portalPinned;   /* a pinned portal can't be dropped */
         bool        autoHeld;       /* was automatically converted from pinned to
                                      * held (see HoldPinnedPortals()) */
     
         /* If not NULL, Executor is active; call ExecutorEnd eventually: */
         QueryDesc  *queryDesc;      /* info needed for executor invocation */
     
         /* If portal returns tuples, this is their tupdesc: */
         TupleDesc   tupDesc;        /* descriptor for result tuples */
         /* and these are the format codes to use for the columns: */
         int16      *formats;        /* a format code for each column */
     
         /*
          * Where we store tuples for a held cursor or a PORTAL_ONE_RETURNING or
          * PORTAL_UTIL_SELECT query.  (A cursor held past the end of its
          * transaction no longer has any active executor state.)
          */
         Tuplestorestate *holdStore; /* store for holdable cursors */
         MemoryContext holdContext;  /* memory containing holdStore */
     
         /*
          * Snapshot under which tuples in the holdStore were read.  We must keep a
          * reference to this snapshot if there is any possibility that the tuples
          * contain TOAST references, because releasing the snapshot could allow
          * recently-dead rows to be vacuumed away, along with any toast data
          * belonging to them.  In the case of a held cursor, we avoid needing to
          * keep such a snapshot by forcibly detoasting the data.
          */
         Snapshot    holdSnapshot;   /* registered snapshot, or NULL if none */
     
         /*
          * atStart, atEnd and portalPos indicate the current cursor position.
          * portalPos is zero before the first row, N after fetching N'th row of
          * query.  After we run off the end, portalPos = # of rows in query, and
          * atEnd is true.  Note that atStart implies portalPos == 0, but not the
          * reverse: we might have backed up only as far as the first row, not to
          * the start.  Also note that various code inspects atStart and atEnd, but
          * only the portal movement routines should touch portalPos.
          */
         bool        atStart;
         bool        atEnd;
         uint64      portalPos;
     
         /* Presentation data, primarily used by the pg_cursors system view */
         TimestampTz creation_time;  /* time at which this portal was defined */
         bool        visible;        /* include this portal in pg_cursors? */
     }           PortalData;
     
    

_2、List_

    
    
     typedef struct ListCell ListCell;
     
     typedef struct List
     {
         NodeTag     type;           /* T_List, T_IntList, or T_OidList */
         int         length;
         ListCell   *head;
         ListCell   *tail;
     } List;
     
     struct ListCell
     {
         union
         {
             void       *ptr_value;
             int         int_value;
             Oid         oid_value;
         }           data;
         ListCell   *next;
     };
    
    

_3、Snapshot_

    
    
    typedef struct SnapshotData *Snapshot;
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
     } SnapshotData;
    
    

**依赖的函数**  
_1、lfirst__ *

    
    
     /*
      * NB: There is an unfortunate legacy from a previous incarnation of
      * the List API: the macro lfirst() was used to mean "the data in this
      * cons cell". To avoid changing every usage of lfirst(), that meaning
      * has been kept. As a result, lfirst() takes a ListCell and returns
      * the data it contains; to get the data in the first cell of a
      * List, use linitial(). Worse, lsecond() is more closely related to
      * linitial() than lfirst(): given a List, lsecond() returns the data
      * in the second cons cell.
      */
    
     #define lnext(lc)               ((lc)->next)
     #define lfirst(lc)              ((lc)->data.ptr_value)
     #define lfirst_int(lc)          ((lc)->data.int_value)
     #define lfirst_oid(lc)          ((lc)->data.oid_value)
     #define lfirst_node(type,lc)    castNode(type, lfirst(lc))
    
     /*
      * castNode(type, ptr) casts ptr to "type *", and if assertions are enabled,
      * verifies that the node has the appropriate type (using its nodeTag()).
      *
      * Use an inline function when assertions are enabled, to avoid multiple
      * evaluations of the ptr argument (which could e.g. be a function call).
      */
     #ifdef USE_ASSERT_CHECKING
     static inline Node *
     castNodeImpl(NodeTag type, void *ptr)
     {
         Assert(ptr == NULL || nodeTag(ptr) == type);
         return (Node *) ptr;
     }
     #define castNode(_type_, nodeptr) ((_type_ *) castNodeImpl(T_##_type_, nodeptr))
     #else
     #define castNode(_type_, nodeptr) ((_type_ *) (nodeptr))
     #endif                          /* USE_ASSERT_CHECKING */
    
    

_2、Snapshot相关_

    
    
    //留待MVCC再行解读
    GetTransactionSnapshot
    RegisterSnapshot
    PushCopiedSnapshot
    UpdateActiveSnapshotCommandId
    PopActiveSnapshot
    

_3、ProcessQuery_

    
    
    //上一节已介绍
    

_4、CommandCounterIncrement_

    
    
     /*
      *  CommandCounterIncrement
      */
     void
     CommandCounterIncrement(void)
     {
         /*
          * If the current value of the command counter hasn't been "used" to mark
          * tuples, we need not increment it, since there's no need to distinguish
          * a read-only command from others.  This helps postpone command counter
          * overflow, and keeps no-op CommandCounterIncrement operations cheap.
          */
         if (currentCommandIdUsed)
         {
             /*
              * Workers synchronize transaction state at the beginning of each
              * parallel operation, so we can't account for new commands after that
              * point.
              */
             if (IsInParallelMode() || IsParallelWorker())
                 elog(ERROR, "cannot start commands during a parallel operation");
     
             currentCommandId += 1;
             if (currentCommandId == InvalidCommandId)
             {
                 currentCommandId -= 1;
                 ereport(ERROR,
                         (errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
                          errmsg("cannot have more than 2^32-2 commands in a transaction")));
             }
             currentCommandIdUsed = false;
     
             /* Propagate new command ID into static snapshots */
             SnapshotSetCommandId(currentCommandId);
     
             /*
              * Make any catalog changes done by the just-completed command visible
              * in the local syscache.  We obviously don't need to do this after a
              * read-only command.  (But see hacks in inval.c to make real sure we
              * don't think a command that queued inval messages was read-only.)
              */
             AtCCI_LocalCache();
         }
     }
    

_5、MemoryContextDeleteChildren_

    
    
     /*
      * MemoryContextDeleteChildren
      *      Delete all the descendants of the named context and release all
      *      space allocated therein.  The named context itself is not touched.
      */
     void
     MemoryContextDeleteChildren(MemoryContext context)
     {
         AssertArg(MemoryContextIsValid(context));
     
         /*
          * MemoryContextDelete will delink the child from me, so just iterate as
          * long as there is a child.
          */
         while (context->firstchild != NULL)
             MemoryContextDelete(context->firstchild);
     }
    

### 二、源码解读

**1、PortalRun**

    
    
    /*
     * PortalRun
     *      Run a portal's query or queries.
     *
     * count <= 0 is interpreted as a no-op: the destination gets started up
     * and shut down, but nothing else happens.  Also, count == FETCH_ALL is
     * interpreted as "all rows".  Note that count is ignored in multi-query
     * situations, where we always run the portal to completion.
     *
     * isTopLevel: true if query is being executed at backend "top level"
     * (that is, directly from a client command message)
     *
     * dest: where to send output of primary (canSetTag) query
     *
     * altdest: where to send output of non-primary queries
     *
     * completionTag: points to a buffer of size COMPLETION_TAG_BUFSIZE
     *      in which to store a command completion status string.
     *      May be NULL if caller doesn't want a status string.
     *
     * Returns true if the portal's execution is complete, false if it was
     * suspended due to exhaustion of the count parameter.
     */
    /*
    输入：
        参照PortalRunMulti
    输出：
        布尔变量，成功true，失败false
    */
    bool
    PortalRun(Portal portal, long count, bool isTopLevel, bool run_once,
              DestReceiver *dest, DestReceiver *altdest,
              char *completionTag)
    {
        bool        result;//返回结果
        uint64      nprocessed;
        ResourceOwner saveTopTransactionResourceOwner;//高层事务资源宿主
        MemoryContext saveTopTransactionContext;//内存上下文
        Portal      saveActivePortal;//活动的Portal
        ResourceOwner saveResourceOwner;
        MemoryContext savePortalContext;
        MemoryContext saveMemoryContext;
    
        AssertArg(PortalIsValid(portal));
    
        TRACE_POSTGRESQL_QUERY_EXECUTE_START();
    
        /* Initialize completion tag to empty string */
        if (completionTag)
            completionTag[0] = '\0';
    
        if (log_executor_stats && portal->strategy != PORTAL_MULTI_QUERY)
        {
            elog(DEBUG3, "PortalRun");
            /* PORTAL_MULTI_QUERY logs its own stats per query */
            ResetUsage();
        }
    
        /*
         * Check for improper portal use, and mark portal active.
         */
        MarkPortalActive(portal);
    
        /* Set run_once flag.  Shouldn't be clear if previously set. */
        Assert(!portal->run_once || run_once);
        portal->run_once = run_once;
    
        /*
         * Set up global portal context pointers.
         *
         * We have to play a special game here to support utility commands like
         * VACUUM and CLUSTER, which internally start and commit transactions.
         * When we are called to execute such a command, CurrentResourceOwner will
         * be pointing to the TopTransactionResourceOwner --- which will be
         * destroyed and replaced in the course of the internal commit and
         * restart.  So we need to be prepared to restore it as pointing to the
         * exit-time TopTransactionResourceOwner.  (Ain't that ugly?  This idea of
         * internally starting whole new transactions is not good.)
         * CurrentMemoryContext has a similar problem, but the other pointers we
         * save here will be NULL or pointing to longer-lived objects.
         */
        //保护现场
        saveTopTransactionResourceOwner = TopTransactionResourceOwner;
        saveTopTransactionContext = TopTransactionContext;
        saveActivePortal = ActivePortal;
        saveResourceOwner = CurrentResourceOwner;
        savePortalContext = PortalContext;
        saveMemoryContext = CurrentMemoryContext;
        PG_TRY();
        {
            ActivePortal = portal;
            if (portal->resowner)
                CurrentResourceOwner = portal->resowner;
            PortalContext = portal->portalContext;
    
            MemoryContextSwitchTo(PortalContext);
    
            switch (portal->strategy)
            {
                case PORTAL_ONE_SELECT:
                case PORTAL_ONE_RETURNING:
                case PORTAL_ONE_MOD_WITH:
                case PORTAL_UTIL_SELECT:
    
                    /*
                     * If we have not yet run the command, do so, storing its
                     * results in the portal's tuplestore.  But we don't do that
                     * for the PORTAL_ONE_SELECT case.
                     */
                    if (portal->strategy != PORTAL_ONE_SELECT && !portal->holdStore)
                        FillPortalStore(portal, isTopLevel);
    
                    /*
                     * Now fetch desired portion of results.
                     */
                    nprocessed = PortalRunSelect(portal, true, count, dest);
    
                    /*
                     * If the portal result contains a command tag and the caller
                     * gave us a pointer to store it, copy it. Patch the "SELECT"
                     * tag to also provide the rowcount.
                     */
                    if (completionTag && portal->commandTag)
                    {
                        if (strcmp(portal->commandTag, "SELECT") == 0)
                            snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
                                     "SELECT " UINT64_FORMAT, nprocessed);
                        else
                            strcpy(completionTag, portal->commandTag);
                    }
    
                    /* Mark portal not active */
                    portal->status = PORTAL_READY;
    
                    /*
                     * Since it's a forward fetch, say DONE iff atEnd is now true.
                     */
                    result = portal->atEnd;
                    break;
    
                case PORTAL_MULTI_QUERY://INSERT语句
                    PortalRunMulti(portal, isTopLevel, false,
                                   dest, altdest, completionTag);
    
                    /* Prevent portal's commands from being re-executed */
                    MarkPortalDone(portal);
    
                    /* Always complete at end of RunMulti */
                    result = true;
                    break;
    
                default:
                    elog(ERROR, "unrecognized portal strategy: %d",
                         (int) portal->strategy);
                    result = false; /* keep compiler quiet */
                    break;
            }
        }
        PG_CATCH();
        {
            /* Uncaught error while executing portal: mark it dead */
            MarkPortalFailed(portal);
    
            /* Restore global vars and propagate error */
            if (saveMemoryContext == saveTopTransactionContext)
                MemoryContextSwitchTo(TopTransactionContext);
            else
                MemoryContextSwitchTo(saveMemoryContext);
            ActivePortal = saveActivePortal;
            if (saveResourceOwner == saveTopTransactionResourceOwner)
                CurrentResourceOwner = TopTransactionResourceOwner;
            else
                CurrentResourceOwner = saveResourceOwner;
            PortalContext = savePortalContext;
    
            PG_RE_THROW();
        }
        PG_END_TRY();
    
        if (saveMemoryContext == saveTopTransactionContext)
            MemoryContextSwitchTo(TopTransactionContext);
        else
            MemoryContextSwitchTo(saveMemoryContext);
        ActivePortal = saveActivePortal;
        if (saveResourceOwner == saveTopTransactionResourceOwner)
            CurrentResourceOwner = TopTransactionResourceOwner;
        else
            CurrentResourceOwner = saveResourceOwner;
        PortalContext = savePortalContext;
    
        if (log_executor_stats && portal->strategy != PORTAL_MULTI_QUERY)
            ShowUsage("EXECUTOR STATISTICS");
    
        TRACE_POSTGRESQL_QUERY_EXECUTE_DONE();
    
        return result;
    }
    
    

**2、PortalRunMulti**

    
    
    /*
     * PortalRunMulti
     *      Execute a portal's queries in the general case (multi queries
     *      or non-SELECT-like queries)
     */
    /*
    输入：
        portal-“门户”数据结构
        isTopLevel-顶层？
        setHoldSnapshot-是否持有快照
        dest-目标端
        altdest-?
        completionTag-完成标记（用于语句执行结果输出）
    输出：
        无
    */
    static void
    PortalRunMulti(Portal portal,
                   bool isTopLevel, bool setHoldSnapshot,
                   DestReceiver *dest, DestReceiver *altdest,
                   char *completionTag)
    {
        bool        active_snapshot_set = false;//活跃snapshot？
        ListCell   *stmtlist_item;//SQL语句，临时变量
    
        /*
         * If the destination is DestRemoteExecute, change to DestNone.  The
         * reason is that the client won't be expecting any tuples, and indeed has
         * no way to know what they are, since there is no provision for Describe
         * to send a RowDescription message when this portal execution strategy is
         * in effect.  This presently will only affect SELECT commands added to
         * non-SELECT queries by rewrite rules: such commands will be executed,
         * but the results will be discarded unless you use "simple Query"
         * protocol.
         */
        if (dest->mydest == DestRemoteExecute)
            dest = None_Receiver;
        if (altdest->mydest == DestRemoteExecute)
            altdest = None_Receiver;
    
        /*
         * Loop to handle the individual queries generated from a single parsetree
         * by analysis and rewrite.
         */
        foreach(stmtlist_item, portal->stmts)//循环处理SQL语句
        {
            PlannedStmt *pstmt = lfirst_node(PlannedStmt, stmtlist_item);//获取已规划的语句
    
            /*
             * If we got a cancel signal in prior command, quit
             */
            CHECK_FOR_INTERRUPTS();
    
            if (pstmt->utilityStmt == NULL)//非“工具类”语句
            {
                /*
                 * process a plannable query.
                 */
                TRACE_POSTGRESQL_QUERY_EXECUTE_START();
    
                if (log_executor_stats)
                    ResetUsage();
    
                /*
                 * Must always have a snapshot for plannable queries.  First time
                 * through, take a new snapshot; for subsequent queries in the
                 * same portal, just update the snapshot's copy of the command
                 * counter.
                 */
                if (!active_snapshot_set)
                {
                    Snapshot    snapshot = GetTransactionSnapshot();//获取事务快照
    
                    /* If told to, register the snapshot and save in portal */
                    if (setHoldSnapshot)
                    {
                        snapshot = RegisterSnapshot(snapshot);
                        portal->holdSnapshot = snapshot;
                    }
    
                    /*
                     * We can't have the holdSnapshot also be the active one,
                     * because UpdateActiveSnapshotCommandId would complain.  So
                     * force an extra snapshot copy.  Plain PushActiveSnapshot
                     * would have copied the transaction snapshot anyway, so this
                     * only adds a copy step when setHoldSnapshot is true.  (It's
                     * okay for the command ID of the active snapshot to diverge
                     * from what holdSnapshot has.)
                     */
                    PushCopiedSnapshot(snapshot);
                    active_snapshot_set = true;
                }
                else
                    UpdateActiveSnapshotCommandId();
    
                //处理查询
                if (pstmt->canSetTag)
                {
                    /* statement can set tag string */
                    ProcessQuery(pstmt,
                                 portal->sourceText,
                                 portal->portalParams,
                                 portal->queryEnv,
                                 dest, completionTag);
                }
                else
                {
                    /* stmt added by rewrite cannot set tag */
                    ProcessQuery(pstmt,
                                 portal->sourceText,
                                 portal->portalParams,
                                 portal->queryEnv,
                                 altdest, NULL);
                }
    
                if (log_executor_stats)
                    ShowUsage("EXECUTOR STATISTICS");
    
                TRACE_POSTGRESQL_QUERY_EXECUTE_DONE();
            }
            else//“工具类”语句
            {
                /*
                 * process utility functions (create, destroy, etc..)
                 *
                 * We must not set a snapshot here for utility commands (if one is
                 * needed, PortalRunUtility will do it).  If a utility command is
                 * alone in a portal then everything's fine.  The only case where
                 * a utility command can be part of a longer list is that rules
                 * are allowed to include NotifyStmt.  NotifyStmt doesn't care
                 * whether it has a snapshot or not, so we just leave the current
                 * snapshot alone if we have one.
                 */
                if (pstmt->canSetTag)
                {
                    Assert(!active_snapshot_set);
                    /* statement can set tag string */
                    PortalRunUtility(portal, pstmt, isTopLevel, false,
                                     dest, completionTag);
                }
                else
                {
                    Assert(IsA(pstmt->utilityStmt, NotifyStmt));
                    /* stmt added by rewrite cannot set tag */
                    PortalRunUtility(portal, pstmt, isTopLevel, false,
                                     altdest, NULL);
                }
            }
    
            /*
             * Increment command counter between queries, but not after the last
             * one.
             */
            if (lnext(stmtlist_item) != NULL)
                CommandCounterIncrement();
    
            /*
             * Clear subsidiary contexts to recover temporary memory.
             */
            Assert(portal->portalContext == CurrentMemoryContext);
    
            MemoryContextDeleteChildren(portal->portalContext);//释放资源
        }
    
        /* Pop the snapshot if we pushed one. */
        if (active_snapshot_set)
            PopActiveSnapshot();
    
        /*
         * If a command completion tag was supplied, use it.  Otherwise use the
         * portal's commandTag as the default completion tag.
         *
         * Exception: Clients expect INSERT/UPDATE/DELETE tags to have counts, so
         * fake them with zeros.  This can happen with DO INSTEAD rules if there
         * is no replacement query of the same type as the original.  We print "0
         * 0" here because technically there is no query of the matching tag type,
         * and printing a non-zero count for a different query type seems wrong,
         * e.g.  an INSERT that does an UPDATE instead should not print "0 1" if
         * one row was updated.  See QueryRewrite(), step 3, for details.
         */
        if (completionTag && completionTag[0] == '\0')//操作提示
        {
            if (portal->commandTag)
                strcpy(completionTag, portal->commandTag);
            if (strcmp(completionTag, "SELECT") == 0)
                sprintf(completionTag, "SELECT 0 0");
            else if (strcmp(completionTag, "INSERT") == 0)
                strcpy(completionTag, "INSERT 0 0");
            else if (strcmp(completionTag, "UPDATE") == 0)
                strcpy(completionTag, "UPDATE 0");
            else if (strcmp(completionTag, "DELETE") == 0)
                strcpy(completionTag, "DELETE 0");
        }
    }
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               2551
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(20,'PortalRun','PortalRun','PortalRun');
    
    

启动gdb，跟踪调试：  
_1、PortalRun_

    
    
    [root@localhost ~]# gdb -p 2551
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b PortalRun
    Breakpoint 1 at 0x8528af: file pquery.c, line 707.
    (gdb) c
    Continuing.
    
    Breakpoint 1, PortalRun (portal=0x2c6f490, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x2ccb4d8, altdest=0x2ccb4d8, completionTag=0x7ffe94ba4940 "") at pquery.c:707
    707     if (completionTag)
    #查看输入参数
    #1、portal
    (gdb) p *portal
    $1 = {name = 0x2c72e98 "", prepStmtName = 0x0, portalContext = 0x2cc1470, resowner = 0x2c3ad10, cleanup = 0x62f15c <PortalCleanup>, createSubid = 1, activeSubid = 1, 
      sourceText = 0x2c09ef0 "insert into t_insert values(20,'PortalRun','PortalRun','PortalRun');", commandTag = 0xb50908 "INSERT", stmts = 0x2ccb4a8, cplan = 0x0, portalParams = 0x0, queryEnv = 0x0, 
      strategy = PORTAL_MULTI_QUERY, cursorOptions = 4, run_once = false, status = PORTAL_READY, portalPinned = false, autoHeld = false, queryDesc = 0x0, tupDesc = 0x0, formats = 0x0, holdStore = 0x0, 
      holdContext = 0x0, holdSnapshot = 0x0, atStart = true, atEnd = true, portalPos = 0, creation_time = 587033564125509, visible = false}
    (gdb) p *(portal->portalContext)
    $2 = {type = T_AllocSetContext, isReset = true, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c6f380, firstchild = 0x0, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb8d2f1 "PortalContext", ident = 0x2c72e98 "", reset_cbs = 0x0}
    (gdb) p *(portal->resowner)
    $3 = {parent = 0x2c35518, firstchild = 0x0, nextchild = 0x0, name = 0xb8d2ff "Portal", bufferarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, catrefarr = {
        itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, catlistrefarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, 
      relrefarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, planrefarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, 
      tupdescarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, snapshotarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, 
        lastidx = 0}, filearr = {itemsarr = 0x0, invalidval = 18446744073709551615, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, dsmarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, 
        nitems = 0, maxitems = 0, lastidx = 0}, jitarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, nlocks = 0, locks = {0x0 <repeats 15 times>}}
    (gdb) p *(portal->resowner->parent)
    $4 = {parent = 0x0, firstchild = 0x2c3ad10, nextchild = 0x0, name = 0xa1b515 "TopTransaction", bufferarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, 
      catrefarr = {itemsarr = 0x2c3b040, invalidval = 0, capacity = 16, nitems = 0, maxitems = 16, lastidx = 4294967295}, catlistrefarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, 
        maxitems = 0, lastidx = 0}, relrefarr = {itemsarr = 0x2c35728, invalidval = 0, capacity = 16, nitems = 0, maxitems = 16, lastidx = 4294967295}, planrefarr = {itemsarr = 0x0, invalidval = 0, 
        capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, tupdescarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, snapshotarr = {itemsarr = 0x0, 
        invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, filearr = {itemsarr = 0x0, invalidval = 18446744073709551615, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, dsmarr = {
        itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, jitarr = {itemsarr = 0x0, invalidval = 0, capacity = 0, nitems = 0, maxitems = 0, lastidx = 0}, nlocks = 1, 
      locks = {0x2c28320, 0x0 <repeats 14 times>}}
    #2、count
    (gdb) p count
    $5 = 9223372036854775807
    #3、isTopLevel
    (gdb) p isTopLevel
    $6 = true
    #4、run_once
    (gdb) p run_once
    $7 = true
    #5、dest
    $8 = (DestReceiver *) 0x2ccb4d8
    (gdb) p *dest
    $9 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    (gdb) 
    #6、altdest
    (gdb) p altdest
    $10 = (DestReceiver *) 0x2ccb4d8
    (gdb) p *altdest
    $11 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    (gdb) 
    #7、completionTag
    (gdb) p completionTag
    $12 = 0x7ffe94ba4940 ""
    #单步调试
    710     if (log_executor_stats && portal->strategy != PORTAL_MULTI_QUERY)
    (gdb) 
    720     MarkPortalActive(portal);
    (gdb) 
    724     portal->run_once = run_once;
    (gdb) 
    740     saveTopTransactionResourceOwner = TopTransactionResourceOwner;
    (gdb) 
    741     saveTopTransactionContext = TopTransactionContext;
    (gdb) 
    742     saveActivePortal = ActivePortal;
    (gdb) 
    743     saveResourceOwner = CurrentResourceOwner;
    (gdb) 
    744     savePortalContext = PortalContext;
    (gdb) 
    745     saveMemoryContext = CurrentMemoryContext;
    (gdb) 
    746     PG_TRY();
    (gdb) 
    748         ActivePortal = portal;
    (gdb) 
    749         if (portal->resowner)
    (gdb) 
    750             CurrentResourceOwner = portal->resowner;
    (gdb) 
    751         PortalContext = portal->portalContext;
    (gdb) 
    753         MemoryContextSwitchTo(PortalContext);
    (gdb) 
    755         switch (portal->strategy)
    (gdb) p portal->strategy
    $13 = PORTAL_MULTI_QUERY
    (gdb) next
    799                 PortalRunMulti(portal, isTopLevel, false,
    (gdb) 
    803                 MarkPortalDone(portal);
    (gdb) 
    806                 result = true;
    (gdb) 
    807                 break;
    (gdb) 
    835     PG_END_TRY();
    (gdb) 
    837     if (saveMemoryContext == saveTopTransactionContext)
    (gdb) 
    838         MemoryContextSwitchTo(TopTransactionContext);
    (gdb) 
    841     ActivePortal = saveActivePortal;
    (gdb) 
    842     if (saveResourceOwner == saveTopTransactionResourceOwner)
    (gdb) 
    843         CurrentResourceOwner = TopTransactionResourceOwner;
    (gdb) 
    846     PortalContext = savePortalContext;
    (gdb) 
    848     if (log_executor_stats && portal->strategy != PORTAL_MULTI_QUERY)
    (gdb) 
    853     return result;
    (gdb) 
    854 }
    #DONE!
    

_2、PortalRunMulti_

    
    
    (gdb) b PortalRunMulti
    Breakpoint 1 at 0x8533df: file pquery.c, line 1210.
    (gdb) c
    Continuing.
    
    Breakpoint 1, PortalRunMulti (portal=0x2c6f490, isTopLevel=true, setHoldSnapshot=false, dest=0x2cbe8f8, altdest=0x2cbe8f8, completionTag=0x7ffe94ba4940 "") at pquery.c:1210
    1210        bool        active_snapshot_set = false;
    #输入参数
    #1、portal
    (gdb) p *portal
    $1 = {name = 0x2c72e98 "", prepStmtName = 0x0, portalContext = 0x2c2d3d0, resowner = 0x2c3ad10, cleanup = 0x62f15c <PortalCleanup>, createSubid = 1, activeSubid = 1, 
      sourceText = 0x2c09ef0 "insert into t_insert values(21,'PortalRunMulti','PortalRunMulti','PortalRunMulti');", commandTag = 0xb50908 "INSERT", stmts = 0x2cbe8c8, cplan = 0x0, portalParams = 0x0, 
      queryEnv = 0x0, strategy = PORTAL_MULTI_QUERY, cursorOptions = 4, run_once = true, status = PORTAL_ACTIVE, portalPinned = false, autoHeld = false, queryDesc = 0x0, tupDesc = 0x0, formats = 0x0, 
      holdStore = 0x0, holdContext = 0x0, holdSnapshot = 0x0, atStart = true, atEnd = true, portalPos = 0, creation_time = 587034112962796, visible = false}
    (gdb) p *(portal->portalContext)
    $2 = {type = T_AllocSetContext, isReset = true, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c6f380, firstchild = 0x0, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb8d2f1 "PortalContext", ident = 0x2c72e98 "", reset_cbs = 0x0}
    #2、isTopLevel
    (gdb) p isTopLevel
    $3 = true
    #3、setHoldSnapshot
    (gdb) p setHoldSnapshot
    $4 = false
    #4、dest
    (gdb) p *dest
    $5 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    #5、altdest
    (gdb) p *altdest
    $6 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    #6、completionTag
    (gdb) p *completionTag
    $7 = 0 '\000'
    #单步调试
    ...
    (gdb) next
    1234            PlannedStmt *pstmt = lfirst_node(PlannedStmt, stmtlist_item);
    (gdb) 
    1239            CHECK_FOR_INTERRUPTS();
    (gdb) p *pstmt
    $12 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x2c0aff8, rtable = 0x2cbe7d8, resultRelations = 0x2cbe878, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x2cbe828, invalItems = 0x0, paramExecTypes = 0x2c31700, utilityStmt = 0x0, stmt_location = 0, stmt_len = 82}
    (gdb) next
    1241            if (pstmt->utilityStmt == NULL)
    (gdb) 
    1248                if (log_executor_stats)
    (gdb) 
    1257                if (!active_snapshot_set)
    (gdb) 
    1259                    Snapshot    snapshot = GetTransactionSnapshot();
    (gdb) 
    1262                    if (setHoldSnapshot)
    (gdb) p *snapshot
    $13 = {satisfies = 0x9f73fc <HeapTupleSatisfiesMVCC>, xmin = 1612887, xmax = 1612887, xip = 0x2c2d1c0, xcnt = 0, subxip = 0x2c81c70, subxcnt = 0, suboverflowed = false, takenDuringRecovery = false, 
      copied = false, curcid = 0, speculativeToken = 0, active_count = 0, regd_count = 0, ph_node = {first_child = 0x0, next_sibling = 0x0, prev_or_parent = 0x0}, whenTaken = 0, lsn = 0}
    (gdb) 
    ...
    (gdb) 
    PortalRun (portal=0x2c6f490, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x2cbe8f8, altdest=0x2cbe8f8, completionTag=0x7ffe94ba4940 "INSERT 0 1") at pquery.c:803
    803                 MarkPortalDone(portal);
    #DONE!
    
    

### 四、小结

1、Portal：门户，类似设计模式中的Facade，对外的统一接口。该数据结构如何构造，有待接下来的上层函数解读。

