本文简单介绍了PG插入数据部分的源码，主要内容包括如何构造PlannedStmt数据结构的实现逻辑，以及其他相关的数据结构，构造PlannedStmt的主函数standard_planner位于planner.c文件中。  
Planned中文直译意思是“已被规划过的”，PlannedStmt意思是已被规划过的Statement（SQL语句），这个数据结构中的元素以及该数据结构的构造是理解后续执行SQL语句的关键。

### 一、源码解读

standard_planner函数,生成PlannedStmt,其中最重要的信息是可用于后续执行SQL语句的planTree.

    
    
     PlannedStmt *
     standard_planner(Query *parse, int cursorOptions, ParamListInfo boundParams)
     {
         PlannedStmt *result;//返回结果
         PlannerGlobal *glob;//全局的Plan信息-Global information for planning/optimization
         double      tuple_fraction;//
         PlannerInfo *root;//每个Query的Plan信息-Per-query information for planning/optimization
         RelOptInfo *final_rel;//Plan中的每个Relation信息-Per-relation information for planning/optimization
         Path       *best_path;//最优路径
         Plan       *top_plan;//最上层的Plan
         ListCell   *lp,//临时变量
                    *lr;
     
         /*
          * Set up global state for this planner invocation.  This data is needed
          * across all levels of sub-Query that might exist in the given command,
          * so we keep it in a separate struct that's linked to by each per-Query
          * PlannerInfo.
          */
         glob = makeNode(PlannerGlobal);//构建PlannerGlobal
       //初始化参数
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
         if ((cursorOptions & CURSOR_OPT_PARALLEL_OK) != 0 &&
             IsUnderPostmaster &&
             parse->commandType == CMD_SELECT &&
             !parse->hasModifyingCTE &&
             max_parallel_workers_per_gather > 0 &&
             !IsParallelWorker() &&
             !IsolationIsSerializable())//并行模式的判断
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
         if (cursorOptions & CURSOR_OPT_FAST_PLAN)
         {
             /*
              * We have no real idea how many tuples the user will ultimately FETCH
              * from a cursor, but it is often the case that he doesn't want 'em
              * all, or would prefer a fast-start plan anyway so that he can
              * process some of the tuples sooner.  Use a GUC parameter to decide
              * what fraction to optimize for.
              */
             tuple_fraction = cursor_tuple_fraction;//使用GUC 参数
     
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
     
         /* primary planning entry point (may recurse for subqueries) */
         root = subquery_planner(glob, parse, NULL,
                                 false, tuple_fraction);//获取PlannerInfo根节点
     
         /* Select best Path and turn it into a Plan */
         final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);//获取顶层的RelOptInfo
         best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);//选择最佳路径
     
         top_plan = create_plan(root, best_path);//生成执行计划
     
         /*
          * If creating a plan for a scrollable cursor, make sure it can run
          * backwards on demand.  Add a Material node at the top at need.
          */
         if (cursorOptions & CURSOR_OPT_SCROLL)
         {
             if (!ExecSupportsBackwardScan(top_plan))
                 top_plan = materialize_finished_plan(top_plan);
         }
     
         /*
          * Optionally add a Gather node for testing purposes, provided this is
          * actually a safe thing to do.
          */
         if (force_parallel_mode != FORCE_PARALLEL_OFF && top_plan->parallel_safe)
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
     
         result->commandType = parse->commandType;//命令类型
         result->queryId = parse->queryId;
         result->hasReturning = (parse->returningList != NIL);
         result->hasModifyingCTE = parse->hasModifyingCTE;
         result->canSetTag = parse->canSetTag;
         result->transientPlan = glob->transientPlan;
         result->dependsOnRole = glob->dependsOnRole;
         result->parallelModeNeeded = glob->parallelModeNeeded;
         result->planTree = top_plan;//执行计划(这是后续执行SQL使用到的最重要的地方)
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
     
    

### 二、基础信息

standard_planner函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、PlannerGlobal_

    
    
    /*----------      * PlannerGlobal
      *      Global information for planning/optimization
      *
      * PlannerGlobal holds state for an entire planner invocation; this state
      * is shared across all levels of sub-Queries that exist in the command being
      * planned.
      *----------      */
     typedef struct PlannerGlobal
     {
       NodeTag     type;
       ParamListInfo boundParams;  /* Param values provided to planner() */
       List       *subplans;       /* Plans for SubPlan nodes */
       List       *subroots;       /* PlannerInfos for SubPlan nodes */
       Bitmapset  *rewindPlanIDs;  /* indices of subplans that require REWIND */
       List       *finalrtable;    /* "flat" rangetable for executor */
       List       *finalrowmarks;  /* "flat" list of PlanRowMarks */
       List       *resultRelations;    /* "flat" list of integer RT indexes */
       List       *nonleafResultRelations; /* "flat" list of integer RT indexes */
       List       *rootResultRelations;    /* "flat" list of integer RT indexes */
       List       *relationOids;   /* OIDs of relations the plan depends on */
       List       *invalItems;     /* other dependencies, as PlanInvalItems */
       List       *paramExecTypes; /* type OIDs for PARAM_EXEC Params */
       Index       lastPHId;       /* highest PlaceHolderVar ID assigned */
       Index       lastRowMarkId;  /* highest PlanRowMark ID assigned */
       int         lastPlanNodeId; /* highest plan node ID assigned */
       bool        transientPlan;  /* redo plan when TransactionXmin changes? */
       bool        dependsOnRole;  /* is plan specific to current role? */
       bool        parallelModeOK; /* parallel mode potentially OK? */
       bool        parallelModeNeeded; /* parallel mode actually required? */
       char        maxParallelHazard;  /* worst PROPARALLEL hazard level */
     } PlannerGlobal;
     
    

_2、PlannerInfo_

    
    
     /*----------      * PlannerInfo
      *      Per-query information for planning/optimization
      *
      * This struct is conventionally called "root" in all the planner routines.
      * It holds links to all of the planner's working state, in addition to the
      * original Query.  Note that at present the planner extensively modifies
      * the passed-in Query data structure; someday that should stop.
      *----------      */
     struct AppendRelInfo;
     
     typedef struct PlannerInfo
     {
         NodeTag     type;
     
         Query      *parse;          /* the Query being planned */
     
         PlannerGlobal *glob;        /* global info for current planner run */
     
         Index       query_level;    /* 1 at the outermost Query */
     
         struct PlannerInfo *parent_root;    /* NULL at outermost Query */
     
         /*
          * plan_params contains the expressions that this query level needs to
          * make available to a lower query level that is currently being planned.
          * outer_params contains the paramIds of PARAM_EXEC Params that outer
          * query levels will make available to this query level.
          */
         List       *plan_params;    /* list of PlannerParamItems, see below */
         Bitmapset  *outer_params;
     
         /*
          * simple_rel_array holds pointers to "base rels" and "other rels" (see
          * comments for RelOptInfo for more info).  It is indexed by rangetable
          * index (so entry 0 is always wasted).  Entries can be NULL when an RTE
          * does not correspond to a base relation, such as a join RTE or an
          * unreferenced view RTE; or if the RelOptInfo hasn't been made yet.
          */
         struct RelOptInfo **simple_rel_array;   /* All 1-rel RelOptInfos */
         int         simple_rel_array_size;  /* allocated size of array */
     
         /*
          * simple_rte_array is the same length as simple_rel_array and holds
          * pointers to the associated rangetable entries.  This lets us avoid
          * rt_fetch(), which can be a bit slow once large inheritance sets have
          * been expanded.
          */
         RangeTblEntry **simple_rte_array;   /* rangetable as an array */
     
         /*
          * append_rel_array is the same length as the above arrays, and holds
          * pointers to the corresponding AppendRelInfo entry indexed by
          * child_relid, or NULL if none.  The array itself is not allocated if
          * append_rel_list is empty.
          */
         struct AppendRelInfo **append_rel_array;
     
         /*
          * all_baserels is a Relids set of all base relids (but not "other"
          * relids) in the query; that is, the Relids identifier of the final join
          * we need to form.  This is computed in make_one_rel, just before we
          * start making Paths.
          */
         Relids      all_baserels;
     
         /*
          * nullable_baserels is a Relids set of base relids that are nullable by
          * some outer join in the jointree; these are rels that are potentially
          * nullable below the WHERE clause, SELECT targetlist, etc.  This is
          * computed in deconstruct_jointree.
          */
         Relids      nullable_baserels;
     
         /*
          * join_rel_list is a list of all join-relation RelOptInfos we have
          * considered in this planning run.  For small problems we just scan the
          * list to do lookups, but when there are many join relations we build a
          * hash table for faster lookups.  The hash table is present and valid
          * when join_rel_hash is not NULL.  Note that we still maintain the list
          * even when using the hash table for lookups; this simplifies life for
          * GEQO.
          */
         List       *join_rel_list;  /* list of join-relation RelOptInfos */
         struct HTAB *join_rel_hash; /* optional hashtable for join relations */
     
         /*
          * When doing a dynamic-programming-style join search, join_rel_level[k]
          * is a list of all join-relation RelOptInfos of level k, and
          * join_cur_level is the current level.  New join-relation RelOptInfos are
          * automatically added to the join_rel_level[join_cur_level] list.
          * join_rel_level is NULL if not in use.
          */
         List      **join_rel_level; /* lists of join-relation RelOptInfos */
         int         join_cur_level; /* index of list being extended */
     
         List       *init_plans;     /* init SubPlans for query */
     
         List       *cte_plan_ids;   /* per-CTE-item list of subplan IDs */
     
         List       *multiexpr_params;   /* List of Lists of Params for MULTIEXPR
                                          * subquery outputs */
     
         List       *eq_classes;     /* list of active EquivalenceClasses */
     
         List       *canon_pathkeys; /* list of "canonical" PathKeys */
     
         List       *left_join_clauses;  /* list of RestrictInfos for mergejoinable
                                          * outer join clauses w/nonnullable var on
                                          * left */
     
         List       *right_join_clauses; /* list of RestrictInfos for mergejoinable
                                          * outer join clauses w/nonnullable var on
                                          * right */
     
         List       *full_join_clauses;  /* list of RestrictInfos for mergejoinable
                                          * full join clauses */
     
         List       *join_info_list; /* list of SpecialJoinInfos */
     
         List       *append_rel_list;    /* list of AppendRelInfos */
     
         List       *rowMarks;       /* list of PlanRowMarks */
     
         List       *placeholder_list;   /* list of PlaceHolderInfos */
     
         List       *fkey_list;      /* list of ForeignKeyOptInfos */
     
         List       *query_pathkeys; /* desired pathkeys for query_planner() */
     
         List       *group_pathkeys; /* groupClause pathkeys, if any */
         List       *window_pathkeys;    /* pathkeys of bottom window, if any */
         List       *distinct_pathkeys;  /* distinctClause pathkeys, if any */
         List       *sort_pathkeys;  /* sortClause pathkeys, if any */
     
         List       *part_schemes;   /* Canonicalised partition schemes used in the
                                      * query. */
     
         List       *initial_rels;   /* RelOptInfos we are now trying to join */
     
         /* Use fetch_upper_rel() to get any particular upper rel */
         List       *upper_rels[UPPERREL_FINAL + 1]; /* upper-rel RelOptInfos */
     
         /* Result tlists chosen by grouping_planner for upper-stage processing */
         struct PathTarget *upper_targets[UPPERREL_FINAL + 1];//参见UpperRelationKind
     
         /*
          * grouping_planner passes back its final processed targetlist here, for
          * use in relabeling the topmost tlist of the finished Plan.
          */
         List       *processed_tlist;
     
         /* Fields filled during create_plan() for use in setrefs.c */
         AttrNumber *grouping_map;   /* for GroupingFunc fixup */
         List       *minmax_aggs;    /* List of MinMaxAggInfos */
     
         MemoryContext planner_cxt;  /* context holding PlannerInfo */
     
         double      total_table_pages;  /* # of pages in all tables of query */
     
         double      tuple_fraction; /* tuple_fraction passed to query_planner */
         double      limit_tuples;   /* limit_tuples passed to query_planner */
     
         Index       qual_security_level;    /* minimum security_level for quals */
         /* Note: qual_security_level is zero if there are no securityQuals */
     
         InheritanceKind inhTargetKind;  /* indicates if the target relation is an
                                          * inheritance child or partition or a
                                          * partitioned table */
         bool        hasJoinRTEs;    /* true if any RTEs are RTE_JOIN kind */
         bool        hasLateralRTEs; /* true if any RTEs are marked LATERAL */
         bool        hasDeletedRTEs; /* true if any RTE was deleted from jointree */
         bool        hasHavingQual;  /* true if havingQual was non-null */
         bool        hasPseudoConstantQuals; /* true if any RestrictInfo has
                                              * pseudoconstant = true */
         bool        hasRecursion;   /* true if planning a recursive WITH item */
     
         /* These fields are used only when hasRecursion is true: */
         int         wt_param_id;    /* PARAM_EXEC ID for the work table */
         struct Path *non_recursive_path;    /* a path for non-recursive term */
     
         /* These fields are workspace for createplan.c */
         Relids      curOuterRels;   /* outer rels above current node */
         List       *curOuterParams; /* not-yet-assigned NestLoopParams */
     
         /* optional private data for join_search_hook, e.g., GEQO */
         void       *join_search_private;
     
         /* Does this query modify any partition key columns? */
         bool        partColsUpdated;
     } PlannerInfo;
     
     /*
      * This enum identifies the different types of "upper" (post-scan/join)
      * relations that we might deal with during planning.
      */
     typedef enum UpperRelationKind
     {
         UPPERREL_SETOP,             /* result of UNION/INTERSECT/EXCEPT, if any */
         UPPERREL_PARTIAL_GROUP_AGG, /* result of partial grouping/aggregation, if
                                      * any */
         UPPERREL_GROUP_AGG,         /* result of grouping/aggregation, if any */
         UPPERREL_WINDOW,            /* result of window functions, if any */
         UPPERREL_DISTINCT,          /* result of "SELECT DISTINCT", if any */
         UPPERREL_ORDERED,           /* result of ORDER BY, if any */
         UPPERREL_FINAL              /* result of any remaining top-level actions */
         /* NB: UPPERREL_FINAL must be last enum entry; it's used to size arrays */
     } UpperRelationKind;
     
    
    

_3、RangeTblEntry_

    
    
     /*--------------------      * RangeTblEntry -      *    A range table is a List of RangeTblEntry nodes.
      *
      *    A range table entry may represent a plain relation, a sub-select in
      *    FROM, or the result of a JOIN clause.  (Only explicit JOIN syntax
      *    produces an RTE, not the implicit join resulting from multiple FROM
      *    items.  This is because we only need the RTE to deal with SQL features
      *    like outer joins and join-output-column aliasing.)  Other special
      *    RTE types also exist, as indicated by RTEKind.
      *
      *    Note that we consider RTE_RELATION to cover anything that has a pg_class
      *    entry.  relkind distinguishes the sub-cases.
      *
      *    alias is an Alias node representing the AS alias-clause attached to the
      *    FROM expression, or NULL if no clause.
      *
      *    eref is the table reference name and column reference names (either
      *    real or aliases).  Note that system columns (OID etc) are not included
      *    in the column list.
      *    eref->aliasname is required to be present, and should generally be used
      *    to identify the RTE for error messages etc.
      *
      *    In RELATION RTEs, the colnames in both alias and eref are indexed by
      *    physical attribute number; this means there must be colname entries for
      *    dropped columns.  When building an RTE we insert empty strings ("") for
      *    dropped columns.  Note however that a stored rule may have nonempty
      *    colnames for columns dropped since the rule was created (and for that
      *    matter the colnames might be out of date due to column renamings).
      *    The same comments apply to FUNCTION RTEs when a function's return type
      *    is a named composite type.
      *
      *    In JOIN RTEs, the colnames in both alias and eref are one-to-one with
      *    joinaliasvars entries.  A JOIN RTE will omit columns of its inputs when
      *    those columns are known to be dropped at parse time.  Again, however,
      *    a stored rule might contain entries for columns dropped since the rule
      *    was created.  (This is only possible for columns not actually referenced
      *    in the rule.)  When loading a stored rule, we replace the joinaliasvars
      *    items for any such columns with null pointers.  (We can't simply delete
      *    them from the joinaliasvars list, because that would affect the attnums
      *    of Vars referencing the rest of the list.)
      *
      *    inh is true for relation references that should be expanded to include
      *    inheritance children, if the rel has any.  This *must* be false for
      *    RTEs other than RTE_RELATION entries.
      *
      *    inFromCl marks those range variables that are listed in the FROM clause.
      *    It's false for RTEs that are added to a query behind the scenes, such
      *    as the NEW and OLD variables for a rule, or the subqueries of a UNION.
      *    This flag is not used anymore during parsing, since the parser now uses
      *    a separate "namespace" data structure to control visibility, but it is
      *    needed by ruleutils.c to determine whether RTEs should be shown in
      *    decompiled queries.
      *
      *    requiredPerms and checkAsUser specify run-time access permissions
      *    checks to be performed at query startup.  The user must have *all*
      *    of the permissions that are OR'd together in requiredPerms (zero
      *    indicates no permissions checking).  If checkAsUser is not zero,
      *    then do the permissions checks using the access rights of that user,
      *    not the current effective user ID.  (This allows rules to act as
      *    setuid gateways.)  Permissions checks only apply to RELATION RTEs.
      *
      *    For SELECT/INSERT/UPDATE permissions, if the user doesn't have
      *    table-wide permissions then it is sufficient to have the permissions
      *    on all columns identified in selectedCols (for SELECT) and/or
      *    insertedCols and/or updatedCols (INSERT with ON CONFLICT DO UPDATE may
      *    have all 3).  selectedCols, insertedCols and updatedCols are bitmapsets,
      *    which cannot have negative integer members, so we subtract
      *    FirstLowInvalidHeapAttributeNumber from column numbers before storing
      *    them in these fields.  A whole-row Var reference is represented by
      *    setting the bit for InvalidAttrNumber.
      *
      *    securityQuals is a list of security barrier quals (boolean expressions),
      *    to be tested in the listed order before returning a row from the
      *    relation.  It is always NIL in parser output.  Entries are added by the
      *    rewriter to implement security-barrier views and/or row-level security.
      *    Note that the planner turns each boolean expression into an implicitly
      *    AND'ed sublist, as is its usual habit with qualification expressions.
      *--------------------      */
     typedef enum RTEKind
     {
         RTE_RELATION,               /* ordinary relation reference */
         RTE_SUBQUERY,               /* subquery in FROM */
         RTE_JOIN,                   /* join */
         RTE_FUNCTION,               /* function in FROM */
         RTE_TABLEFUNC,              /* TableFunc(.., column list) */
         RTE_VALUES,                 /* VALUES (<exprlist>), (<exprlist>), ... */
         RTE_CTE,                    /* common table expr (WITH list element) */
         RTE_NAMEDTUPLESTORE         /* tuplestore, e.g. for AFTER triggers */
     } RTEKind;
     
     typedef struct RangeTblEntry
     {
         NodeTag     type;
     
         RTEKind     rtekind;        /* see above */
     
         /*
          * XXX the fields applicable to only some rte kinds should be merged into
          * a union.  I didn't do this yet because the diffs would impact a lot of
          * code that is being actively worked on.  FIXME someday.
          */
     
         /*
          * Fields valid for a plain relation RTE (else zero):
          *
          * As a special case, RTE_NAMEDTUPLESTORE can also set relid to indicate
          * that the tuple format of the tuplestore is the same as the referenced
          * relation.  This allows plans referencing AFTER trigger transition
          * tables to be invalidated if the underlying table is altered.
          */
         Oid         relid;          /* OID of the relation */
         char        relkind;        /* relation kind (see pg_class.relkind) */
         struct TableSampleClause *tablesample;  /* sampling info, or NULL */
     
         /*
          * Fields valid for a subquery RTE (else NULL):
          */
         Query      *subquery;       /* the sub-query */
         bool        security_barrier;   /* is from security_barrier view? */
     
         /*
          * Fields valid for a join RTE (else NULL/zero):
          *
          * joinaliasvars is a list of (usually) Vars corresponding to the columns
          * of the join result.  An alias Var referencing column K of the join
          * result can be replaced by the K'th element of joinaliasvars --- but to
          * simplify the task of reverse-listing aliases correctly, we do not do
          * that until planning time.  In detail: an element of joinaliasvars can
          * be a Var of one of the join's input relations, or such a Var with an
          * implicit coercion to the join's output column type, or a COALESCE
          * expression containing the two input column Vars (possibly coerced).
          * Within a Query loaded from a stored rule, it is also possible for
          * joinaliasvars items to be null pointers, which are placeholders for
          * (necessarily unreferenced) columns dropped since the rule was made.
          * Also, once planning begins, joinaliasvars items can be almost anything,
          * as a result of subquery-flattening substitutions.
          */
         JoinType    jointype;       /* type of join */
         List       *joinaliasvars;  /* list of alias-var expansions */
     
         /*
          * Fields valid for a function RTE (else NIL/zero):
          *
          * When funcordinality is true, the eref->colnames list includes an alias
          * for the ordinality column.  The ordinality column is otherwise
          * implicit, and must be accounted for "by hand" in places such as
          * expandRTE().
          */
         List       *functions;      /* list of RangeTblFunction nodes */
         bool        funcordinality; /* is this called WITH ORDINALITY? */
     
         /*
          * Fields valid for a TableFunc RTE (else NULL):
          */
         TableFunc  *tablefunc;
     
         /*
          * Fields valid for a values RTE (else NIL):
          */
         List       *values_lists;   /* list of expression lists */
     
         /*
          * Fields valid for a CTE RTE (else NULL/zero):
          */
         char       *ctename;        /* name of the WITH list item */
         Index       ctelevelsup;    /* number of query levels up */
         bool        self_reference; /* is this a recursive self-reference? */
     
         /*
          * Fields valid for table functions, values, CTE and ENR RTEs (else NIL):
          *
          * We need these for CTE RTEs so that the types of self-referential
          * columns are well-defined.  For VALUES RTEs, storing these explicitly
          * saves having to re-determine the info by scanning the values_lists. For
          * ENRs, we store the types explicitly here (we could get the information
          * from the catalogs if 'relid' was supplied, but we'd still need these
          * for TupleDesc-based ENRs, so we might as well always store the type
          * info here).
          *
          * For ENRs only, we have to consider the possibility of dropped columns.
          * A dropped column is included in these lists, but it will have zeroes in
          * all three lists (as well as an empty-string entry in eref).  Testing
          * for zero coltype is the standard way to detect a dropped column.
          */
         List       *coltypes;       /* OID list of column type OIDs */
         List       *coltypmods;     /* integer list of column typmods */
         List       *colcollations;  /* OID list of column collation OIDs */
     
         /*
          * Fields valid for ENR RTEs (else NULL/zero):
          */
         char       *enrname;        /* name of ephemeral named relation */
         double      enrtuples;      /* estimated or actual from caller */
     
         /*
          * Fields valid in all RTEs:
          */
         Alias      *alias;          /* user-written alias clause, if any */
         Alias      *eref;           /* expanded reference names */
         bool        lateral;        /* subquery, function, or values is LATERAL? */
         bool        inh;            /* inheritance requested? */
         bool        inFromCl;       /* present in FROM clause? */
         AclMode     requiredPerms;  /* bitmask of required access permissions */
         Oid         checkAsUser;    /* if valid, check access as this role */
         Bitmapset  *selectedCols;   /* columns needing SELECT permission */
         Bitmapset  *insertedCols;   /* columns needing INSERT permission */
         Bitmapset  *updatedCols;    /* columns needing UPDATE permission */
         List       *securityQuals;  /* security barrier quals to apply, if any */
     } RangeTblEntry;
     
    

_4、TargetEntry_

    
    
     /*--------------------      * TargetEntry -      *     a target entry (used in query target lists)
      *
      * Strictly speaking, a TargetEntry isn't an expression node (since it can't
      * be evaluated by ExecEvalExpr).  But we treat it as one anyway, since in
      * very many places it's convenient to process a whole query targetlist as a
      * single expression tree.
      *
      * In a SELECT's targetlist, resno should always be equal to the item's
      * ordinal position (counting from 1).  However, in an INSERT or UPDATE
      * targetlist, resno represents the attribute number of the destination
      * column for the item; so there may be missing or out-of-order resnos.
      * It is even legal to have duplicated resnos; consider
      *      UPDATE table SET arraycol[1] = ..., arraycol[2] = ..., ...
      * The two meanings come together in the executor, because the planner
      * transforms INSERT/UPDATE tlists into a normalized form with exactly
      * one entry for each column of the destination table.  Before that's
      * happened, however, it is risky to assume that resno == position.
      * Generally get_tle_by_resno() should be used rather than list_nth()
      * to fetch tlist entries by resno, and only in SELECT should you assume
      * that resno is a unique identifier.
      *
      * resname is required to represent the correct column name in non-resjunk
      * entries of top-level SELECT targetlists, since it will be used as the
      * column title sent to the frontend.  In most other contexts it is only
      * a debugging aid, and may be wrong or even NULL.  (In particular, it may
      * be wrong in a tlist from a stored rule, if the referenced column has been
      * renamed by ALTER TABLE since the rule was made.  Also, the planner tends
      * to store NULL rather than look up a valid name for tlist entries in
      * non-toplevel plan nodes.)  In resjunk entries, resname should be either
      * a specific system-generated name (such as "ctid") or NULL; anything else
      * risks confusing ExecGetJunkAttribute!
      *
      * ressortgroupref is used in the representation of ORDER BY, GROUP BY, and
      * DISTINCT items.  Targetlist entries with ressortgroupref=0 are not
      * sort/group items.  If ressortgroupref>0, then this item is an ORDER BY,
      * GROUP BY, and/or DISTINCT target value.  No two entries in a targetlist
      * may have the same nonzero ressortgroupref --- but there is no particular
      * meaning to the nonzero values, except as tags.  (For example, one must
      * not assume that lower ressortgroupref means a more significant sort key.)
      * The order of the associated SortGroupClause lists determine the semantics.
      *
      * resorigtbl/resorigcol identify the source of the column, if it is a
      * simple reference to a column of a base table (or view).  If it is not
      * a simple reference, these fields are zeroes.
      *
      * If resjunk is true then the column is a working column (such as a sort key)
      * that should be removed from the final output of the query.  Resjunk columns
      * must have resnos that cannot duplicate any regular column's resno.  Also
      * note that there are places that assume resjunk columns come after non-junk
      * columns.
      *--------------------      */
     typedef struct TargetEntry
     {
         Expr        xpr;
         Expr       *expr;           /* expression to evaluate */
         AttrNumber  resno;          /* attribute number (see notes above) */
         char       *resname;        /* name of the column (could be NULL) */
         Index       ressortgroupref;    /* nonzero if referenced by a sort/group
                                          * clause */
         Oid         resorigtbl;     /* OID of column's source table */
         AttrNumber  resorigcol;     /* column's number in source table */
         bool        resjunk;        /* set to true to eliminate the attribute from
                                      * final target list */
     } TargetEntry;
      
    

_5、RelOptInfo_

    
    
     /*----------      * RelOptInfo
      *      Per-relation information for planning/optimization
      *
      * For planning purposes, a "base rel" is either a plain relation (a table)
      * or the output of a sub-SELECT or function that appears in the range table.
      * In either case it is uniquely identified by an RT index.  A "joinrel"
      * is the joining of two or more base rels.  A joinrel is identified by
      * the set of RT indexes for its component baserels.  We create RelOptInfo
      * nodes for each baserel and joinrel, and store them in the PlannerInfo's
      * simple_rel_array and join_rel_list respectively.
      *
      * Note that there is only one joinrel for any given set of component
      * baserels, no matter what order we assemble them in; so an unordered
      * set is the right datatype to identify it with.
      *
      * We also have "other rels", which are like base rels in that they refer to
      * single RT indexes; but they are not part of the join tree, and are given
      * a different RelOptKind to identify them.
      * Currently the only kind of otherrels are those made for member relations
      * of an "append relation", that is an inheritance set or UNION ALL subquery.
      * An append relation has a parent RTE that is a base rel, which represents
      * the entire append relation.  The member RTEs are otherrels.  The parent
      * is present in the query join tree but the members are not.  The member
      * RTEs and otherrels are used to plan the scans of the individual tables or
      * subqueries of the append set; then the parent baserel is given Append
      * and/or MergeAppend paths comprising the best paths for the individual
      * member rels.  (See comments for AppendRelInfo for more information.)
      *
      * At one time we also made otherrels to represent join RTEs, for use in
      * handling join alias Vars.  Currently this is not needed because all join
      * alias Vars are expanded to non-aliased form during preprocess_expression.
      *
      * We also have relations representing joins between child relations of
      * different partitioned tables. These relations are not added to
      * join_rel_level lists as they are not joined directly by the dynamic
      * programming algorithm.
      *
      * There is also a RelOptKind for "upper" relations, which are RelOptInfos
      * that describe post-scan/join processing steps, such as aggregation.
      * Many of the fields in these RelOptInfos are meaningless, but their Path
      * fields always hold Paths showing ways to do that processing step.
      *
      * Lastly, there is a RelOptKind for "dead" relations, which are base rels
      * that we have proven we don't need to join after all.
      *
      * Parts of this data structure are specific to various scan and join
      * mechanisms.  It didn't seem worth creating new node types for them.
      *
      *      relids - Set of base-relation identifiers; it is a base relation
      *              if there is just one, a join relation if more than one
      *      rows - estimated number of tuples in the relation after restriction
      *             clauses have been applied (ie, output rows of a plan for it)
      *      consider_startup - true if there is any value in keeping plain paths for
      *                         this rel on the basis of having cheap startup cost
      *      consider_param_startup - the same for parameterized paths
      *      reltarget - Default Path output tlist for this rel; normally contains
      *                  Var and PlaceHolderVar nodes for the values we need to
      *                  output from this relation.
      *                  List is in no particular order, but all rels of an
      *                  appendrel set must use corresponding orders.
      *                  NOTE: in an appendrel child relation, may contain
      *                  arbitrary expressions pulled up from a subquery!
      *      pathlist - List of Path nodes, one for each potentially useful
      *                 method of generating the relation
      *      ppilist - ParamPathInfo nodes for parameterized Paths, if any
      *      cheapest_startup_path - the pathlist member with lowest startup cost
      *          (regardless of ordering) among the unparameterized paths;
      *          or NULL if there is no unparameterized path
      *      cheapest_total_path - the pathlist member with lowest total cost
      *          (regardless of ordering) among the unparameterized paths;
      *          or if there is no unparameterized path, the path with lowest
      *          total cost among the paths with minimum parameterization
      *      cheapest_unique_path - for caching cheapest path to produce unique
      *          (no duplicates) output from relation; NULL if not yet requested
      *      cheapest_parameterized_paths - best paths for their parameterizations;
      *          always includes cheapest_total_path, even if that's unparameterized
      *      direct_lateral_relids - rels this rel has direct LATERAL references to
      *      lateral_relids - required outer rels for LATERAL, as a Relids set
      *          (includes both direct and indirect lateral references)
      *
      * If the relation is a base relation it will have these fields set:
      *
      *      relid - RTE index (this is redundant with the relids field, but
      *              is provided for convenience of access)
      *      rtekind - copy of RTE's rtekind field
      *      min_attr, max_attr - range of valid AttrNumbers for rel
      *      attr_needed - array of bitmapsets indicating the highest joinrel
      *              in which each attribute is needed; if bit 0 is set then
      *              the attribute is needed as part of final targetlist
      *      attr_widths - cache space for per-attribute width estimates;
      *                    zero means not computed yet
      *      lateral_vars - lateral cross-references of rel, if any (list of
      *                     Vars and PlaceHolderVars)
      *      lateral_referencers - relids of rels that reference this one laterally
      *              (includes both direct and indirect lateral references)
      *      indexlist - list of IndexOptInfo nodes for relation's indexes
      *                  (always NIL if it's not a table)
      *      pages - number of disk pages in relation (zero if not a table)
      *      tuples - number of tuples in relation (not considering restrictions)
      *      allvisfrac - fraction of disk pages that are marked all-visible
      *      subroot - PlannerInfo for subquery (NULL if it's not a subquery)
      *      subplan_params - list of PlannerParamItems to be passed to subquery
      *
      *      Note: for a subquery, tuples and subroot are not set immediately
      *      upon creation of the RelOptInfo object; they are filled in when
      *      set_subquery_pathlist processes the object.
      *
      *      For otherrels that are appendrel members, these fields are filled
      *      in just as for a baserel, except we don't bother with lateral_vars.
      *
      * If the relation is either a foreign table or a join of foreign tables that
      * all belong to the same foreign server and are assigned to the same user to
      * check access permissions as (cf checkAsUser), these fields will be set:
      *
      *      serverid - OID of foreign server, if foreign table (else InvalidOid)
      *      userid - OID of user to check access as (InvalidOid means current user)
      *      useridiscurrent - we've assumed that userid equals current user
      *      fdwroutine - function hooks for FDW, if foreign table (else NULL)
      *      fdw_private - private state for FDW, if foreign table (else NULL)
      *
      * Two fields are used to cache knowledge acquired during the join search
      * about whether this rel is provably unique when being joined to given other
      * relation(s), ie, it can have at most one row matching any given row from
      * that join relation.  Currently we only attempt such proofs, and thus only
      * populate these fields, for base rels; but someday they might be used for
      * join rels too:
      *
      *      unique_for_rels - list of Relid sets, each one being a set of other
      *                  rels for which this one has been proven unique
      *      non_unique_for_rels - list of Relid sets, each one being a set of
      *                  other rels for which we have tried and failed to prove
      *                  this one unique
      *
      * The presence of the following fields depends on the restrictions
      * and joins that the relation participates in:
      *
      *      baserestrictinfo - List of RestrictInfo nodes, containing info about
      *                  each non-join qualification clause in which this relation
      *                  participates (only used for base rels)
      *      baserestrictcost - Estimated cost of evaluating the baserestrictinfo
      *                  clauses at a single tuple (only used for base rels)
      *      baserestrict_min_security - Smallest security_level found among
      *                  clauses in baserestrictinfo
      *      joininfo  - List of RestrictInfo nodes, containing info about each
      *                  join clause in which this relation participates (but
      *                  note this excludes clauses that might be derivable from
      *                  EquivalenceClasses)
      *      has_eclass_joins - flag that EquivalenceClass joins are possible
      *
      * Note: Keeping a restrictinfo list in the RelOptInfo is useful only for
      * base rels, because for a join rel the set of clauses that are treated as
      * restrict clauses varies depending on which sub-relations we choose to join.
      * (For example, in a 3-base-rel join, a clause relating rels 1 and 2 must be
      * treated as a restrictclause if we join {1} and {2 3} to make {1 2 3}; but
      * if we join {1 2} and {3} then that clause will be a restrictclause in {1 2}
      * and should not be processed again at the level of {1 2 3}.)  Therefore,
      * the restrictinfo list in the join case appears in individual JoinPaths
      * (field joinrestrictinfo), not in the parent relation.  But it's OK for
      * the RelOptInfo to store the joininfo list, because that is the same
      * for a given rel no matter how we form it.
      *
      * We store baserestrictcost in the RelOptInfo (for base relations) because
      * we know we will need it at least once (to price the sequential scan)
      * and may need it multiple times to price index scans.
      *
      * If the relation is partitioned, these fields will be set:
      *
      *      part_scheme - Partitioning scheme of the relation
      *      nparts - Number of partitions
      *      boundinfo - Partition bounds
      *      partition_qual - Partition constraint if not the root
      *      part_rels - RelOptInfos for each partition
      *      partexprs, nullable_partexprs - Partition key expressions
      *      partitioned_child_rels - RT indexes of unpruned partitions of
      *                               this relation that are partitioned tables
      *                               themselves, in hierarchical order
      *
      * Note: A base relation always has only one set of partition keys, but a join
      * relation may have as many sets of partition keys as the number of relations
      * being joined. partexprs and nullable_partexprs are arrays containing
      * part_scheme->partnatts elements each. Each of these elements is a list of
      * partition key expressions.  For a base relation each list in partexprs
      * contains only one expression and nullable_partexprs is not populated. For a
      * join relation, partexprs and nullable_partexprs contain partition key
      * expressions from non-nullable and nullable relations resp. Lists at any
      * given position in those arrays together contain as many elements as the
      * number of joining relations.
      *----------      */
     typedef enum RelOptKind
     {
         RELOPT_BASEREL,
         RELOPT_JOINREL,
         RELOPT_OTHER_MEMBER_REL,
         RELOPT_OTHER_JOINREL,
         RELOPT_UPPER_REL,
         RELOPT_OTHER_UPPER_REL,
         RELOPT_DEADREL
     } RelOptKind;
     
     /*
      * Is the given relation a simple relation i.e a base or "other" member
      * relation?
      */
     #define IS_SIMPLE_REL(rel) \
         ((rel)->reloptkind == RELOPT_BASEREL || \
          (rel)->reloptkind == RELOPT_OTHER_MEMBER_REL)
     
     /* Is the given relation a join relation? */
     #define IS_JOIN_REL(rel)    \
         ((rel)->reloptkind == RELOPT_JOINREL || \
          (rel)->reloptkind == RELOPT_OTHER_JOINREL)
     
     /* Is the given relation an upper relation? */
     #define IS_UPPER_REL(rel)   \
         ((rel)->reloptkind == RELOPT_UPPER_REL || \
          (rel)->reloptkind == RELOPT_OTHER_UPPER_REL)
     
     /* Is the given relation an "other" relation? */
     #define IS_OTHER_REL(rel) \
         ((rel)->reloptkind == RELOPT_OTHER_MEMBER_REL || \
          (rel)->reloptkind == RELOPT_OTHER_JOINREL || \
          (rel)->reloptkind == RELOPT_OTHER_UPPER_REL)
     
     typedef struct RelOptInfo
     {
         NodeTag     type;
     
         RelOptKind  reloptkind;
     
         /* all relations included in this RelOptInfo */
         Relids      relids;         /* set of base relids (rangetable indexes) */
     
         /* size estimates generated by planner */
         double      rows;           /* estimated number of result tuples */
     
         /* per-relation planner control flags */
         bool        consider_startup;   /* keep cheap-startup-cost paths? */
         bool        consider_param_startup; /* ditto, for parameterized paths? */
         bool        consider_parallel;  /* consider parallel paths? */
     
         /* default result targetlist for Paths scanning this relation */
         struct PathTarget *reltarget;   /* list of Vars/Exprs, cost, width */
     
         /* materialization information */
         List       *pathlist;       /* Path structures */
         List       *ppilist;        /* ParamPathInfos used in pathlist */
         List       *partial_pathlist;   /* partial Paths */
         struct Path *cheapest_startup_path;
         struct Path *cheapest_total_path;
         struct Path *cheapest_unique_path;
         List       *cheapest_parameterized_paths;
     
         /* parameterization information needed for both base rels and join rels */
         /* (see also lateral_vars and lateral_referencers) */
         Relids      direct_lateral_relids;  /* rels directly laterally referenced */
         Relids      lateral_relids; /* minimum parameterization of rel */
     
         /* information about a base rel (not set for join rels!) */
         Index       relid;
         Oid         reltablespace;  /* containing tablespace */
         RTEKind     rtekind;        /* RELATION, SUBQUERY, FUNCTION, etc */
         AttrNumber  min_attr;       /* smallest attrno of rel (often <0) */
         AttrNumber  max_attr;       /* largest attrno of rel */
         Relids     *attr_needed;    /* array indexed [min_attr .. max_attr] */
         int32      *attr_widths;    /* array indexed [min_attr .. max_attr] */
         List       *lateral_vars;   /* LATERAL Vars and PHVs referenced by rel */
         Relids      lateral_referencers;    /* rels that reference me laterally */
         List       *indexlist;      /* list of IndexOptInfo */
         List       *statlist;       /* list of StatisticExtInfo */
         BlockNumber pages;          /* size estimates derived from pg_class */
         double      tuples;
         double      allvisfrac;
         PlannerInfo *subroot;       /* if subquery */
         List       *subplan_params; /* if subquery */
         int         rel_parallel_workers;   /* wanted number of parallel workers */
     
         /* Information about foreign tables and foreign joins */
         Oid         serverid;       /* identifies server for the table or join */
         Oid         userid;         /* identifies user to check access as */
         bool        useridiscurrent;    /* join is only valid for current user */
         /* use "struct FdwRoutine" to avoid including fdwapi.h here */
         struct FdwRoutine *fdwroutine;
         void       *fdw_private;
     
         /* cache space for remembering if we have proven this relation unique */
         List       *unique_for_rels;    /* known unique for these other relid
                                          * set(s) */
         List       *non_unique_for_rels;    /* known not unique for these set(s) */
     
         /* used by various scans and joins: */
         List       *baserestrictinfo;   /* RestrictInfo structures (if base rel) */
         QualCost    baserestrictcost;   /* cost of evaluating the above */
         Index       baserestrict_min_security;  /* min security_level found in
                                                  * baserestrictinfo */
         List       *joininfo;       /* RestrictInfo structures for join clauses
                                      * involving this rel */
         bool        has_eclass_joins;   /* T means joininfo is incomplete */
     
         /* used by "other" relations */
         Relids      top_parent_relids;  /* Relids of topmost parents */
     
         /* used for partitioned relations */
         PartitionScheme part_scheme;    /* Partitioning scheme. */
         int         nparts;         /* number of partitions */
         struct PartitionBoundInfoData *boundinfo;   /* Partition bounds */
         List       *partition_qual; /* partition constraint */
         struct RelOptInfo **part_rels;  /* Array of RelOptInfos of partitions,
                                          * stored in the same order of bounds */
         List      **partexprs;      /* Non-nullable partition key expressions. */
         List      **nullable_partexprs; /* Nullable partition key expressions. */
         List       *partitioned_child_rels; /* List of RT indexes. */
     } RelOptInfo;
     
    

_6、Path_

    
    
     /*
      * Type "Path" is used as-is for sequential-scan paths, as well as some other
      * simple plan types that we don't need any extra information in the path for.
      * For other path types it is the first component of a larger struct.
      *
      * "pathtype" is the NodeTag of the Plan node we could build from this Path.
      * It is partially redundant with the Path's NodeTag, but allows us to use
      * the same Path type for multiple Plan types when there is no need to
      * distinguish the Plan type during path processing.
      *
      * "parent" identifies the relation this Path scans, and "pathtarget"
      * describes the precise set of output columns the Path would compute.
      * In simple cases all Paths for a given rel share the same targetlist,
      * which we represent by having path->pathtarget equal to parent->reltarget.
      *
      * "param_info", if not NULL, links to a ParamPathInfo that identifies outer
      * relation(s) that provide parameter values to each scan of this path.
      * That means this path can only be joined to those rels by means of nestloop
      * joins with this path on the inside.  Also note that a parameterized path
      * is responsible for testing all "movable" joinclauses involving this rel
      * and the specified outer rel(s).
      *
      * "rows" is the same as parent->rows in simple paths, but in parameterized
      * paths and UniquePaths it can be less than parent->rows, reflecting the
      * fact that we've filtered by extra join conditions or removed duplicates.
      *
      * "pathkeys" is a List of PathKey nodes (see above), describing the sort
      * ordering of the path's output rows.
      */
     typedef struct Path
     {
         NodeTag     type;
     
         NodeTag     pathtype;       /* tag identifying scan/join method */
     
         RelOptInfo *parent;         /* the relation this path can build */
         PathTarget *pathtarget;     /* list of Vars/Exprs, cost, width */
     
         ParamPathInfo *param_info;  /* parameterization info, or NULL if none */
     
         bool        parallel_aware; /* engage parallel-aware logic? */
         bool        parallel_safe;  /* OK to use as part of parallel plan? */
         int         parallel_workers;   /* desired # of workers; 0 = not parallel */
     
         /* estimated size/costs for path (see costsize.c for more info) */
         double      rows;           /* estimated number of result tuples */
         Cost        startup_cost;   /* cost expended before fetching any tuples */
         Cost        total_cost;     /* total cost (assuming all tuples fetched) */
     
         List       *pathkeys;       /* sort ordering of path's output */
         /* pathkeys is a List of PathKey nodes; see above */
     } Path;
     
    /*
      * PathKeys
      *
      * The sort ordering of a path is represented by a list of PathKey nodes.
      * An empty list implies no known ordering.  Otherwise the first item
      * represents the primary sort key, the second the first secondary sort key,
      * etc.  The value being sorted is represented by linking to an
      * EquivalenceClass containing that value and including pk_opfamily among its
      * ec_opfamilies.  The EquivalenceClass tells which collation to use, too.
      * This is a convenient method because it makes it trivial to detect
      * equivalent and closely-related orderings. (See optimizer/README for more
      * information.)
      *
      * Note: pk_strategy is either BTLessStrategyNumber (for ASC) or
      * BTGreaterStrategyNumber (for DESC).  We assume that all ordering-capable
      * index types will use btree-compatible strategy numbers.
      */
     typedef struct PathKey
     {
         NodeTag     type;
     
         EquivalenceClass *pk_eclass;    /* the value that is ordered */
         Oid         pk_opfamily;    /* btree opfamily defining the ordering */
         int         pk_strategy;    /* sort direction (ASC or DESC) */
         bool        pk_nulls_first; /* do NULLs come before normal values? */
     } PathKey;
     
     
     
     /*
      * PathTarget
      *
      * This struct contains what we need to know during planning about the
      * targetlist (output columns) that a Path will compute.  Each RelOptInfo
      * includes a default PathTarget, which its individual Paths may simply
      * reference.  However, in some cases a Path may compute outputs different
      * from other Paths, and in that case we make a custom PathTarget for it.
      * For example, an indexscan might return index expressions that would
      * otherwise need to be explicitly calculated.  (Note also that "upper"
      * relations generally don't have useful default PathTargets.)
      *
      * exprs contains bare expressions; they do not have TargetEntry nodes on top,
      * though those will appear in finished Plans.
      *
      * sortgrouprefs[] is an array of the same length as exprs, containing the
      * corresponding sort/group refnos, or zeroes for expressions not referenced
      * by sort/group clauses.  If sortgrouprefs is NULL (which it generally is in
      * RelOptInfo.reltarget targets; only upper-level Paths contain this info),
      * we have not identified sort/group columns in this tlist.  This allows us to
      * deal with sort/group refnos when needed with less expense than including
      * TargetEntry nodes in the exprs list.
      */
     typedef struct PathTarget
     {
         NodeTag     type;
         List       *exprs;          /* list of expressions to be computed */
         Index      *sortgrouprefs;  /* corresponding sort/group refnos, or 0 */
         QualCost    cost;           /* cost of evaluating the expressions */
         int         width;          /* estimated avg width of result tuples */
     } PathTarget;
     
     /* Convenience macro to get a sort/group refno from a PathTarget */
     #define get_pathtarget_sortgroupref(target, colno) \
         ((target)->sortgrouprefs ? (target)->sortgrouprefs[colno] : (Index) 0)
    
    
    

_7、ModifyTable_

    
    
     /* ----------------      *   ModifyTable node -      *      Apply rows produced by subplan(s) to result table(s),
      *      by inserting, updating, or deleting.
      *
      * Note that rowMarks and epqParam are presumed to be valid for all the
      * subplan(s); they can't contain any info that varies across subplans.
      * ----------------      */
     typedef struct ModifyTable
     {
         Plan        plan;
         CmdType     operation;      /* INSERT, UPDATE, or DELETE */
         bool        canSetTag;      /* do we set the command tag/es_processed? */
         Index       nominalRelation;    /* Parent RT index for use of EXPLAIN */
         /* RT indexes of non-leaf tables in a partition tree */
         List       *partitioned_rels;
         bool        partColsUpdated;    /* some part key in hierarchy updated */
         List       *resultRelations;    /* integer list of RT indexes */
         int         resultRelIndex; /* index of first resultRel in plan's list */
         int         rootResultRelIndex; /* index of the partitioned table root */
         List       *plans;          /* plan(s) producing source data */
         List       *withCheckOptionLists;   /* per-target-table WCO lists */
         List       *returningLists; /* per-target-table RETURNING tlists */
         List       *fdwPrivLists;   /* per-target-table FDW private data lists */
         Bitmapset  *fdwDirectModifyPlans;   /* indices of FDW DM plans */
         List       *rowMarks;       /* PlanRowMarks (non-locking only) */
         int         epqParam;       /* ID of Param for EvalPlanQual re-eval */
         OnConflictAction onConflictAction;  /* ON CONFLICT action */
         List       *arbiterIndexes; /* List of ON CONFLICT arbiter index OIDs  */
         List       *onConflictSet;  /* SET for INSERT ON CONFLICT DO UPDATE */
         Node       *onConflictWhere;    /* WHERE for ON CONFLICT UPDATE */
         Index       exclRelRTI;     /* RTI of the EXCLUDED pseudo relation */
         List       *exclRelTlist;   /* tlist of the EXCLUDED pseudo relation */
     } ModifyTable;
    

**依赖的函数**  
_1、grouping_planner_

    
    
    /*--------------------      * grouping_planner
      *    Perform planning steps related to grouping, aggregation, etc.
      *
      * This function adds all required top-level processing to the scan/join
      * Path(s) produced by query_planner.
      *
      * If inheritance_update is true, we're being called from inheritance_planner
      * and should not include a ModifyTable step in the resulting Path(s).
      * (inheritance_planner will create a single ModifyTable node covering all the
      * target tables.)
      *
      * tuple_fraction is the fraction of tuples we expect will be retrieved.
      * tuple_fraction is interpreted as follows:
      *    0: expect all tuples to be retrieved (normal case)
      *    0 < tuple_fraction < 1: expect the given fraction of tuples available
      *      from the plan to be retrieved
      *    tuple_fraction >= 1: tuple_fraction is the absolute number of tuples
      *      expected to be retrieved (ie, a LIMIT specification)
      *
      * Returns nothing; the useful output is in the Paths we attach to the
      * (UPPERREL_FINAL, NULL) upperrel in *root.  In addition,
      * root->processed_tlist contains the final processed targetlist.
      *
      * Note that we have not done set_cheapest() on the final rel; it's convenient
      * to leave this to the caller.
      *--------------------      */
     static void
     grouping_planner(PlannerInfo *root, bool inheritance_update,
                      double tuple_fraction)
     {
         Query      *parse = root->parse;
         List       *tlist;
         int64       offset_est = 0;
         int64       count_est = 0;
         double      limit_tuples = -1.0;
         bool        have_postponed_srfs = false;
         PathTarget *final_target;
         List       *final_targets;
         List       *final_targets_contain_srfs;
         bool        final_target_parallel_safe;
         RelOptInfo *current_rel;
         RelOptInfo *final_rel;
         ListCell   *lc;
     
         /* Tweak caller-supplied tuple_fraction if have LIMIT/OFFSET */
         if (parse->limitCount || parse->limitOffset)
         {
             tuple_fraction = preprocess_limit(root, tuple_fraction,
                                               &offset_est, &count_est);
     
             /*
              * If we have a known LIMIT, and don't have an unknown OFFSET, we can
              * estimate the effects of using a bounded sort.
              */
             if (count_est > 0 && offset_est >= 0)
                 limit_tuples = (double) count_est + (double) offset_est;
         }
     
         /* Make tuple_fraction accessible to lower-level routines */
         root->tuple_fraction = tuple_fraction;
     
         if (parse->setOperations)
         {
             /*
              * If there's a top-level ORDER BY, assume we have to fetch all the
              * tuples.  This might be too simplistic given all the hackery below
              * to possibly avoid the sort; but the odds of accurate estimates here
              * are pretty low anyway.  XXX try to get rid of this in favor of
              * letting plan_set_operations generate both fast-start and
              * cheapest-total paths.
              */
             if (parse->sortClause)
                 root->tuple_fraction = 0.0;
     
             /*
              * Construct Paths for set operations.  The results will not need any
              * work except perhaps a top-level sort and/or LIMIT.  Note that any
              * special work for recursive unions is the responsibility of
              * plan_set_operations.
              */
             current_rel = plan_set_operations(root);
     
             /*
              * We should not need to call preprocess_targetlist, since we must be
              * in a SELECT query node.  Instead, use the targetlist returned by
              * plan_set_operations (since this tells whether it returned any
              * resjunk columns!), and transfer any sort key information from the
              * original tlist.
              */
             Assert(parse->commandType == CMD_SELECT);
     
             tlist = root->processed_tlist;  /* from plan_set_operations */
     
             /* for safety, copy processed_tlist instead of modifying in-place */
             tlist = postprocess_setop_tlist(copyObject(tlist), parse->targetList);
     
             /* Save aside the final decorated tlist */
             root->processed_tlist = tlist;
     
             /* Also extract the PathTarget form of the setop result tlist */
             final_target = current_rel->cheapest_total_path->pathtarget;
     
             /* And check whether it's parallel safe */
             final_target_parallel_safe =
                 is_parallel_safe(root, (Node *) final_target->exprs);
     
             /* The setop result tlist couldn't contain any SRFs */
             Assert(!parse->hasTargetSRFs);
             final_targets = final_targets_contain_srfs = NIL;
     
             /*
              * Can't handle FOR [KEY] UPDATE/SHARE here (parser should have
              * checked already, but let's make sure).
              */
             if (parse->rowMarks)
                 ereport(ERROR,
                         (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                 /*------                   translator: %s is a SQL row locking clause such as FOR UPDATE */
                          errmsg("%s is not allowed with UNION/INTERSECT/EXCEPT",
                                 LCS_asString(linitial_node(RowMarkClause,
                                                            parse->rowMarks)->strength))));
     
             /*
              * Calculate pathkeys that represent result ordering requirements
              */
             Assert(parse->distinctClause == NIL);
             root->sort_pathkeys = make_pathkeys_for_sortclauses(root,
                                                                 parse->sortClause,
                                                                 tlist);
         }
         else
         {
             /* No set operations, do regular planning */
             PathTarget *sort_input_target;
             List       *sort_input_targets;
             List       *sort_input_targets_contain_srfs;
             bool        sort_input_target_parallel_safe;
             PathTarget *grouping_target;
             List       *grouping_targets;
             List       *grouping_targets_contain_srfs;
             bool        grouping_target_parallel_safe;
             PathTarget *scanjoin_target;
             List       *scanjoin_targets;
             List       *scanjoin_targets_contain_srfs;
             bool        scanjoin_target_parallel_safe;
             bool        scanjoin_target_same_exprs;
             bool        have_grouping;
             AggClauseCosts agg_costs;
             WindowFuncLists *wflists = NULL;
             List       *activeWindows = NIL;
             grouping_sets_data *gset_data = NULL;
             standard_qp_extra qp_extra;
     
             /* A recursive query should always have setOperations */
             Assert(!root->hasRecursion);
     
             /* Preprocess grouping sets and GROUP BY clause, if any */
             if (parse->groupingSets)
             {
                 gset_data = preprocess_grouping_sets(root);
             }
             else
             {
                 /* Preprocess regular GROUP BY clause, if any */
                 if (parse->groupClause)
                     parse->groupClause = preprocess_groupclause(root, NIL);
             }
     
             /* Preprocess targetlist */
             tlist = preprocess_targetlist(root);
     
             /*
              * We are now done hacking up the query's targetlist.  Most of the
              * remaining planning work will be done with the PathTarget
              * representation of tlists, but save aside the full representation so
              * that we can transfer its decoration (resnames etc) to the topmost
              * tlist of the finished Plan.
              */
             root->processed_tlist = tlist;
     
             /*
              * Collect statistics about aggregates for estimating costs, and mark
              * all the aggregates with resolved aggtranstypes.  We must do this
              * before slicing and dicing the tlist into various pathtargets, else
              * some copies of the Aggref nodes might escape being marked with the
              * correct transtypes.
              *
              * Note: currently, we do not detect duplicate aggregates here.  This
              * may result in somewhat-overestimated cost, which is fine for our
              * purposes since all Paths will get charged the same.  But at some
              * point we might wish to do that detection in the planner, rather
              * than during executor startup.
              */
             MemSet(&agg_costs, 0, sizeof(AggClauseCosts));
             if (parse->hasAggs)
             {
                 get_agg_clause_costs(root, (Node *) tlist, AGGSPLIT_SIMPLE,
                                      &agg_costs);
                 get_agg_clause_costs(root, parse->havingQual, AGGSPLIT_SIMPLE,
                                      &agg_costs);
             }
     
             /*
              * Locate any window functions in the tlist.  (We don't need to look
              * anywhere else, since expressions used in ORDER BY will be in there
              * too.)  Note that they could all have been eliminated by constant
              * folding, in which case we don't need to do any more work.
              */
             if (parse->hasWindowFuncs)
             {
                 wflists = find_window_functions((Node *) tlist,
                                                 list_length(parse->windowClause));
                 if (wflists->numWindowFuncs > 0)
                     activeWindows = select_active_windows(root, wflists);
                 else
                     parse->hasWindowFuncs = false;
             }
     
             /*
              * Preprocess MIN/MAX aggregates, if any.  Note: be careful about
              * adding logic between here and the query_planner() call.  Anything
              * that is needed in MIN/MAX-optimizable cases will have to be
              * duplicated in planagg.c.
              */
             if (parse->hasAggs)
                 preprocess_minmax_aggregates(root, tlist);
     
             /*
              * Figure out whether there's a hard limit on the number of rows that
              * query_planner's result subplan needs to return.  Even if we know a
              * hard limit overall, it doesn't apply if the query has any
              * grouping/aggregation operations, or SRFs in the tlist.
              */
             if (parse->groupClause ||
                 parse->groupingSets ||
                 parse->distinctClause ||
                 parse->hasAggs ||
                 parse->hasWindowFuncs ||
                 parse->hasTargetSRFs ||
                 root->hasHavingQual)
                 root->limit_tuples = -1.0;
             else
                 root->limit_tuples = limit_tuples;
     
             /* Set up data needed by standard_qp_callback */
             qp_extra.tlist = tlist;
             qp_extra.activeWindows = activeWindows;
             qp_extra.groupClause = (gset_data
                                     ? (gset_data->rollups ? linitial_node(RollupData, gset_data->rollups)->groupClause : NIL)
                                     : parse->groupClause);
     
             /*
              * Generate the best unsorted and presorted paths for the scan/join
              * portion of this Query, ie the processing represented by the
              * FROM/WHERE clauses.  (Note there may not be any presorted paths.)
              * We also generate (in standard_qp_callback) pathkey representations
              * of the query's sort clause, distinct clause, etc.
              */
             current_rel = query_planner(root, tlist,
                                         standard_qp_callback, &qp_extra);
     
             /*
              * Convert the query's result tlist into PathTarget format.
              *
              * Note: it's desirable to not do this till after query_planner(),
              * because the target width estimates can use per-Var width numbers
              * that were obtained within query_planner().
              */
             final_target = create_pathtarget(root, tlist);
             final_target_parallel_safe =
                 is_parallel_safe(root, (Node *) final_target->exprs);
     
             /*
              * If ORDER BY was given, consider whether we should use a post-sort
              * projection, and compute the adjusted target for preceding steps if
              * so.
              */
             if (parse->sortClause)
             {
                 sort_input_target = make_sort_input_target(root,
                                                            final_target,
                                                            &have_postponed_srfs);
                 sort_input_target_parallel_safe =
                     is_parallel_safe(root, (Node *) sort_input_target->exprs);
             }
             else
             {
                 sort_input_target = final_target;
                 sort_input_target_parallel_safe = final_target_parallel_safe;
             }
     
             /*
              * If we have window functions to deal with, the output from any
              * grouping step needs to be what the window functions want;
              * otherwise, it should be sort_input_target.
              */
             if (activeWindows)
             {
                 grouping_target = make_window_input_target(root,
                                                            final_target,
                                                            activeWindows);
                 grouping_target_parallel_safe =
                     is_parallel_safe(root, (Node *) grouping_target->exprs);
             }
             else
             {
                 grouping_target = sort_input_target;
                 grouping_target_parallel_safe = sort_input_target_parallel_safe;
             }
     
             /*
              * If we have grouping or aggregation to do, the topmost scan/join
              * plan node must emit what the grouping step wants; otherwise, it
              * should emit grouping_target.
              */
             have_grouping = (parse->groupClause || parse->groupingSets ||
                              parse->hasAggs || root->hasHavingQual);
             if (have_grouping)
             {
                 scanjoin_target = make_group_input_target(root, final_target);
                 scanjoin_target_parallel_safe =
                     is_parallel_safe(root, (Node *) grouping_target->exprs);
             }
             else
             {
                 scanjoin_target = grouping_target;
                 scanjoin_target_parallel_safe = grouping_target_parallel_safe;
             }
     
             /*
              * If there are any SRFs in the targetlist, we must separate each of
              * these PathTargets into SRF-computing and SRF-free targets.  Replace
              * each of the named targets with a SRF-free version, and remember the
              * list of additional projection steps we need to add afterwards.
              */
             if (parse->hasTargetSRFs)
             {
                 /* final_target doesn't recompute any SRFs in sort_input_target */
                 split_pathtarget_at_srfs(root, final_target, sort_input_target,
                                          &final_targets,
                                          &final_targets_contain_srfs);
                 final_target = linitial_node(PathTarget, final_targets);
                 Assert(!linitial_int(final_targets_contain_srfs));
                 /* likewise for sort_input_target vs. grouping_target */
                 split_pathtarget_at_srfs(root, sort_input_target, grouping_target,
                                          &sort_input_targets,
                                          &sort_input_targets_contain_srfs);
                 sort_input_target = linitial_node(PathTarget, sort_input_targets);
                 Assert(!linitial_int(sort_input_targets_contain_srfs));
                 /* likewise for grouping_target vs. scanjoin_target */
                 split_pathtarget_at_srfs(root, grouping_target, scanjoin_target,
                                          &grouping_targets,
                                          &grouping_targets_contain_srfs);
                 grouping_target = linitial_node(PathTarget, grouping_targets);
                 Assert(!linitial_int(grouping_targets_contain_srfs));
                 /* scanjoin_target will not have any SRFs precomputed for it */
                 split_pathtarget_at_srfs(root, scanjoin_target, NULL,
                                          &scanjoin_targets,
                                          &scanjoin_targets_contain_srfs);
                 scanjoin_target = linitial_node(PathTarget, scanjoin_targets);
                 Assert(!linitial_int(scanjoin_targets_contain_srfs));
             }
             else
             {
                 /* initialize lists; for most of these, dummy values are OK */
                 final_targets = final_targets_contain_srfs = NIL;
                 sort_input_targets = sort_input_targets_contain_srfs = NIL;
                 grouping_targets = grouping_targets_contain_srfs = NIL;
                 scanjoin_targets = list_make1(scanjoin_target);
                 scanjoin_targets_contain_srfs = NIL;
             }
     
             /* Apply scan/join target. */
             scanjoin_target_same_exprs = list_length(scanjoin_targets) == 1
                 && equal(scanjoin_target->exprs, current_rel->reltarget->exprs);
             apply_scanjoin_target_to_paths(root, current_rel, scanjoin_targets,
                                            scanjoin_targets_contain_srfs,
                                            scanjoin_target_parallel_safe,
                                            scanjoin_target_same_exprs);
     
             /*
              * Save the various upper-rel PathTargets we just computed into
              * root->upper_targets[].  The core code doesn't use this, but it
              * provides a convenient place for extensions to get at the info.  For
              * consistency, we save all the intermediate targets, even though some
              * of the corresponding upperrels might not be needed for this query.
              */
             root->upper_targets[UPPERREL_FINAL] = final_target;
             root->upper_targets[UPPERREL_WINDOW] = sort_input_target;
             root->upper_targets[UPPERREL_GROUP_AGG] = grouping_target;
     
             /*
              * If we have grouping and/or aggregation, consider ways to implement
              * that.  We build a new upperrel representing the output of this
              * phase.
              */
             if (have_grouping)
             {
                 current_rel = create_grouping_paths(root,
                                                     current_rel,
                                                     grouping_target,
                                                     grouping_target_parallel_safe,
                                                     &agg_costs,
                                                     gset_data);
                 /* Fix things up if grouping_target contains SRFs */
                 if (parse->hasTargetSRFs)
                     adjust_paths_for_srfs(root, current_rel,
                                           grouping_targets,
                                           grouping_targets_contain_srfs);
             }
     
             /*
              * If we have window functions, consider ways to implement those.  We
              * build a new upperrel representing the output of this phase.
              */
             if (activeWindows)
             {
                 current_rel = create_window_paths(root,
                                                   current_rel,
                                                   grouping_target,
                                                   sort_input_target,
                                                   sort_input_target_parallel_safe,
                                                   tlist,
                                                   wflists,
                                                   activeWindows);
                 /* Fix things up if sort_input_target contains SRFs */
                 if (parse->hasTargetSRFs)
                     adjust_paths_for_srfs(root, current_rel,
                                           sort_input_targets,
                                           sort_input_targets_contain_srfs);
             }
     
             /*
              * If there is a DISTINCT clause, consider ways to implement that. We
              * build a new upperrel representing the output of this phase.
              */
             if (parse->distinctClause)
             {
                 current_rel = create_distinct_paths(root,
                                                     current_rel);
             }
         }                           /* end of if (setOperations) */
     
         /*
          * If ORDER BY was given, consider ways to implement that, and generate a
          * new upperrel containing only paths that emit the correct ordering and
          * project the correct final_target.  We can apply the original
          * limit_tuples limit in sort costing here, but only if there are no
          * postponed SRFs.
          */
         if (parse->sortClause)
         {
             current_rel = create_ordered_paths(root,
                                                current_rel,
                                                final_target,
                                                final_target_parallel_safe,
                                                have_postponed_srfs ? -1.0 :
                                                limit_tuples);
             /* Fix things up if final_target contains SRFs */
             if (parse->hasTargetSRFs)
                 adjust_paths_for_srfs(root, current_rel,
                                       final_targets,
                                       final_targets_contain_srfs);
         }
     
         /*
          * Now we are prepared to build the final-output upperrel.
          */
         final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
     
         /*
          * If the input rel is marked consider_parallel and there's nothing that's
          * not parallel-safe in the LIMIT clause, then the final_rel can be marked
          * consider_parallel as well.  Note that if the query has rowMarks or is
          * not a SELECT, consider_parallel will be false for every relation in the
          * query.
          */
         if (current_rel->consider_parallel &&
             is_parallel_safe(root, parse->limitOffset) &&
             is_parallel_safe(root, parse->limitCount))
             final_rel->consider_parallel = true;
     
         /*
          * If the current_rel belongs to a single FDW, so does the final_rel.
          */
         final_rel->serverid = current_rel->serverid;
         final_rel->userid = current_rel->userid;
         final_rel->useridiscurrent = current_rel->useridiscurrent;
         final_rel->fdwroutine = current_rel->fdwroutine;
     
         /*
          * Generate paths for the final_rel.  Insert all surviving paths, with
          * LockRows, Limit, and/or ModifyTable steps added if needed.
          */
         foreach(lc, current_rel->pathlist)
         {
             Path       *path = (Path *) lfirst(lc);
     
             /*
              * If there is a FOR [KEY] UPDATE/SHARE clause, add the LockRows node.
              * (Note: we intentionally test parse->rowMarks not root->rowMarks
              * here.  If there are only non-locking rowmarks, they should be
              * handled by the ModifyTable node instead.  However, root->rowMarks
              * is what goes into the LockRows node.)
              */
             if (parse->rowMarks)
             {
                 path = (Path *) create_lockrows_path(root, final_rel, path,
                                                      root->rowMarks,
                                                      SS_assign_special_param(root));
             }
     
             /*
              * If there is a LIMIT/OFFSET clause, add the LIMIT node.
              */
             if (limit_needed(parse))
             {
                 path = (Path *) create_limit_path(root, final_rel, path,
                                                   parse->limitOffset,
                                                   parse->limitCount,
                                                   offset_est, count_est);
             }
     
             /*
              * If this is an INSERT/UPDATE/DELETE, and we're not being called from
              * inheritance_planner, add the ModifyTable node.
              */
             if (parse->commandType != CMD_SELECT && !inheritance_update)
             {
                 List       *withCheckOptionLists;
                 List       *returningLists;
                 List       *rowMarks;
     
                 /*
                  * Set up the WITH CHECK OPTION and RETURNING lists-of-lists, if
                  * needed.
                  */
                 if (parse->withCheckOptions)
                     withCheckOptionLists = list_make1(parse->withCheckOptions);
                 else
                     withCheckOptionLists = NIL;
     
                 if (parse->returningList)
                     returningLists = list_make1(parse->returningList);
                 else
                     returningLists = NIL;
     
                 /*
                  * If there was a FOR [KEY] UPDATE/SHARE clause, the LockRows node
                  * will have dealt with fetching non-locked marked rows, else we
                  * need to have ModifyTable do that.
                  */
                 if (parse->rowMarks)
                     rowMarks = NIL;
                 else
                     rowMarks = root->rowMarks;
     
                 path = (Path *)
                     create_modifytable_path(root, final_rel,
                                             parse->commandType,
                                             parse->canSetTag,
                                             parse->resultRelation,
                                             NIL,
                                             false,
                                             list_make1_int(parse->resultRelation),
                                             list_make1(path),
                                             list_make1(root),
                                             withCheckOptionLists,
                                             returningLists,
                                             rowMarks,
                                             parse->onConflict,
                                             SS_assign_special_param(root));
             }
     
             /* And shove it into final_rel */
             add_path(final_rel, path);
         }
     
         /*
          * Generate partial paths for final_rel, too, if outer query levels might
          * be able to make use of them.
          */
         if (final_rel->consider_parallel && root->query_level > 1 &&
             !limit_needed(parse))
         {
             Assert(!parse->rowMarks && parse->commandType == CMD_SELECT);
             foreach(lc, current_rel->partial_pathlist)
             {
                 Path       *partial_path = (Path *) lfirst(lc);
     
                 add_partial_path(final_rel, partial_path);
             }
         }
     
         /*
          * If there is an FDW that's responsible for all baserels of the query,
          * let it consider adding ForeignPaths.
          */
         if (final_rel->fdwroutine &&
             final_rel->fdwroutine->GetForeignUpperPaths)
             final_rel->fdwroutine->GetForeignUpperPaths(root, UPPERREL_FINAL,
                                                         current_rel, final_rel,
                                                         NULL);
     
         /* Let extensions possibly add some more paths */
         if (create_upper_paths_hook)
             (*create_upper_paths_hook) (root, UPPERREL_FINAL,
                                         current_rel, final_rel, NULL);
     
         /* Note: currently, we leave it to callers to do set_cheapest() */
     }
    
     /*
      * query_planner
      *    Generate a path (that is, a simplified plan) for a basic query,
      *    which may involve joins but not any fancier features.
      *
      * Since query_planner does not handle the toplevel processing (grouping,
      * sorting, etc) it cannot select the best path by itself.  Instead, it
      * returns the RelOptInfo for the top level of joining, and the caller
      * (grouping_planner) can choose among the surviving paths for the rel.
      *
      * root describes the query to plan
      * tlist is the target list the query should produce
      *      (this is NOT necessarily root->parse->targetList!)
      * qp_callback is a function to compute query_pathkeys once it's safe to do so
      * qp_extra is optional extra data to pass to qp_callback
      *
      * Note: the PlannerInfo node also includes a query_pathkeys field, which
      * tells query_planner the sort order that is desired in the final output
      * plan.  This value is *not* available at call time, but is computed by
      * qp_callback once we have completed merging the query's equivalence classes.
      * (We cannot construct canonical pathkeys until that's done.)
      */
     RelOptInfo *
     query_planner(PlannerInfo *root, List *tlist,
                   query_pathkeys_callback qp_callback, void *qp_extra)
     {
         Query      *parse = root->parse;
         List       *joinlist;
         RelOptInfo *final_rel;
         Index       rti;
         double      total_pages;
     
         /*
          * If the query has an empty join tree, then it's something easy like
          * "SELECT 2+2;" or "INSERT ... VALUES()".  Fall through quickly.
          */
         if (parse->jointree->fromlist == NIL)
         {
             /* We need a dummy joinrel to describe the empty set of baserels */
             final_rel = build_empty_join_rel(root);
     
             /*
              * If query allows parallelism in general, check whether the quals are
              * parallel-restricted.  (We need not check final_rel->reltarget
              * because it's empty at this point.  Anything parallel-restricted in
              * the query tlist will be dealt with later.)
              */
             if (root->glob->parallelModeOK)
                 final_rel->consider_parallel =
                     is_parallel_safe(root, parse->jointree->quals);
     
             /* The only path for it is a trivial Result path */
             add_path(final_rel, (Path *)
                      create_result_path(root, final_rel,
                                         final_rel->reltarget,
                                         (List *) parse->jointree->quals));
     
             /* Select cheapest path (pretty easy in this case...) */
             set_cheapest(final_rel);
     
             /*
              * We still are required to call qp_callback, in case it's something
              * like "SELECT 2+2 ORDER BY 1".
              */
             root->canon_pathkeys = NIL;
             (*qp_callback) (root, qp_extra);
     
             return final_rel;
      }
    
       /*
      * create_modifytable_path
      *    Creates a pathnode that represents performing INSERT/UPDATE/DELETE mods
      *
      * 'rel' is the parent relation associated with the result
      * 'operation' is the operation type
      * 'canSetTag' is true if we set the command tag/es_processed
      * 'nominalRelation' is the parent RT index for use of EXPLAIN
      * 'partitioned_rels' is an integer list of RT indexes of non-leaf tables in
      *      the partition tree, if this is an UPDATE/DELETE to a partitioned table.
      *      Otherwise NIL.
      * 'partColsUpdated' is true if any partitioning columns are being updated,
      *      either from the target relation or a descendent partitioned table.
      * 'resultRelations' is an integer list of actual RT indexes of target rel(s)
      * 'subpaths' is a list of Path(s) producing source data (one per rel)
      * 'subroots' is a list of PlannerInfo structs (one per rel)
      * 'withCheckOptionLists' is a list of WCO lists (one per rel)
      * 'returningLists' is a list of RETURNING tlists (one per rel)
      * 'rowMarks' is a list of PlanRowMarks (non-locking only)
      * 'onconflict' is the ON CONFLICT clause, or NULL
      * 'epqParam' is the ID of Param for EvalPlanQual re-eval
      */
     ModifyTablePath *
     create_modifytable_path(PlannerInfo *root, RelOptInfo *rel,
                             CmdType operation, bool canSetTag,
                             Index nominalRelation, List *partitioned_rels,
                             bool partColsUpdated,
                             List *resultRelations, List *subpaths,
                             List *subroots,
                             List *withCheckOptionLists, List *returningLists,
                             List *rowMarks, OnConflictExpr *onconflict,
                             int epqParam)
     {
         ModifyTablePath *pathnode = makeNode(ModifyTablePath);
         double      total_size;
         ListCell   *lc;
     
         Assert(list_length(resultRelations) == list_length(subpaths));
         Assert(list_length(resultRelations) == list_length(subroots));
         Assert(withCheckOptionLists == NIL ||
                list_length(resultRelations) == list_length(withCheckOptionLists));
         Assert(returningLists == NIL ||
                list_length(resultRelations) == list_length(returningLists));
     
         pathnode->path.pathtype = T_ModifyTable;
         pathnode->path.parent = rel;
         /* pathtarget is not interesting, just make it minimally valid */
         pathnode->path.pathtarget = rel->reltarget;
         /* For now, assume we are above any joins, so no parameterization */
         pathnode->path.param_info = NULL;
         pathnode->path.parallel_aware = false;
         pathnode->path.parallel_safe = false;
         pathnode->path.parallel_workers = 0;
         pathnode->path.pathkeys = NIL;
     
         /*
          * Compute cost & rowcount as sum of subpath costs & rowcounts.
          *
          * Currently, we don't charge anything extra for the actual table
          * modification work, nor for the WITH CHECK OPTIONS or RETURNING
          * expressions if any.  It would only be window dressing, since
          * ModifyTable is always a top-level node and there is no way for the
          * costs to change any higher-level planning choices.  But we might want
          * to make it look better sometime.
          */
         pathnode->path.startup_cost = 0;
         pathnode->path.total_cost = 0;
         pathnode->path.rows = 0;
         total_size = 0;
         foreach(lc, subpaths)
         {
             Path       *subpath = (Path *) lfirst(lc);
     
             if (lc == list_head(subpaths))  /* first node? */
                 pathnode->path.startup_cost = subpath->startup_cost;
             pathnode->path.total_cost += subpath->total_cost;
             pathnode->path.rows += subpath->rows;
             total_size += subpath->pathtarget->width * subpath->rows;
         }
     
         /*
          * Set width to the average width of the subpath outputs.  XXX this is
          * totally wrong: we should report zero if no RETURNING, else an average
          * of the RETURNING tlist widths.  But it's what happened historically,
          * and improving it is a task for another day.
          */
         if (pathnode->path.rows > 0)
             total_size /= pathnode->path.rows;
         pathnode->path.pathtarget->width = rint(total_size);
     
         pathnode->operation = operation;
         pathnode->canSetTag = canSetTag;
         pathnode->nominalRelation = nominalRelation;
         pathnode->partitioned_rels = list_copy(partitioned_rels);
         pathnode->partColsUpdated = partColsUpdated;
         pathnode->resultRelations = resultRelations;
         pathnode->subpaths = subpaths;
         pathnode->subroots = subroots;
         pathnode->withCheckOptionLists = withCheckOptionLists;
         pathnode->returningLists = returningLists;
         pathnode->rowMarks = rowMarks;
         pathnode->onconflict = onconflict;
         pathnode->epqParam = epqParam;
     
         return pathnode;
     }
     
    

_2、subquery_planner_

    
    
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
             SS_process_ctes(root);//With 语句
     
         /*
          * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
          * to transform them into joins.  Note that this step does not descend
          * into subqueries; if we pull up any subqueries below, their SubLinks are
          * processed just before pulling them up.
          */
         if (parse->hasSubLinks)
             pull_up_sublinks(root); //转换ANY/EXISTS为JOIN
     
         /*
          * Scan the rangetable for set-returning functions, and inline them if
          * possible (producing subqueries that might get pulled up next).
          * Recursion issues here are handled in the same way as for SubLinks.
          */
         inline_set_returning_functions(root);
     
         /*
          * Check to see if any subqueries in the jointree can be merged into this
          * query.
          */
         pull_up_subqueries(root);//
     
         /*
          * If this is a simple UNION ALL query, flatten it into an appendrel. We
          * do this now because it requires applying pull_up_subqueries to the leaf
          * queries of the UNION ALL, which weren't touched above because they
          * weren't referenced by the jointree (they will be after we do this).
          */
         if (parse->setOperations)
             flatten_simple_union_all(root);
     
         /*
          * Detect whether any rangetable entries are RTE_JOIN kind; if not, we can
          * avoid the expense of doing flatten_join_alias_vars().  Also check for
          * outer joins --- if none, we can skip reduce_outer_joins().  And check
          * for LATERAL RTEs, too.  This must be done after we have done
          * pull_up_subqueries(), of course.
          */
         root->hasJoinRTEs = false;
         root->hasLateralRTEs = false;
         hasOuterJoins = false;
         foreach(l, parse->rtable)
         {
             RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
     
             if (rte->rtekind == RTE_JOIN)
             {
                 root->hasJoinRTEs = true;
                 if (IS_OUTER_JOIN(rte->jointype))
                     hasOuterJoins = true;
             }
             if (rte->lateral)
                 root->hasLateralRTEs = true;
         }
     
         /*
          * Preprocess RowMark information.  We need to do this after subquery
          * pullup (so that all non-inherited RTEs are present) and before
          * inheritance expansion (so that the info is available for
          * expand_inherited_tables to examine and modify).
          */
         preprocess_rowmarks(root);
     
         /*
          * Expand any rangetable entries that are inheritance sets into "append
          * relations".  This can add entries to the rangetable, but they must be
          * plain base relations not joins, so it's OK (and marginally more
          * efficient) to do it after checking for join RTEs.  We must do it after
          * pulling up subqueries, else we'd fail to handle inherited tables in
          * subqueries.
          */
         expand_inherited_tables(root);
     
         /*
          * Set hasHavingQual to remember if HAVING clause is present.  Needed
          * because preprocess_expression will reduce a constant-true condition to
          * an empty qual list ... but "HAVING TRUE" is not a semantic no-op.
          */
         root->hasHavingQual = (parse->havingQual != NULL);
     
         /* Clear this flag; might get set in distribute_qual_to_rels */
         root->hasPseudoConstantQuals = false;
     
         /*
          * Do expression preprocessing on targetlist and quals, as well as other
          * random expressions in the querytree.  Note that we do not need to
          * handle sort/group expressions explicitly, because they are actually
          * part of the targetlist.
          */
         parse->targetList = (List *)
             preprocess_expression(root, (Node *) parse->targetList,
                                   EXPRKIND_TARGET);
     
         /* Constant-folding might have removed all set-returning functions */
         if (parse->hasTargetSRFs)
             parse->hasTargetSRFs = expression_returns_set((Node *) parse->targetList);
     
         newWithCheckOptions = NIL;
         foreach(l, parse->withCheckOptions)
         {
             WithCheckOption *wco = lfirst_node(WithCheckOption, l);
     
             wco->qual = preprocess_expression(root, wco->qual,
                                               EXPRKIND_QUAL);
             if (wco->qual != NULL)
                 newWithCheckOptions = lappend(newWithCheckOptions, wco);
         }
         parse->withCheckOptions = newWithCheckOptions;
     
         parse->returningList = (List *)
             preprocess_expression(root, (Node *) parse->returningList,
                                   EXPRKIND_TARGET);
     
         preprocess_qual_conditions(root, (Node *) parse->jointree);
     
         parse->havingQual = preprocess_expression(root, parse->havingQual,
                                                   EXPRKIND_QUAL);
     
         foreach(l, parse->windowClause)
         {
             WindowClause *wc = lfirst_node(WindowClause, l);
     
             /* partitionClause/orderClause are sort/group expressions */
             wc->startOffset = preprocess_expression(root, wc->startOffset,
                                                     EXPRKIND_LIMIT);
             wc->endOffset = preprocess_expression(root, wc->endOffset,
                                                   EXPRKIND_LIMIT);
         }
     
         parse->limitOffset = preprocess_expression(root, parse->limitOffset,
                                                    EXPRKIND_LIMIT);
         parse->limitCount = preprocess_expression(root, parse->limitCount,
                                                   EXPRKIND_LIMIT);
     
         if (parse->onConflict)
         {
             parse->onConflict->arbiterElems = (List *)
                 preprocess_expression(root,
                                       (Node *) parse->onConflict->arbiterElems,
                                       EXPRKIND_ARBITER_ELEM);
             parse->onConflict->arbiterWhere =
                 preprocess_expression(root,
                                       parse->onConflict->arbiterWhere,
                                       EXPRKIND_QUAL);
             parse->onConflict->onConflictSet = (List *)
                 preprocess_expression(root,
                                       (Node *) parse->onConflict->onConflictSet,
                                       EXPRKIND_TARGET);
             parse->onConflict->onConflictWhere =
                 preprocess_expression(root,
                                       parse->onConflict->onConflictWhere,
                                       EXPRKIND_QUAL);
             /* exclRelTlist contains only Vars, so no preprocessing needed */
         }
     
         root->append_rel_list = (List *)
             preprocess_expression(root, (Node *) root->append_rel_list,
                                   EXPRKIND_APPINFO);
     
         /* Also need to preprocess expressions within RTEs */
         foreach(l, parse->rtable)
         {
             RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
             int         kind;
             ListCell   *lcsq;
     
             if (rte->rtekind == RTE_RELATION)
             {
                 if (rte->tablesample)
                     rte->tablesample = (TableSampleClause *)
                         preprocess_expression(root,
                                               (Node *) rte->tablesample,
                                               EXPRKIND_TABLESAMPLE);
             }
             else if (rte->rtekind == RTE_SUBQUERY)
             {
                 /*
                  * We don't want to do all preprocessing yet on the subquery's
                  * expressions, since that will happen when we plan it.  But if it
                  * contains any join aliases of our level, those have to get
                  * expanded now, because planning of the subquery won't do it.
                  * That's only possible if the subquery is LATERAL.
                  */
                 if (rte->lateral && root->hasJoinRTEs)
                     rte->subquery = (Query *)
                         flatten_join_alias_vars(root, (Node *) rte->subquery);
             }
             else if (rte->rtekind == RTE_FUNCTION)
             {
                 /* Preprocess the function expression(s) fully */
                 kind = rte->lateral ? EXPRKIND_RTFUNC_LATERAL : EXPRKIND_RTFUNC;
                 rte->functions = (List *)
                     preprocess_expression(root, (Node *) rte->functions, kind);
             }
             else if (rte->rtekind == RTE_TABLEFUNC)
             {
                 /* Preprocess the function expression(s) fully */
                 kind = rte->lateral ? EXPRKIND_TABLEFUNC_LATERAL : EXPRKIND_TABLEFUNC;
                 rte->tablefunc = (TableFunc *)
                     preprocess_expression(root, (Node *) rte->tablefunc, kind);
             }
             else if (rte->rtekind == RTE_VALUES)
             {
                 /* Preprocess the values lists fully */
                 kind = rte->lateral ? EXPRKIND_VALUES_LATERAL : EXPRKIND_VALUES;
                 rte->values_lists = (List *)
                     preprocess_expression(root, (Node *) rte->values_lists, kind);
             }
     
             /*
              * Process each element of the securityQuals list as if it were a
              * separate qual expression (as indeed it is).  We need to do it this
              * way to get proper canonicalization of AND/OR structure.  Note that
              * this converts each element into an implicit-AND sublist.
              */
             foreach(lcsq, rte->securityQuals)
             {
                 lfirst(lcsq) = preprocess_expression(root,
                                                      (Node *) lfirst(lcsq),
                                                      EXPRKIND_QUAL);
             }
         }
     
         /*
          * Now that we are done preprocessing expressions, and in particular done
          * flattening join alias variables, get rid of the joinaliasvars lists.
          * They no longer match what expressions in the rest of the tree look
          * like, because we have not preprocessed expressions in those lists (and
          * do not want to; for example, expanding a SubLink there would result in
          * a useless unreferenced subplan).  Leaving them in place simply creates
          * a hazard for later scans of the tree.  We could try to prevent that by
          * using QTW_IGNORE_JOINALIASES in every tree scan done after this point,
          * but that doesn't sound very reliable.
          */
         if (root->hasJoinRTEs)
         {
             foreach(l, parse->rtable)
             {
                 RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
     
                 rte->joinaliasvars = NIL;
             }
         }
     
         /*
          * In some cases we may want to transfer a HAVING clause into WHERE. We
          * cannot do so if the HAVING clause contains aggregates (obviously) or
          * volatile functions (since a HAVING clause is supposed to be executed
          * only once per group).  We also can't do this if there are any nonempty
          * grouping sets; moving such a clause into WHERE would potentially change
          * the results, if any referenced column isn't present in all the grouping
          * sets.  (If there are only empty grouping sets, then the HAVING clause
          * must be degenerate as discussed below.)
          *
          * Also, it may be that the clause is so expensive to execute that we're
          * better off doing it only once per group, despite the loss of
          * selectivity.  This is hard to estimate short of doing the entire
          * planning process twice, so we use a heuristic: clauses containing
          * subplans are left in HAVING.  Otherwise, we move or copy the HAVING
          * clause into WHERE, in hopes of eliminating tuples before aggregation
          * instead of after.
          *
          * If the query has explicit grouping then we can simply move such a
          * clause into WHERE; any group that fails the clause will not be in the
          * output because none of its tuples will reach the grouping or
          * aggregation stage.  Otherwise we must have a degenerate (variable-free)
          * HAVING clause, which we put in WHERE so that query_planner() can use it
          * in a gating Result node, but also keep in HAVING to ensure that we
          * don't emit a bogus aggregated row. (This could be done better, but it
          * seems not worth optimizing.)
          *
          * Note that both havingQual and parse->jointree->quals are in
          * implicitly-ANDed-list form at this point, even though they are declared
          * as Node *.
          */
         newHaving = NIL;
         foreach(l, (List *) parse->havingQual)
         {
             Node       *havingclause = (Node *) lfirst(l);
     
             if ((parse->groupClause && parse->groupingSets) ||
                 contain_agg_clause(havingclause) ||
                 contain_volatile_functions(havingclause) ||
                 contain_subplans(havingclause))
             {
                 /* keep it in HAVING */
                 newHaving = lappend(newHaving, havingclause);
             }
             else if (parse->groupClause && !parse->groupingSets)
             {
                 /* move it to WHERE */
                 parse->jointree->quals = (Node *)
                     lappend((List *) parse->jointree->quals, havingclause);
             }
             else
             {
                 /* put a copy in WHERE, keep it in HAVING */
                 parse->jointree->quals = (Node *)
                     lappend((List *) parse->jointree->quals,
                             copyObject(havingclause));
                 newHaving = lappend(newHaving, havingclause);
             }
         }
         parse->havingQual = (Node *) newHaving;
     
         /* Remove any redundant GROUP BY columns */
         remove_useless_groupby_columns(root);
     
         /*
          * If we have any outer joins, try to reduce them to plain inner joins.
          * This step is most easily done after we've done expression
          * preprocessing.
          */
         if (hasOuterJoins)
             reduce_outer_joins(root);
     
         /*
          * Do the main planning.  If we have an inherited target relation, that
          * needs special processing, else go straight to grouping_planner.
          */
         if (parse->resultRelation &&
             rt_fetch(parse->resultRelation, parse->rtable)->inh)
             inheritance_planner(root);
         else
             grouping_planner(root, false, tuple_fraction);
     
         /*
          * Capture the set of outer-level param IDs we have access to, for use in
          * extParam/allParam calculations later.
          */
         SS_identify_outer_params(root);
     
         /*
          * If any initPlans were created in this query level, adjust the surviving
          * Paths' costs and parallel-safety flags to account for them.  The
          * initPlans won't actually get attached to the plan tree till
          * create_plan() runs, but we must include their effects now.
          */
         final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
         SS_charge_for_initplans(root, final_rel);
     
         /*
          * Make sure we've identified the cheapest Path for the final rel.  (By
          * doing this here not in grouping_planner, we include initPlan costs in
          * the decision, though it's unlikely that will change anything.)
          */
         set_cheapest(final_rel);
     
         return root;
     }
     
    

_3.create_plan_

    
    
     /*
      * create_plan
      *    Creates the access plan for a query by recursively processing the
      *    desired tree of pathnodes, starting at the node 'best_path'.  For
      *    every pathnode found, we create a corresponding plan node containing
      *    appropriate id, target list, and qualification information.
      *
      *    The tlists and quals in the plan tree are still in planner format,
      *    ie, Vars still correspond to the parser's numbering.  This will be
      *    fixed later by setrefs.c.
      *
      *    best_path is the best access path
      *
      *    Returns a Plan tree.
      */
     Plan *
     create_plan(PlannerInfo *root, Path *best_path)
     {
         Plan       *plan;
     
         /* plan_params should not be in use in current query level */
         Assert(root->plan_params == NIL);
     
         /* Initialize this module's private workspace in PlannerInfo */
         root->curOuterRels = NULL;
         root->curOuterParams = NIL;
     
         /* Recursively process the path tree, demanding the correct tlist result */
         plan = create_plan_recurse(root, best_path, CP_EXACT_TLIST);
     
         /*
          * Make sure the topmost plan node's targetlist exposes the original
          * column names and other decorative info.  Targetlists generated within
          * the planner don't bother with that stuff, but we must have it on the
          * top-level tlist seen at execution time.  However, ModifyTable plan
          * nodes don't have a tlist matching the querytree targetlist.
          */
         if (!IsA(plan, ModifyTable))
             apply_tlist_labeling(plan->targetlist, root->processed_tlist);
     
         /*
          * Attach any initPlans created in this query level to the topmost plan
          * node.  (In principle the initplans could go in any plan node at or
          * above where they're referenced, but there seems no reason to put them
          * any lower than the topmost node for the query level.  Also, see
          * comments for SS_finalize_plan before you try to change this.)
          */
         SS_attach_initplans(root, plan);
     
         /* Check we successfully assigned all NestLoopParams to plan nodes */
         if (root->curOuterParams != NIL)
             elog(ERROR, "failed to assign all NestLoopParams to plan nodes");
     
         /*
          * Reset plan_params to ensure param IDs used for nestloop params are not
          * re-used later
          */
         root->plan_params = NIL;
     
         return plan;
     }
    
    /*
      * create_plan_recurse
      *    Recursive guts of create_plan().
      */
     static Plan *
     create_plan_recurse(PlannerInfo *root, Path *best_path, int flags)
     {
         Plan       *plan;
     
         /* Guard against stack overflow due to overly complex plans */
         check_stack_depth();
     
         switch (best_path->pathtype)
         {
             case T_SeqScan:
             case T_SampleScan:
             case T_IndexScan:
             case T_IndexOnlyScan:
             case T_BitmapHeapScan:
             case T_TidScan:
             case T_SubqueryScan:
             case T_FunctionScan:
             case T_TableFuncScan:
             case T_ValuesScan:
             case T_CteScan:
             case T_WorkTableScan:
             case T_NamedTuplestoreScan:
             case T_ForeignScan:
             case T_CustomScan:
                 plan = create_scan_plan(root, best_path, flags);
                 break;
             case T_HashJoin:
             case T_MergeJoin:
             case T_NestLoop:
                 plan = create_join_plan(root,
                                         (JoinPath *) best_path);
                 break;
             case T_Append:
                 plan = create_append_plan(root,
                                           (AppendPath *) best_path);
                 break;
             case T_MergeAppend:
                 plan = create_merge_append_plan(root,
                                                 (MergeAppendPath *) best_path);
                 break;
             case T_Result:
                 if (IsA(best_path, ProjectionPath))
                 {
                     plan = create_projection_plan(root,
                                                   (ProjectionPath *) best_path,
                                                   flags);
                 }
                 else if (IsA(best_path, MinMaxAggPath))
                 {
                     plan = (Plan *) create_minmaxagg_plan(root,
                                                           (MinMaxAggPath *) best_path);
                 }
                 else
                 {
                     Assert(IsA(best_path, ResultPath));
                     plan = (Plan *) create_result_plan(root,
                                                        (ResultPath *) best_path);
                 }
                 break;
             case T_ProjectSet:
                 plan = (Plan *) create_project_set_plan(root,
                                                         (ProjectSetPath *) best_path);
                 break;
             case T_Material:
                 plan = (Plan *) create_material_plan(root,
                                                      (MaterialPath *) best_path,
                                                      flags);
                 break;
             case T_Unique:
                 if (IsA(best_path, UpperUniquePath))
                 {
                     plan = (Plan *) create_upper_unique_plan(root,
                                                              (UpperUniquePath *) best_path,
                                                              flags);
                 }
                 else
                 {
                     Assert(IsA(best_path, UniquePath));
                     plan = create_unique_plan(root,
                                               (UniquePath *) best_path,
                                               flags);
                 }
                 break;
             case T_Gather:
                 plan = (Plan *) create_gather_plan(root,
                                                    (GatherPath *) best_path);
                 break;
             case T_Sort:
                 plan = (Plan *) create_sort_plan(root,
                                                  (SortPath *) best_path,
                                                  flags);
                 break;
             case T_Group:
                 plan = (Plan *) create_group_plan(root,
                                                   (GroupPath *) best_path);
                 break;
             case T_Agg:
                 if (IsA(best_path, GroupingSetsPath))
                     plan = create_groupingsets_plan(root,
                                                     (GroupingSetsPath *) best_path);
                 else
                 {
                     Assert(IsA(best_path, AggPath));
                     plan = (Plan *) create_agg_plan(root,
                                                     (AggPath *) best_path);
                 }
                 break;
             case T_WindowAgg:
                 plan = (Plan *) create_windowagg_plan(root,
                                                       (WindowAggPath *) best_path);
                 break;
             case T_SetOp:
                 plan = (Plan *) create_setop_plan(root,
                                                   (SetOpPath *) best_path,
                                                   flags);
                 break;
             case T_RecursiveUnion:
                 plan = (Plan *) create_recursiveunion_plan(root,
                                                            (RecursiveUnionPath *) best_path);
                 break;
             case T_LockRows:
                 plan = (Plan *) create_lockrows_plan(root,
                                                      (LockRowsPath *) best_path,
                                                      flags);
                 break;
             case T_ModifyTable:
                 plan = (Plan *) create_modifytable_plan(root,
                                                         (ModifyTablePath *) best_path);
                 break;
             case T_Limit:
                 plan = (Plan *) create_limit_plan(root,
                                                   (LimitPath *) best_path,
                                                   flags);
                 break;
             case T_GatherMerge:
                 plan = (Plan *) create_gather_merge_plan(root,
                                                          (GatherMergePath *) best_path);
                 break;
             default:
                 elog(ERROR, "unrecognized node type: %d",
                      (int) best_path->pathtype);
                 plan = NULL;        /* keep compiler quiet */
                 break;
         }
     
         return plan;
     }
    
     /*
      * create_modifytable_plan
      *    Create a ModifyTable plan for 'best_path'.
      *
      *    Returns a Plan node.
      */
     static ModifyTable *
     create_modifytable_plan(PlannerInfo *root, ModifyTablePath *best_path)
     {
         ModifyTable *plan;
         List       *subplans = NIL;
         ListCell   *subpaths,
                    *subroots;
     
         /* Build the plan for each input path */
         forboth(subpaths, best_path->subpaths,
                 subroots, best_path->subroots)
         {
             Path       *subpath = (Path *) lfirst(subpaths);
             PlannerInfo *subroot = (PlannerInfo *) lfirst(subroots);
             Plan       *subplan;
     
             /*
              * In an inherited UPDATE/DELETE, reference the per-child modified
              * subroot while creating Plans from Paths for the child rel.  This is
              * a kluge, but otherwise it's too hard to ensure that Plan creation
              * functions (particularly in FDWs) don't depend on the contents of
              * "root" matching what they saw at Path creation time.  The main
              * downside is that creation functions for Plans that might appear
              * below a ModifyTable cannot expect to modify the contents of "root"
              * and have it "stick" for subsequent processing such as setrefs.c.
              * That's not great, but it seems better than the alternative.
              */
             subplan = create_plan_recurse(subroot, subpath, CP_EXACT_TLIST);
     
             /* Transfer resname/resjunk labeling, too, to keep executor happy */
             apply_tlist_labeling(subplan->targetlist, subroot->processed_tlist);
     
             subplans = lappend(subplans, subplan);
         }
     
         plan = make_modifytable(root,
                                 best_path->operation,
                                 best_path->canSetTag,
                                 best_path->nominalRelation,
                                 best_path->partitioned_rels,
                                 best_path->partColsUpdated,
                                 best_path->resultRelations,
                                 subplans,
                                 best_path->subroots,
                                 best_path->withCheckOptionLists,
                                 best_path->returningLists,
                                 best_path->rowMarks,
                                 best_path->onconflict,
                                 best_path->epqParam);
     
         copy_generic_path_info(&plan->plan, &best_path->path);
     
         return plan;
     }
    
     /*
      * make_modifytable
      *    Build a ModifyTable plan node
      */
     static ModifyTable *
     make_modifytable(PlannerInfo *root,
                      CmdType operation, bool canSetTag,
                      Index nominalRelation, List *partitioned_rels,
                      bool partColsUpdated,
                      List *resultRelations, List *subplans, List *subroots,
                      List *withCheckOptionLists, List *returningLists,
                      List *rowMarks, OnConflictExpr *onconflict, int epqParam)
     {
         ModifyTable *node = makeNode(ModifyTable);
         List       *fdw_private_list;
         Bitmapset  *direct_modify_plans;
         ListCell   *lc;
         ListCell   *lc2;
         int         i;
     
         Assert(list_length(resultRelations) == list_length(subplans));
         Assert(list_length(resultRelations) == list_length(subroots));
         Assert(withCheckOptionLists == NIL ||
                list_length(resultRelations) == list_length(withCheckOptionLists));
         Assert(returningLists == NIL ||
                list_length(resultRelations) == list_length(returningLists));
     
         node->plan.lefttree = NULL;
         node->plan.righttree = NULL;
         node->plan.qual = NIL;
         /* setrefs.c will fill in the targetlist, if needed */
         node->plan.targetlist = NIL;
     
         node->operation = operation;
         node->canSetTag = canSetTag;
         node->nominalRelation = nominalRelation;
         node->partitioned_rels = flatten_partitioned_rels(partitioned_rels);
         node->partColsUpdated = partColsUpdated;
         node->resultRelations = resultRelations;
         node->resultRelIndex = -1;  /* will be set correctly in setrefs.c */
         node->rootResultRelIndex = -1;  /* will be set correctly in setrefs.c */
         node->plans = subplans;
         if (!onconflict)
         {
             node->onConflictAction = ONCONFLICT_NONE;
             node->onConflictSet = NIL;
             node->onConflictWhere = NULL;
             node->arbiterIndexes = NIL;
             node->exclRelRTI = 0;
             node->exclRelTlist = NIL;
         }
         else
         {
             node->onConflictAction = onconflict->action;
             node->onConflictSet = onconflict->onConflictSet;
             node->onConflictWhere = onconflict->onConflictWhere;
     
             /*
              * If a set of unique index inference elements was provided (an
              * INSERT...ON CONFLICT "inference specification"), then infer
              * appropriate unique indexes (or throw an error if none are
              * available).
              */
             node->arbiterIndexes = infer_arbiter_indexes(root);
     
             node->exclRelRTI = onconflict->exclRelIndex;
             node->exclRelTlist = onconflict->exclRelTlist;
         }
         node->withCheckOptionLists = withCheckOptionLists;
         node->returningLists = returningLists;
         node->rowMarks = rowMarks;
         node->epqParam = epqParam;
     
         /*
          * For each result relation that is a foreign table, allow the FDW to
          * construct private plan data, and accumulate it all into a list.
          */
         fdw_private_list = NIL;
         direct_modify_plans = NULL;
         i = 0;
         forboth(lc, resultRelations, lc2, subroots)
         {
             Index       rti = lfirst_int(lc);
             PlannerInfo *subroot = lfirst_node(PlannerInfo, lc2);
             FdwRoutine *fdwroutine;
             List       *fdw_private;
             bool        direct_modify;
     
             /*
              * If possible, we want to get the FdwRoutine from our RelOptInfo for
              * the table.  But sometimes we don't have a RelOptInfo and must get
              * it the hard way.  (In INSERT, the target relation is not scanned,
              * so it's not a baserel; and there are also corner cases for
              * updatable views where the target rel isn't a baserel.)
              */
             if (rti < subroot->simple_rel_array_size &&
                 subroot->simple_rel_array[rti] != NULL)
             {
                 RelOptInfo *resultRel = subroot->simple_rel_array[rti];
     
                 fdwroutine = resultRel->fdwroutine;
             }
             else
             {
                 RangeTblEntry *rte = planner_rt_fetch(rti, subroot);
     
                 Assert(rte->rtekind == RTE_RELATION);
                 if (rte->relkind == RELKIND_FOREIGN_TABLE)
                     fdwroutine = GetFdwRoutineByRelId(rte->relid);
                 else
                     fdwroutine = NULL;
             }
     
             /*
              * Try to modify the foreign table directly if (1) the FDW provides
              * callback functions needed for that, (2) there are no row-level
              * triggers on the foreign table, and (3) there are no WITH CHECK
              * OPTIONs from parent views.
              */
             direct_modify = false;
             if (fdwroutine != NULL &&
                 fdwroutine->PlanDirectModify != NULL &&
                 fdwroutine->BeginDirectModify != NULL &&
                 fdwroutine->IterateDirectModify != NULL &&
                 fdwroutine->EndDirectModify != NULL &&
                 withCheckOptionLists == NIL &&
                 !has_row_triggers(subroot, rti, operation))
                 direct_modify = fdwroutine->PlanDirectModify(subroot, node, rti, i);
             if (direct_modify)
                 direct_modify_plans = bms_add_member(direct_modify_plans, i);
     
             if (!direct_modify &&
                 fdwroutine != NULL &&
                 fdwroutine->PlanForeignModify != NULL)
                 fdw_private = fdwroutine->PlanForeignModify(subroot, node, rti, i);
             else
                 fdw_private = NIL;
             fdw_private_list = lappend(fdw_private_list, fdw_private);
             i++;
         }
         node->fdwPrivLists = fdw_private_list;
         node->fdwDirectModifyPlans = direct_modify_plans;
     
         return node;
     }
     
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# insert into t_insert values(1000,'I am test','I am test','I am test');
    (挂起)
    

启动gdb，跟踪调试：

**_standard_planner_**

    
    
    [root@localhost ~]# gdb -p 1610
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    #跟踪进入subquery_planner(见后)
    (gdb) n
    409   final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
    (gdb) 
    410   best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);
    #最优路径,INSERT语句,Plan为T_ModifyTable
    (gdb) p *best_path
    $51 = {type = T_ModifyTablePath, pathtype = T_ModifyTable, parent = 0x21c40a0, pathtarget = 0x21c42b0, 
      param_info = 0x0, parallel_aware = false, parallel_safe = false, parallel_workers = 0, rows = 1, 
      startup_cost = 0, total_cost = 0.01, pathkeys = 0x0}
    (gdb) 
    412   top_plan = create_plan(root, best_path);
    (gdb) step
    create_plan (root=0x21c2cb0, best_path=0x219dd88) at createplan.c:323
    323   root->curOuterRels = NULL;
    (gdb) n
    324   root->curOuterParams = NIL;
    (gdb) 
    327   plan = create_plan_recurse(root, best_path, CP_EXACT_TLIST);
    (gdb) 
    336   if (!IsA(plan, ModifyTable))
    #plan可用于后续的执行
    (gdb) p *plan
    $53 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, 
      parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    
    

**_subquery_planner_**

    
    
    [root@localhost ~]# gdb -p 1610
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b subquery_planner
    Breakpoint 1 at 0x76a0bb: file planner.c, line 606.
    (gdb) c
    Continuing.
    ...
    Breakpoint 1, subquery_planner (glob=0x21c2c20, parse=0x219de98, parent_root=0x0, hasRecursion=false, 
        tuple_fraction=0) at planner.c:606
    606     root = makeNode(PlannerInfo);
    #输入参数
    #1,glob
    (gdb) p *glob
    $1 = {type = T_PlannerGlobal, boundParams = 0x0, subplans = 0x0, subroots = 0x0, rewindPlanIDs = 0x0, 
      finalrtable = 0x0, finalrowmarks = 0x0, resultRelations = 0x0, nonleafResultRelations = 0x0, 
      rootResultRelations = 0x0, relationOids = 0x0, invalItems = 0x0, paramExecTypes = 0x0, lastPHId = 0, 
      lastRowMarkId = 0, lastPlanNodeId = 0, transientPlan = false, dependsOnRole = false, 
      parallelModeOK = false, parallelModeNeeded = false, maxParallelHazard = 117 'u'}
    #2,parse
    #Query结构体
    (gdb) p *parse
    $2 = {type = T_Query, commandType = CMD_INSERT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 1, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x219e2b8, jointree = 0x21c2aa0, 
      targetList = 0x21c2b20, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 69}
    #targetList中的元素为TargetEntry *
    #在insert语句中,是数据表列
    (gdb) p *(parse->targetList)
    $3 = {type = T_List, length = 4, head = 0x21c2b00, tail = 0x21c2b90}
    (gdb)  p *((TargetEntry *)(parse->targetList->head->data.ptr_value))
    $4 = {xpr = {type = T_TargetEntry}, expr = 0x219e5e8, resno = 1, resname = 0x219e338 "id", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    #rtable中的元素是RangeTblEntry *
    #在insert操作中,是数据表
    (gdb)  p *(parse->rtable)
    $5 = {type = T_List, length = 1, head = 0x219e298, tail = 0x219e298}
    (gdb) p *((RangeTblEntry *)(parse->rtable->head->data.ptr_value))
    $6 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26731, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x219e0b8, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 1, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x21c2938, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(((RangeTblEntry *)(parse->rtable->head->data.ptr_value))->insertedCols)
    $7 = {nwords = 1, words = 0x21c293c}
    #3,parent_root
    (gdb) p *parent_root
    Cannot access memory at address 0x0
    #4,hasRecursion
    (gdb) p hasRecursion
    $9 = false
    #5,tuple_fraction
    (gdb) p tuple_fraction
    $10 = 0
    ...
    639     if (parse->cteList)
    (gdb) 
    648     if (parse->hasSubLinks)
    (gdb) 
    656     inline_set_returning_functions(root);
    (gdb) 
    662     pull_up_subqueries(root);
    (gdb) 
    670     if (parse->setOperations)
    (gdb) 
    680     root->hasJoinRTEs = false;
    (gdb) 
    681     root->hasLateralRTEs = false;
    682     hasOuterJoins = false;
    (gdb) 
    683     foreach(l, parse->rtable)
    (gdb) 
    685         RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
    (gdb) 
    687         if (rte->rtekind == RTE_JOIN)
    (gdb) p *rte
    $11 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26731, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x219e0b8, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 1, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x21c2938, updatedCols = 0x0, 
      securityQuals = 0x0}
    ...
    731     parse->targetList = (List *)
    (gdb) 
    736     if (parse->hasTargetSRFs)
    (gdb) p *((TargetEntry *)(parse->targetList->head->data.ptr_value))
    $12 = {xpr = {type = T_TargetEntry}, expr = 0x21c3110, resno = 1, resname = 0x219e338 "id", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    (gdb) p *(((TargetEntry *)(parse->targetList->head->data.ptr_value))->expr)
    $13 = {type = T_Const}
    ...
    #进入grouping_planner函数,此函数生成root->upper_rels & upper_targets
    #注意upper_rels,grouping_planner函数执行完毕,upper_rels最后一个元素会填入相应的值
    (gdb) p *root
    $22 = {type = T_PlannerInfo, parse = 0x219de98, glob = 0x21c2c20, query_level = 1, parent_root = 0x0, 
      plan_params = 0x0, outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, 
      simple_rte_array = 0x0, all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x0, 
      join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, 
      multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, left_join_clauses = 0x0, 
      right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 
        0x0, 0x0, 0x0}, processed_tlist = 0x0, grouping_map = 0x0, minmax_aggs = 0x0, planner_cxt = 0x219cde0, 
      total_table_pages = 0, tuple_fraction = 0, limit_tuples = 0, qual_security_level = 0, 
      inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, hasLateralRTEs = false, hasDeletedRTEs = false, 
      hasHavingQual = false, hasPseudoConstantQuals = false, hasRecursion = false, wt_param_id = -1, 
      non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, join_search_private = 0x0, 
      partColsUpdated = false}
    (gdb) p inheritance_update
    $23 = false
    (gdb) p inheritance_update
    $24 = false
    (gdb) p tuple_fraction
    $25 = 0
    (gdb) 
    ...
    (gdb) 
    1808      tlist = preprocess_targetlist(root);
    (gdb) p *root
    $27 = {type = T_PlannerInfo, parse = 0x219de98, glob = 0x21c2c20, query_level = 1, parent_root = 0x0, 
      plan_params = 0x0, outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, 
      simple_rte_array = 0x0, all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x0, 
      join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, 
      multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, left_join_clauses = 0x0, 
      right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 
        0x0, 0x0, 0x0}, processed_tlist = 0x21c39e0, grouping_map = 0x0, minmax_aggs = 0x0, 
      planner_cxt = 0x219cde0, total_table_pages = 0, tuple_fraction = 0, limit_tuples = 0, 
      qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, hasLateralRTEs = false, 
      hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, hasRecursion = false, 
      wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    #processed_tlist中的元素为TargetEntry *,也就是字段Column
    (gdb) p *(root->processed_tlist)
    $28 = {type = T_List, length = 4, head = 0x21c39c0, tail = 0x21c3a50}
    (gdb) p *(root->processed_tlist->head)
    $29 = {data = {ptr_value = 0x21c30c0, int_value = 35401920, oid_value = 35401920}, next = 0x21c3a10}
    (gdb) p *(TargetEntry *)(root->processed_tlist->head.data->ptr_value)
    $30 = {xpr = {type = T_TargetEntry}, expr = 0x21c3110, resno = 1, resname = 0x219e338 "id", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    ...
    2026      root->upper_targets[UPPERREL_FINAL] = final_target;
    (gdb) 
    2027      root->upper_targets[UPPERREL_WINDOW] = sort_input_target;
    (gdb) 
    2028      root->upper_targets[UPPERREL_GROUP_AGG] = grouping_target;
    (gdb) 
    2035      if (have_grouping)
    (gdb) p *final_target
    $45 = {type = T_PathTarget, exprs = 0x21c3ee0, sortgrouprefs = 0x21c3ea0, cost = {startup = 0, 
    ...
    (gdb) 
    2197          create_modifytable_path(root, final_rel,
    (gdb) 
    2200                      parse->resultRelation,
    (gdb) p *root
    $49 = {type = T_PlannerInfo, parse = 0x219de98, glob = 0x21c2c20, query_level = 1, parent_root = 0x0, 
      plan_params = 0x0, outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, 
      simple_rte_array = 0x0, all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x21c3cf0, 
      join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, 
      multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, left_join_clauses = 0x0, 
      right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x21c4320}, upper_targets = {0x0, 0x0, 
        0x21c3e50, 0x21c3e50, 0x0, 0x0, 0x21c3e50}, processed_tlist = 0x21c39e0, grouping_map = 0x0, 
      minmax_aggs = 0x0, planner_cxt = 0x219cde0, total_table_pages = 0, tuple_fraction = 0, limit_tuples = -1, 
      qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, hasLateralRTEs = false, 
      hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, hasRecursion = false, 
      wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    (gdb) finish
    Run till exit from #0  grouping_planner (root=0x21c2cb0, inheritance_update=false, tuple_fraction=0)
        at planner.c:2200
    subquery_planner (glob=0x21c2c20, parse=0x219de98, parent_root=0x0, hasRecursion=false, tuple_fraction=0)
        at planner.c:972
    #退出grouping_planner函数
    ...
    #最终的返回值
    #INSERT VALUES语句相对比较简单,没有复杂的JOIN/WITH/HAVING/GROUP等语句,这里只是简单的返回一个root节点
    (gdb) p *root
    $17 = {type = T_PlannerInfo, parse = 0x219de98, glob = 0x21c2c20, query_level = 1, parent_root = 0x0, 
      plan_params = 0x0, outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, 
      simple_rte_array = 0x0, all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x21c3cf0, 
      join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, 
      multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, left_join_clauses = 0x0, 
      right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x21c4320}, upper_targets = {0x0, 0x0, 
        0x21c3e50, 0x21c3e50, 0x0, 0x0, 0x21c3e50}, processed_tlist = 0x21c39e0, grouping_map = 0x0, 
      minmax_aggs = 0x0, planner_cxt = 0x219cde0, total_table_pages = 0, tuple_fraction = 0, limit_tuples = -1, 
      qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, hasLateralRTEs = false, 
      hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, hasRecursion = false, 
      wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    

### 四、小结

1.重要的数据结构:PlannedStmt/PlannerGlobal/PlannerInfo/RelOptInfo/Path  
2.重要的函数:subquery_planner/grouping_planner/create_plan

