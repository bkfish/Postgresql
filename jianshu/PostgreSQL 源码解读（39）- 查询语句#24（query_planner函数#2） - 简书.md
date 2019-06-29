上一小节介绍了函数query_planner对简单语句(SELECT
2+2;)的处理逻辑,本节介绍了函数query_planner函数除此之外的其他主处理逻辑。

### 一、重要的数据结构

在query_planner中,对root(PlannerInfo)结构进行初始化和处理,为后续的计划作准备.  
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
     
     
    

### 二、源码解读

本节介绍query_planner的主流程以及setup_simple_rel_arrays和setup_append_rel_array两个子函数的实现逻辑.  
**query_planner**

    
    
     /*
      * query_planner
      *    Generate a path (that is, a simplified plan) for a basic query,
      *    which may involve joins but not any fancier features.
      *
      * 为一个基本的查询(可能涉及连接)生成访问路径(也可以视为一个简化的计划).
      *
      * Since query_planner does not handle the toplevel processing (grouping,
      * sorting, etc) it cannot select the best path by itself.  Instead, it
      * returns the RelOptInfo for the top level of joining, and the caller
      * (grouping_planner) can choose among the surviving paths for the rel.
      *
      * query_planner不会处理顶层的处理过程(如最后的分组/排序等操作),因此,不能选择最优的访问路径
      * 该函数会返回RelOptInfo给最高层的连接,grouping_planner可以在剩下的路径中进行选择
      *
      * root describes the query to plan
      * tlist is the target list the query should produce
      *      (this is NOT necessarily root->parse->targetList!)
      * qp_callback is a function to compute query_pathkeys once it's safe to do so
      * qp_extra is optional extra data to pass to qp_callback
      *
      * root是计划信息/tlist是投影列
      * qp_callback是计算query_pathkeys的函数/qp_extra是传递给qp_callback的函数
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
         Query      *parse = root->parse;//查询树
         List       *joinlist;
         RelOptInfo *final_rel;//结果
         Index       rti;//RTE的index
         double      total_pages;//总pages数
     
         /*
          * If the query has an empty join tree, then it's something easy like
          * "SELECT 2+2;" or "INSERT ... VALUES()".  Fall through quickly.
          */
         if (parse->jointree->fromlist == NIL)//简单SQL,无FROM/WHERE语句
         {
             /* We need a dummy joinrel to describe the empty set of baserels */
             final_rel = build_empty_join_rel(root);//创建返回结果
     
             /*
              * If query allows parallelism in general, check whether the quals are
              * parallel-restricted.  (We need not check final_rel->reltarget
              * because it's empty at this point.  Anything parallel-restricted in
              * the query tlist will be dealt with later.)
              */
             if (root->glob->parallelModeOK)//并行模式?
                 final_rel->consider_parallel =
                     is_parallel_safe(root, parse->jointree->quals);
     
             /* The only path for it is a trivial Result path */
             add_path(final_rel, (Path *)
                      create_result_path(root, final_rel,
                                         final_rel->reltarget,
                                         (List *) parse->jointree->quals));//添加访问路径
     
             /* Select cheapest path (pretty easy in this case...) */
             set_cheapest(final_rel);//选择最优的访问路径
     
             /*
              * We still are required to call qp_callback, in case it's something
              * like "SELECT 2+2 ORDER BY 1".
              */
             root->canon_pathkeys = NIL;
             (*qp_callback) (root, qp_extra);//回调函数
     
             return final_rel;//返回
         }
     
     
         /*
          * Init planner lists to empty.
          *
          * NOTE: append_rel_list was set up by subquery_planner, so do not touch
          * here.
          */
         root->join_rel_list = NIL;//初始化PlannerInfo
         root->join_rel_hash = NULL;
         root->join_rel_level = NULL;
         root->join_cur_level = 0;
         root->canon_pathkeys = NIL;
         root->left_join_clauses = NIL;
         root->right_join_clauses = NIL;
         root->full_join_clauses = NIL;
         root->join_info_list = NIL;
         root->placeholder_list = NIL;
         root->fkey_list = NIL;
         root->initial_rels = NIL;
     
         /*
          * Make a flattened version of the rangetable for faster access (this is
          * OK because the rangetable won't change any more), and set up an empty
          * array for indexing base relations.
          */
         setup_simple_rel_arrays(root);//初始化PlannerInfo->simple_rel/rte_array&size
     
         /*
          * Populate append_rel_array with each AppendRelInfo to allow direct
          * lookups by child relid.
          */
         setup_append_rel_array(root);//初始化PlannerInfo->append_rel_array(通过append_rel_list)
     
         /*
          * Construct RelOptInfo nodes for all base relations in query, and
          * indirectly for all appendrel member relations ("other rels").  This
          * will give us a RelOptInfo for every "simple" (non-join) rel involved in
          * the query.
          *
          * Note: the reason we find the rels by searching the jointree and
          * appendrel list, rather than just scanning the rangetable, is that the
          * rangetable may contain RTEs for rels not actively part of the query,
          * for example views.  We don't want to make RelOptInfos for them.
          */
         add_base_rels_to_query(root, (Node *) parse->jointree);//构建RelOptInfo节点
     
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
     
         joinlist = deconstruct_jointree(root);//重构jointree
     
         /*
          * Reconsider any postponed outer-join quals now that we have built up
          * equivalence classes.  (This could result in further additions or
          * mergings of classes.)
          */
         reconsider_outer_join_clauses(root);//已创建等价类,那么需要重新考虑被推后处理的外连接表达式
     
         /*
          * If we formed any equivalence classes, generate additional restriction
          * clauses as appropriate.  (Implied join clauses are formed on-the-fly
          * later.)
          */
         generate_base_implied_equalities(root);//等价类构建后,生成因此外加的约束语句
     
         /*
          * We have completed merging equivalence sets, so it's now possible to
          * generate pathkeys in canonical form; so compute query_pathkeys and
          * other pathkeys fields in PlannerInfo.
          */
         (*qp_callback) (root, qp_extra);//调用回调函数
     
         /*
          * Examine any "placeholder" expressions generated during subquery pullup.
          * Make sure that the Vars they need are marked as needed at the relevant
          * join level.  This must be done before join removal because it might
          * cause Vars or placeholders to be needed above a join when they weren't
          * so marked before.
          */
         fix_placeholder_input_needed_levels(root);//检查在子查询上拉时生成的PH表达式,确保Vars是OK的
     
         /*
          * Remove any useless outer joins.  Ideally this would be done during
          * jointree preprocessing, but the necessary information isn't available
          * until we've built baserel data structures and classified qual clauses.
          */
         joinlist = remove_useless_joins(root, joinlist);//清除无用的外连接
     
         /*
          * Also, reduce any semijoins with unique inner rels to plain inner joins.
          * Likewise, this can't be done until now for lack of needed info.
          */
         reduce_unique_semijoins(root);//消除半连接
     
         /*
          * Now distribute "placeholders" to base rels as needed.  This has to be
          * done after join removal because removal could change whether a
          * placeholder is evaluable at a base rel.
          */
         add_placeholders_to_base_rels(root);//在"base rels"中添加PH
     
         /*
          * Construct the lateral reference sets now that we have finalized
          * PlaceHolderVar eval levels.
          */
         create_lateral_join_info(root);//创建Lateral连接信息
     
         /*
          * Match foreign keys to equivalence classes and join quals.  This must be
          * done after finalizing equivalence classes, and it's useful to wait till
          * after join removal so that we can skip processing foreign keys
          * involving removed relations.
          */
         match_foreign_keys_to_quals(root);//匹配外键信息
     
         /*
          * Look for join OR clauses that we can extract single-relation
          * restriction OR clauses from.
          */
         extract_restriction_or_clauses(root);//在OR语句中抽取约束条件
     
         /*
          * We should now have size estimates for every actual table involved in
          * the query, and we also know which if any have been deleted from the
          * query by join removal; so we can compute total_table_pages.
          *
          * Note that appendrels are not double-counted here, even though we don't
          * bother to distinguish RelOptInfos for appendrel parents, because the
          * parents will still have size zero.
          *
          * XXX if a table is self-joined, we will count it once per appearance,
          * which perhaps is the wrong thing ... but that's not completely clear,
          * and detecting self-joins here is difficult, so ignore it for now.
          */
         total_pages = 0;
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//计算总pages
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
     
             if (brel == NULL)
                 continue;
     
             Assert(brel->relid == rti); /* sanity check on array */
     
             if (IS_SIMPLE_REL(brel))
                 total_pages += (double) brel->pages;
         }
         root->total_table_pages = total_pages;//赋值
     
         /*
          * Ready to do the primary planning.
          */
         final_rel = make_one_rel(root, joinlist);//执行主要的计划过程
     
         /* Check that we got at least one usable path */
         if (!final_rel || !final_rel->cheapest_total_path ||
             final_rel->cheapest_total_path->param_info != NULL)
             elog(ERROR, "failed to construct the join relation");//检查
     
         return final_rel;//返回结果
     }
    

**setup_simple_rel_arrays**  
初始化setup_simple_rel_arrays(注意:[0]无用)和setup_simple_rel_arrays

    
    
     /*
      * setup_simple_rel_arrays
      *    Prepare the arrays we use for quickly accessing base relations.
      */
     void
     setup_simple_rel_arrays(PlannerInfo *root)
     {
         Index       rti;
         ListCell   *lc;
     
         /* Arrays are accessed using RT indexes (1..N) */
         root->simple_rel_array_size = list_length(root->parse->rtable) + 1;
     
         /* simple_rel_array is initialized to all NULLs */
         root->simple_rel_array = (RelOptInfo **)
             palloc0(root->simple_rel_array_size * sizeof(RelOptInfo *));
     
         /* simple_rte_array is an array equivalent of the rtable list */
         root->simple_rte_array = (RangeTblEntry **)
             palloc0(root->simple_rel_array_size * sizeof(RangeTblEntry *));
         rti = 1;
         foreach(lc, root->parse->rtable)
         {
             RangeTblEntry *rte = (RangeTblEntry *) lfirst(lc);
     
             root->simple_rte_array[rti++] = rte;
         }
     }
    

**setup_append_rel_array**  
源码比较简单,读取append_rel_list中的信息初始化append_rel_array

    
    
     /*
      * setup_append_rel_array
      *      Populate the append_rel_array to allow direct lookups of
      *      AppendRelInfos by child relid.
      *
      * The array remains unallocated if there are no AppendRelInfos.
      */
     void
     setup_append_rel_array(PlannerInfo *root)
     {
         ListCell   *lc;
         int         size = list_length(root->parse->rtable) + 1;
     
         if (root->append_rel_list == NIL)
         {
             root->append_rel_array = NULL;
             return;
         }
     
         root->append_rel_array = (AppendRelInfo **)
             palloc0(size * sizeof(AppendRelInfo *));
     
         foreach(lc, root->append_rel_list)
         {
             AppendRelInfo *appinfo = lfirst_node(AppendRelInfo, lc);
             int         child_relid = appinfo->child_relid;
     
             /* Sanity check */
             Assert(child_relid < size);
     
             if (root->append_rel_array[child_relid])
                 elog(ERROR, "child relation already exists");
     
             root->append_rel_array[child_relid] = appinfo;
         }
     }
    

### 三、参考资料

[planmain.c](https://doxygen.postgresql.org/planmain_8c_source.html)

