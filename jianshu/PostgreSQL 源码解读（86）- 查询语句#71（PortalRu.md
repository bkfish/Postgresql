本节介绍了PortalRun->PortalRunSelect函数的实现逻辑，该函数执行以PORTAL_ONE_SELECT模式运行的SQL。

### 一、数据结构

**Portal**  
对于Portals(客户端请求)，有几种执行策略，具体取决于要执行什么查询。  
(注意:无论什么情况下，一个Portal只执行一个source-SQL查询，因此从用户的角度来看只产生一个结果。

    
    
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

PortalRun->PortalRunSelect函数执行以PORTAL_ONE_SELECT模式运行的SQL.

    
    
    /*
     * PortalRunSelect
     *      Execute a portal's query in PORTAL_ONE_SELECT mode, and also
     *      when fetching from a completed holdStore in PORTAL_ONE_RETURNING,
     *      PORTAL_ONE_MOD_WITH, and PORTAL_UTIL_SELECT cases.
     *       执行以PORTAL_ONE_SELECT模式运行的SQL,同时处理PORTAL_ONE_RETURNING/
     *      PORTAL_ONE_MOD_WITH/PORTAL_UTIL_SELECT这几种模式下完成holdStore后的数据提取
     *
     * This handles simple N-rows-forward-or-backward cases.  For more complex
     * nonsequential access to a portal, see PortalRunFetch.
     * 这将处理简单的n行前向或后向情况。
     * 有关对门户的更复杂的非顺序访问，请参阅PortalRunFetch。
     * 
     * count <= 0 is interpreted as a no-op: the destination gets started up
     * and shut down, but nothing else happens.  Also, count == FETCH_ALL is
     * interpreted as "all rows".  (cf FetchStmt.howMany)
     * count <= 0被解释为一个no-op:目标启动并关闭，但是没有发生其他事情。
     * 另外，count == FETCH_ALL被解释为“所有行”。(cf FetchStmt.howMany)
     * 
     * Caller must already have validated the Portal and done appropriate
     * setup (cf. PortalRun).
     * 调用者必须完成Portal的校验以及相关的配置.
     *
     * Returns number of rows processed (suitable for use in result tag)
     * 返回已处理的行数.
     */
    static uint64
    PortalRunSelect(Portal portal,
                    bool forward,
                    long count,
                    DestReceiver *dest)
    {
        QueryDesc  *queryDesc;
        ScanDirection direction;
        uint64      nprocessed;
    
        /*
         * NB: queryDesc will be NULL if we are fetching from a held cursor or a
         * completed utility query; can't use it in that path.
         * 注意:从已持有的游标或者已完成的工具类查询中返回时,queryDesc有可能是NULL.
         */
        queryDesc = portal->queryDesc;
    
        /* Caller messed up if we have neither a ready query nor held data. */
        //确保queryDescbuweiNULL或者持有提取的数据
        Assert(queryDesc || portal->holdStore);
    
        /*
         * Force the queryDesc destination to the right thing.  This supports
         * MOVE, for example, which will pass in dest = DestNone.  This is okay to
         * change as long as we do it on every fetch.  (The  must not
         * assume that dest never changes.)
         * 确保queryDesc目的地是正确的地方。
         * 例如，它支持MOVE，它将传入dest = DestNone。
         * 只要在每次取回时都这样做，这是可以改变的。(Executor不能假定dest永不改变。)
         */
        if (queryDesc)
            queryDesc->dest = dest;//设置dest
    
        /*
         * Determine which direction to go in, and check to see if we're already
         * at the end of the available tuples in that direction.  If so, set the
         * direction to NoMovement to avoid trying to fetch any tuples.  (This
         * check exists because not all plan node types are robust about being
         * called again if they've already returned NULL once.)  Then call the
         * executor (we must not skip this, because the destination needs to see a
         * setup and shutdown even if no tuples are available).  Finally, update
         * the portal position state depending on the number of tuples that were
         * retrieved.
         * 确定要进入的方向，并检查是否已经在该方向的可用元组的末尾。
         * 如果是这样，则将方向设置为NoMovement，以避免试图再次获取任何元组。
         * (之所以存在这种检查，是因为不是所有的计划节点类型都能够在已经返回NULL时再次调用。)
         * 然后调用executor(我们不能跳过这一步，因为目标需要看到设置和关闭，即使没有元组可用)。
         * 最后，根据检索到的元组数量更新Portal的数据位置状态。
         */
        if (forward)//前向
        {
            if (portal->atEnd || count <= 0)
            {
                //已到末尾或者行计数小于等于0
                direction = NoMovementScanDirection;
                count = 0;          /* don't pass negative count to executor */
            }
            else
                direction = ForwardScanDirection;//前向扫描
    
            /* In the executor, zero count processes all rows */
            //在executor中,count=0意味着提取所有行
            if (count == FETCH_ALL)
                count = 0;
    
            if (portal->holdStore)
                //持有提取后的数据游标
                nprocessed = RunFromStore(portal, direction, (uint64) count, dest);
            else
            {
                //没有持有游标(数据)
                PushActiveSnapshot(queryDesc->snapshot);//快照入栈
                ExecutorRun(queryDesc, direction, (uint64) count,
                            portal->run_once);//开始执行
                nprocessed = queryDesc->estate->es_processed;//结果行数
                PopActiveSnapshot();//快照出栈
            }
    
            if (!ScanDirectionIsNoMovement(direction))//扫描方向可移动
            {
                if (nprocessed > 0)//扫描行数>0
                    portal->atStart = false;    /* 可以向前移动了;OK to go backward now */
                if (count == 0 || nprocessed < (uint64) count)
                    //count为0或者行数小于传入的计数器
                    portal->atEnd = true;   /* 已完成扫描;we retrieved 'em all */
                portal->portalPos += nprocessed;//位置移动(+处理行数)
            }
        }
        else//非前向(后向)
        {
            if (portal->cursorOptions & CURSOR_OPT_NO_SCROLL)//如游标不可移动,报错
                ereport(ERROR,
                        (errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
                         errmsg("cursor can only scan forward"),
                         errhint("Declare it with SCROLL option to enable backward scan.")));
    
            if (portal->atStart || count <= 0)
            {
                //处于开始或者count小于等于0
                direction = NoMovementScanDirection;
                count = 0;          /* don't pass negative count to executor */
            }
            else
                //往后扫描
                direction = BackwardScanDirection;
    
            /* In the executor, zero count processes all rows */
            //参见forward=T的注释
            if (count == FETCH_ALL)
                count = 0;
    
            if (portal->holdStore)
                nprocessed = RunFromStore(portal, direction, (uint64) count, dest);
            else
            {
                PushActiveSnapshot(queryDesc->snapshot);
                ExecutorRun(queryDesc, direction, (uint64) count,
                            portal->run_once);
                nprocessed = queryDesc->estate->es_processed;
                PopActiveSnapshot();
            }
    
            if (!ScanDirectionIsNoMovement(direction))
            {
                if (nprocessed > 0 && portal->atEnd)
                {
                    portal->atEnd = false;  /* OK to go forward now */
                    portal->portalPos++;    /* adjust for endpoint case */
                }
                if (count == 0 || nprocessed < (uint64) count)
                {
                    portal->atStart = true; /* we retrieved 'em all */
                    portal->portalPos = 0;
                }
                else
                {
                    portal->portalPos -= nprocessed;
                }
            }
        }
    
        return nprocessed;
    }
    
    
    /*
     * RunFromStore
     *      Fetch tuples from the portal's tuple store.
     *      从Portal的tuple store中提取元组.
     *
     * Calling conventions are similar to ExecutorRun, except that we
     * do not depend on having a queryDesc or estate.  Therefore we return the
     * number of tuples processed as the result, not in estate->es_processed.
     * 该函数的调用约定类似于ExecutorRun，只是不依赖于是否拥有queryDesc或estate。
     * 因此，返回处理的元组的数量作为结果，而不是在estate->es_processed中返回。
     * 
     * One difference from ExecutorRun is that the destination receiver functions
     * are run in the caller's memory context (since we have no estate).  Watch
     * out for memory leaks.
     * 与ExecutorRun不同的是，目标接收器函数在调用者的内存上下文中运行(因为没有estate)。
     * 需注意内存泄漏!!!
     */
    static uint64
    RunFromStore(Portal portal, ScanDirection direction, uint64 count,
                 DestReceiver *dest)
    {
        uint64      current_tuple_count = 0;
        TupleTableSlot *slot;//元组表slot
    
        slot = MakeSingleTupleTableSlot(portal->tupDesc);
    
        dest->rStartup(dest, CMD_SELECT, portal->tupDesc);//目标启动
    
        if (ScanDirectionIsNoMovement(direction))//无法移动
        {
            /* do nothing except start/stop the destination */
            //不需要做任何事情
        }
        else
        {
            bool        forward = ScanDirectionIsForward(direction);//是否前向扫描
    
            for (;;)//循环
            {
                MemoryContext oldcontext;//内存上下文
                bool        ok;
    
                oldcontext = MemoryContextSwitchTo(portal->holdContext);//切换至相应的内存上下文
    
                ok = tuplestore_gettupleslot(portal->holdStore, forward, false,
                                             slot);//获取元组
    
                MemoryContextSwitchTo(oldcontext);//切换回原上下文
    
                if (!ok)
                    break;//如出错,则跳出循环
    
                /*
                 * If we are not able to send the tuple, we assume the destination
                 * has closed and no more tuples can be sent. If that's the case,
                 * end the loop.
                 * 如果不能发送元组到目标端，那么我们假设目标端已经关闭，不能发送更多元组。
                 * 如果是这样，结束循环。
                 */
                if (!dest->receiveSlot(slot, dest))
                    break;
    
                ExecClearTuple(slot);//执行清理
    
                /*
                 * check our tuple count.. if we've processed the proper number
                 * then quit, else loop again and process more tuples. Zero count
                 * means no limit.
                 * 检查元组计数…如果处理了正确的计数，那么退出，
                 * 否则再次循环并处理更多元组。零计数意味着没有限制。
                 */
                current_tuple_count++;
                if (count && count == current_tuple_count)
                    break;
            }
        }
    
        dest->rShutdown(dest);//关闭目标端
    
        ExecDropSingleTupleTableSlot(slot);//清除slot
    
        return current_tuple_count;//返回行数
    }
     
    
    /* ----------------------------------------------------------------     *      ExecutorRun
     *      ExecutorRun函数
     *
     *      This is the main routine of the executor module. It accepts
     *      the query descriptor from the traffic cop and executes the
     *      query plan.
     *      这是executor模块的主要实现例程。它接受traffic cop的查询描述符并执行查询计划。
     *
     *      ExecutorStart must have been called already.
     *      在此之前,已调用ExecutorStart函数.
     *  
     *      If direction is NoMovementScanDirection then nothing is done
     *      except to start up/shut down the destination.  Otherwise,
     *      we retrieve up to 'count' tuples in the specified direction.
     *      如果方向是NoMovementScanDirection，那么除了启动/关闭目标之外什么也不做。
     *      否则，在指定的方向上检索指定数量“count”的元组。
     *
     *      Note: count = 0 is interpreted as no portal limit, i.e., run to
     *      completion.  Also note that the count limit is only applied to
     *      retrieved tuples, not for instance to those inserted/updated/deleted
     *      by a ModifyTable plan node.
     *      注意:count = 0被解释为没有限制，即，运行到完成。
     *      还要注意，计数限制只适用于检索到的元组，而不适用于由ModifyTable计划节点插入/更新/删除的元组。
     *
     *      There is no return value, but output tuples (if any) are sent to
     *      the destination receiver specified in the QueryDesc; and the number
     *      of tuples processed at the top level can be found in
     *      estate->es_processed.
     *      没有返回值，但是输出元组(如果有的话)被发送到QueryDesc中指定的目标接收器;
     *      在顶层处理的元组数量可以在estate-> es_processing中找到。
     *
     *      We provide a function hook variable that lets loadable plugins
     *      get control when ExecutorRun is called.  Such a plugin would
     *      normally call standard_ExecutorRun().
     *      我们提供了一个钩子函数变量，可以让插件在调用ExecutorRun时获得控制权。
     *      这样的插件通常会调用standard_ExecutorRun()函数。
     *
     * ----------------------------------------------------------------     */
    void
    ExecutorRun(QueryDesc *queryDesc,
                ScanDirection direction, uint64 count,
                bool execute_once)
    {
        if (ExecutorRun_hook)
            (*ExecutorRun_hook) (queryDesc, direction, count, execute_once);//钩子函数
        else
            standard_ExecutorRun(queryDesc, direction, count, execute_once);//标准函数
    }
    
    void
    standard_ExecutorRun(QueryDesc *queryDesc,
                         ScanDirection direction, uint64 count, bool execute_once)
    {
        EState     *estate;//全局执行状态
        CmdType     operation;//命令类型
        DestReceiver *dest;//接收器
        bool        sendTuples;//是否需要传输元组
        MemoryContext oldcontext;//内存上下文
    
        /* sanity checks */
        Assert(queryDesc != NULL);//校验queryDesc不能为NULL
    
        estate = queryDesc->estate;//获取执行状态
    
        Assert(estate != NULL);//执行状态不能为NULL
        Assert(!(estate->es_top_eflags & EXEC_FLAG_EXPLAIN_ONLY));//eflags标记不能为EXEC_FLAG_EXPLAIN_ONLY
    
        /*
         * Switch into per-query memory context
         * 切换内存上下文
         */
        oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
    
        /* Allow instrumentation of Executor overall runtime */
        //允许全程instrumentation
        if (queryDesc->totaltime)
            InstrStartNode(queryDesc->totaltime);
    
        /*
         * extract information from the query descriptor and the query feature.
         * 从查询描述符和查询特性中提取信息。
         */
        operation = queryDesc->operation;
        dest = queryDesc->dest;
    
        /*
         * startup tuple receiver, if we will be emitting tuples
         * 如需发送元组,则启动元组接收器
         */
        estate->es_processed = 0;
        estate->es_lastoid = InvalidOid;
    
        sendTuples = (operation == CMD_SELECT ||
                      queryDesc->plannedstmt->hasReturning);
    
        if (sendTuples)//如需发送元组
            dest->rStartup(dest, operation, queryDesc->tupDesc);
    
        /*
         * run plan
         * 执行Plan
         */
        if (!ScanDirectionIsNoMovement(direction))//如非ScanDirectionIsNoMovement
        {
            if (execute_once && queryDesc->already_executed)//校验
                elog(ERROR, "can't re-execute query flagged for single execution");
            queryDesc->already_executed = true;//修改标记
    
            ExecutePlan(estate,
                        queryDesc->planstate,
                        queryDesc->plannedstmt->parallelModeNeeded,
                        operation,
                        sendTuples,
                        count,
                        direction,
                        dest,
                        execute_once);//执行Plan
        }
    
        /*
         * shutdown tuple receiver, if we started it
         * 如启动了元组接收器,则关闭它
         */
        if (sendTuples)
            dest->rShutdown(dest);
    
        if (queryDesc->totaltime)//收集时间
            InstrStopNode(queryDesc->totaltime, estate->es_processed);
    
        MemoryContextSwitchTo(oldcontext);//切换内存上下文
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
    
    

启动gdb,设置断点,进入PortalRunSelect

    
    
    (gdb) b PortalRunSelect
    Breakpoint 1 at 0x8cc0e8: file pquery.c, line 888.
    (gdb) c
    Continuing.
    
    Breakpoint 1, PortalRunSelect (portal=0x1af2468, forward=true, count=9223372036854775807, dest=0x1b74668) at pquery.c:888
    warning: Source file is more recent than executable.
    888     queryDesc = portal->queryDesc;
    (gdb) 
    

查看输入参数portal&dest,forward为T表示前向扫描  
portal:未命名的Portal,holdStore为NULL,atStart = true, atEnd = false, portalPos = 0  
dest:接收器slot为printtup

    
    
    (gdb) p *portal
    $1 = {name = 0x1af5e90 "", prepStmtName = 0x0, portalContext = 0x1b795d0, resowner = 0x1abde80, 
      cleanup = 0x6711b6 <PortalCleanup>, createSubid = 1, activeSubid = 1, 
      sourceText = 0x1a8ceb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      commandTag = 0xc5eed5 "SELECT", stmts = 0x1b74630, cplan = 0x0, portalParams = 0x0, queryEnv = 0x0, 
      strategy = PORTAL_ONE_SELECT, cursorOptions = 4, run_once = true, status = PORTAL_ACTIVE, portalPinned = false, 
      autoHeld = false, queryDesc = 0x1b796e8, tupDesc = 0x1b867d8, formats = 0x1b79780, holdStore = 0x0, holdContext = 0x0, 
      holdSnapshot = 0x0, atStart = true, atEnd = false, portalPos = 0, creation_time = 595566906253867, visible = false}
    (gdb) p *dest
    $2 = {receiveSlot = 0x48cc00 <printtup>, rStartup = 0x48c5c1 <printtup_startup>, rShutdown = 0x48d02e <printtup_shutdown>, 
      rDestroy = 0x48d0a7 <printtup_destroy>, mydest = DestRemote}
    

校验并设置dest

    
    
    (gdb) n
    891     Assert(queryDesc || portal->holdStore);
    (gdb) 
    899     if (queryDesc)
    (gdb) 
    900         queryDesc->dest = dest;
    

前向扫描

    
    
    (gdb) n
    913     if (forward)
    (gdb) 
    915         if (portal->atEnd || count <= 0)
    

进入ExecutorRun

    
    
    ...
    (gdb) 
    932             ExecutorRun(queryDesc, direction, (uint64) count,
    (gdb) step
    ExecutorRun (queryDesc=0x1b796e8, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:304
    warning: Source file is more recent than executable.
    304     if (ExecutorRun_hook)
    

进入standard_ExecutorRun

    
    
    (gdb) n
    307         standard_ExecutorRun(queryDesc, direction, count, execute_once);
    (gdb) step
    standard_ExecutorRun (queryDesc=0x1b796e8, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:321
    321     Assert(queryDesc != NULL);
    

standard_ExecutorRun->校验并切换上下文

    
    
    321     Assert(queryDesc != NULL);
    (gdb) n
    323     estate = queryDesc->estate;
    (gdb) 
    325     Assert(estate != NULL);
    (gdb) 
    326     Assert(!(estate->es_top_eflags & EXEC_FLAG_EXPLAIN_ONLY));
    (gdb) 
    331     oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
    (gdb) 
    

standard_ExecutorRun->变量赋值,判断是否需要传输元组

    
    
    (gdb) 
    334     if (queryDesc->totaltime)
    (gdb) n
    340     operation = queryDesc->operation;
    (gdb) 
    341     dest = queryDesc->dest;
    (gdb) p operation
    $3 = CMD_SELECT
    (gdb) n
    346     estate->es_processed = 0;
    (gdb) 
    347     estate->es_lastoid = InvalidOid;
    (gdb) 
    349     sendTuples = (operation == CMD_SELECT ||
    (gdb) 
    352     if (sendTuples)
    (gdb) 
    353         dest->rStartup(dest, operation, queryDesc->tupDesc);
    (gdb) p sendTuples
    $4 = true
    (gdb) 
    

standard_ExecutorRun->执行计划(ExecutePlan函数下节介绍)

    
    
    (gdb) n
    358     if (!ScanDirectionIsNoMovement(direction))
    (gdb) 
    360         if (execute_once && queryDesc->already_executed)
    (gdb) 
    362         queryDesc->already_executed = true;
    (gdb) 
    364         ExecutePlan(estate,
    (gdb) 
    

standard_ExecutorRun->关闭资源并切换上下文

    
    
    (gdb) 
    378     if (sendTuples)
    (gdb) n
    379         dest->rShutdown(dest);
    (gdb) 
    381     if (queryDesc->totaltime)
    (gdb) 
    384     MemoryContextSwitchTo(oldcontext);
    (gdb) 
    385 }
    (gdb) 
    

standard_ExecutorRun->回到PortalRunSelect

    
    
    (gdb) n
    ExecutorRun (queryDesc=0x1b796e8, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:308
    308 }
    (gdb) 
    PortalRunSelect (portal=0x1af2468, forward=true, count=0, dest=0x1b74668) at pquery.c:934
    934             nprocessed = queryDesc->estate->es_processed;
    

快照出栈,修改状态atStart/atEnd等

    
    
    (gdb) n
    935             PopActiveSnapshot();
    (gdb) 
    938         if (!ScanDirectionIsNoMovement(direction))
    (gdb) 
    940             if (nprocessed > 0)
    (gdb) p nprocessed
    $6 = 99991
    (gdb) n
    941                 portal->atStart = false;    /* OK to go backward now */
    (gdb) 
    942             if (count == 0 || nprocessed < (uint64) count)
    (gdb) 
    

完成调用

    
    
    (gdb) n
    943                 portal->atEnd = true;   /* we retrieved 'em all */
    (gdb) p count
    $7 = 0
    (gdb) n
    944             portal->portalPos += nprocessed;
    (gdb) 
    997     return nprocessed;
    (gdb) 
    998 }
    (gdb) n
    PortalRun (portal=0x1af2468, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x1b74668, altdest=0x1b74668, 
        completionTag=0x7ffc5ff58740 "") at pquery.c:780
    780                 if (completionTag && portal->commandTag)
    (gdb) p nprocessed
    $8 = 99991
    

DONE!

### 四、参考资料

[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

