本文简单介绍了PG查询逻辑优化中的子查询链接(subLink),主要内容包括子链接的基本概念，介绍了主函数以及使用gdb跟踪分析了上拉函数的部分处理逻辑。  
上拉子链接的目的是为了把子链接改写为半连接或反半连接的方式提升性能。

### 一、子链接简介

按官方文档的介绍,子链接Sublink代表的是出现在表达式(可能会出现组合运算符)中的子查询,子查询的类型包括:  
_EXISTS_SUBLINK_  
语法:EXISTS(SELECT ...)

    
    
    select * 
    from t_dwxx a
    where exists (select b.dwbh from t_grxx b where a.dwbh = b.dwbh);
    

_ALL_SUBLINK_  
语法:(lefthand) op ALL (SELECT ...)

    
    
    select * 
    from t_dwxx a
    where dwbh > all (select b.dwbh from t_grxx b);
    

_ANY_SUBLINK_  
语法:(lefthand) op ANY (SELECT ...)

    
    
    select * 
    from t_dwxx a
    where dwbh = any (select b.dwbh from t_grxx b);
    

_ROWCOMPARE_SUBLINK_  
语法:(lefthand) op (SELECT ...)

    
    
    select * 
    from t_dwxx a
    where dwbh > (select max(b.dwbh) from t_grxx b);
    

_EXPR_SUBLINK_  
语法:(SELECT with single targetlist item ...)

    
    
    select *,(select max(dwbh) from t_grxx) 
    from t_dwxx a;
    

_MULTIEXPR_SUBLINK_  
语法:(SELECT with multiple targetlist items ...)

_ARRAY_SUBLINK_  
语法:ARRAY(SELECT with single targetlist item ...)

_CTE_SUBLINK_  
语法:  
WITH query (never actually part of an expression)

官方说明:

    
    
    /*
     * SubLink
     *
     * A SubLink represents a subselect appearing in an expression, and in some
     * cases also the combining operator(s) just above it.  The subLinkType
     * indicates the form of the expression represented:
     *  EXISTS_SUBLINK    EXISTS(SELECT ...)
     *  ALL_SUBLINK     (lefthand) op ALL (SELECT ...)
     *  ANY_SUBLINK     (lefthand) op ANY (SELECT ...)
     *  ROWCOMPARE_SUBLINK  (lefthand) op (SELECT ...)
     *  EXPR_SUBLINK    (SELECT with single targetlist item ...)
     *  MULTIEXPR_SUBLINK (SELECT with multiple targetlist items ...)
     *  ARRAY_SUBLINK   ARRAY(SELECT with single targetlist item ...)
     *  CTE_SUBLINK     WITH query (never actually part of an expression)
     * For ALL, ANY, and ROWCOMPARE, the lefthand is a list of expressions of the
     * same length as the subselect's targetlist.  ROWCOMPARE will *always* have
     * a list with more than one entry; if the subselect has just one target
     * then the parser will create an EXPR_SUBLINK instead (and any operator
     * above the subselect will be represented separately).
     * ROWCOMPARE, EXPR, and MULTIEXPR require the subselect to deliver at most
     * one row (if it returns no rows, the result is NULL).
     * ALL, ANY, and ROWCOMPARE require the combining operators to deliver boolean
     * results.  ALL and ANY combine the per-row results using AND and OR
     * semantics respectively.
     * ARRAY requires just one target column, and creates an array of the target
     * column's type using any number of rows resulting from the subselect.
     * 
     * 子链接属于Expr Node,但并不意味着子链接是可以执行的.子链接必须在计划期间通过子计划节点在表达式树中替换
     * SubLink is classed as an Expr node, but it is not actually executable;
     * it must be replaced in the expression tree by a SubPlan node during
     * planning.
     *
     * NOTE: in the raw output of gram.y, testexpr contains just the raw form
     * of the lefthand expression (if any), and operName is the String name of
     * the combining operator.  Also, subselect is a raw parsetree.  During parse
     * analysis, the parser transforms testexpr into a complete boolean expression
     * that compares the lefthand value(s) to PARAM_SUBLINK nodes representing the
     * output columns of the subselect.  And subselect is transformed to a Query.
     * This is the representation seen in saved rules and in the rewriter.
     *
     * In EXISTS, EXPR, MULTIEXPR, and ARRAY SubLinks, testexpr and operName
     * are unused and are always null.
     *
     * subLinkId is currently used only for MULTIEXPR SubLinks, and is zero in
     * other SubLinks.  This number identifies different multiple-assignment
     * subqueries within an UPDATE statement's SET list.  It is unique only
     * within a particular targetlist.  The output column(s) of the MULTIEXPR
     * are referenced by PARAM_MULTIEXPR Params appearing elsewhere in the tlist.
     *
     * The CTE_SUBLINK case never occurs in actual SubLink nodes, but it is used
     * in SubPlans generated for WITH subqueries.
     */
    typedef enum SubLinkType
    {
      EXISTS_SUBLINK,
      ALL_SUBLINK,
      ANY_SUBLINK,
      ROWCOMPARE_SUBLINK,
      EXPR_SUBLINK,
      MULTIEXPR_SUBLINK,
      ARRAY_SUBLINK,
      CTE_SUBLINK         /* for SubPlans only */
    } SubLinkType;
    
    
    typedef struct SubLink
    {
      Expr    xpr;
      SubLinkType subLinkType;  /* see above */
      int     subLinkId;    /* ID (1..n); 0 if not MULTIEXPR */
      Node     *testexpr;   /* outer-query test for ALL/ANY/ROWCOMPARE */
      List     *operName;   /* originally specified operator name */
      Node     *subselect;    /* subselect as Query* or raw parsetree */
      int     location;   /* token location, or -1 if unknown */
    } SubLink;
    
    

### 二、源码解读

**standard_planner函数**

    
    
    /*****************************************************************************
      *
      *     Query optimizer entry point
      *
      * To support loadable plugins that monitor or modify planner behavior,
      * we provide a hook variable that lets a plugin get control before and
      * after the standard planning process.  The plugin would normally call
      * standard_planner().
      *
      * Note to plugin authors: standard_planner() scribbles on its Query input,
      * so you'd better copy that data structure if you want to plan more than once.
      *
      *****************************************************************************/
     PlannedStmt *
     planner(Query *parse, int cursorOptions, ParamListInfo boundParams)
     {
         PlannedStmt *result;
     
         if (planner_hook)
             result = (*planner_hook) (parse, cursorOptions, boundParams); //钩子函数,可以实现定制化开发
         else
             result = standard_planner(parse, cursorOptions, boundParams);//PG的标准实现
         return result;
     }
     
     /*
     输入参数:
        parse-查询树
        cursorOptions-游标处理选项,见本节中的数据结构
        boundParams-绑定变量?
     输出参数:
     */
     PlannedStmt *
     standard_planner(Query *parse, int cursorOptions, ParamListInfo boundParams)
     {
         PlannedStmt *result;//最终结果
         PlannerGlobal *glob;//全局优化信息
         double      tuple_fraction;//优化过程中元组的采样率
         PlannerInfo *root;//执行计划的根节点
         RelOptInfo *final_rel;//优化后的Relation信息
         Path       *best_path;//最优路径
         Plan       *top_plan;//顶层计划
         ListCell   *lp,//临时变量
                    *lr;
     
         /*
          * Set up global state for this planner invocation.  This data is needed
          * across all levels of sub-Query that might exist in the given command,
          * so we keep it in a separate struct that's linked to by each per-Query
          * PlannerInfo.
          */
         //初始化全局优化信息 
         glob = makeNode(PlannerGlobal);
     
         glob->boundParams = boundParams;
         glob->subplans = NIL;
         glob->subroots = NIL;
         glob->rewindPlanIDs = NULL;
         glob->finalrtable = NIL;
         glob->finalrowmarks = NIL;
         glob->resultRelations = NIL;
         glob->nonleafResultRelations = NIL;
         glob->rootResultRelations = NIL;
         glob->relationOids = NIL;
         glob->invalItems = NIL;
         glob->paramExecTypes = NIL;
         glob->lastPHId = 0;
         glob->lastRowMarkId = 0;
         glob->lastPlanNodeId = 0;
         glob->transientPlan = false;
         glob->dependsOnRole = false;
     
         /*
          * Assess whether it's feasible to use parallel mode for this query. We
          * can't do this in a standalone backend, or if the command will try to
          * modify any data, or if this is a cursor operation, or if GUCs are set
          * to values that don't permit parallelism, or if parallel-unsafe
          * functions are present in the query tree.
          *
          * (Note that we do allow CREATE TABLE AS, SELECT INTO, and CREATE
          * MATERIALIZED VIEW to use parallel plans, but this is safe only because
          * the command is writing into a completely new table which workers won't
          * be able to see.  If the workers could see the table, the fact that
          * group locking would cause them to ignore the leader's heavyweight
          * relation extension lock and GIN page locks would make this unsafe.
          * We'll have to fix that somehow if we want to allow parallel inserts in
          * general; updates and deletes have additional problems especially around
          * combo CIDs.)
          *
          * For now, we don't try to use parallel mode if we're running inside a
          * parallel worker.  We might eventually be able to relax this
          * restriction, but for now it seems best not to have parallel workers
          * trying to create their own parallel workers.
          *
          * We can't use parallelism in serializable mode because the predicate
          * locking code is not parallel-aware.  It's not catastrophic if someone
          * tries to run a parallel plan in serializable mode; it just won't get
          * any workers and will run serially.  But it seems like a good heuristic
          * to assume that the same serialization level will be in effect at plan
          * time and execution time, so don't generate a parallel plan if we're in
          * serializable mode.
          */
         //判断是否可以使用并行模式
         //并行是很大的一个话题,有待以后再谈 
         if ((cursorOptions & CURSOR_OPT_PARALLEL_OK) != 0 &&
             IsUnderPostmaster &&
             parse->commandType == CMD_SELECT &&
             !parse->hasModifyingCTE &&
             max_parallel_workers_per_gather > 0 &&
             !IsParallelWorker() &&
             !IsolationIsSerializable())
         {
             /* all the cheap tests pass, so scan the query tree */
             glob->maxParallelHazard = max_parallel_hazard(parse);
             glob->parallelModeOK = (glob->maxParallelHazard != PROPARALLEL_UNSAFE);
         }
         else
         {
             /* skip the query tree scan, just assume it's unsafe */
             glob->maxParallelHazard = PROPARALLEL_UNSAFE;
             glob->parallelModeOK = false;
         }
     
         /*
          * glob->parallelModeNeeded is normally set to false here and changed to
          * true during plan creation if a Gather or Gather Merge plan is actually
          * created (cf. create_gather_plan, create_gather_merge_plan).
          *
          * However, if force_parallel_mode = on or force_parallel_mode = regress,
          * then we impose parallel mode whenever it's safe to do so, even if the
          * final plan doesn't use parallelism.  It's not safe to do so if the
          * query contains anything parallel-unsafe; parallelModeOK will be false
          * in that case.  Note that parallelModeOK can't change after this point.
          * Otherwise, everything in the query is either parallel-safe or
          * parallel-restricted, and in either case it should be OK to impose
          * parallel-mode restrictions.  If that ends up breaking something, then
          * either some function the user included in the query is incorrectly
          * labelled as parallel-safe or parallel-restricted when in reality it's
          * parallel-unsafe, or else the query planner itself has a bug.
          */
         glob->parallelModeNeeded = glob->parallelModeOK &&
             (force_parallel_mode != FORCE_PARALLEL_OFF);
     
         /* Determine what fraction of the plan is likely to be scanned */
         if (cursorOptions & CURSOR_OPT_FAST_PLAN)//fast-start plan?
         {
             /*
              * We have no real idea how many tuples the user will ultimately FETCH
              * from a cursor, but it is often the case that he doesn't want 'em
              * all, or would prefer a fast-start plan anyway so that he can
              * process some of the tuples sooner.  Use a GUC parameter to decide
              * what fraction to optimize for.
              */
             tuple_fraction = cursor_tuple_fraction;
     
             /*
              * We document cursor_tuple_fraction as simply being a fraction, which
              * means the edge cases 0 and 1 have to be treated specially here.  We
              * convert 1 to 0 ("all the tuples") and 0 to a very small fraction.
              */
             if (tuple_fraction >= 1.0)
                 tuple_fraction = 0.0;
             else if (tuple_fraction <= 0.0)
                 tuple_fraction = 1e-10;
         }
         else
         {
             /* Default assumption is we need all the tuples */
             tuple_fraction = 0.0;
         }
         //以上:set up for recursive handling of subqueries,为子查询配置处理器(递归方式)
         /* primary planning entry point (may recurse for subqueries) */
         root = subquery_planner(glob, parse, NULL,
                                 false, tuple_fraction);//进入子查询优化器
     
         /* Select best Path and turn it into a Plan */
         final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);//获得最上层的优化后的Relation
         best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);//获得最佳路径
     
         top_plan = create_plan(root, best_path);//创建计划
     
         /*
          * If creating a plan for a scrollable cursor, make sure it can run
          * backwards on demand.  Add a Material node at the top at need.
          */
         if (cursorOptions & CURSOR_OPT_SCROLL)
         {
             if (!ExecSupportsBackwardScan(top_plan))
                 top_plan = materialize_finished_plan(top_plan);//如果是Scroll游标,在前台执行,那么物化之(数据不能长期占用内存,但又要满足Scroll的要求,需要物化至磁盘中)
         }
     
         /*
          * Optionally add a Gather node for testing purposes, provided this is
          * actually a safe thing to do.
          */
         if (force_parallel_mode != FORCE_PARALLEL_OFF && top_plan->parallel_safe)//并行执行,添加Gather节点
         {
             Gather     *gather = makeNode(Gather);
     
             /*
              * If there are any initPlans attached to the formerly-top plan node,
              * move them up to the Gather node; same as we do for Material node in
              * materialize_finished_plan.
              */
             gather->plan.initPlan = top_plan->initPlan;
             top_plan->initPlan = NIL;
     
             gather->plan.targetlist = top_plan->targetlist;
             gather->plan.qual = NIL;
             gather->plan.lefttree = top_plan;
             gather->plan.righttree = NULL;
             gather->num_workers = 1;
             gather->single_copy = true;
             gather->invisible = (force_parallel_mode == FORCE_PARALLEL_REGRESS);
     
             /*
              * Since this Gather has no parallel-aware descendants to signal to,
              * we don't need a rescan Param.
              */
             gather->rescan_param = -1;
     
             /*
              * Ideally we'd use cost_gather here, but setting up dummy path data
              * to satisfy it doesn't seem much cleaner than knowing what it does.
              */
             gather->plan.startup_cost = top_plan->startup_cost +
                 parallel_setup_cost;
             gather->plan.total_cost = top_plan->total_cost +
                 parallel_setup_cost + parallel_tuple_cost * top_plan->plan_rows;
             gather->plan.plan_rows = top_plan->plan_rows;
             gather->plan.plan_width = top_plan->plan_width;
             gather->plan.parallel_aware = false;
             gather->plan.parallel_safe = false;
     
             /* use parallel mode for parallel plans. */
             root->glob->parallelModeNeeded = true;
     
             top_plan = &gather->plan;
         }
     
         /*
          * If any Params were generated, run through the plan tree and compute
          * each plan node's extParam/allParam sets.  Ideally we'd merge this into
          * set_plan_references' tree traversal, but for now it has to be separate
          * because we need to visit subplans before not after main plan.
          */
         if (glob->paramExecTypes != NIL)
         {
             Assert(list_length(glob->subplans) == list_length(glob->subroots));
             forboth(lp, glob->subplans, lr, glob->subroots)
             {
                 Plan       *subplan = (Plan *) lfirst(lp);
                 PlannerInfo *subroot = lfirst_node(PlannerInfo, lr);
     
                 SS_finalize_plan(subroot, subplan);
             }
             SS_finalize_plan(root, top_plan);
         }
     
         /* final cleanup of the plan */
         Assert(glob->finalrtable == NIL);
         Assert(glob->finalrowmarks == NIL);
         Assert(glob->resultRelations == NIL);
         Assert(glob->nonleafResultRelations == NIL);
         Assert(glob->rootResultRelations == NIL);
         top_plan = set_plan_references(root, top_plan);
         /* ... and the subplans (both regular subplans and initplans) */
         Assert(list_length(glob->subplans) == list_length(glob->subroots));
         forboth(lp, glob->subplans, lr, glob->subroots)
         {
             Plan       *subplan = (Plan *) lfirst(lp);
             PlannerInfo *subroot = lfirst_node(PlannerInfo, lr);
     
             lfirst(lp) = set_plan_references(subroot, subplan);
         }
     
         /* build the PlannedStmt result */
         result = makeNode(PlannedStmt);
     
         result->commandType = parse->commandType;
         result->queryId = parse->queryId;
         result->hasReturning = (parse->returningList != NIL);
         result->hasModifyingCTE = parse->hasModifyingCTE;
         result->canSetTag = parse->canSetTag;
         result->transientPlan = glob->transientPlan;
         result->dependsOnRole = glob->dependsOnRole;
         result->parallelModeNeeded = glob->parallelModeNeeded;
         result->planTree = top_plan;
         result->rtable = glob->finalrtable;
         result->resultRelations = glob->resultRelations;
         result->nonleafResultRelations = glob->nonleafResultRelations;
         result->rootResultRelations = glob->rootResultRelations;
         result->subplans = glob->subplans;
         result->rewindPlanIDs = glob->rewindPlanIDs;
         result->rowMarks = glob->finalrowmarks;
         result->relationOids = glob->relationOids;
         result->invalItems = glob->invalItems;
         result->paramExecTypes = glob->paramExecTypes;
         /* utilityStmt should be null, but we might as well copy it */
         result->utilityStmt = parse->utilityStmt;
         result->stmt_location = parse->stmt_location;
         result->stmt_len = parse->stmt_len;
     
         result->jitFlags = PGJIT_NONE;
         if (jit_enabled && jit_above_cost >= 0 &&
             top_plan->total_cost > jit_above_cost)
         {
             result->jitFlags |= PGJIT_PERFORM;
     
             /*
              * Decide how much effort should be put into generating better code.
              */
             if (jit_optimize_above_cost >= 0 &&
                 top_plan->total_cost > jit_optimize_above_cost)
                 result->jitFlags |= PGJIT_OPT3;
             if (jit_inline_above_cost >= 0 &&
                 top_plan->total_cost > jit_inline_above_cost)
                 result->jitFlags |= PGJIT_INLINE;
     
             /*
              * Decide which operations should be JITed.
              */
             if (jit_expressions)
                 result->jitFlags |= PGJIT_EXPR;
             if (jit_tuple_deforming)
                 result->jitFlags |= PGJIT_DEFORM;
         }
     
         return result;
     }
    

**subquery_planner**

    
    
     /*--------------------      * subquery_planner
      *    Invokes the planner on a subquery.  We recurse to here for each
      *    sub-SELECT found in the query tree.
      *
      * glob is the global state for the current planner run.
      * parse is the querytree produced by the parser & rewriter.
      * parent_root is the immediate parent Query's info (NULL at the top level).
      * hasRecursion is true if this is a recursive WITH query.
      * tuple_fraction is the fraction of tuples we expect will be retrieved.
      * tuple_fraction is interpreted as explained for grouping_planner, below.
      *
      * Basically, this routine does the stuff that should only be done once
      * per Query object.  It then calls grouping_planner.  At one time,
      * grouping_planner could be invoked recursively on the same Query object;
      * that's not currently true, but we keep the separation between the two
      * routines anyway, in case we need it again someday.
      *
      * subquery_planner will be called recursively to handle sub-Query nodes
      * found within the query's expressions and rangetable.
      *
      * Returns the PlannerInfo struct ("root") that contains all data generated
      * while planning the subquery.  In particular, the Path(s) attached to
      * the (UPPERREL_FINAL, NULL) upperrel represent our conclusions about the
      * cheapest way(s) to implement the query.  The top level will select the
      * best Path and pass it through createplan.c to produce a finished Plan.
      *--------------------      */
    /*
    输入:
        glob-PlannerGlobal
        parse-Query结构体指针
        parent_root-父PlannerInfo Root节点
        hasRecursion-是否递归?
        tuple_fraction-扫描Tuple比例
    输出:
        PlannerInfo指针
    */
     PlannerInfo *
     subquery_planner(PlannerGlobal *glob, Query *parse,
                      PlannerInfo *parent_root,
                      bool hasRecursion, double tuple_fraction)
     {
         PlannerInfo *root;//返回值
         List       *newWithCheckOptions;//
         List       *newHaving;//Having子句
         bool        hasOuterJoins;//是否存在Outer Join?
         RelOptInfo *final_rel;//
         ListCell   *l;//临时变量
     
         /* Create a PlannerInfo data structure for this subquery */
         root = makeNode(PlannerInfo);//构造返回值
         root->parse = parse;
         root->glob = glob;
         root->query_level = parent_root ? parent_root->query_level + 1 : 1;
         root->parent_root = parent_root;
         root->plan_params = NIL;
         root->outer_params = NULL;
         root->planner_cxt = CurrentMemoryContext;
         root->init_plans = NIL;
         root->cte_plan_ids = NIL;
         root->multiexpr_params = NIL;
         root->eq_classes = NIL;
         root->append_rel_list = NIL;
         root->rowMarks = NIL;
         memset(root->upper_rels, 0, sizeof(root->upper_rels));
         memset(root->upper_targets, 0, sizeof(root->upper_targets));
         root->processed_tlist = NIL;
         root->grouping_map = NULL;
         root->minmax_aggs = NIL;
         root->qual_security_level = 0;
         root->inhTargetKind = INHKIND_NONE;
         root->hasRecursion = hasRecursion;
         if (hasRecursion)
             root->wt_param_id = SS_assign_special_param(root);
         else
             root->wt_param_id = -1;
         root->non_recursive_path = NULL;
         root->partColsUpdated = false;
     
         /*
          * If there is a WITH list, process each WITH query and build an initplan
          * SubPlan structure for it.
          */
         if (parse->cteList)
             SS_process_ctes(root);//处理With 语句
     
         /*
          * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
          * to transform them into joins.  Note that this step does not descend
          * into subqueries; if we pull up any subqueries below, their SubLinks are
          * processed just before pulling them up.
          */
         if (parse->hasSubLinks)
             pull_up_sublinks(root); //上拉子链接
     
         //其他内容...
    
         return root;
     }
     
    

**pull_up_sublinks**

    
    
    下一小节介绍
    

### 三、基础信息

**数据结构/宏定义**  
_1、cursorOptions_

    
    
     /* ----------------------      *      Declare Cursor Statement
      *
      * The "query" field is initially a raw parse tree, and is converted to a
      * Query node during parse analysis.  Note that rewriting and planning
      * of the query are always postponed until execution.
      * ----------------------      */
     #define CURSOR_OPT_BINARY       0x0001  /* BINARY */
     #define CURSOR_OPT_SCROLL       0x0002  /* SCROLL explicitly given */
     #define CURSOR_OPT_NO_SCROLL    0x0004  /* NO SCROLL explicitly given */
     #define CURSOR_OPT_INSENSITIVE  0x0008  /* INSENSITIVE */
     #define CURSOR_OPT_HOLD         0x0010  /* WITH HOLD */
     /* these planner-control flags do not correspond to any SQL grammar: */
     #define CURSOR_OPT_FAST_PLAN    0x0020  /* prefer fast-start plan */
     #define CURSOR_OPT_GENERIC_PLAN 0x0040  /* force use of generic plan */
     #define CURSOR_OPT_CUSTOM_PLAN  0x0080  /* force use of custom 
     #define CURSOR_OPT_PARALLEL_OK  0x0100  /* parallel mode OK */
    

_2、Relids_

    
    
     /*
      * Relids
      *      Set of relation identifiers (indexes into the rangetable).
      */
     typedef Bitmapset *Relids;
     /* The unit size can be adjusted by changing these three declarations: */
     #define BITS_PER_BITMAPWORD 32
     typedef uint32 bitmapword;      /* must be an unsigned type */
     typedef int32 signedbitmapword; /* must be the matching signed type */
     
     typedef struct Bitmapset
     {
         int         nwords;         /* number of words in array */
         bitmapword  words[FLEXIBLE_ARRAY_MEMBER];   /* really [nwords] */
     } Bitmapset; 
    

### 四、跟踪分析

测试脚本:

    
    
    testdb=# explain select * 
    from t_dwxx a
    where dwbh > all (select b.dwbh from t_grxx b);
                                    QUERY PLAN                                
    --------------------------------------------------------------------------     Seq Scan on t_dwxx a  (cost=0.00..1498.00 rows=80 width=474)
       Filter: (SubPlan 1)
       SubPlan 1
         ->  Materialize  (cost=0.00..17.35 rows=490 width=38)
               ->  Seq Scan on t_grxx b  (cost=0.00..14.90 rows=490 width=38)
    (5 rows)
    

启动gdb跟踪:

    
    
    (gdb) b pull_up_sublinks
    Breakpoint 1 at 0x77cbc6: file prepjointree.c, line 157.
    (gdb) c
    Continuing.
    
    Breakpoint 1, pull_up_sublinks (root=0x126fd48) at prepjointree.c:157
    157                          (Node *) root->parse->jointree,
    (gdb) 
    #查看输入参数
    (gdb) p *root
    $1 = {type = T_PlannerInfo, parse = 0x11b43d8, glob = 0x11b4e00, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, simple_rte_array = 0x0, all_baserels = 0x0, 
      nullable_baserels = 0x0, join_rel_list = 0x0, join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, 
      init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, 
      left_join_clauses = 0x0, right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, initial_rels = 0x0, 
      upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
      processed_tlist = 0x0, grouping_map = 0x0, minmax_aggs = 0x0, planner_cxt = 0x11b2e90, total_table_pages = 0, 
      tuple_fraction = 0, limit_tuples = 0, qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, 
      hasLateralRTEs = false, hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, 
      hasRecursion = false, wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    #jointree的类型为FromExpr  
    (gdb) p *root->parse->jointree
    $3 = {type = T_FromExpr, fromlist = 0x11b4870, quals = 0x11b40e8}
    (gdb) n
    156   jtnode = pull_up_sublinks_jointree_recurse(root,
    #进入pull_up_sublinks_jointree_recurse函数
    (gdb) step
    pull_up_sublinks_jointree_recurse (root=0x126fd48, jtnode=0x1282ea0, relids=0x7ffe830ab9b0) at prepjointree.c:180
    180   if (jtnode == NULL)
    #查看参数
    #1.root与pull_up_sublinks函数的输入一致
    #2.jtnode类型为FromExpr
    #3.relids
    (gdb) p *jtnode
    $7 = {type = T_FromExpr}
    (gdb) p *(FromExpr *)jtnode
    $8 = {type = T_FromExpr, fromlist = 0x11b4870, quals = 0x11b40e8}
    #FromExpr->fromlist中的的元素类型为RangeTblRef
    (gdb) p *((FromExpr *)jtnode)->fromlist
    $9 = {type = T_List, length = 1, head = 0x11b4850, tail = 0x11b4850}
    (gdb) p *(Node *)((FromExpr *)jtnode)->fromlist->head->data.ptr_value
    $10 = {type = T_RangeTblRef}
    ...
    #进入FromExpr分支
    191   else if (IsA(jtnode, FromExpr))
    ...
    201     foreach(l, f->fromlist)
    (gdb) 
    207                              lfirst(l),
    (gdb) p *l
    $11 = {data = {ptr_value = 0x11b4838, int_value = 18565176, oid_value = 18565176}, next = 0x0}
    (gdb) p *(RangeTblRef *)l->data.ptr_value
    $13 = {type = T_RangeTblRef, rtindex = 1}
    #递归调用pull_up_sublinks_jointree_recurse
    (gdb) n
    206       newchild = pull_up_sublinks_jointree_recurse(root,
    (gdb) step
    pull_up_sublinks_jointree_recurse (root=0x126fd48, jtnode=0x11b4838, relids=0x7ffe830ab940) at prepjointree.c:180
    180   if (jtnode == NULL)
    #输入参数
    #1.root与pull_up_sublinks函数的输入一致
    #2.jtnode类型为RangeTblRef
    #3.relids
    #进入RangeTblRef分支
    (gdb) n
    184   else if (IsA(jtnode, RangeTblRef))
    (gdb) n
    186     int     varno = ((RangeTblRef *) jtnode)->rtindex;
    (gdb) 
    188     *relids = bms_make_singleton(varno);
    (gdb) 
    312   return jtnode;
    (gdb) p *relids
    $16 = (Relids) 0x12704c8
    ...
    (gdb) p *newchild
    $19 = {type = T_RangeTblRef}
    (gdb) n
    210       frelids = bms_join(frelids, childrelids);
    (gdb) p *frelids
    $25 = {nwords = 1, words = 0x12704cc}
    (gdb) p *frelids->words
    $26 = 2
    ...
    #进入pull_up_sublinks_qual_recurse
    (gdb) n
    217     newf->quals = pull_up_sublinks_qual_recurse(root, f->quals,
    (gdb) step
    pull_up_sublinks_qual_recurse (root=0x126fd48, node=0x11b40e8, jtlink1=0x7ffe830ab948, available_rels1=0x12704c8, 
        jtlink2=0x0, available_rels2=0x0) at prepjointree.c:335
    335   if (node == NULL)
    #输入参数
    #1.root与上述无异
    #2.node
    (gdb) p *node
    $35 = {type = T_SubLink}
    (gdb) p *(SubLink *)node
    $36 = {xpr = {type = T_SubLink}, subLinkType = ALL_SUBLINK, subLinkId = 0, testexpr = 0x1282e00, operName = 0x11b3cf8, 
      subselect = 0x1282578, location = 35}
    #3.jtlink1,指针数组
    (gdb) p *(RangeTblRef *)((FromExpr *)jtlink1[0])->fromlist->head->data->ptr_value
    $45 = {type = T_RangeTblRef, rtindex = 1}
    #4.available_rels1,可用的Relids
    (gdb) p *available_rels1
    $47 = {nwords = 1, words = 0x12704cc}
    (gdb) p *available_rels1->words
    $48 = 2
    #5/6,NULL值
    (gdb) n
    337   if (IsA(node, SubLink))
    (gdb) 
    339     SubLink    *sublink = (SubLink *) node;
    (gdb) 
    344     if (sublink->subLinkType == ANY_SUBLINK)
    (gdb) 
    398     else if (sublink->subLinkType == EXISTS_SUBLINK)
    (gdb) p sublink->subLinkType
    $49 = ALL_SUBLINK
    #非ANY/EXISTS链接,退出
    (gdb) n
    453     return node;
    (gdb) 
    #回到pull_up_sublinks_jointree_recurse
    (gdb) 
    pull_up_sublinks_jointree_recurse (root=0x126fd48, jtnode=0x1282ea0, relids=0x7ffe830ab9b0) at prepjointree.c:230
    230     *relids = frelids;
    (gdb) 
    231     jtnode = jtlink;
    (gdb) 
    312   return jtnode;
    #返回的jtnode为RangeTblRef
    (gdb) p *(RangeTblRef *)((FromExpr *)jtnode)->fromlist->head->data.ptr_value
    $55 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) n
    313 }
    (gdb) 
    pull_up_sublinks (root=0x126fd48) at prepjointree.c:164
    164   if (IsA(jtnode, FromExpr))
    (gdb) 
    165     root->parse->jointree = (FromExpr *) jtnode;
    (gdb) p *root->parse->jointree
    $56 = {type = T_FromExpr, fromlist = 0x11b4870, quals = 0x11b40e8}
    (gdb) p *root->parse->jointree->fromlist
    $57 = {type = T_List, length = 1, head = 0x11b4850, tail = 0x11b4850}
    (gdb) n
    168 }
    (gdb) 
    subquery_planner (glob=0x11b4e00, parse=0x11b43d8, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:656
    656   inline_set_returning_functions(root);
    (gdb) c
    Continuing.
    #ANY类型并没有做上拉操作
    

### 五、小结

1、基本概念：subLink的基本概念以及基本分类；  
2、函数：Planner的主函数简要介绍；  
3、数据结构：SubLink、SubLinkType等结构；  
4、上拉操作：只对ANY/EXISTS进行处理，其他类型如ALL暂不处理。

