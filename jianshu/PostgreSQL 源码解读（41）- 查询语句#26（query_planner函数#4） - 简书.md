上一小节介绍了函数query_planner中子函数add_base_rels_to_query的实现逻辑,本节继续介绍其中的子函数build_base_rel_tlists/find_placeholders_in_jointree/find_lateral_references,这几个子函数的目的仍然是完善RelOptInfo结构体信息。

query_planner代码片段:

    
    
         //...
         /*
          * Examine the targetlist and join tree, adding entries to baserel
          * targetlists for all referenced Vars, and generating PlaceHolderInfo
          * entries for all referenced PlaceHolderVars.  Restrict and join clauses
          * are added to appropriate lists belonging to the mentioned relations. We
          * also build EquivalenceClasses for provably equivalent expressions. The
          * SpecialJoinInfo list is also built to hold information about join order
          * restrictions.  Finally, we form a target joinlist for make_one_rel() to
          * work from.
          */
         build_base_rel_tlists(root, tlist);//构建"base rels"的投影列
     
         find_placeholders_in_jointree(root);//处理jointree中的PHI
     
         find_lateral_references(root);//处理jointree中Lateral依赖
     
         //...
    

### 一、重要的数据结构

**PlannerInfo**  
PlannerInfo贯穿整个构建查询计划的全过程.  
build_base_rel_tlists、find_placeholders_in_jointree和find_lateral_references函数完善了PlannerInfo->placeholder_list链表.

    
    
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
RelOptInfo结构体贯彻逻辑优化和物理优化过程的始终.  
build_base_rel_tlists完善了结构体中attr_needed和reltarget变量,find_lateral_references函数完善了结构体中lateral_vars变量.

    
    
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
    
    

### 二、源码解读

**基本概念**  
_PlaceHolder_  
PlaceHolder即占位符,常用于减少SQL的parse过程提高性能.  
如JDBC中常用的PreparedStatement:

    
    
    String sql = "select * from t_dwxx where dwbh = ? and dwmc = ?";
    PreparedStatement pstmt = connection.preparestatement(sql);
    pstmt.setstring(1,'1001');
    pstmt.setstring(2,'测试');
    resultset rs = ps.executequery(); 
    

可以认为,其中的?所代表的是占位符.

在psql中,使用set命令定义变量,在SQL语句中使用占位符:

    
    
    testdb=# \set v1 '\'1001\''
    testdb=# select * from t_dwxx where dwbh = :v1;
       dwmc    | dwbh |        dwdz        
    -----------+------+--------------------     X有限公司 | 1001 | 广东省广州市荔湾区
    (1 row)
    
    

**build_base_rel_tlists**

    
    
    /*
    ******************************** build_base_rel_tlists *****************
    */
    
    /*
      * build_base_rel_tlists
      *    Add targetlist entries for each var needed in the query's final tlist
      *    (and HAVING clause, if any) to the appropriate base relations.
      * 
      * 把最终的投影列信息添加到合适的"base rels"中.
      *
      * We mark such vars as needed by "relation 0" to ensure that they will
      * propagate up through all join plan steps.
      */
     void
     build_base_rel_tlists(PlannerInfo *root, List *final_tlist)
     {
         List       *tlist_vars = pull_var_clause((Node *) final_tlist,
                                                  PVC_RECURSE_AGGREGATES |
                                                  PVC_RECURSE_WINDOWFUNCS |
                                                  PVC_INCLUDE_PLACEHOLDERS);//获取投影列
     
         if (tlist_vars != NIL)
         {
             //添加到相应的Relation's targetlist(如不存在)
             //标记其为连接或者最终输出所需要
             add_vars_to_targetlist(root, tlist_vars, bms_make_singleton(0), true);
             list_free(tlist_vars);
         }
     
         /*
          * If there's a HAVING clause, we'll need the Vars it uses, too.  Note
          * that HAVING can contain Aggrefs but not WindowFuncs.
          */
         if (root->parse->havingQual)//如存在Having子句,顶层的Having是在查询语句的后期才执行,需保留需要的Vars
         {
             List       *having_vars = pull_var_clause(root->parse->havingQual,
                                                       PVC_RECURSE_AGGREGATES |
                                                       PVC_INCLUDE_PLACEHOLDERS);
     
             if (having_vars != NIL)
             {
                 add_vars_to_targetlist(root, having_vars,
                                        bms_make_singleton(0), true);
                 list_free(having_vars);
             }
         }
     }
    
     /*
      * add_vars_to_targetlist
      *    For each variable appearing in the list, add it to the owning
      *    relation's targetlist if not already present, and mark the variable
      *    as being needed for the indicated join (or for final output if
      *    where_needed includes "relation 0").
      *
      *    The list may also contain PlaceHolderVars.  These don't necessarily
      *    have a single owning relation; we keep their attr_needed info in
      *    root->placeholder_list instead.  If create_new_ph is true, it's OK
      *    to create new PlaceHolderInfos; otherwise, the PlaceHolderInfos must
      *    already exist, and we should only update their ph_needed.  (This should
      *    be true before deconstruct_jointree begins, and false after that.)
      */
     void
     add_vars_to_targetlist(PlannerInfo *root, List *vars,
                            Relids where_needed, bool create_new_ph)
     {
         ListCell   *temp;
     
         Assert(!bms_is_empty(where_needed));
     
         foreach(temp, vars)
         {
             Node       *node = (Node *) lfirst(temp);
     
             if (IsA(node, Var))
             {
                 Var        *var = (Var *) node;//属性Var
                 RelOptInfo *rel = find_base_rel(root, var->varno);//找到相应的RelOptInfo
                 int         attno = var->varattno;//属性编号
     
                 if (bms_is_subset(where_needed, rel->relids))//where_needed是否rel的子集?
                     continue;//是,继续循环
                 Assert(attno >= rel->min_attr && attno <= rel->max_attr);
                 attno -= rel->min_attr;
                 if (rel->attr_needed[attno] == NULL)
                 {
                     /* Variable not yet requested, so add to rel's targetlist */
                     /* XXX is copyObject necessary here? */
                     rel->reltarget->exprs = lappend(rel->reltarget->exprs,
                                                     copyObject(var));//TODO,添加到rel->reltarget->exprs 中
                     /* reltarget cost and width will be computed later */
                 }
                 rel->attr_needed[attno] = bms_add_members(rel->attr_needed[attno],
                                                           where_needed);//where_needed添加到bitmapset中
             }
             else if (IsA(node, PlaceHolderVar))
             {
                 PlaceHolderVar *phv = (PlaceHolderVar *) node;
                 PlaceHolderInfo *phinfo = find_placeholder_info(root, phv,
                                                                 create_new_ph);
     
                 phinfo->ph_needed = bms_add_members(phinfo->ph_needed,
                                                     where_needed);//添加PHV
             }
             else
                 elog(ERROR, "unrecognized node type: %d", (int) nodeTag(node));
         }
     }
    

**find_placeholders_in_jointree**

    
    
    /*
    ******************************** find_placeholders_in_jointree *****************
    */
     /*
      * find_placeholders_in_jointree
      *      Search the jointree for PlaceHolderVars, and build PlaceHolderInfos
      *
      * 搜索jointree中的PHV,并且构建PHInfos
      *
      * We don't need to look at the targetlist because build_base_rel_tlists()
      * will already have made entries for any PHVs in the tlist.
      *
      * This is called before we begin deconstruct_jointree.  Once we begin
      * deconstruct_jointree, all active placeholders must be present in
      * root->placeholder_list, because make_outerjoininfo and
      * update_placeholder_eval_levels require this info to be available
      * while we crawl up the join tree.
      */
     void
     find_placeholders_in_jointree(PlannerInfo *root)
     {
         /* We need do nothing if the query contains no PlaceHolderVars */
         if (root->glob->lastPHId != 0)
         {
             /* Start recursion at top of jointree */
             Assert(root->parse->jointree != NULL &&
                    IsA(root->parse->jointree, FromExpr));
             find_placeholders_recurse(root, (Node *) root->parse->jointree);//递归处理
         }
     }
     
     /*
      * find_placeholders_recurse
      *    One recursion level of find_placeholders_in_jointree.
      *
      * jtnode is the current jointree node to examine.
      */
     static void
     find_placeholders_recurse(PlannerInfo *root, Node *jtnode)
     {
         if (jtnode == NULL)
             return;
         if (IsA(jtnode, RangeTblRef))//RTR
         {
             /* No quals to deal with here */
         }
         else if (IsA(jtnode, FromExpr))//FromExpr
         {
             FromExpr   *f = (FromExpr *) jtnode;
             ListCell   *l;
     
             /*
              * First, recurse to handle child joins.
              */
             foreach(l, f->fromlist)
             {
                 find_placeholders_recurse(root, lfirst(l));
             }
     
             /*
              * Now process the top-level quals.
              */
             find_placeholders_in_expr(root, f->quals);//在表达式中搜索
         }
         else if (IsA(jtnode, JoinExpr))//JoinExpr
         {
             JoinExpr   *j = (JoinExpr *) jtnode;
     
             /*
              * First, recurse to handle child joins.
              */
             find_placeholders_recurse(root, j->larg);
             find_placeholders_recurse(root, j->rarg);
     
             /* Process the qual clauses */
             find_placeholders_in_expr(root, j->quals);//在表达式中搜索
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
     }
     
     /*
      * find_placeholders_in_expr
      *      Find all PlaceHolderVars in the given expression, and create
      *      PlaceHolderInfo entries for them.
      */
     static void
     find_placeholders_in_expr(PlannerInfo *root, Node *expr)
     {
         List       *vars;
         ListCell   *vl;
     
         /*
          * pull_var_clause does more than we need here, but it'll do and it's
          * convenient to use.
          */
         vars = pull_var_clause(expr,
                                PVC_RECURSE_AGGREGATES |
                                PVC_RECURSE_WINDOWFUNCS |
                                PVC_INCLUDE_PLACEHOLDERS);//遍历Vars,得到PH链表
         foreach(vl, vars)
         {
             PlaceHolderVar *phv = (PlaceHolderVar *) lfirst(vl);
     
             /* Ignore any plain Vars */
             if (!IsA(phv, PlaceHolderVar))
                 continue;
     
             /* Create a PlaceHolderInfo entry if there's not one already */
             (void) find_placeholder_info(root, phv, true);//创建PHInfo
         }
         list_free(vars);
     }
    
     /*
      * find_placeholder_info
      *      Fetch the PlaceHolderInfo for the given PHV
      *
      * If the PlaceHolderInfo doesn't exist yet, create it if create_new_ph is
      * true, else throw an error.
      *
      * This is separate from make_placeholder_expr because subquery pullup has
      * to make PlaceHolderVars for expressions that might not be used at all in
      * the upper query, or might not remain after const-expression simplification.
      * We build PlaceHolderInfos only for PHVs that are still present in the
      * simplified query passed to query_planner().
      *
      * Note: this should only be called after query_planner() has started.  Also,
      * create_new_ph must not be true after deconstruct_jointree begins, because
      * make_outerjoininfo assumes that we already know about all placeholders.
      */
     PlaceHolderInfo *
     find_placeholder_info(PlannerInfo *root, PlaceHolderVar *phv,
                           bool create_new_ph)
     {
         PlaceHolderInfo *phinfo;//结果
         Relids      rels_used;
         ListCell   *lc;
     
         /* if this ever isn't true, we'd need to be able to look in parent lists */
         Assert(phv->phlevelsup == 0);
     
         foreach(lc, root->placeholder_list)//已存在,返回
         {
             phinfo = (PlaceHolderInfo *) lfirst(lc);
             if (phinfo->phid == phv->phid)
                 return phinfo;
         }
     
         /* Not found, so create it */
         if (!create_new_ph)
             elog(ERROR, "too late to create a new PlaceHolderInfo");
     
         phinfo = makeNode(PlaceHolderInfo);//构建PHInfo
     
         phinfo->phid = phv->phid;
         phinfo->ph_var = copyObject(phv);
     
         /*
          * Any referenced rels that are outside the PHV's syntactic scope are
          * LATERAL references, which should be included in ph_lateral but not in
          * ph_eval_at.  If no referenced rels are within the syntactic scope,
          * force evaluation at the syntactic location.
          */
         rels_used = pull_varnos((Node *) phv->phexpr);
         phinfo->ph_lateral = bms_difference(rels_used, phv->phrels);
         if (bms_is_empty(phinfo->ph_lateral))
             phinfo->ph_lateral = NULL;  /* make it exactly NULL if empty */
         phinfo->ph_eval_at = bms_int_members(rels_used, phv->phrels);
         /* If no contained vars, force evaluation at syntactic location */
         if (bms_is_empty(phinfo->ph_eval_at))
         {
             phinfo->ph_eval_at = bms_copy(phv->phrels);
             Assert(!bms_is_empty(phinfo->ph_eval_at));
         }
         /* ph_eval_at may change later, see update_placeholder_eval_levels */
         phinfo->ph_needed = NULL;   /* initially it's unused */
         /* for the moment, estimate width using just the datatype info */
         phinfo->ph_width = get_typavgwidth(exprType((Node *) phv->phexpr),
                                            exprTypmod((Node *) phv->phexpr));
     
         root->placeholder_list = lappend(root->placeholder_list, phinfo);//添加到优化器信息中
     
         /*
          * The PHV's contained expression may contain other, lower-level PHVs.  We
          * now know we need to get those into the PlaceHolderInfo list, too, so we
          * may as well do that immediately.
          */
         find_placeholders_in_expr(root, (Node *) phinfo->ph_var->phexpr);//如存在子表达式,递归进去
     
         return phinfo;
     }
    

**find_lateral_references**

    
    
    /*
    ************************ find_lateral_references *****************************
    */
    
     /*
      * find_lateral_references
      *    For each LATERAL subquery, extract all its references to Vars and
      *    PlaceHolderVars of the current query level, and make sure those values
      *    will be available for evaluation of the subquery.
      *
      * 对于LATERAL子查询,获取当前查询层次的Vars&PHVars,并且确保这些值在解析子查询时是可用的
      *
      * While later planning steps ensure that the Var/PHV source rels are on the
      * outside of nestloops relative to the LATERAL subquery, we also need to
      * ensure that the Vars/PHVs propagate up to the nestloop join level; this
      * means setting suitable where_needed values for them.
      *
      * Note that this only deals with lateral references in unflattened LATERAL
      * subqueries.  When we flatten a LATERAL subquery, its lateral references
      * become plain Vars in the parent query, but they may have to be wrapped in
      * PlaceHolderVars if they need to be forced NULL by outer joins that don't
      * also null the LATERAL subquery.  That's all handled elsewhere.
      *
      * This has to run before deconstruct_jointree, since it might result in
      * creation of PlaceHolderInfos.
      */
     void
     find_lateral_references(PlannerInfo *root)
     {
         Index       rti;
     
         /* We need do nothing if the query contains no LATERAL RTEs */
         if (!root->hasLateralRTEs)
             return;
     
         /*
          * Examine all baserels (the rel array has been set up by now).
          */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//遍历RelOptInfo
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (brel == NULL)
                 continue;
     
             Assert(brel->relid == rti); /* sanity check on array */
     
             /*
              * This bit is less obvious than it might look.  We ignore appendrel
              * otherrels and consider only their parent baserels.  In a case where
              * a LATERAL-containing UNION ALL subquery was pulled up, it is the
              * otherrel that is actually going to be in the plan.  However, we
              * want to mark all its lateral references as needed by the parent,
              * because it is the parent's relid that will be used for join
              * planning purposes.  And the parent's RTE will contain all the
              * lateral references we need to know, since the pulled-up member is
              * nothing but a copy of parts of the original RTE's subquery.  We
              * could visit the parent's children instead and transform their
              * references back to the parent's relid, but it would be much more
              * complicated for no real gain.  (Important here is that the child
              * members have not yet received any processing beyond being pulled
              * up.)  Similarly, in appendrels created by inheritance expansion,
              * it's sufficient to look at the parent relation.
              */
     
             /* ignore RTEs that are "other rels" */
             if (brel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             extract_lateral_references(root, brel, rti);//获取LATERAL依赖
         }
     }
     
     static void
     extract_lateral_references(PlannerInfo *root, RelOptInfo *brel, Index rtindex)
     {
         RangeTblEntry *rte = root->simple_rte_array[rtindex];//相应的RTE
         List       *vars;
         List       *newvars;
         Relids      where_needed;
         ListCell   *lc;
     
         /* No cross-references are possible if it's not LATERAL */
         if (!rte->lateral)//非LATERAL,退出
             return;
     
         /* Fetch the appropriate variables */
         //获取相应的Vars
         if (rte->rtekind == RTE_RELATION)//基表
             vars = pull_vars_of_level((Node *) rte->tablesample, 0);
         else if (rte->rtekind == RTE_SUBQUERY)//子查询
             vars = pull_vars_of_level((Node *) rte->subquery, 1);
         else if (rte->rtekind == RTE_FUNCTION)//函数
             vars = pull_vars_of_level((Node *) rte->functions, 0);
         else if (rte->rtekind == RTE_TABLEFUNC)//TABLEFUNC
             vars = pull_vars_of_level((Node *) rte->tablefunc, 0);
         else if (rte->rtekind == RTE_VALUES)//VALUES
             vars = pull_vars_of_level((Node *) rte->values_lists, 0);
         else
         {
             Assert(false);
             return;                 /* keep compiler quiet */
         }
     
         if (vars == NIL)
             return;                 /* nothing to do */
     
         /* Copy each Var (or PlaceHolderVar) and adjust it to match our level */
         newvars = NIL;
         foreach(lc, vars)//遍历Vars
         {
             Node       *node = (Node *) lfirst(lc);
     
             node = copyObject(node);
             if (IsA(node, Var))//Var
             {
                 Var        *var = (Var *) node;
     
                 /* Adjustment is easy since it's just one node */
                 var->varlevelsup = 0;
             }
             else if (IsA(node, PlaceHolderVar))//PHVar
             {
                 PlaceHolderVar *phv = (PlaceHolderVar *) node;
                 int         levelsup = phv->phlevelsup;
     
                 /* Have to work harder to adjust the contained expression too */
                 if (levelsup != 0)
                     IncrementVarSublevelsUp(node, -levelsup, 0);//调整其中的表达式
     
                 /*
                  * If we pulled the PHV out of a subquery RTE, its expression
                  * needs to be preprocessed.  subquery_planner() already did this
                  * for level-zero PHVs in function and values RTEs, though.
                  */
                 if (levelsup > 0)
                     phv->phexpr = preprocess_phv_expression(root, phv->phexpr);//预处理PHVar表达式
             }
             else
                 Assert(false);
             newvars = lappend(newvars, node);//添加到新的结果Vars中
         }
     
         list_free(vars);
     
         /*
          * We mark the Vars as being "needed" at the LATERAL RTE.  This is a bit
          * of a cheat: a more formal approach would be to mark each one as needed
          * at the join of the LATERAL RTE with its source RTE.  But it will work,
          * and it's much less tedious than computing a separate where_needed for
          * each Var.
          */
         where_needed = bms_make_singleton(rtindex);//获取Rel编号
     
         /*
          * Push Vars into their source relations' targetlists, and PHVs into
          * root->placeholder_list.
          */
         add_vars_to_targetlist(root, newvars, where_needed, true);//添加到相应的Rel中
     
         /* Remember the lateral references for create_lateral_join_info */
         brel->lateral_vars = newvars;//RelOptInfo赋值
     }
    
     /*
      * pull_vars_of_level
      *      Create a list of all Vars (and PlaceHolderVars) referencing the
      *      specified query level in the given parsetree.
      *
      * Caution: the Vars are not copied, only linked into the list.
      */
     List *
     pull_vars_of_level(Node *node, int levelsup)
     {
         pull_vars_context context;
     
         context.vars = NIL;
         context.sublevels_up = levelsup;
     
         /*
          * Must be prepared to start with a Query or a bare expression tree; if
          * it's a Query, we don't want to increment sublevels_up.
          */
         query_or_expression_tree_walker(node,
                                         pull_vars_walker,
                                         (void *) &context,
                                         0);//调用XX_walker函数遍历
     
         return context.vars;
     }
     
     static bool
     pull_vars_walker(Node *node, pull_vars_context *context)//遍历函数
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))
         {
             Var        *var = (Var *) node;
     
             if (var->varlevelsup == context->sublevels_up)
                 context->vars = lappend(context->vars, var);
             return false;
         }
         if (IsA(node, PlaceHolderVar))
         {
             PlaceHolderVar *phv = (PlaceHolderVar *) node;
     
             if (phv->phlevelsup == context->sublevels_up)
                 context->vars = lappend(context->vars, phv);
             /* we don't want to look into the contained expression */
             return false;
         }
         if (IsA(node, Query))
         {
             /* Recurse into RTE subquery or not-yet-planned sublink subquery */
             bool        result;
     
             context->sublevels_up++;
             result = query_tree_walker((Query *) node, pull_vars_walker,
                                        (void *) context, 0);
             context->sublevels_up--;
             return result;
         }
         return expression_tree_walker(node, pull_vars_walker,
                                       (void *) context);
     }
    

**公共部分**

    
    
    /*
    ************************ 公共 *****************************
    */
     /*
      * pull_var_clause
      *    Recursively pulls all Var nodes from an expression clause.
      *
      * 递归的方式推送(pull)所有的Var节点
      *
      *    Aggrefs are handled according to these bits in 'flags':
      *      PVC_INCLUDE_AGGREGATES      include Aggrefs in output list
      *      PVC_RECURSE_AGGREGATES      recurse into Aggref arguments
      *      neither flag                throw error if Aggref found
      *    Vars within an Aggref's expression are included in the result only
      *    when PVC_RECURSE_AGGREGATES is specified.
      *
      *    WindowFuncs are handled according to these bits in 'flags':
      *      PVC_INCLUDE_WINDOWFUNCS     include WindowFuncs in output list
      *      PVC_RECURSE_WINDOWFUNCS     recurse into WindowFunc arguments
      *      neither flag                throw error if WindowFunc found
      *    Vars within a WindowFunc's expression are included in the result only
      *    when PVC_RECURSE_WINDOWFUNCS is specified.
      *
      *    PlaceHolderVars are handled according to these bits in 'flags':
      *      PVC_INCLUDE_PLACEHOLDERS    include PlaceHolderVars in output list
      *      PVC_RECURSE_PLACEHOLDERS    recurse into PlaceHolderVar arguments
      *      neither flag                throw error if PlaceHolderVar found
      *    Vars within a PHV's expression are included in the result only
      *    when PVC_RECURSE_PLACEHOLDERS is specified.
      *
      *    GroupingFuncs are treated mostly like Aggrefs, and so do not need
      *    their own flag bits.
      *
      *    CurrentOfExpr nodes are ignored in all cases.
      *
      *    Upper-level vars (with varlevelsup > 0) should not be seen here,
      *    likewise for upper-level Aggrefs and PlaceHolderVars.
      *
      *    Returns list of nodes found.  Note the nodes themselves are not
      *    copied, only referenced.
      *
      * Does not examine subqueries, therefore must only be used after reduction
      * of sublinks to subplans!
      */
     List *
     pull_var_clause(Node *node, int flags)
     {
         pull_var_clause_context context;//上下文
     
         //互斥选项检测
         /* Assert that caller has not specified inconsistent flags */
         Assert((flags & (PVC_INCLUDE_AGGREGATES | PVC_RECURSE_AGGREGATES))
                != (PVC_INCLUDE_AGGREGATES | PVC_RECURSE_AGGREGATES));
         Assert((flags & (PVC_INCLUDE_WINDOWFUNCS | PVC_RECURSE_WINDOWFUNCS))
                != (PVC_INCLUDE_WINDOWFUNCS | PVC_RECURSE_WINDOWFUNCS));
         Assert((flags & (PVC_INCLUDE_PLACEHOLDERS | PVC_RECURSE_PLACEHOLDERS))
                != (PVC_INCLUDE_PLACEHOLDERS | PVC_RECURSE_PLACEHOLDERS));
     
         context.varlist = NIL;
         context.flags = flags;
     
         pull_var_clause_walker(node, &context);//调用XX_walker函数遍历,结果保存在context.varlist中
         return context.varlist;
     }
     
     static bool
     pull_var_clause_walker(Node *node, pull_var_clause_context *context)
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))//Var类型
         {
             if (((Var *) node)->varlevelsup != 0)//非本级Var
                 elog(ERROR, "Upper-level Var found where not expected");
             context->varlist = lappend(context->varlist, node);//添加到结果链表中
             return false;
         }
         else if (IsA(node, Aggref))//聚合
         {
             if (((Aggref *) node)->agglevelsup != 0)
                 elog(ERROR, "Upper-level Aggref found where not expected");
             if (context->flags & PVC_INCLUDE_AGGREGATES)//包含聚合
             {
                 context->varlist = lappend(context->varlist, node);//添加到结果
                 /* we do NOT descend into the contained expression */
                 return false;
             }
             else if (context->flags & PVC_RECURSE_AGGREGATES)//递归搜索
             {
                 /* fall through to recurse into the aggregate's arguments */
             }
             else
                 elog(ERROR, "Aggref found where not expected");
         }
         else if (IsA(node, GroupingFunc))//分组
         {
             if (((GroupingFunc *) node)->agglevelsup != 0)
                 elog(ERROR, "Upper-level GROUPING found where not expected");
             if (context->flags & PVC_INCLUDE_AGGREGATES)//包含标记
             {
                 context->varlist = lappend(context->varlist, node);
                 /* we do NOT descend into the contained expression */
                 return false;
             }
             else if (context->flags & PVC_RECURSE_AGGREGATES)//递归标记
             {
                 /*
                  * We do NOT descend into the contained expression, even if the
                  * caller asked for it, because we never actually evaluate it -                  * the result is driven entirely off the associated GROUP BY
                  * clause, so we never need to extract the actual Vars here.
                  */
                 return false;//直接返回,需与GROUP BY语句一起
             }
             else
                 elog(ERROR, "GROUPING found where not expected");
         }
         else if (IsA(node, WindowFunc))//窗口函数
         {
             /* WindowFuncs have no levelsup field to check ... */
             if (context->flags & PVC_INCLUDE_WINDOWFUNCS)//包含标记
             {
                 context->varlist = lappend(context->varlist, node);
                 /* we do NOT descend into the contained expressions */
                 return false;
             }
             else if (context->flags & PVC_RECURSE_WINDOWFUNCS)//递归标记
             {
                 /* fall through to recurse into the windowfunc's arguments */
             }
             else
                 elog(ERROR, "WindowFunc found where not expected");
         }
         else if (IsA(node, PlaceHolderVar))//PH
         {
             if (((PlaceHolderVar *) node)->phlevelsup != 0)
                 elog(ERROR, "Upper-level PlaceHolderVar found where not expected");
             if (context->flags & PVC_INCLUDE_PLACEHOLDERS)
             {
                 context->varlist = lappend(context->varlist, node);
                 /* we do NOT descend into the contained expression */
                 return false;
             }
             else if (context->flags & PVC_RECURSE_PLACEHOLDERS)
             {
                 /* fall through to recurse into the placeholder's expression */
             }
             else
                 elog(ERROR, "PlaceHolderVar found where not expected");
         }
         return expression_tree_walker(node, pull_var_clause_walker,
                                       (void *) context);//表达式解析
     }
     
    
    

### 三、跟踪分析

重点考察root->simple_rel_array[n]->attr_needed、root->simple_rel_array[n]->reltarget、root->placeholder_list、root->simple_rel_array[n]->lateral_vars.  
启动gdb跟踪

    
    
    (gdb) b build_base_rel_tlists
    Breakpoint 1 at 0x76551b: file initsplan.c, line 153.
    (gdb) c
    Continuing.
    
    Breakpoint 1, build_base_rel_tlists (root=0x171ae40, final_tlist=0x1734750) at initsplan.c:153
    153     List       *tlist_vars = pull_var_clause((Node *) final_tlist,
    (gdb) 
    

final_tlist是最终的输出列(投影列),一共有5个,分别是t_dwxx.dwmc/dwbh/dwdz,t_grxx.grbh,t_jfxx.je

    
    
    (gdb) p *final_tlist
    $1 = {type = T_List, length = 5, head = 0x1734730, tail = 0x1734a60}
    (gdb) p *(Node *)final_tlist->head->data.ptr_value
    $2 = {type = T_TargetEntry}
    (gdb) p *(TargetEntry *)final_tlist->head->data.ptr_value
    $3 = {xpr = {type = T_TargetEntry}, expr = 0x17346e0, resno = 1, resname = 0x171a5a0 "dwmc", ressortgroupref = 0, 
      resorigtbl = 16394, resorigcol = 1, resjunk = false}
    
    

跟踪函数build_base_rel_tlists

    
    
    (gdb) n
    158     if (tlist_vars != NIL)
    (gdb) 
    160         add_vars_to_targetlist(root, tlist_vars, bms_make_singleton(0), true);
    #5个Vars
    (gdb) p *tlist_vars
    $4 = {type = T_List, length = 5, head = 0x17369e0, tail = 0x1737cb8}
    (gdb) p *(Var *)tlist_vars->head->data.ptr_value
    $5 = {xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 1043, vartypmod = 104, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 1, location = 7}
    

执行函数build_base_rel_tlists,检查final_rel->attr_needed和final_rel->reltarget

    
    
    (gdb) finish
    Run till exit from #0  build_base_rel_tlists (root=0x171ae40, final_tlist=0x1734750) at initsplan.c:160
    query_planner (root=0x171ae40, tlist=0x1734750, qp_callback=0x76e97d <standard_qp_callback>, qp_extra=0x7ffe20fc33d0)
        at planmain.c:152
    152     find_placeholders_in_jointree(root);
    
    

检查root内存结构

    
    
    (gdb) p *root
    $15 = {type = T_PlannerInfo, parse = 0x1711680, glob = 0x1732118, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x1736578, simple_rel_array_size = 6, simple_rte_array = 0x17365c8, 
      all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x0, join_rel_hash = 0x0, join_rel_level = 0x0, 
      join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, 
      left_join_clauses = 0x0, right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, 
      rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, 
      window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, initial_rels = 0x0, 
      upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
      processed_tlist = 0x1734750, grouping_map = 0x0, minmax_aggs = 0x0, planner_cxt = 0x165a040, total_table_pages = 0, 
      tuple_fraction = 0, limit_tuples = -1, qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = true, 
      hasLateralRTEs = true, hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, 
      hasRecursion = false, wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    

RelOptInfo数组,注意数组的第0(下标)个元素为NULL(无用),有用的元素下标从1开始:  
第1个元素是基础关系(相应的RTE=t_dwxx),第2个元素为NULL(相应的RTE=子查询),第3个元素为基础关系(相应的RTE=t_grxx),第4个元素为基础关系(相应的RTE=t_jfxx),第5个元素为NULL(相应的RTE=连接)

    
    
    (gdb) p *root->simple_rel_array[0]
    Cannot access memory at address 0x0
    (gdb) p *root->simple_rel_array[1]
    $31 = {type = T_RelOptInfo, reloptkind = RELOPT_BASEREL, relids = 0x1736828, rows = 0, consider_startup = false, 
      consider_param_startup = false, consider_parallel = false, reltarget = 0x1736840, pathlist = 0x0, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 1, reltablespace = 0, 
      rtekind = RTE_RELATION, min_attr = -7, max_attr = 3, attr_needed = 0x1736890, attr_widths = 0x1736920, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x1736cc8, statlist = 0x0, pages = 1, tuples = 3, 
      allvisfrac = 0, subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, 
      useridiscurrent = false, fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, 
      baserestrictinfo = 0x0, baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, 
      joininfo = 0x0, has_eclass_joins = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, 
      partition_qual = 0x0, part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) p *root->simple_rel_array[2]
    Cannot access memory at address 0x0
    (gdb) p *root->simple_rel_array[3]
    $32 = {type = T_RelOptInfo, reloptkind = RELOPT_BASEREL, relids = 0x17377f8, rows = 0, consider_startup = false, 
      consider_param_startup = false, consider_parallel = false, reltarget = 0x1737810, pathlist = 0x0, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 3, reltablespace = 0, 
      rtekind = RTE_RELATION, min_attr = -7, max_attr = 5, attr_needed = 0x1737860, attr_widths = 0x17378f0, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 10, tuples = 400, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, 
      has_eclass_joins = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, 
      part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) p *root->simple_rel_array[4]
    $33 = {type = T_RelOptInfo, reloptkind = RELOPT_BASEREL, relids = 0x1737b50, rows = 0, consider_startup = false, 
      consider_param_startup = false, consider_parallel = false, reltarget = 0x1737b68, pathlist = 0x0, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 4, reltablespace = 0, 
      rtekind = RTE_RELATION, min_attr = -7, max_attr = 3, attr_needed = 0x1737bb8, attr_widths = 0x1737c48, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 10, tuples = 720, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, 
      has_eclass_joins = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, 
      part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) p *root->simple_rel_array[5]
    Cannot access memory at address 0x0
    

查看root->simple_rel_array[n]->attr_needed、root->simple_rel_array[n]->reltarget的内存结构,以第1个元素为例:

    
    
    #attr_needed(类型为Relids)为NULL
    (gdb) p *root->simple_rel_array[1]->attr_needed
    $47 = (Relids) 0x0
    #reltarget->exprs为3个Var的链表
    (gdb) p *root->simple_rel_array[1]->reltarget->exprs
    $38 = {type = T_List, length = 3, head = 0x1737d40, tail = 0x1737e80}
    (gdb) p *(Var *)root->simple_rel_array[1]->reltarget->exprs->head->data.ptr_value
    $40 = {xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 1043, vartypmod = 104, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 1, location = 7}
    

继续执行,调用函数find_placeholders_in_jointree

    
    
    152     find_placeholders_in_jointree(root);
    (gdb) step
    find_placeholders_in_jointree (root=0x171ae40) at placeholder.c:148
    148     if (root->glob->lastPHId != 0)
    (gdb) n
    155 }
    (gdb) n
    154     find_lateral_references(root);
    #链表为NULL
    #psql中的:v1似乎不是占位符,理解有偏差//TODO...
    (gdb) p root->placeholder_list
    $48 = (List *) 0x0
    

下面调用子函数find_lateral_references,由于RelOptInfo中的3个不为NULL的元素对应的lateral均为FALSE(只有子查询的lateral为true,但子查询对应的RelOptInfo为NULL),因此root->simple_rel_array[1/3/4/5]->lateral_vars均为NULL

### 四、参考资料

[initsplan.c](https://doxygen.postgresql.org/initsplan_8c_source.html)

