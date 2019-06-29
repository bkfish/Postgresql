本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel函数中的sort_inner_and_outer函数，该函数尝试构造merge
join访问路径。  
merge join的算法实现伪代码如下:  
READ data_set_1 SORT BY JOIN KEY TO temp_ds1  
READ data_set_2 SORT BY JOIN KEY TO temp_ds2  
READ ds1_row FROM temp_ds1  
READ ds2_row FROM temp_ds2  
WHILE NOT eof ON temp_ds1,temp_ds2 LOOP  
IF ( temp_ds1.key = temp_ds2.key ) OUTPUT JOIN ds1_row,ds2_row  
ELSIF ( temp_ds1.key <= temp_ds2.key ) READ ds1_row FROM temp_ds1  
ELSIF ( temp_ds1.key => temp_ds2.key ) READ ds2_row FROM temp_ds2  
END LOOP

### 一、数据结构

**Cost相关**  
注意:实际使用的参数值通过系统配置文件定义,而不是这里的常量定义!

    
    
     typedef double Cost; /* execution cost (in page-access units) */
    
     /* defaults for costsize.c's Cost parameters */
     /* NB: cost-estimation code should use the variables, not these constants! */
     /* 注意:实际值通过系统配置文件定义,而不是这里的常量定义! */
     /* If you change these, update backend/utils/misc/postgresql.sample.conf */
     #define DEFAULT_SEQ_PAGE_COST  1.0       //顺序扫描page的成本
     #define DEFAULT_RANDOM_PAGE_COST  4.0      //随机扫描page的成本
     #define DEFAULT_CPU_TUPLE_COST  0.01     //处理一个元组的CPU成本
     #define DEFAULT_CPU_INDEX_TUPLE_COST 0.005   //处理一个索引元组的CPU成本
     #define DEFAULT_CPU_OPERATOR_COST  0.0025    //执行一次操作或函数的CPU成本
     #define DEFAULT_PARALLEL_TUPLE_COST 0.1    //并行执行,从一个worker传输一个元组到另一个worker的成本
     #define DEFAULT_PARALLEL_SETUP_COST  1000.0  //构建并行执行环境的成本
     
     #define DEFAULT_EFFECTIVE_CACHE_SIZE  524288    /*先前已有介绍, measured in pages */
    
     double      seq_page_cost = DEFAULT_SEQ_PAGE_COST;
     double      random_page_cost = DEFAULT_RANDOM_PAGE_COST;
     double      cpu_tuple_cost = DEFAULT_CPU_TUPLE_COST;
     double      cpu_index_tuple_cost = DEFAULT_CPU_INDEX_TUPLE_COST;
     double      cpu_operator_cost = DEFAULT_CPU_OPERATOR_COST;
     double      parallel_tuple_cost = DEFAULT_PARALLEL_TUPLE_COST;
     double      parallel_setup_cost = DEFAULT_PARALLEL_SETUP_COST;
     
     int         effective_cache_size = DEFAULT_EFFECTIVE_CACHE_SIZE;
     Cost        disable_cost = 1.0e10;//1后面10个0,通过设置一个巨大的成本,让优化器自动放弃此路径
     
     int         max_parallel_workers_per_gather = 2;//每次gather使用的worker数
    

### 二、源码解读

sort_inner_and_outer函数尝试构造merge join访问路径.  
构造过程中的成本估算实现函数initial_cost_mergejoin和final_cost_mergejoin在下一节介绍.

    
    
    //------------------------------------------------ sort_inner_and_outer
    
    /*
     * sort_inner_and_outer
     *    Create mergejoin join paths by explicitly sorting both the outer and
     *    inner join relations on each available merge ordering.
     *    显式排序outer和inner表,创建mergejoin连接路径.
     *
     * 'joinrel' is the join relation
     * joinrel-连接后新生成的relation
     * 'outerrel' is the outer join relation
     * outerrel-参与连接的outer relation(俗称外表,驱动表)
     * 'innerrel' is the inner join relation
     * innerrel-参与连接的inner relation(俗称内表)
     * 'jointype' is the type of join to do
     * jointype-连接类型
     * 'extra' contains additional input values
     * extra-额外的输入参数
     */
    static void
    sort_inner_and_outer(PlannerInfo *root,
                         RelOptInfo *joinrel,
                         RelOptInfo *outerrel,
                         RelOptInfo *innerrel,
                         JoinType jointype,
                         JoinPathExtraData *extra)
    {
        JoinType    save_jointype = jointype;
        Path       *outer_path;
        Path       *inner_path;
        Path       *cheapest_partial_outer = NULL;
        Path       *cheapest_safe_inner = NULL;
        List       *all_pathkeys;
        ListCell   *l;
    
        /*
         * We only consider the cheapest-total-cost input paths, since we are
         * assuming here that a sort is required.  We will consider
         * cheapest-startup-cost input paths later, and only if they don't need a
         * sort.
         * 由于我们假定排序是必须的,只考虑总成本最低的路径。
         * 后续会尝试启动成本最低的路径，并且只在它们不需要排序的情况下。
         *
         * This function intentionally does not consider parameterized input
         * paths, except when the cheapest-total is parameterized.  If we did so,
         * we'd have a combinatorial explosion of mergejoin paths of dubious
         * value.  This interacts with decisions elsewhere that also discriminate
         * against mergejoins with parameterized inputs; see comments in
         * src/backend/optimizer/README.
         * 该函数特意不考虑参数化输入路径，除非成本最低路径是参数化的。
         * 如果考虑了参数化输入路径,mergejoin将有一个数量巨大的组合,成本上划不来。
         * 而且这将与其他地方的决策相互作用，这些决策同样会"歧视"使用参数化输入的mergejoin;
         * 请参阅src/backend/optimizer/README中的注释。
         */
        outer_path = outerrel->cheapest_total_path;
        inner_path = innerrel->cheapest_total_path;
    
        /*
         * If either cheapest-total path is parameterized by the other rel, we
         * can't use a mergejoin.  (There's no use looking for alternative input
         * paths, since these should already be the least-parameterized available
         * paths.)
         * 如果任意一个总成本最低的路径使用了参数化访问路径，那么不能使用合并连接。
         * (没有必要寻找替代的输入路径，因为这些路径已经是参数化最少的可用路径了。)
         */
        if (PATH_PARAM_BY_REL(outer_path, innerrel) ||
            PATH_PARAM_BY_REL(inner_path, outerrel))
            return;
    
        /*
         * If unique-ification is requested, do it and then handle as a plain
         * inner join.
         * 如果需要保证唯一,创建唯一性访问路径并设置为JOIN_INNER连接类型
         */
        if (jointype == JOIN_UNIQUE_OUTER)
        {
            outer_path = (Path *) create_unique_path(root, outerrel,
                                                     outer_path, extra->sjinfo);
            Assert(outer_path);
            jointype = JOIN_INNER;
        }
        else if (jointype == JOIN_UNIQUE_INNER)
        {
            inner_path = (Path *) create_unique_path(root, innerrel,
                                                     inner_path, extra->sjinfo);
            Assert(inner_path);
            jointype = JOIN_INNER;
        }
    
        /*
         * If the joinrel is parallel-safe, we may be able to consider a partial
         * merge join.  However, we can't handle JOIN_UNIQUE_OUTER, because the
         * outer path will be partial, and therefore we won't be able to properly
         * guarantee uniqueness.  Similarly, we can't handle JOIN_FULL and
         * JOIN_RIGHT, because they can produce false null extended rows.  Also,
         * the resulting path must not be parameterized.
         * 如果连接可以并行执行，那么可以考虑并行合并连接。
         * 但是，PG不能处理JOIN_UNIQUE_OUTER，因为外部路径是部分的，因此不能正确地保证惟一性。
         * 类似地，PG不能处理JOIN_FULL和JOIN_RIGHT，因为它们会产生假空扩展行。
         * 此外，生成的路径不能被参数化。
         */
        if (joinrel->consider_parallel &&
            save_jointype != JOIN_UNIQUE_OUTER &&
            save_jointype != JOIN_FULL &&
            save_jointype != JOIN_RIGHT &&
            outerrel->partial_pathlist != NIL &&
            bms_is_empty(joinrel->lateral_relids))
        {
            cheapest_partial_outer = (Path *) linitial(outerrel->partial_pathlist);
    
            if (inner_path->parallel_safe)
                cheapest_safe_inner = inner_path;
            else if (save_jointype != JOIN_UNIQUE_INNER)
                cheapest_safe_inner =
                    get_cheapest_parallel_safe_total_inner(innerrel->pathlist);
        }
    
        /*
         * Each possible ordering of the available mergejoin clauses will generate
         * a differently-sorted result path at essentially the same cost.  We have
         * no basis for choosing one over another at this level of joining, but
         * some sort orders may be more useful than others for higher-level
         * mergejoins, so it's worth considering multiple orderings.
         * 可用的mergejoin子句的每一种可能的排序都将以基本相同的代价生成不同的排序结果路径。
         * 在这种连接级别上，我们没有选择一个排序的基础，但是对于更高层的合并，
         * 某些排序顺序可能比其他更有用，因此值得考虑多种排序的顺序。
         *
         * Actually, it's not quite true that every mergeclause ordering will
         * generate a different path order, because some of the clauses may be
         * partially redundant (refer to the same EquivalenceClasses).  Therefore,
         * what we do is convert the mergeclause list to a list of canonical
         * pathkeys, and then consider different orderings of the pathkeys.
         * 实际上，并不是每个mergeclause都会生成不同的路径顺序，因为有些子句可能是部分冗余的(请参阅等价类)。
         * 因此，要做的是将mergeclause列表转换为一个规范路径键列表，然后考虑路径键的不同顺序。
         *
         * Generating a path for *every* permutation of the pathkeys doesn't seem
         * like a winning strategy; the cost in planning time is too high. For
         * now, we generate one path for each pathkey, listing that pathkey first
         * and the rest in random order.  This should allow at least a one-clause
         * mergejoin without re-sorting against any other possible mergejoin
         * partner path.  But if we've not guessed the right ordering of secondary
         * keys, we may end up evaluating clauses as qpquals when they could have
         * been done as mergeclauses.  (In practice, it's rare that there's more
         * than two or three mergeclauses, so expending a huge amount of thought
         * on that is probably not worth it.)
         * 为每个路径键的排列生成路径似乎不是一个好的策略,因为计划时间成本太高了。
         * 现在，我们为每个pathkey生成一个路径，首先列出这个pathkey，然后按随机顺序列出其余的路径。
         * 这应该至少允许一个单子句的mergejoin，而不需要对任何其他可能的mergejoin路径进行重新排序。
         * 但如果没有正确判断二级键的顺序，那么可能会以qpquals的形式计算条件子句，而这些子句本可以作为mergeclauses完成。
         * (实际上，很少有两个或三个以上的mergeclauses，所以花大量时间在这上面可能不值得)。
         *
         * The pathkey order returned by select_outer_pathkeys_for_merge() has
         * some heuristics behind it (see that function), so be sure to try it
         * exactly as-is as well as making variants.
         * 通过调用select_outer_pathkeys_for_merge函数返回的排序键顺序pathkey
         * 背后有一些启发式函数(请参阅该函数)，所以一定要按原样尝试它，并创建变体。
         */
        all_pathkeys = select_outer_pathkeys_for_merge(root,
                                                       extra->mergeclause_list,
                                                       joinrel);
    
        foreach(l, all_pathkeys)//遍历所有可用的排序键
        {
            List       *front_pathkey = (List *) lfirst(l);
            List       *cur_mergeclauses;
            List       *outerkeys;
            List       *innerkeys;
            List       *merge_pathkeys;
    
            /* Make a pathkey list with this guy first */
            if (l != list_head(all_pathkeys))
                outerkeys = lcons(front_pathkey,
                                  list_delete_ptr(list_copy(all_pathkeys),
                                                  front_pathkey));
            else
                outerkeys = all_pathkeys;   /* no work at first one... */
    
            /* Sort the mergeclauses into the corresponding ordering */
            cur_mergeclauses =
                find_mergeclauses_for_outer_pathkeys(root,
                                                     outerkeys,
                                                     extra->mergeclause_list);
    
            /* Should have used them all... */
            Assert(list_length(cur_mergeclauses) == list_length(extra->mergeclause_list));
    
            /* Build sort pathkeys for the inner side */
            innerkeys = make_inner_pathkeys_for_merge(root,
                                                      cur_mergeclauses,
                                                      outerkeys);
    
            /* Build pathkeys representing output sort order */
            merge_pathkeys = build_join_pathkeys(root, joinrel, jointype,
                                                 outerkeys);
    
            /*
             * And now we can make the path.
             *
             * Note: it's possible that the cheapest paths will already be sorted
             * properly.  try_mergejoin_path will detect that case and suppress an
             * explicit sort step, so we needn't do so here.
             * 注意:成本最低的路径可能已被正确排序。
             *      try_mergejoin_path将检测这种情况并禁止显式排序步骤，因此在这里不需要这样做。
             */
            try_mergejoin_path(root,
                               joinrel,
                               outer_path,
                               inner_path,
                               merge_pathkeys,
                               cur_mergeclauses,
                               outerkeys,
                               innerkeys,
                               jointype,
                               extra,
                               false);
    
            /*
             * If we have partial outer and parallel safe inner path then try
             * partial mergejoin path.
             * 并行处理
             */
            if (cheapest_partial_outer && cheapest_safe_inner)
                try_partial_mergejoin_path(root,
                                           joinrel,
                                           cheapest_partial_outer,
                                           cheapest_safe_inner,
                                           merge_pathkeys,
                                           cur_mergeclauses,
                                           outerkeys,
                                           innerkeys,
                                           jointype,
                                           extra);
        }
    }
    
    
    //----------------------------------- try_mergejoin_path
    /*
     * try_mergejoin_path
     *    Consider a merge join path; if it appears useful, push it into
     *    the joinrel's pathlist via add_path().
     *    尝试merge join path,如可用,则把该路径通过add_path函数放在joinrel的pathlist链表中
     */
    static void
    try_mergejoin_path(PlannerInfo *root,
                       RelOptInfo *joinrel,
                       Path *outer_path,
                       Path *inner_path,
                       List *pathkeys,
                       List *mergeclauses,
                       List *outersortkeys,
                       List *innersortkeys,
                       JoinType jointype,
                       JoinPathExtraData *extra,
                       bool is_partial)
    {
        Relids      required_outer;
        JoinCostWorkspace workspace;
    
        if (is_partial)
        {
            try_partial_mergejoin_path(root,
                                       joinrel,
                                       outer_path,
                                       inner_path,
                                       pathkeys,
                                       mergeclauses,
                                       outersortkeys,
                                       innersortkeys,
                                       jointype,
                                       extra);//并行执行
            return;
        }
    
        /*
         * Check to see if proposed path is still parameterized, and reject if the
         * parameterization wouldn't be sensible.
         * 检查建议的路径是否仍然参数化，如果参数化不合理，则舍弃。
         */
        required_outer = calc_non_nestloop_required_outer(outer_path,
                                                          inner_path);
        if (required_outer &&
            !bms_overlap(required_outer, extra->param_source_rels))
        {
            /* Waste no memory when we reject a path here */
            bms_free(required_outer);
            return;
        }
    
        /*
         * If the given paths are already well enough ordered, we can skip doing
         * an explicit sort.
         * 如果给定的路径已完成排序,则跳过显式排序.
         */
        if (outersortkeys &&
            pathkeys_contained_in(outersortkeys, outer_path->pathkeys))
            outersortkeys = NIL;
        if (innersortkeys &&
            pathkeys_contained_in(innersortkeys, inner_path->pathkeys))
            innersortkeys = NIL;
    
        /*
         * See comments in try_nestloop_path().
         */
        initial_cost_mergejoin(root, &workspace, jointype, mergeclauses,
                               outer_path, inner_path,
                               outersortkeys, innersortkeys,
                               extra);//初始化mergejoin
    
        if (add_path_precheck(joinrel,
                              workspace.startup_cost, workspace.total_cost,
                              pathkeys, required_outer))//执行前置检查
        {
            add_path(joinrel, (Path *)
                     create_mergejoin_path(root,
                                           joinrel,
                                           jointype,
                                           &workspace,
                                           extra,
                                           outer_path,
                                           inner_path,
                                           extra->restrictlist,
                                           pathkeys,
                                           required_outer,
                                           mergeclauses,
                                           outersortkeys,
                                           innersortkeys));//创建并添加路径
        }
        else
        {
            /* Waste no memory when we reject a path here */
            bms_free(required_outer);
        }
    }
    
    
    //----------------------- create_mergejoin_path
     
     /*
      * create_mergejoin_path
      *    Creates a pathnode corresponding to a mergejoin join between
      *    two relations
      *    创建merge join访问路径
      *
      * 'joinrel' is the join relation
      * joinrel-连接后新生成的relation
      * 'jointype' is the type of join required
      * jointype-连接类型
      * 'workspace' is the result from initial_cost_mergejoin
      * workspace-通过函数initial_cost_mergejoin返回的结果
      * 'extra' contains various information about the join
      * extra-额外的输入参数
      * 'outer_path' is the outer path
      * outer_path-outer relation访问路径
      * 'inner_path' is the inner path
      * inner_path-inner relation访问路径
      * 'restrict_clauses' are the RestrictInfo nodes to apply at the join
      * restrict_clauses-约束条件子句
      * 'pathkeys' are the path keys of the new join path
      * pathkeys-排序键
      * 'required_outer' is the set of required outer rels
      * required_outer-如需要outer rels,则在此保存relids
      * 'mergeclauses' are the RestrictInfo nodes to use as merge clauses
      *      (this should be a subset of the restrict_clauses list)
      * mergeclauses-合并连接条件子句
      * 'outersortkeys' are the sort varkeys for the outer relation
      * outersortkeys-outer relation的排序键
      * 'innersortkeys' are the sort varkeys for the inner relation
      * innersortkeys-inner relation的排序键
      */
     MergePath *
     create_mergejoin_path(PlannerInfo *root,
                           RelOptInfo *joinrel,
                           JoinType jointype,
                           JoinCostWorkspace *workspace,
                           JoinPathExtraData *extra,
                           Path *outer_path,
                           Path *inner_path,
                           List *restrict_clauses,
                           List *pathkeys,
                           Relids required_outer,
                           List *mergeclauses,
                           List *outersortkeys,
                           List *innersortkeys)
     {
         MergePath  *pathnode = makeNode(MergePath);
     
         pathnode->jpath.path.pathtype = T_MergeJoin;
         pathnode->jpath.path.parent = joinrel;
         pathnode->jpath.path.pathtarget = joinrel->reltarget;
         pathnode->jpath.path.param_info =
             get_joinrel_parampathinfo(root,
                                       joinrel,
                                       outer_path,
                                       inner_path,
                                       extra->sjinfo,
                                       required_outer,
                                       &restrict_clauses);
         pathnode->jpath.path.parallel_aware = false;
         pathnode->jpath.path.parallel_safe = joinrel->consider_parallel &&
             outer_path->parallel_safe && inner_path->parallel_safe;
         /* This is a foolish way to estimate parallel_workers, but for now... */
         pathnode->jpath.path.parallel_workers = outer_path->parallel_workers;
         pathnode->jpath.path.pathkeys = pathkeys;
         pathnode->jpath.jointype = jointype;
         pathnode->jpath.inner_unique = extra->inner_unique;
         pathnode->jpath.outerjoinpath = outer_path;
         pathnode->jpath.innerjoinpath = inner_path;
         pathnode->jpath.joinrestrictinfo = restrict_clauses;
         pathnode->path_mergeclauses = mergeclauses;
         pathnode->outersortkeys = outersortkeys;
         pathnode->innersortkeys = innersortkeys;
         /* pathnode->skip_mark_restore will be set by final_cost_mergejoin */
         /* pathnode->materialize_inner will be set by final_cost_mergejoin */
     
         final_cost_mergejoin(root, pathnode, workspace, extra);//估算成本
     
         return pathnode;
     }
    

### 三、跟踪分析

测试脚本如下

    
    
    select a.*,b.grbh,b.je 
    from t_dwxx a,
        lateral (select t1.dwbh,t1.grbh,t2.je 
         from t_grxx t1 
              inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    order by b.dwbh;
    
    

启动gdb,设置断点

    
    
    (gdb) b sort_inner_and_outer
    Breakpoint 1 at 0x7af63a: file joinpath.c, line 888.
    (gdb) c
    Continuing.
    
    Breakpoint 1, sort_inner_and_outer (root=0x1a4a278, joinrel=0x1aa7180, outerrel=0x1a55700, innerrel=0x1a56c30, 
        jointype=JOIN_INNER, extra=0x7ffca933f880) at joinpath.c:888
    888     JoinType    save_jointype = jointype;
    (gdb) 
    

新生成的joinrel是1号和3号RTE的连接,类型为JOIN_INNER

    
    
    (gdb) p *joinrel->relids->words
    $1 = 10
    (gdb) p jointype
    $2 = JOIN_INNER
    

获取排序键,PathKey中的等价类EC,成员为t_grxx.dwbh和t_dwxx.dwbh

    
    
    ...
    (gdb) 
    993     all_pathkeys = select_outer_pathkeys_for_merge(root,
    (gdb) n
    997     foreach(l, all_pathkeys)
    (gdb) p *all_pathkeys
    $3 = {type = T_List, length = 1, head = 0x1a69490, tail = 0x1a69490}
    (gdb) p *(PathKey *)all_pathkeys->head->data.ptr_value
    $5 = {type = T_PathKey, pk_eclass = 0x1a60e08, pk_opfamily = 1994, pk_strategy = 1, pk_nulls_first = false}
    ...
    (gdb) set $rt=(RelabelType *)((EquivalenceMember *)$ec->ec_members->head->data.ptr_value)->em_expr
    (gdb) p *$rt->arg
    $14 = {type = T_Var}
    (gdb) p *(Var *)$rt->arg
    $15 = {xpr = {type = T_Var}, varno = 3, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 1, location = 208}
    (gdb) set $rt2=(RelabelType *)((EquivalenceMember *)$ec->ec_members->head->next->data.ptr_value)->em_expr
    (gdb) p *(Var *)$rt2->arg
    $16 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 24, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 2, location = 218}
    

开始遍历all_pathkeys

    
    
    (gdb) n
    999         List       *front_pathkey = (List *) lfirst(l);
    

获取连接条件子句,t_dwxx.dwbh=t_grxx.dwbh

    
    
    (gdb) p *cur_mergeclauses
    $17 = {type = T_List, length = 1, head = 0x1a694f0, tail = 0x1a694f0}
    

构建outer和inner relation的排序键

    
    
    (gdb) p *(PathKey *)innerkeys->head->data.ptr_value
    $22 = {type = T_PathKey, pk_eclass = 0x1a60e08, pk_opfamily = 1994, pk_strategy = 1, pk_nulls_first = false}
    (gdb) p *(PathKey *)merge_pathkeys->head->data.ptr_value
    $25 = {type = T_PathKey, pk_eclass = 0x1a60e08, pk_opfamily = 1994, pk_strategy = 1, pk_nulls_first = false}
    

尝试merge join,进入函数try_mergejoin_path

    
    
    (gdb) 
    1038            try_mergejoin_path(root,
    (gdb) step
    try_mergejoin_path (root=0x1a4dcc0, joinrel=0x1a68e20, outer_path=0x1a62288, inner_path=0x1a62320, pathkeys=0x1a694b8, 
        mergeclauses=0x1a69518, outersortkeys=0x1a694b8, innersortkeys=0x1a69578, jointype=JOIN_INNER, extra=0x7ffca933f880, 
        is_partial=false) at joinpath.c:572
    572     if (is_partial)
    
    

初始merge join成本

    
    
    ...
    (gdb) 
    615     initial_cost_mergejoin(root, &workspace, jointype, mergeclauses,
    (gdb) p workspace
    $26 = {startup_cost = 10861.483356195882, total_cost = 11134.203356195881, run_cost = 24.997499999999999, 
      inner_run_cost = 247.72250000000003, inner_rescan_run_cost = 1.3627136827435593e-316, outer_rows = 9999, 
      inner_rows = 100000, outer_skip_rows = 0, inner_skip_rows = 911, numbuckets = 27665584, numbatches = 0, 
      inner_rows_total = 1.3681950446447804e-316}
    

构造merge join

    
    
    ...
    (gdb) n
    625                  create_mergejoin_path(root,
    (gdb) 
    624         add_path(joinrel, (Path *)
    (gdb) 
    644 }
    (gdb) p *joinrel->pathlist
    $28 = {type = T_List, length = 1, head = 0x1a6a180, tail = 0x1a6a180}
    (gdb) p *(Node *)joinrel->pathlist->head->data.ptr_value
    $29 = {type = T_MergePath}
    (gdb) p *(MergePath *)joinrel->pathlist->head->data.ptr_value
    $30 = {jpath = {path = {type = T_MergePath, pathtype = T_MergeJoin, parent = 0x1a68e20, pathtarget = 0x1a69058, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 10863.760856195882, total_cost = 12409.200856195883, pathkeys = 0x1a694b8}, jointype = JOIN_INNER, 
        inner_unique = false, outerjoinpath = 0x1a62288, innerjoinpath = 0x1a62320, joinrestrictinfo = 0x1a692f8}, 
      path_mergeclauses = 0x1a69518, outersortkeys = 0x1a694b8, innersortkeys = 0x1a69578, skip_mark_restore = false, 
      materialize_inner = false}
    
    

完成调用

    
    
    (gdb) n
    sort_inner_and_outer (root=0x1a4dcc0, joinrel=0x1a68e20, outerrel=0x1a4d700, innerrel=0x1a4d918, jointype=JOIN_INNER, 
        extra=0x7ffca933f880) at joinpath.c:1054
    1054            if (cheapest_partial_outer && cheapest_safe_inner)
    (gdb) 
    997     foreach(l, all_pathkeys)
    (gdb) 
    1066    }
    (gdb) n
    add_paths_to_joinrel (root=0x1a4dcc0, joinrel=0x1a68e20, outerrel=0x1a4d700, innerrel=0x1a4d918, jointype=JOIN_INNER, 
        sjinfo=0x7ffca933f970, restrictlist=0x1a692f8) at joinpath.c:279
    279     if (mergejoin_allowed)
    (gdb) 
    280         match_unsorted_outer(root, joinrel, outerrel, innerrel,
    ...
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

