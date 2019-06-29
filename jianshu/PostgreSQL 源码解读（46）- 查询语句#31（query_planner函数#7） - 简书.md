先前的章节已介绍了函数query_planner中子函数reconsider_outer_join_clauses和generate_base_implied_equalities的主要实现逻辑,本节继续介绍query_planner中qp_callback（回调函数）、fix_placeholder_input_needed_levels函数的实现逻辑。

query_planner代码片段:

    
    
         //...
         /*
          * We have completed merging equivalence sets, so it's now possible to
          * generate pathkeys in canonical form; so compute query_pathkeys and
          * other pathkeys fields in PlannerInfo.
          */
         (*qp_callback) (root, qp_extra);//调用回调函数,处理PathKeys
     
         /*
          * Examine any "placeholder" expressions generated during subquery pullup.
          * Make sure that the Vars they need are marked as needed at the relevant
          * join level.  This must be done before join removal because it might
          * cause Vars or placeholders to be needed above a join when they weren't
          * so marked before.
          */
         fix_placeholder_input_needed_levels(root);//检查在子查询上拉时生成的PH表达式,确保Vars是OK的
     
    
         //...
    

### 一、数据结构

PlannerInfo与RelOptInfo结构体贯彻逻辑优化和物理优化过程的始终.  
**PlannerInfo**

    
    
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
         NodeTag     type;//Node标识
     
         Query      *parse;          /* 查询树,the Query being planned */
     
         PlannerGlobal *glob;        /* 当前的planner全局信息,global info for current planner run */
     
         Index       query_level;    /* 查询层次,1标识最高层,1 at the outermost Query */
     
         struct PlannerInfo *parent_root;    /* 如为子计划,则这里存储父计划器指针,NULL标识最高层,NULL at outermost Query */
     
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
         /* RelOptInfo数组,存储"base rels",比如基表/子查询等.该数组与RTE的顺序一一对应,而且是从1开始,因此[0]无用 */
         struct RelOptInfo **simple_rel_array;   /* All 1-rel RelOptInfos */
         int         simple_rel_array_size;  /* 数组大小,allocated size of array */
     
         /*
          * simple_rte_array is the same length as simple_rel_array and holds
          * pointers to the associated rangetable entries.  This lets us avoid
          * rt_fetch(), which can be a bit slow once large inheritance sets have
          * been expanded.
          */
         RangeTblEntry **simple_rte_array;   /* RTE数组,rangetable as an array */
     
         /*
          * append_rel_array is the same length as the above arrays, and holds
          * pointers to the corresponding AppendRelInfo entry indexed by
          * child_relid, or NULL if none.  The array itself is not allocated if
          * append_rel_list is empty.
          */
         struct AppendRelInfo **append_rel_array;//先前已介绍,在处理集合操作如UNION ALL时使用
     
         /*
          * all_baserels is a Relids set of all base relids (but not "other"
          * relids) in the query; that is, the Relids identifier of the final join
          * we need to form.  This is computed in make_one_rel, just before we
          * start making Paths.
          */
         Relids      all_baserels;//"base rels"
     
         /*
          * nullable_baserels is a Relids set of base relids that are nullable by
          * some outer join in the jointree; these are rels that are potentially
          * nullable below the WHERE clause, SELECT targetlist, etc.  This is
          * computed in deconstruct_jointree.
          */
         Relids      nullable_baserels;//Nullable-side端的"base rels"
     
         /*
          * join_rel_list is a list of all join-relation RelOptInfos we have
          * considered in this planning run.  For small problems we just scan the
          * list to do lookups, but when there are many join relations we build a
          * hash table for faster lookups.  The hash table is present and valid
          * when join_rel_hash is not NULL.  Note that we still maintain the list
          * even when using the hash table for lookups; this simplifies life for
          * GEQO.
          */
         List       *join_rel_list;  /* 参与连接的Relation的RelOptInfo链表,list of join-relation RelOptInfos */
         struct HTAB *join_rel_hash; /* 可加快链表访问的hash表,optional hashtable for join relations */
     
         /*
          * When doing a dynamic-programming-style join search, join_rel_level[k]
          * is a list of all join-relation RelOptInfos of level k, and
          * join_cur_level is the current level.  New join-relation RelOptInfos are
          * automatically added to the join_rel_level[join_cur_level] list.
          * join_rel_level is NULL if not in use.
          */
         List      **join_rel_level; /* RelOptInfo指针链表数组,k层的join存储在[k]中,lists of join-relation RelOptInfos */
         int         join_cur_level; /* 当前的join层次,index of list being extended */
     
         List       *init_plans;     /* 查询的初始化计划链表,init SubPlans for query */
     
         List       *cte_plan_ids;   /* CTE子计划ID链表,per-CTE-item list of subplan IDs */
     
         List       *multiexpr_params;   /* List of Lists of Params for MULTIEXPR
                                          * subquery outputs */
     
         List       *eq_classes;     /* 活动的等价类链表,list of active EquivalenceClasses */
     
         List       *canon_pathkeys; /* 规范化PathKey链表,list of "canonical" PathKeys */
     
         List       *left_join_clauses;  /* 外连接约束条件链表(左),list of RestrictInfos for mergejoinable
                                          * outer join clauses w/nonnullable var on
                                          * left */
     
         List       *right_join_clauses; /* 外连接约束条件链表(右),list of RestrictInfos for mergejoinable
                                          * outer join clauses w/nonnullable var on
                                          * right */
     
         List       *full_join_clauses;  /* 全连接约束条件链表,list of RestrictInfos for mergejoinable
                                          * full join clauses */
     
         List       *join_info_list; /* 特殊连接信息链表,list of SpecialJoinInfos */
     
         List       *append_rel_list;    /* AppendRelInfo链表,list of AppendRelInfos */
     
         List       *rowMarks;       /* list of PlanRowMarks */
     
         List       *placeholder_list;   /* PHI链表,list of PlaceHolderInfos */
     
         List       *fkey_list;      /* 外键信息链表,list of ForeignKeyOptInfos */
     
         List       *query_pathkeys; /* uery_planner()要求的PathKeys,desired pathkeys for query_planner() */
     
         List       *group_pathkeys; /* groupClause pathkeys, if any */
         List       *window_pathkeys;    /* pathkeys of bottom window, if any */
         List       *distinct_pathkeys;  /* distinctClause pathkeys, if any */
         List       *sort_pathkeys;  /* sortClause pathkeys, if any */
     
         List       *part_schemes;   /* 已规范化的分区Schema,Canonicalised partition schemes used in the
                                      * query. */
     
         List       *initial_rels;   /* 尝试连接的RelOptInfo链表,RelOptInfos we are now trying to join */
     
         /* Use fetch_upper_rel() to get any particular upper rel */
         List       *upper_rels[UPPERREL_FINAL + 1]; /* 上层的RelOptInfo链表, upper-rel RelOptInfos */
     
         /* Result tlists chosen by grouping_planner for upper-stage processing */
         struct PathTarget *upper_targets[UPPERREL_FINAL + 1];//
     
         /*
          * grouping_planner passes back its final processed targetlist here, for
          * use in relabeling the topmost tlist of the finished Plan.
          */
         List       *processed_tlist;//最后需处理的投影列
     
         /* Fields filled during create_plan() for use in setrefs.c */
         AttrNumber *grouping_map;   /* for GroupingFunc fixup */
         List       *minmax_aggs;    /* List of MinMaxAggInfos */
     
         MemoryContext planner_cxt;  /* 内存上下文,context holding PlannerInfo */
     
         double      total_table_pages;  /* 所有的pages,# of pages in all tables of query */
     
         double      tuple_fraction; /* query_planner输入参数:元组处理比例,tuple_fraction passed to query_planner */
         double      limit_tuples;   /* query_planner输入参数:limit_tuples passed to query_planner */
     
         Index       qual_security_level;    /* 表达式的最新安全等级,minimum security_level for quals */
         /* Note: qual_security_level is zero if there are no securityQuals */
     
         InheritanceKind inhTargetKind;  /* indicates if the target relation is an
                                          * inheritance child or partition or a
                                          * partitioned table */
         bool        hasJoinRTEs;    /* 存在RTE_JOIN的RTE,true if any RTEs are RTE_JOIN kind */
         bool        hasLateralRTEs; /* 存在标记为LATERAL的RTE,true if any RTEs are marked LATERAL */
         bool        hasDeletedRTEs; /* 存在已在jointree删除的RTE,true if any RTE was deleted from jointree */
         bool        hasHavingQual;  /* 存在Having子句,true if havingQual was non-null */
         bool        hasPseudoConstantQuals; /* true if any RestrictInfo has
                                              * pseudoconstant = true */
         bool        hasRecursion;   /* 递归语句,true if planning a recursive WITH item */
     
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
     
    

**RelOptInfo**

    
    
     typedef struct RelOptInfo
     {
         NodeTag     type;//节点标识
     
         RelOptKind  reloptkind;//RelOpt类型
     
         /* all relations included in this RelOptInfo */
         Relids      relids;         /*Relids(rtindex)集合 set of base relids (rangetable indexes) */
     
         /* size estimates generated by planner */
         double      rows;           /*结果元组的估算数量 estimated number of result tuples */
     
         /* per-relation planner control flags */
         bool        consider_startup;   /*是否考虑启动成本?是,需要保留启动成本低的路径 keep cheap-startup-cost paths? */
         bool        consider_param_startup; /*是否考虑参数化?的路径 ditto, for parameterized paths? */
         bool        consider_parallel;  /*是否考虑并行处理路径 consider parallel paths? */
     
         /* default result targetlist for Paths scanning this relation */
         struct PathTarget *reltarget;   /*扫描该Relation时默认的结果 list of Vars/Exprs, cost, width */
     
         /* materialization information */
         List       *pathlist;       /*访问路径链表 Path structures */
         List       *ppilist;        /*路径链表中使用参数化路径进行 ParamPathInfos used in pathlist */
         List       *partial_pathlist;   /* partial Paths */
         struct Path *cheapest_startup_path;//代价最低的启动路径
         struct Path *cheapest_total_path;//代价最低的整体路径
         struct Path *cheapest_unique_path;//代价最低的获取唯一值的路径
         List       *cheapest_parameterized_paths;//代价最低的参数化?路径链表
     
         /* parameterization information needed for both base rels and join rels */
         /* (see also lateral_vars and lateral_referencers) */
         Relids      direct_lateral_relids;  /*使用lateral语法,需依赖的Relids rels directly laterally referenced */
         Relids      lateral_relids; /* minimum parameterization of rel */
     
         /* information about a base rel (not set for join rels!) */
         //reloptkind=RELOPT_BASEREL时使用的数据结构
         Index       relid;          /* Relation ID */
         Oid         reltablespace;  /* 表空间 containing tablespace */
         RTEKind     rtekind;        /* 基表?子查询?还是函数等等?RELATION, SUBQUERY, FUNCTION, etc */
         AttrNumber  min_attr;       /* 最小的属性编号 smallest attrno of rel (often <0) */
         AttrNumber  max_attr;       /* 最大的属性编号 largest attrno of rel */
         Relids     *attr_needed;    /* 数组 array indexed [min_attr .. max_attr] */
         int32      *attr_widths;    /* 属性宽度 array indexed [min_attr .. max_attr] */
         List       *lateral_vars;   /* 关系依赖的Vars/PHVs LATERAL Vars and PHVs referenced by rel */
         Relids      lateral_referencers;    /*依赖该关系的Relids rels that reference me laterally */
         List       *indexlist;      /* 该关系的IndexOptInfo链表 list of IndexOptInfo */
         List       *statlist;       /* 统计信息链表 list of StatisticExtInfo */
         BlockNumber pages;          /* 块数 size estimates derived from pg_class */
         double      tuples;         /* 元组数 */
         double      allvisfrac;     /* ? */
         PlannerInfo *subroot;       /* 如为子查询,存储子查询的root if subquery */
         List       *subplan_params; /* 如为子查询,存储子查询的参数 if subquery */
         int         rel_parallel_workers;   /* 并行执行,需要多少个workers? wanted number of parallel workers */
     
         /* Information about foreign tables and foreign joins */
         //FWD相关信息
         Oid         serverid;       /* identifies server for the table or join */
         Oid         userid;         /* identifies user to check access as */
         bool        useridiscurrent;    /* join is only valid for current user */
         /* use "struct FdwRoutine" to avoid including fdwapi.h here */
         struct FdwRoutine *fdwroutine;
         void       *fdw_private;
     
         /* cache space for remembering if we have proven this relation unique */
         //已知的,可保证唯一的Relids链表
         List       *unique_for_rels;    /* known unique for these other relid
                                          * set(s) */
         List       *non_unique_for_rels;    /* 已知的,不唯一的Relids链表 known not unique for these set(s) */
     
         /* used by various scans and joins: */
         List       *baserestrictinfo;   /* 如为基本关系,存储约束条件 RestrictInfo structures (if base rel) */
         QualCost    baserestrictcost;   /* 解析约束表达式的成本? cost of evaluating the above */
         Index       baserestrict_min_security;  /* 最低安全等级 min security_level found in
                                                  * baserestrictinfo */
         List       *joininfo;       /* 连接语句的约束条件信息 RestrictInfo structures for join clauses
                                      * involving this rel */
         bool        has_eclass_joins;   /* 是否存在等价类连接? T means joininfo is incomplete */
     
         /* used by partitionwise joins: */
         bool        consider_partitionwise_join;    /* 分区? consider partitionwise
                                                      * join paths? (if
                                                      * partitioned rel) */
         Relids      top_parent_relids;  /* Relids of topmost parents (if "other"
                                          * rel) */
     
         /* used for partitioned relations */
         //分区表使用
         PartitionScheme part_scheme;    /* 分区的schema Partitioning scheme. */
         int         nparts;         /* 分区数 number of partitions */
         struct PartitionBoundInfoData *boundinfo;   /* 分区边界信息 Partition bounds */
         List       *partition_qual; /* 分区约束 partition constraint */
         struct RelOptInfo **part_rels;  /* 分区的RelOptInfo数组 Array of RelOptInfos of partitions,
                                          * stored in the same order of bounds */
         List      **partexprs;      /* 非空分区键表达式 Non-nullable partition key expressions. */
         List      **nullable_partexprs; /* 可为空的分区键表达式 Nullable partition key expressions. */
         List       *partitioned_child_rels; /* RT Indexes链表 List of RT indexes. */
     } RelOptInfo;
    
    

**PlaceHolderInfo**

    
    
    /*
      * For each distinct placeholder expression generated during planning, we
      * store a PlaceHolderInfo node in the PlannerInfo node's placeholder_list.
      * This stores info that is needed centrally rather than in each copy of the
      * PlaceHolderVar.  The phid fields identify which PlaceHolderInfo goes with
      * each PlaceHolderVar.  Note that phid is unique throughout a planner run,
      * not just within a query level --- this is so that we need not reassign ID's
      * when pulling a subquery into its parent.
      *
      * The idea is to evaluate the expression at (only) the ph_eval_at join level,
      * then allow it to bubble up like a Var until the ph_needed join level.
      * ph_needed has the same definition as attr_needed for a regular Var.
      *
      * The PlaceHolderVar's expression might contain LATERAL references to vars
      * coming from outside its syntactic scope.  If so, those rels are *not*
      * included in ph_eval_at, but they are recorded in ph_lateral.
      *
      * Notice that when ph_eval_at is a join rather than a single baserel, the
      * PlaceHolderInfo may create constraints on join order: the ph_eval_at join
      * has to be formed below any outer joins that should null the PlaceHolderVar.
      *
      * We create a PlaceHolderInfo only after determining that the PlaceHolderVar
      * is actually referenced in the plan tree, so that unreferenced placeholders
      * don't result in unnecessary constraints on join order.
      */
     
     typedef struct PlaceHolderInfo
     {
         NodeTag     type;
     
         Index       phid;           /* PH的ID,ID for PH (unique within planner run) */
         PlaceHolderVar *ph_var;     /* copy of PlaceHolderVar tree */
         Relids      ph_eval_at;     /* lowest level we can evaluate value at */
         Relids      ph_lateral;     /* relids of contained lateral refs, if any */
         Relids      ph_needed;      /* highest level the value is needed at */
         int32       ph_width;       /* estimated attribute width */
     } PlaceHolderInfo;
    

### 二、源码解读

**standard_qp_callback**  
标准的query_planner回调函数,在生成计划的期间处理query_pathkeys和其他pathkeys

    
    
     /*
      * Compute query_pathkeys and other pathkeys during plan generation
      */
     static void
     standard_qp_callback(PlannerInfo *root, void *extra)
     {
         Query      *parse = root->parse;//查询树
         standard_qp_extra *qp_extra = (standard_qp_extra *) extra;//参数
         List       *tlist = qp_extra->tlist;
         List       *activeWindows = qp_extra->activeWindows;
     
         /*
          * Calculate pathkeys that represent grouping/ordering requirements.  The
          * sortClause is certainly sort-able, but GROUP BY and DISTINCT might not
          * be, in which case we just leave their pathkeys empty.
          */
         if (qp_extra->groupClause &&
             grouping_is_sortable(qp_extra->groupClause))//group语句&要求排序
             root->group_pathkeys =
                 make_pathkeys_for_sortclauses(root,
                                               qp_extra->groupClause,
                                               tlist);//构建pathkeys
         else
             root->group_pathkeys = NIL;
     
         /* We consider only the first (bottom) window in pathkeys logic */
         if (activeWindows != NIL)//窗口函数
         {
             WindowClause *wc = linitial_node(WindowClause, activeWindows);
     
             root->window_pathkeys = make_pathkeys_for_window(root,
                                                              wc,
                                                              tlist);
         }
         else
             root->window_pathkeys = NIL;
     
         if (parse->distinctClause &&
             grouping_is_sortable(parse->distinctClause))//存在distinct语句&按相关字段排序
             root->distinct_pathkeys =
                 make_pathkeys_for_sortclauses(root,
                                               parse->distinctClause,
                                               tlist);//构建pathkeys
         else
             root->distinct_pathkeys = NIL;
     
         root->sort_pathkeys =
             make_pathkeys_for_sortclauses(root,
                                           parse->sortClause,
                                           tlist);//构建常规的排序pathkeys
     
         /*
          * Figure out whether we want a sorted result from query_planner.
          *
          * If we have a sortable GROUP BY clause, then we want a result sorted
          * properly for grouping.  Otherwise, if we have window functions to
          * evaluate, we try to sort for the first window.  Otherwise, if there's a
          * sortable DISTINCT clause that's more rigorous than the ORDER BY clause,
          * we try to produce output that's sufficiently well sorted for the
          * DISTINCT.  Otherwise, if there is an ORDER BY clause, we want to sort
          * by the ORDER BY clause.
          *
          * Note: if we have both ORDER BY and GROUP BY, and ORDER BY is a superset
          * of GROUP BY, it would be tempting to request sort by ORDER BY --- but
          * that might just leave us failing to exploit an available sort order at
          * all.  Needs more thought.  The choice for DISTINCT versus ORDER BY is
          * much easier, since we know that the parser ensured that one is a
          * superset of the other.
          */
         if (root->groupremove_useless_joins_pathkeys)
             root->query_pathkeys = root->group_pathkeys;
         else if (root->window_pathkeys)
             root->query_pathkeys = root->window_pathkeys;
         else if (list_length(root->distinct_pathkeys) >
                  list_length(root->sort_pathkeys))
             root->query_pathkeys = root->distinct_pathkeys;
         else if (root->sort_pathkeys)
             root->query_pathkeys = root->sort_pathkeys;
         else
             root->query_pathkeys = NIL;
     }
    
    /*
      * make_pathkeys_for_sortclauses
      *      Generate a pathkeys list that represents the sort order specified
      *      by a list of SortGroupClauses
      *
      * The resulting PathKeys are always in canonical form.  (Actually, there
      * is no longer any code anywhere that creates non-canonical PathKeys.)
      *
      * We assume that root->nullable_baserels is the set of base relids that could
      * have gone to NULL below the SortGroupClause expressions.  This is okay if
      * the expressions came from the query's top level (ORDER BY, DISTINCT, etc)
      * and if this function is only invoked after deconstruct_jointree.  In the
      * future we might have to make callers pass in the appropriate
      * nullable-relids set, but for now it seems unnecessary.
      *
      * 'sortclauses' is a list of SortGroupClause nodes
      * 'tlist' is the targetlist to find the referenced tlist entries in
      */
     List *
     make_pathkeys_for_sortclauses(PlannerInfo *root,
                                   List *sortclauses,
                                   List *tlist)
     {
         List       *pathkeys = NIL;
         ListCell   *l;
     
         foreach(l, sortclauses)
         {
             SortGroupClause *sortcl = (SortGroupClause *) lfirst(l);
             Expr       *sortkey;
             PathKey    *pathkey;
     
             sortkey = (Expr *) get_sortgroupclause_expr(sortcl, tlist);
             Assert(OidIsValid(sortcl->sortop));
             pathkey = make_pathkey_from_sortop(root,
                                                sortkey,
                                                root->nullable_baserels,
                                                sortcl->sortop,
                                                sortcl->nulls_first,
                                                sortcl->tleSortGroupRef,
                                                true);
     
             /* Canonical form eliminates redundant ordering keys */
             if (!pathkey_is_redundant(pathkey, pathkeys))//不是多余的Key的情况下,才保留
                 pathkeys = lappend(pathkeys, pathkey);
         }
         return pathkeys;
     }
    
    

测试脚本:

    
    
    testdb=# select t1.dwbh,t2.grbh
    from t_dwxx t1 left join t_grxx t2 on t1.dwbh = t2.dwbh
    where t1.dwbh = '1001'
    order by t1.dwbh;
    

跟踪分析,进入make_pathkeys_for_sortclauses函数:

    
    
    ...
    (gdb) step
    make_pathkeys_for_sortclauses (root=0x1702958, sortclauses=0x170d068, tlist=0x1746758) at pathkeys.c:878
    878   List     *pathkeys = NIL;
    (gdb) p *(SortGroupClause *)sortclauses->head->data.ptr_value
    $5 = {type = T_SortGroupClause, tleSortGroupRef = 1, eqop = 98, sortop = 664, nulls_first = false, hashable = true}
    
    ...
    (gdb) n
    889     pathkey = make_pathkey_from_sortop(root,
    (gdb) 
    898     if (!pathkey_is_redundant(pathkey, pathkeys))
    (gdb) p *pathkey
    $11 = {type = T_PathKey, pk_eclass = 0x17486c0, pk_opfamily = 1994, pk_strategy = 1, pk_nulls_first = false}
    
    

函数pathkey_is_redundant通过等价类判断排序是否多余(redundant),在本例中,已存在限制条件dwbh='1001',因此该排序是多余的,返回NULL.

    
    
    901   return pathkeys;
    (gdb) p *pathkeys
    Cannot access memory at address 0x0
    

**fix_placeholder_input_needed_levels**

    
    
     /*
      * fix_placeholder_input_needed_levels
      *      Adjust the "needed at" levels for placeholder inputs
      *
      * This is called after we've finished determining the eval_at levels for
      * all placeholders.  We need to make sure that all vars and placeholders
      * needed to evaluate each placeholder will be available at the scan or join
      * level where the evaluation will be done.  (It might seem that scan-level
      * evaluations aren't interesting, but that's not so: a LATERAL reference
      * within a placeholder's expression needs to cause the referenced var or
      * placeholder to be marked as needed in the scan where it's evaluated.)
      * Note that this loop can have side-effects on the ph_needed sets of other
      * PlaceHolderInfos; that's okay because we don't examine ph_needed here, so
      * there are no ordering issues to worry about.
      */
     void
     fix_placeholder_input_needed_levels(PlannerInfo *root)
     {
         ListCell   *lc;
     
         foreach(lc, root->placeholder_list)//遍历链表
         {
             PlaceHolderInfo *phinfo = (PlaceHolderInfo *) lfirst(lc);
             List       *vars = pull_var_clause((Node *) phinfo->ph_var->phexpr,
                                                PVC_RECURSE_AGGREGATES |
                                                PVC_RECURSE_WINDOWFUNCS |
                                                PVC_INCLUDE_PLACEHOLDERS);//获取Vars
     
             add_vars_to_targetlist(root, vars, phinfo->ph_eval_at, false);//添加到投影列中
             list_free(vars);
         }
     }
    

考察下面的SQL语句:

    
    
    testdb=# explain verbose select t1.dwbh,t2.grbh,t2.constant_field
    from t_dwxx t1 left join (select a.dwbh,a.grbh,'TEST' as constant_field from t_grxx a) t2 on t1.dwbh = t2.dwbh  
    where t1.dwbh = '1001'
    order by t1.dwbh;
                                   QUERY PLAN                               
    ------------------------------------------------------------------------     Nested Loop Left Join  (cost=0.00..16.06 rows=2 width=108)
       Output: t1.dwbh, a.grbh, ('TEST'::text) -- PlaceHolderVar
       Join Filter: ((t1.dwbh)::text = (a.dwbh)::text)
       ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.04 rows=1 width=38)
             Output: t1.dwmc, t1.dwbh, t1.dwdz
             Filter: ((t1.dwbh)::text = '1001'::text)
       ->  Seq Scan on public.t_grxx a  (cost=0.00..15.00 rows=2 width=108)
             Output: a.grbh, a.dwbh, 'TEST'::text
             Filter: ((a.dwbh)::text = '1001'::text)
    (9 rows)
    

子查询上拉与t_dwxx进行连接,上拉过程中,不能简单的把子查询中的"'TEST' as
constant_field"作为上层查询的Var来看待(如果不作特殊处理,跟外连接就不等价了,因为该值有可能是NULL),PG因此引入了PlaceHolderVar这么一个Var来对这种变量进行特殊处理.

跟踪分析:

    
    
    (gdb) b planmain.c:161
    Breakpoint 1 at 0x769602: file planmain.c, line 161.
    (gdb) c
    Continuing.
    
    Breakpoint 1, query_planner (root=0x1702b08, tlist=0x1749c20, qp_callback=0x76e97d <standard_qp_callback>, 
        qp_extra=0x7ffd35e059c0) at planmain.c:163
    163   reconsider_outer_join_clauses(root);
    
    

注意root中的placeholder_list

    
    
    (gdb) p *root
    $1 = {type = T_PlannerInfo, ..., placeholder_list = 0x174bf00, ...}
    

查看其内存结构:

    
    
    #1个PHV
    (gdb) p *root->placeholder_list
    $2 = {type = T_List, length = 1, head = 0x174bee0, tail = 0x174bee0}
    (gdb) p *(Node *)root->placeholder_list->head->data.ptr_value
    $3 = {type = T_PlaceHolderInfo}
    (gdb) p *(PlaceHolderInfo *)root->placeholder_list->head->data.ptr_value
    $4 = {type = T_PlaceHolderInfo, phid = 1, ph_var = 0x174be18, ph_eval_at = 0x174bec8, ph_lateral = 0x0, 
      ph_needed = 0x174bf30, ph_width = 32}
    (gdb) set $phi=(PlaceHolderInfo *)root->placeholder_list->head->data.ptr_value
    (gdb) p *$phi->ph_var
    $5 = {xpr = {type = T_PlaceHolderVar}, phexpr = 0x174be48, phrels = 0x174beb0, phid = 1, phlevelsup = 0}
    #该PHV位于编号为4的RTE中
    (gdb) p *$phi->ph_eval_at
    $6 = {nwords = 1, words = 0x174becc}
    (gdb) p *$phi->ph_eval_at->words
    $7 = 16
    (gdb) p *$phi->ph_needed
    $8 = {nwords = 1, words = 0x174bf34}
    #该PHV在编号为1的RTE中需要用到
    (gdb) p *$phi->ph_needed->words
    $9 = 1
    

### 三、参考资料

[planmain.c](https://doxygen.postgresql.org/planmain_8c.html)  
[what exactly is a PlaceHolderVAr](https://www.postgresql.org/message-id/23076.1277179362%40sss.pgh.pa.us)

