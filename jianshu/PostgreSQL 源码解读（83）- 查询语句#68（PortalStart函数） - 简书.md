本节介绍了PortalStart函数，该函数在create_simple_query中被调用，用于执行前初始化portal结构体中的相关信息。

### 一、数据结构

**Portal**  
包括场景PortalStrategy枚举定义/PortalStatus状态定义/PortalData结构体.Portal是PortalData结构体指针,详见代码注释.

    
    
    /*
     * We have several execution strategies for Portals, depending on what
     * query or queries are to be executed.  (Note: in all cases, a Portal
     * executes just a single source-SQL query, and thus produces just a
     * single result from the user's viewpoint.  However, the rule rewriter
     * may expand the single source query to zero or many actual queries.)
     * 对于Portals(客户端请求)，有几种执行策略，具体取决于要执行什么查询。
     * (注意:无论什么情况下，一个Portal只执行一个source-SQL查询，因此从用户的角度来看只产生一个结果。
     * 但是，规则重写器可以将单个源查询扩展为零或多个实际查询。
     * 
     * PORTAL_ONE_SELECT: the portal contains one single SELECT query.  We run
     * the Executor incrementally as results are demanded.  This strategy also
     * supports holdable cursors (the Executor results can be dumped into a
     * tuplestore for access after transaction completion).
     * PORTAL_ONE_SELECT: 包含一个SELECT查询。
     *                    按需要的结果重复(递增)地运行执行器。
     *                    该策略还支持可持有游标(执行器结果可以在事务完成后转储到tuplestore中进行访问)。
     * 
     * PORTAL_ONE_RETURNING: the portal contains a single INSERT/UPDATE/DELETE
     * query with a RETURNING clause (plus possibly auxiliary queries added by
     * rule rewriting).  On first execution, we run the portal to completion
     * and dump the primary query's results into the portal tuplestore; the
     * results are then returned to the client as demanded.  (We can't support
     * suspension of the query partway through, because the AFTER TRIGGER code
     * can't cope, and also because we don't want to risk failing to execute
     * all the auxiliary queries.)
     * PORTAL_ONE_RETURNING: 包含一个带有RETURNING子句的INSERT/UPDATE/DELETE查询
                             (可能还包括由规则重写添加的辅助查询)。
     *                       在第一次执行时，运行Portal来完成并将主查询的结果转储到Portal的tuplestore中;
     *                       然后根据需要将结果返回给客户端。
     *                       (我们不能支持半途中断的查询，因为AFTER触发器代码无法处理，
     *                       也因为不想冒执行所有辅助查询失败的风险)。
     * 
     * PORTAL_ONE_MOD_WITH: the portal contains one single SELECT query, but
     * it has data-modifying CTEs.  This is currently treated the same as the
     * PORTAL_ONE_RETURNING case because of the possibility of needing to fire
     * triggers.  It may act more like PORTAL_ONE_SELECT in future.
     * PORTAL_ONE_MOD_WITH: 只包含一个SELECT查询，但它具有数据修改的CTEs。
     *                      这与PORTAL_ONE_RETURNING的情况相同，因为可能需要触发触发器。将来它的行为可能更像PORTAL_ONE_SELECT。
     * 
     * PORTAL_UTIL_SELECT: the portal contains a utility statement that returns
     * a SELECT-like result (for example, EXPLAIN or SHOW).  On first execution,
     * we run the statement and dump its results into the portal tuplestore;
     * the results are then returned to the client as demanded.
     * PORTAL_UTIL_SELECT: 包含一个实用程序语句，该语句返回一个类似SELECT的结果(例如，EXPLAIN或SHOW)。
     *                     在第一次执行时，运行语句并将其结果转储到portal tuplestore;然后根据需要将结果返回给客户端。
     * 
     * PORTAL_MULTI_QUERY: all other cases.  Here, we do not support partial
     * execution: the portal's queries will be run to completion on first call.
     * PORTAL_MULTI_QUERY: 除上述情况外的其他情况。
     *                     在这里，不支持部分执行:Portal的查询语句将在第一次调用时运行到完成。
     */
    typedef enum PortalStrategy
    {
        PORTAL_ONE_SELECT,
        PORTAL_ONE_RETURNING,
        PORTAL_ONE_MOD_WITH,
        PORTAL_UTIL_SELECT,
        PORTAL_MULTI_QUERY
    } PortalStrategy;
    
    /*
     * A portal is always in one of these states.  It is possible to transit
     * from ACTIVE back to READY if the query is not run to completion;
     * otherwise we never back up in status.
     * Portal总是处于这些状态中的之一。
     * 如果查询没有运行到完成，则可以从活动状态转回准备状态;否则永远不会后退。
     */
    typedef enum PortalStatus
    {
        PORTAL_NEW,                 /* 刚创建;freshly created */
        PORTAL_DEFINED,             /* PortalDefineQuery完成;PortalDefineQuery done */
        PORTAL_READY,               /* PortalStart完成;PortalStart complete, can run it */
        PORTAL_ACTIVE,              /* Portal正在运行;portal is running (can't delete it) */
        PORTAL_DONE,                /* Portal已经完成;portal is finished (don't re-run it) */
        PORTAL_FAILED               /* Portal出现错误;portal got error (can't re-run it) */
    } PortalStatus;
    
    typedef struct PortalData *Portal;//结构体指针
    
    typedef struct PortalData
    {
        /* Bookkeeping data */
        const char *name;           /* portal的名称;portal's name */
        const char *prepStmtName;   /* 已完成准备的源语句;source prepared statement (NULL if none) */
        MemoryContext portalContext;    /* 内存上下文;subsidiary memory for portal */
        ResourceOwner resowner;     /* 资源的owner;resources owned by portal */
        void        (*cleanup) (Portal portal); /* cleanup钩子函数;cleanup hook */
    
        /*
         * State data for remembering which subtransaction(s) the portal was
         * created or used in.  If the portal is held over from a previous
         * transaction, both subxids are InvalidSubTransactionId.  Otherwise,
         * createSubid is the creating subxact and activeSubid is the last subxact
         * in which we ran the portal.
         * 状态数据，用于记住在哪个子事务中创建或使用Portal。
         * 如果Portal是从以前的事务中持有的，那么两个subxids都应该是InvalidSubTransactionId。
         * 否则，createSubid是正在创建的subxact，而activeSubid是运行Portal的最后一个subxact。
         */
        SubTransactionId createSubid;   /* 正在创建的subxact;the creating subxact */
        SubTransactionId activeSubid;   /* 活动的最后一个subxact;the last subxact with activity */
    
        /* The query or queries the portal will execute */
        //portal将会执行的查询
        const char *sourceText;     /* 查询的源文本;text of query (as of 8.4, never NULL) */
        const char *commandTag;     /* 源查询的命令tag;command tag for original query */
        List       *stmts;          /* PlannedStmt链表;list of PlannedStmts */
        CachedPlan *cplan;          /* 缓存的PlannedStmts;CachedPlan, if stmts are from one */
    
        ParamListInfo portalParams; /* 传递给查询的参数;params to pass to query */
        QueryEnvironment *queryEnv; /* 查询的执行环境;environment for query */
    
        /* Features/options */
        PortalStrategy strategy;    /* 场景;see above */
        int         cursorOptions;  /* DECLARE CURSOR选项位;DECLARE CURSOR option bits */
        bool        run_once;       /* 是否只执行一次;portal will only be run once */
    
        /* Status data */
        PortalStatus status;        /* Portal的状态;see above */
        bool        portalPinned;   /* 是否不能被清除;a pinned portal can't be dropped */
        bool        autoHeld;       /* 是否自动从pinned到held;was automatically converted from pinned to
                                     * held (see HoldPinnedPortals()) */
    
        /* If not NULL, Executor is active; call ExecutorEnd eventually: */
        //如不为NULL,执行器处于活动状态
        QueryDesc  *queryDesc;      /* 执行器需要使用的信息;info needed for executor invocation */
    
        /* If portal returns tuples, this is their tupdesc: */
        //如Portal需要返回元组,这是元组的描述
        TupleDesc   tupDesc;        /* 结果元组的描述;descriptor for result tuples */
        /* and these are the format codes to use for the columns: */
        //列信息的格式码
        int16      *formats;        /* 每一列的格式码;a format code for each column */
    
        /*
         * Where we store tuples for a held cursor or a PORTAL_ONE_RETURNING or
         * PORTAL_UTIL_SELECT query.  (A cursor held past the end of its
         * transaction no longer has any active executor state.)
         * 在这里，为持有的游标或PORTAL_ONE_RETURNING或PORTAL_UTIL_SELECT存储元组。
         * (在事务结束后持有的游标不再具有任何活动执行器状态。)
         */
        Tuplestorestate *holdStore; /* 存储持有的游标信息;store for holdable cursors */
        MemoryContext holdContext;  /* 持有holdStore的内存上下文;memory containing holdStore */
    
        /*
         * Snapshot under which tuples in the holdStore were read.  We must keep a
         * reference to this snapshot if there is any possibility that the tuples
         * contain TOAST references, because releasing the snapshot could allow
         * recently-dead rows to be vacuumed away, along with any toast data
         * belonging to them.  In the case of a held cursor, we avoid needing to
         * keep such a snapshot by forcibly detoasting the data.
         * 读取holdStore中元组的Snapshot。
         * 如果元组包含TOAST引用的可能性存在，那么必须保持对该快照的引用，
         * 因为释放快照可能会使最近废弃的行与属于它们的TOAST数据一起被清除。
         * 对于持有的游标，通过强制解压数据来避免需要保留这样的快照。
         */
        Snapshot    holdSnapshot;   /* 已注册的快照信息,如无则为NULL;registered snapshot, or NULL if none */
    
        /*
         * atStart, atEnd and portalPos indicate the current cursor position.
         * portalPos is zero before the first row, N after fetching N'th row of
         * query.  After we run off the end, portalPos = # of rows in query, and
         * atEnd is true.  Note that atStart implies portalPos == 0, but not the
         * reverse: we might have backed up only as far as the first row, not to
         * the start.  Also note that various code inspects atStart and atEnd, but
         * only the portal movement routines should touch portalPos.
         * atStart、atEnd和portalPos表示当前光标的位置。
         * portalPos在第一行之前为0，在获取第N行查询后为N。
         * 在运行结束后，portalPos = #查询中的行号，atEnd为T。
         * 注意，atStart表示portalPos == 0，但不是相反:我们可能只回到到第一行，而不是开始。
         * 还要注意，各种代码在开始和结束时都要检查，但是只有Portal移动例程应该访问portalPos。
         */
        bool        atStart;//处于开始位置?
        bool        atEnd;//处于结束位置?
        uint64      portalPos;//实际行号
    
        /* Presentation data, primarily used by the pg_cursors system view */
        //用于表示的数据，主要由pg_cursors系统视图使用
        TimestampTz creation_time;  /* portal定义的时间;time at which this portal was defined */
        bool        visible;        /* 是否在pg_cursors中可见? include this portal in pg_cursors? */
    }           PortalData;
    
    /*
     * PortalIsValid
     *      True iff portal is valid.
     *      判断Portal是否有效
     */
    #define PortalIsValid(p) PointerIsValid(p)
    
    

**QueryDesc**  
QueryDesc封装了执行器执行查询所需的所有内容。

    
    
    /* ----------------     *      query descriptor:
     *
     *  a QueryDesc encapsulates everything that the executor
     *  needs to execute the query.
     *  QueryDesc封装了执行器执行查询所需的所有内容。
     *
     *  For the convenience of SQL-language functions, we also support QueryDescs
     *  containing utility statements; these must not be passed to the executor
     *  however.
     *  为了使用SQL函数，还需要支持包含实用语句的QueryDescs;
     *  但是，这些内容不能传递给执行程序。
     * ---------------------     */
    typedef struct QueryDesc
    {
        /* These fields are provided by CreateQueryDesc */
        //以下变量由CreateQueryDesc函数设置
        CmdType     operation;      /* 操作类型,如CMD_SELECT等;CMD_SELECT, CMD_UPDATE, etc. */
        PlannedStmt *plannedstmt;   /* 已规划的语句,规划器的输出;planner's output (could be utility, too) */
        const char *sourceText;     /* 源SQL文本;source text of the query */
        Snapshot    snapshot;       /* 查询使用的快照;snapshot to use for query */
        Snapshot    crosscheck_snapshot;    /* RI 更新/删除交叉检查快照;crosscheck for RI update/delete */
        DestReceiver *dest;         /* 元组输出的接收器;the destination for tuple output */
        ParamListInfo params;       /* 需传入的参数值;param values being passed in */
        QueryEnvironment *queryEnv; /* 查询环境变量;query environment passed in */
        int         instrument_options; /* InstrumentOption选项;OR of InstrumentOption flags */
    
        /* These fields are set by ExecutorStart */
        //以下变量由ExecutorStart函数设置
        TupleDesc   tupDesc;        /* 结果元组tuples描述;descriptor for result tuples */
        EState     *estate;         /* 执行器状态;executor's query-wide state */
        PlanState  *planstate;      /* per-plan-node状态树;tree of per-plan-node state */
    
        /* This field is set by ExecutorRun */
        //以下变量由ExecutorRun设置
        bool        already_executed;   /* 先前已执行,则为T;true if previously executed */
    
        /* This is always set NULL by the core system, but plugins can change it */
        //内核设置为NULL,可由插件修改
        struct Instrumentation *totaltime;  /* ExecutorRun函数所花费的时间;total time spent in ExecutorRun */
    } QueryDesc;
    
    

### 二、源码解读

**PortalStart**  
PortalStart函数的作用是在执行SQL语句前初始化portal结构体中的相关信息，其中有2个重要数据结构的初始化:  
1.调用CreateQueryDesc函数结构体QueryDesc  
2.调用ExecutorStart函数初始化结构体EState,ExecutorStart函数调用InitPlan(下一节介绍)初始化计划状态树.

    
    
    /*
     * PortalStart
     *      Prepare a portal for execution.
     *      执行前初始化portal结构体中的相关信息
     * 
     * Caller must already have created the portal, done PortalDefineQuery(),
     * and adjusted portal options if needed.
     * 调用者必须已经创建了portal,完成PortalDefineQuery()函数的调用,并且已经调整了portal中的相关选项
     * 
     * If parameters are needed by the query, they must be passed in "params"
     * (caller is responsible for giving them appropriate lifetime).
     * 如果查询需要提供参数,通过"params"参数传入
     * (调用者负责参数的生命周期管理)
     * 
     * The caller can also provide an initial set of "eflags" to be passed to
     * ExecutorStart (but note these can be modified internally, and they are
     * currently only honored for PORTAL_ONE_SELECT portals).  Most callers
     * should simply pass zero.
     * 调用者需要提供"eflags"变量的初始化集合,该参数用于传递给函数ExecutorStart
     * (要注意eflags可以在内部修改,它们目前只在PORTAL_ONE_SELECT中才会被使用)
     * 大多数的调用者只应该传递参数0
     *
     * The caller can optionally pass a snapshot to be used; pass InvalidSnapshot
     * for the normal behavior of setting a new snapshot.  This parameter is
     * presently ignored for non-PORTAL_ONE_SELECT portals (it's only intended
     * to be used for cursors).
     * 调用者可以选择传递要使用的快照;为设置新快照的正常行为传递InvalidSnapshot。
     * 这个参数目前仅用于PORTAL_ONE_SELECT使用(用于游标)。
     * 
     * On return, portal is ready to accept PortalRun() calls, and the result
     * tupdesc (if any) is known.
     * 该函数返回时,portal已做好接收PortalRun()调用返回的准备,结果tupdesc是已知的.
     */
    void
    PortalStart(Portal portal, ParamListInfo params,
                int eflags, Snapshot snapshot)
    {
        Portal      saveActivePortal;
        ResourceOwner saveResourceOwner;
        MemoryContext savePortalContext;
        MemoryContext oldContext;
        QueryDesc  *queryDesc;
        int         myeflags;
    
        AssertArg(PortalIsValid(portal));
        AssertState(portal->status == PORTAL_DEFINED);
    
        /*
         * Set up global portal context pointers.
         * 设置全局portal上下文指针
         */
        //保护"现场"
        saveActivePortal = ActivePortal;
        saveResourceOwner = CurrentResourceOwner;
        savePortalContext = PortalContext;
        PG_TRY();
        {
            ActivePortal = portal;
            if (portal->resowner)
                CurrentResourceOwner = portal->resowner;
            PortalContext = portal->portalContext;
    
            oldContext = MemoryContextSwitchTo(PortalContext);
    
            /* Must remember portal param list, if any */
            //记录传递的参数信息
            portal->portalParams = params;
    
            /*
             * Determine the portal execution strategy
             * 确定portal执行场景
             */
            portal->strategy = ChoosePortalStrategy(portal->stmts);
    
            /*
             * Fire her up according to the strategy
             * 根据场景触发相应的处理
             */
            switch (portal->strategy)
            {
                case PORTAL_ONE_SELECT://PORTAL_ONE_SELECT
    
                    /* Must set snapshot before starting executor. */
                    //在开始执行前必须设置快照snapshot
                    if (snapshot)
                        PushActiveSnapshot(snapshot);
                    else
                        PushActiveSnapshot(GetTransactionSnapshot());
    
                    /*
                     * Create QueryDesc in portal's context; for the moment, set
                     * the destination to DestNone.
                     * 在portal上下文中创建QueryDesc,同时设置接收的目标为DestNone
                     */
                    queryDesc = CreateQueryDesc(linitial_node(PlannedStmt, portal->stmts),
                                                portal->sourceText,
                                                GetActiveSnapshot(),
                                                InvalidSnapshot,
                                                None_Receiver,
                                                params,
                                                portal->queryEnv,
                                                0);
    
                    /*
                     * If it's a scrollable cursor, executor needs to support
                     * REWIND and backwards scan, as well as whatever the caller
                     * might've asked for.
                     * 游标可滚动,执行器需要支持REWIND和向后的扫描
                     */
                    if (portal->cursorOptions & CURSOR_OPT_SCROLL)
                        myeflags = eflags | EXEC_FLAG_REWIND | EXEC_FLAG_BACKWARD;
                    else
                        myeflags = eflags;
    
                    /*
                     * Call ExecutorStart to prepare the plan for execution
                     * 调用ExecutorStart,为执行做准备
                     */
                    ExecutorStart(queryDesc, myeflags);
    
                    /*
                     * This tells PortalCleanup to shut down the executor
                     * 告知PortalCleanup关闭执行器
                     */
                    portal->queryDesc = queryDesc;
    
                    /*
                     * Remember tuple descriptor (computed by ExecutorStart)
                     * 记录tuple描述符(queryDesc->tupDesc)
                     */
                    portal->tupDesc = queryDesc->tupDesc;
    
                    /*
                     * Reset cursor position data to "start of query"
                     * 重置游标位置数据为"开始查询"
                     */
                    portal->atStart = true;//开始的位置
                    portal->atEnd = false;  /* 允许可获取数据;allow fetches */
                    portal->portalPos = 0;//游标位置
    
                    PopActiveSnapshot();
                    break;
    
                case PORTAL_ONE_RETURNING:
                case PORTAL_ONE_MOD_WITH:
    
                    /*
                     * We don't start the executor until we are told to run the
                     * portal.  We do need to set up the result tupdesc.
                     * 执行器在调用的时候才会启动,需要配置结果tupdesc.
                     * 
                     */
                    {
                        PlannedStmt *pstmt;
    
                        pstmt = PortalGetPrimaryStmt(portal);//获取主stmt
                        portal->tupDesc =
                            ExecCleanTypeFromTL(pstmt->planTree->targetlist,
                                                false);//设置元组描述符
                    }
    
                    /*
                     * Reset cursor position data to "start of query"
                     * 重置游标位置
                     */
                    portal->atStart = true;
                    portal->atEnd = false;  /* allow fetches */
                    portal->portalPos = 0;
                    break;
    
                case PORTAL_UTIL_SELECT://PORTAL_UTIL_SELECT
    
                    /*
                     * We don't set snapshot here, because PortalRunUtility will
                     * take care of it if needed.
                     */
                    {
                        PlannedStmt *pstmt = PortalGetPrimaryStmt(portal);
    
                        Assert(pstmt->commandType == CMD_UTILITY);
                        portal->tupDesc = UtilityTupleDescriptor(pstmt->utilityStmt);
                    }
    
                    /*
                     * Reset cursor position data to "start of query"
                     */
                    portal->atStart = true;
                    portal->atEnd = false;  /* allow fetches */
                    portal->portalPos = 0;
                    break;
    
                case PORTAL_MULTI_QUERY://PORTAL_MULTI_QUERY
                    /* Need do nothing now */
                    portal->tupDesc = NULL;
                    break;
            }
        }
        PG_CATCH();
        {
            /* Uncaught error while executing portal: mark it dead */
            MarkPortalFailed(portal);
    
            /* Restore global vars and propagate error */
            ActivePortal = saveActivePortal;
            CurrentResourceOwner = saveResourceOwner;
            PortalContext = savePortalContext;
    
            PG_RE_THROW();
        }
        PG_END_TRY();
    
        MemoryContextSwitchTo(oldContext);
    
        ActivePortal = saveActivePortal;
        CurrentResourceOwner = saveResourceOwner;
        PortalContext = savePortalContext;
    
        portal->status = PORTAL_READY;
    }
    
    
    /*
     * ChoosePortalStrategy
     *      Select portal execution strategy given the intended statement list.
     *      根据预期的语句链表选择portal执行策略。
     *
     * The list elements can be Querys or PlannedStmts.
     * That's more general than portals need, but plancache.c uses this too.
     * 链表中的元素可以是Query或者是PlannedStmt.
     * 这比portal需要的更普遍，plancache.c也用这个。
     * 
     *
     * See the comments in portal.h.
     * 参见portal.h中的注释.
     */
    PortalStrategy
    ChoosePortalStrategy(List *stmts)
    {
        int         nSetTag;
        ListCell   *lc;
    
        /*
         * PORTAL_ONE_SELECT and PORTAL_UTIL_SELECT need only consider the
         * single-statement case, since there are no rewrite rules that can add
         * auxiliary queries to a SELECT or a utility command. PORTAL_ONE_MOD_WITH
         * likewise allows only one top-level statement.
         * PORTAL_ONE_SELECT和PORTAL_UTIL_SELECT只需要考虑单语句的情况，
         * 因为没有可以向SELECT或实用程序命令添加辅助查询的重写规则。
         * PORTAL_ONE_MOD_WITH同样只允许一个最上层语句。
         */
        if (list_length(stmts) == 1)//只有1条语句
        {
            Node       *stmt = (Node *) linitial(stmts);//获取stmt
    
            if (IsA(stmt, Query))//Query
            {
                Query      *query = (Query *) stmt;
    
                if (query->canSetTag)
                {
                    if (query->commandType == CMD_SELECT)//查询命令
                    {
                        if (query->hasModifyingCTE)
                            return PORTAL_ONE_MOD_WITH;//存在可更新的CTE-->PORTAL_ONE_MOD_WITH
                        else
                            return PORTAL_ONE_SELECT;//单个查询语句
                    }
                    if (query->commandType == CMD_UTILITY)//工具语句
                    {
                        if (UtilityReturnsTuples(query->utilityStmt))//返回元组
                            return PORTAL_UTIL_SELECT;//PORTAL_UTIL_SELECT
                        /* it can't be ONE_RETURNING, so give up */
                        return PORTAL_MULTI_QUERY;//返回PORTAL_MULTI_QUERY
                    }
                }
            }
            else if (IsA(stmt, PlannedStmt))//PlannedStmt,参见Query处理逻辑
            {
                PlannedStmt *pstmt = (PlannedStmt *) stmt;
    
                if (pstmt->canSetTag)
                {
                    if (pstmt->commandType == CMD_SELECT)
                    {
                        if (pstmt->hasModifyingCTE)
                            return PORTAL_ONE_MOD_WITH;
                        else
                            return PORTAL_ONE_SELECT;
                    }
                    if (pstmt->commandType == CMD_UTILITY)
                    {
                        if (UtilityReturnsTuples(pstmt->utilityStmt))
                            return PORTAL_UTIL_SELECT;
                        /* it can't be ONE_RETURNING, so give up */
                        return PORTAL_MULTI_QUERY;
                    }
                }
            }
            else
                elog(ERROR, "unrecognized node type: %d", (int) nodeTag(stmt));
        }
    
        //存在多条语句
        /*
         * PORTAL_ONE_RETURNING has to allow auxiliary queries added by rewrite.
         * Choose PORTAL_ONE_RETURNING if there is exactly one canSetTag query and
         * it has a RETURNING list.
         * PORTAL_ONE_RETURNING必须允许通过重写添加辅助查询。
         * 如果只有一个canSetTag查询，并且它有一个RETURNING链表，那么选择PORTAL_ONE_RETURNING。
         */
        nSetTag = 0;
        foreach(lc, stmts)//遍历
        {
            Node       *stmt = (Node *) lfirst(lc);
    
            if (IsA(stmt, Query))
            {
                Query      *query = (Query *) stmt;
    
                if (query->canSetTag)
                {
                    if (++nSetTag > 1)
                        return PORTAL_MULTI_QUERY;  /* no need to look further */
                    if (query->commandType == CMD_UTILITY ||
                        query->returningList == NIL)
                        return PORTAL_MULTI_QUERY;  /* no need to look further */
                }
            }
            else if (IsA(stmt, PlannedStmt))
            {
                PlannedStmt *pstmt = (PlannedStmt *) stmt;
    
                if (pstmt->canSetTag)
                {
                    if (++nSetTag > 1)
                        return PORTAL_MULTI_QUERY;  /* no need to look further */
                    if (pstmt->commandType == CMD_UTILITY ||
                        !pstmt->hasReturning)
                        return PORTAL_MULTI_QUERY;  /* no need to look further */
                }
            }
            else
                elog(ERROR, "unrecognized node type: %d", (int) nodeTag(stmt));
        }
        if (nSetTag == 1)
            return PORTAL_ONE_RETURNING;
    
        /* Else, it's the general case... */
        //通常的情况
        return PORTAL_MULTI_QUERY;
    }
    
    
    /*
     * CreateQueryDesc
     * 构造QueryDesc结构体
     */
    QueryDesc *
    CreateQueryDesc(PlannedStmt *plannedstmt,
                    const char *sourceText,
                    Snapshot snapshot,
                    Snapshot crosscheck_snapshot,
                    DestReceiver *dest,
                    ParamListInfo params,
                    QueryEnvironment *queryEnv,
                    int instrument_options)
    {
        QueryDesc  *qd = (QueryDesc *) palloc(sizeof(QueryDesc));
    
        qd->operation = plannedstmt->commandType;   /* 操作类型;operation */
        qd->plannedstmt = plannedstmt;  /* 已规划的SQL语句;plan */
        qd->sourceText = sourceText;    /* 源SQL文本;query text */
        qd->snapshot = RegisterSnapshot(snapshot);  /* 快照;snapshot */
        /* RI check snapshot */
        qd->crosscheck_snapshot = RegisterSnapshot(crosscheck_snapshot);
        qd->dest = dest;            /* 输出的目标端;output dest */
        qd->params = params;        /* 传入到查询语句中的参数值;parameter values passed into query */
        qd->queryEnv = queryEnv;    //查询环境变量
        qd->instrument_options = instrument_options;    /* 是否需要instrumentation;instrumentation wanted? */
    
        /* null these fields until set by ExecutorStart */
        qd->tupDesc = NULL;//初始化为NULL
        qd->estate = NULL;
        qd->planstate = NULL;
        qd->totaltime = NULL;
    
        /* not yet executed */
        qd->already_executed = false;//未执行
    
        return qd;
    }
    
    
    /* ----------------------------------------------------------------     *      ExecutorStart
     *
     *      This routine must be called at the beginning of any execution of any
     *      query plan
     *      ExecutorStart必须在执行开始前调用.
     *
     * Takes a QueryDesc previously created by CreateQueryDesc (which is separate
     * only because some places use QueryDescs for utility commands).  The tupDesc
     * field of the QueryDesc is filled in to describe the tuples that will be
     * returned, and the internal fields (estate and planstate) are set up.
     * 获取先前由CreateQueryDesc创建的QueryDesc(该数据结构是独立的，只是因为有些地方使用QueryDesc来执行实用命令)。
     * 填充QueryDesc的tupDesc字段，以描述将要返回的元组，并设置内部字段(estate和planstate)。
     * 
     * eflags contains flag bits as described in executor.h.
     * eflags存储标志位(在executor.h中有说明)
     * 
     * NB: the CurrentMemoryContext when this is called will become the parent
     * of the per-query context used for this Executor invocation.
     * 注意:CurrentMemoryContext会成为每个执行查询的上下文的parent
     *
     * We provide a function hook variable that lets loadable plugins
     * get control when ExecutorStart is called.  Such a plugin would
     * normally call standard_ExecutorStart().
     * 我们提供了一个函数钩子变量，可以让可加载插件在调用ExecutorStart时获得控制权。
     * 这样的插件通常会调用standard_ExecutorStart()函数。
     *
     * ----------------------------------------------------------------     */
    void
    ExecutorStart(QueryDesc *queryDesc, int eflags)
    {
        if (ExecutorStart_hook)//存在钩子函数
            (*ExecutorStart_hook) (queryDesc, eflags);
        else
            standard_ExecutorStart(queryDesc, eflags);
    }
    
    void
    standard_ExecutorStart(QueryDesc *queryDesc, int eflags)
    {
        EState     *estate;
        MemoryContext oldcontext;
    
        /* sanity checks: queryDesc must not be started already */
        Assert(queryDesc != NULL);
        Assert(queryDesc->estate == NULL);
    
        /*
         * If the transaction is read-only, we need to check if any writes are
         * planned to non-temporary tables.  EXPLAIN is considered read-only.
         * 如果事务是只读的，需要检查是否计划对非临时表进行写操作。
         * EXPLAIN命令被认为是只读的。
         * 
         * Don't allow writes in parallel mode.  Supporting UPDATE and DELETE
         * would require (a) storing the combocid hash in shared memory, rather
         * than synchronizing it just once at the start of parallelism, and (b) an
         * alternative to heap_update()'s reliance on xmax for mutual exclusion.
         * INSERT may have no such troubles, but we forbid it to simplify the
         * checks.
         * 不要在并行模式下写。
         * 支持更新和删除需要:
         *   (a)在共享内存中存储combocid散列，而不是在并行性开始时只同步一次;
         *   (b) heap_update()依赖xmax实现互斥的替代方法。
         * INSERT可能没有这样的麻烦，但我们禁止它简化检查。
         * 
         * We have lower-level defenses in CommandCounterIncrement and elsewhere
         * against performing unsafe operations in parallel mode, but this gives a
         * more user-friendly error message.
         * 在CommandCounterIncrement和其他地方，对于在并行模式下执行不安全的操作，
         * PG有较低级别的防御，这里提供了更用户友好的错误消息。
         */
        if ((XactReadOnly || IsInParallelMode()) &&
            !(eflags & EXEC_FLAG_EXPLAIN_ONLY))
            ExecCheckXactReadOnly(queryDesc->plannedstmt);
    
        /*
         * Build EState, switch into per-query memory context for startup.
         * 构建EState,切换至每个查询的上下文中,准备开启执行
         */
        estate = CreateExecutorState();
        queryDesc->estate = estate;
    
        oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
    
        /*
         * Fill in external parameters, if any, from queryDesc; and allocate
         * workspace for internal parameters
         * 填充queryDesc的外部参数(如有);并为内部参数分配工作区
         */
        estate->es_param_list_info = queryDesc->params;
    
        if (queryDesc->plannedstmt->paramExecTypes != NIL)
        {
            int         nParamExec;
    
            nParamExec = list_length(queryDesc->plannedstmt->paramExecTypes);
            estate->es_param_exec_vals = (ParamExecData *)
                palloc0(nParamExec * sizeof(ParamExecData));
        }
    
        estate->es_sourceText = queryDesc->sourceText;
    
        /*
         * Fill in the query environment, if any, from queryDesc.
         * 填充查询执行环境,从queryDesc中获得
         */
        estate->es_queryEnv = queryDesc->queryEnv;
    
        /*
         * If non-read-only query, set the command ID to mark output tuples with
         * 非只读查询,设置命令ID
         */
        switch (queryDesc->operation)
        {
            case CMD_SELECT:
    
                /*
                 * SELECT FOR [KEY] UPDATE/SHARE and modifying CTEs need to mark
                 * tuples
                 * SELECT FOR [KEY] UPDATE/SHARE和正在更新的CTEs需要标记元组
                 */
                if (queryDesc->plannedstmt->rowMarks != NIL ||
                    queryDesc->plannedstmt->hasModifyingCTE)
                    estate->es_output_cid = GetCurrentCommandId(true);
    
                /*
                 * A SELECT without modifying CTEs can't possibly queue triggers,
                 * so force skip-triggers mode. This is just a marginal efficiency
                 * hack, since AfterTriggerBeginQuery/AfterTriggerEndQuery aren't
                 * all that expensive, but we might as well do it.
                 * 不带更新CTEs的SELECT不可能执行触发器,因此强制为EXEC_FLAG_SKIP_TRIGGERS标记.
                 * 这只是一个边际效益问题，因为AfterTriggerBeginQuery/AfterTriggerEndQuery成本并不高，但不妨这样做。
                 */
                if (!queryDesc->plannedstmt->hasModifyingCTE)
                    eflags |= EXEC_FLAG_SKIP_TRIGGERS;
                break;
    
            case CMD_INSERT:
            case CMD_DELETE:
            case CMD_UPDATE:
                estate->es_output_cid = GetCurrentCommandId(true);
                break;
    
            default:
                elog(ERROR, "unrecognized operation code: %d",
                     (int) queryDesc->operation);
                break;
        }
    
        /*
         * Copy other important information into the EState
         * 拷贝其他重要的信息到EState数据结构中
         */
        estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot);
        estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot);
        estate->es_top_eflags = eflags;
        estate->es_instrument = queryDesc->instrument_options;
        estate->es_jit_flags = queryDesc->plannedstmt->jitFlags;
    
        /*
         * Set up an AFTER-trigger statement context, unless told not to, or
         * unless it's EXPLAIN-only mode (when ExecutorFinish won't be called).
         * 设置AFTER-trigger语句上下文,除非明确不需要执行此操作或者是EXPLAIN-only模式
         */
        if (!(eflags & (EXEC_FLAG_SKIP_TRIGGERS | EXEC_FLAG_EXPLAIN_ONLY)))
            AfterTriggerBeginQuery();
    
        /*
         * Initialize the plan state tree
         * 初始化计划状态树
         */
        InitPlan(queryDesc, eflags);
    
        MemoryContextSwitchTo(oldcontext);
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# order by dw.dwbh;
                                            QUERY PLAN                                        
    ------------------------------------------------------------------------------------------     Sort  (cost=20070.93..20320.93 rows=100000 width=47)
       Sort Key: dw.dwbh
       ->  Hash Join  (cost=3754.00..8689.61 rows=100000 width=47)
             Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
             ->  Hash Join  (cost=3465.00..8138.00 rows=100000 width=31)
                   Hash Cond: ((jf.grbh)::text = (gr.grbh)::text)
                   ->  Seq Scan on t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                   ->  Hash  (cost=1726.00..1726.00 rows=100000 width=16)
                         ->  Seq Scan on t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
             ->  Hash  (cost=164.00..164.00 rows=10000 width=20)
                   ->  Seq Scan on t_dwxx dw  (cost=0.00..164.00 rows=10000 width=20)
    (11 rows)
    
    

启动gdb,设置断点,进入PortalStart函数

    
    
    (gdb) b PortalStart
    Breakpoint 1 at 0x8cb67b: file pquery.c, line 455.
    (gdb) c
    Continuing.
    
    Breakpoint 1, PortalStart (portal=0x25cd468, params=0x0, eflags=0, snapshot=0x0) at pquery.c:455
    455     AssertArg(PortalIsValid(portal));
    

校验并保护现场

    
    
    455     AssertArg(PortalIsValid(portal));
    (gdb) n
    456     AssertState(portal->status == PORTAL_DEFINED);
    (gdb) 
    461     saveActivePortal = ActivePortal;
    (gdb) 
    462     saveResourceOwner = CurrentResourceOwner;
    (gdb) 
    463     savePortalContext = PortalContext;
    

设置内存上下文,资源owner等信息

    
    
    466         ActivePortal = portal;
    (gdb) 
    467         if (portal->resowner)
    (gdb) 
    468             CurrentResourceOwner = portal->resowner;
    (gdb) 
    469         PortalContext = portal->portalContext;
    (gdb) 
    471         oldContext = MemoryContextSwitchTo(PortalContext);
    (gdb) 
    474         portal->portalParams = params;
    

场景为PORTAL_ONE_SELECT

    
    
    (gdb) p portal->strategy
    $1 = PORTAL_ONE_SELECT
    

根据strategy,进入相应的处理分支(PORTAL_ONE_SELECT)  
设置快照

    
    
    489                 if (snapshot)
    (gdb) 
    492                     PushActiveSnapshot(GetTransactionSnapshot());
    

创建QueryDesc结构体

    
    
    (gdb) 
    498                 queryDesc = CreateQueryDesc(linitial_node(PlannedStmt, portal->stmts),
    

查看queryDesc结构体信息

    
    
    (gdb) n
    512                 if (portal->cursorOptions & CURSOR_OPT_SCROLL)
    (gdb) p *queryDesc
    $2 = {operation = CMD_SELECT, plannedstmt = 0x2650df0, 
      sourceText = 0x2567eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      snapshot = 0x260ce10, crosscheck_snapshot = 0x0, dest = 0xf8f280 <donothingDR>, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x0, estate = 0x0, planstate = 0x0, already_executed = false, totaltime = 0x0}
    

设置标记位

    
    
    (gdb) n
    515                     myeflags = eflags;
    (gdb) p eflags
    $3 = 0
    

进入ExecutorStart函数

    
    
    (gdb) n
    147         standard_ExecutorStart(queryDesc, eflags);
    (gdb) step
    standard_ExecutorStart (queryDesc=0x2657f68, eflags=0) at execMain.c:157
    157     Assert(queryDesc != NULL);
    

ExecutorStart-->执行相关校验和判断

    
    
    157     Assert(queryDesc != NULL);
    (gdb) n
    158     Assert(queryDesc->estate == NULL);
    (gdb) 
    175     if ((XactReadOnly || IsInParallelMode()) &&
    

ExecutorStart-->创建EState,初始化EState结构体

    
    
    (gdb) 
    182     estate = CreateExecutorState();
    (gdb) 
    183     queryDesc->estate = estate;
    (gdb) p *estate
    $4 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x0, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x0, es_plannedstmt = 0x0, es_sourceText = 0x0, es_junkFilter = 0x0, es_output_cid = 0, 
      es_result_relations = 0x0, es_num_result_relations = 0, es_result_relation_info = 0x0, es_root_result_relations = 0x0, 
      es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x0, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, 
      es_param_exec_vals = 0x0, es_queryEnv = 0x0, es_query_cxt = 0x2653e30, es_tupleTable = 0x0, es_rowMarks = 0x0, 
      es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x0, 
      es_subplanstates = 0x0, es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, 
      es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0, es_jit_worker_instr = 0x0}
    

ExecutorStart-->EState结构体中的变量赋值

    
    
    (gdb) n
    185     oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
    (gdb) 
    191     estate->es_param_list_info = queryDesc->params;
    (gdb) 
    193     if (queryDesc->plannedstmt->paramExecTypes != NIL)
    (gdb) 
    202     estate->es_sourceText = queryDesc->sourceText;
    (gdb) 
    207     estate->es_queryEnv = queryDesc->queryEnv;
    

ExecutorStart-->根据queryDesc->operation的不同执行的处理

    
    
    (gdb) 
    212     switch (queryDesc->operation)
    (gdb) 
    220             if (queryDesc->plannedstmt->rowMarks != NIL ||
    (gdb) p queryDesc->operation
    $5 = CMD_SELECT
    (gdb) n
    221                 queryDesc->plannedstmt->hasModifyingCTE)
    (gdb) 
    220             if (queryDesc->plannedstmt->rowMarks != NIL ||
    (gdb) 
    230             if (!queryDesc->plannedstmt->hasModifyingCTE)
    (gdb) 
    231                 eflags |= EXEC_FLAG_SKIP_TRIGGERS;
    (gdb) 
    232             break;
    

ExecutorStart-->设置快照

    
    
    (gdb) n
    249     estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot);
    (gdb) p *queryDesc->snapshot
    $6 = {satisfies = 0xa923ca <HeapTupleSatisfiesMVCC>, xmin = 1689, xmax = 1689, xip = 0x0, xcnt = 0, subxip = 0x0, 
      subxcnt = 0, suboverflowed = false, takenDuringRecovery = false, copied = true, curcid = 0, speculativeToken = 0, 
      active_count = 1, regd_count = 1, ph_node = {first_child = 0x0, next_sibling = 0x0, prev_or_parent = 0x0}, whenTaken = 0, 
      lsn = 0}
    

ExecutorStart-->设置其他EState中的变量

    
    
    (gdb) n
    250     estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot);
    (gdb) p *queryDesc->crosscheck_snapshot
    Cannot access memory at address 0x0
    (gdb) n
    251     estate->es_top_eflags = eflags;
    (gdb) 
    252     estate->es_instrument = queryDesc->instrument_options;
    (gdb) 
    253     estate->es_jit_flags = queryDesc->plannedstmt->jitFlags;
    (gdb) 
    259     if (!(eflags & (EXEC_FLAG_SKIP_TRIGGERS | EXEC_FLAG_EXPLAIN_ONLY)))
    

ExecutorStart-->执行InitPlan

    
    
    (gdb) 
    265     InitPlan(queryDesc, eflags);
    (gdb) 
    267     MemoryContextSwitchTo(oldcontext);
    (gdb) 
    268 }
    

ExecutorStart-->查看QueryDesc和EState

    
    
    (gdb) p *queryDesc
    $7 = {operation = CMD_SELECT, plannedstmt = 0x2650df0, 
      sourceText = 0x2567eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      snapshot = 0x25e46c0, crosscheck_snapshot = 0x0, dest = 0xf8f280 <donothingDR>, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x2665058, estate = 0x2653f48, planstate = 0x2654160, already_executed = false, 
      totaltime = 0x0}
    (gdb) p *estate
    $8 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x25e46c0, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x264ec98, es_plannedstmt = 0x2650df0, 
      es_sourceText = 0x2567eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x0, es_num_result_relations = 0, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x0, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x0, 
      es_queryEnv = 0x0, es_query_cxt = 0x2653e30, es_tupleTable = 0x2654af8, es_rowMarks = 0x0, es_processed = 0, 
      es_lastoid = 0, es_top_eflags = 16, es_instrument = 0, es_finished = false, es_exprcontexts = 0x2654550, 
      es_subplanstates = 0x0, es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, 
      es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0, es_jit_worker_instr = 0x0}
    

ExecutorStart-->回到PortalStart

    
    
    (gdb) n
    ExecutorStart (queryDesc=0x2657f68, eflags=0) at execMain.c:148
    148 }
    (gdb) n
    PortalStart (portal=0x25cd468, params=0x0, eflags=0, snapshot=0x0) at pquery.c:525
    525                 portal->queryDesc = queryDesc;
    

设置portal中的变量atStart等

    
    
    525                 portal->queryDesc = queryDesc;
    (gdb) n
    530                 portal->tupDesc = queryDesc->tupDesc;
    (gdb) 
    535                 portal->atStart = true;
    (gdb) 
    536                 portal->atEnd = false;  /* allow fetches */
    (gdb) 
    537                 portal->portalPos = 0;
    (gdb) 
    539                 PopActiveSnapshot();
    (gdb) 
    540                 break;
    

执行完毕,回到exec_simple_query

    
    
    (gdb) 
    613     portal->status = PORTAL_READY;
    (gdb) 
    614 }
    (gdb) 
    exec_simple_query (
        query_string=0x2567eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>...) at postgres.c:1091
    warning: Source file is more recent than executable.
    1091            format = 0;             /* TEXT is default */
    (gdb) 
    

DONE!

### 四、参考资料

[postgres.c](https://doxygen.postgresql.org/postgres_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

