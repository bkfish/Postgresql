本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel函数中的match_unsorted_outer函数，该函数尝试构造nested
loop join访问路径。

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

nested loop join的算法实现伪代码如下:  
FOR row#1 IN (select * from dataset#1) LOOP  
FOR row#2 IN (select * from dataset#2 where row#1 is matched) LOOP  
output values from row#1 and row#2  
END LOOP  
END LOOP

match_unsorted_outer函数通过在每个可能的外表访问路径上使用迭代替换或合并连接的方式尝试构造连接访问路径。

    
    
    //------------------------------------------------ match_unsorted_outer
    
    /*
     * match_unsorted_outer
     *    Creates possible join paths for processing a single join relation
     *    'joinrel' by employing either iterative substitution or
     *    mergejoining on each of its possible outer paths (considering
     *    only outer paths that are already ordered well enough for merging).
     *    通过在每个可能的外表访问路径上使用迭代替换或合并连接(只考虑已经排序得足够好,用于合并的外表访问路径)，
     *    为处理最终的连接关系“joinrel”创建可能的连接路径。
     *
     * We always generate a nestloop path for each available outer path.
     * In fact we may generate as many as five: one on the cheapest-total-cost
     * inner path, one on the same with materialization, one on the
     * cheapest-startup-cost inner path (if different), one on the
     * cheapest-total inner-indexscan path (if any), and one on the
     * cheapest-startup inner-indexscan path (if different).
     * 我们总是为每个可用的外表访问路径生成一个nested loop访问路径。
     * 实际上，可能会生成多达5个:
     * 一个是内表访问成本最低的路径，
     * 一个是物化但总成本最低的路径，
     * 一个是启动成本最低的内表访问路径(如果不同的话)，
     * 一个是成本最低的内表索引扫描路径(如果有的话)，
     * 一个是启动成本最低的内表索引扫描路径(如果不同的话)。
     *
     * We also consider mergejoins if mergejoin clauses are available.  See
     * detailed comments in generate_mergejoin_paths.
     * 如果mergejoin条件子句可用，也会考虑merge join。详细请参阅generate_mergejoin_paths中的注释。
     * 
     * 'joinrel' is the join relation
     * 'outerrel' is the outer join relation
     * 'innerrel' is the inner join relation
     * 'jointype' is the type of join to do
     * 'extra' contains additional input values
     */
    static void
    match_unsorted_outer(PlannerInfo *root,//优化器信息
                         RelOptInfo *joinrel,//连接生成的relation
                         RelOptInfo *outerrel,//外表
                         RelOptInfo *innerrel,//内表
                         JoinType jointype,//连接类型
                         JoinPathExtraData *extra)//额外的信息
    {
        JoinType    save_jointype = jointype;
        bool        nestjoinOK;
        bool        useallclauses;
        Path       *inner_cheapest_total = innerrel->cheapest_total_path;
        Path       *matpath = NULL;
        ListCell   *lc1;
    
        /*
         * Nestloop only supports inner, left, semi, and anti joins.  Also, if we
         * are doing a right or full mergejoin, we must use *all* the mergeclauses
         * as join clauses, else we will not have a valid plan.  (Although these
         * two flags are currently inverses, keep them separate for clarity and
         * possible future changes.)
         * Nestloop只支持内连接、左连接、半连接和反连接。
         * 此外，如果正在做一个右连接或全合并连接，我们必须使用所有的merge clauses条件作为连接子句，
         * 否则我们将没有一个有效的计划。
         * (尽管这两个标志目前是反向的，但为了清晰和可能的未来变化，请将它们分开。)
         */
        switch (jointype)
        {
            case JOIN_INNER:
            case JOIN_LEFT:
            case JOIN_SEMI:
            case JOIN_ANTI:
                nestjoinOK = true;
                useallclauses = false;
                break;
            case JOIN_RIGHT:
            case JOIN_FULL:
                nestjoinOK = false;
                useallclauses = true;
                break;
            case JOIN_UNIQUE_OUTER:
            case JOIN_UNIQUE_INNER:
                jointype = JOIN_INNER;
                nestjoinOK = true;
                useallclauses = false;
                break;
            default:
                elog(ERROR, "unrecognized join type: %d",
                     (int) jointype);
                nestjoinOK = false; /* keep compiler quiet */
                useallclauses = false;
                break;
        }
    
        /*
         * If inner_cheapest_total is parameterized by the outer rel, ignore it;
         * we will consider it below as a member of cheapest_parameterized_paths,
         * but the other possibilities considered in this routine aren't usable.
         * 如果inner_cheapest_total被外表rel参数化，则忽略它;
         * 下面将把它看作cheapest_parameterized_paths的成员，但是这个处理过程中的其他尝试则不可用此信息。
         */
        if (PATH_PARAM_BY_REL(inner_cheapest_total, outerrel))
            inner_cheapest_total = NULL;
    
        /*
         * If we need to unique-ify the inner path, we will consider only the
         * cheapest-total inner.
         * 如果需要唯一化内部路径，将只考虑成本最低的内表访问路径。
         */
        if (save_jointype == JOIN_UNIQUE_INNER)
        {
            /* No way to do this with an inner path parameterized by outer rel */
            //除了使用外表rel参数化的内表访问路径,没有其他办法.
            if (inner_cheapest_total == NULL)
                return;
            inner_cheapest_total = (Path *)
                create_unique_path(root, innerrel, inner_cheapest_total, extra->sjinfo);
            Assert(inner_cheapest_total);
        }
        else if (nestjoinOK)
        {
            /*
             * Consider materializing the cheapest inner path, unless
             * enable_material is off or the path in question materializes its
             * output anyway.
             * 尝试实现成本最低的内表访问路径，除非enable_material关闭或该路径以其他方式物化其输出。
             */
            if (enable_material && inner_cheapest_total != NULL &&
                !ExecMaterializesOutput(inner_cheapest_total->pathtype))
                matpath = (Path *)
                    create_material_path(innerrel, inner_cheapest_total);
        }
    
        foreach(lc1, outerrel->pathlist)//遍历外表访问路径
        {
            Path       *outerpath = (Path *) lfirst(lc1);
            List       *merge_pathkeys;
    
            /*
             * We cannot use an outer path that is parameterized by the inner rel.
             * 不能使用使用内表参数化的外表访问路径
             */
            if (PATH_PARAM_BY_REL(outerpath, innerrel))
                continue;
    
            /*
             * If we need to unique-ify the outer path, it's pointless to consider
             * any but the cheapest outer.  (XXX we don't consider parameterized
             * outers, nor inners, for unique-ified cases.  Should we?)
             * 如果需要唯一化外表访问路径，那么只考虑成本最低的外表访问路径是毫无意义的。
             * 不考虑参数化的outers，也不考虑inners，因为它们是独特的情况。那我们应该做的是?)
             */
            if (save_jointype == JOIN_UNIQUE_OUTER)
            {
                if (outerpath != outerrel->cheapest_total_path)
                    continue;
                outerpath = (Path *) create_unique_path(root, outerrel,
                                                        outerpath, extra->sjinfo);
                Assert(outerpath);
            }
    
            /*
             * The result will have this sort order (even if it is implemented as
             * a nestloop, and even if some of the mergeclauses are implemented by
             * qpquals rather than as true mergeclauses):
             * 最终结果将会有这样的排序顺序(即使它是作为nestloop实现的，
             * 即使一些mergeclauses是由qpquals而不是作为真正的mergeclauses实现的):
             */
            merge_pathkeys = build_join_pathkeys(root, joinrel, jointype,
                                                 outerpath->pathkeys);
    
            if (save_jointype == JOIN_UNIQUE_INNER)
            {
                /*
                 * Consider nestloop join, but only with the unique-ified cheapest
                 * inner path
                 * 尝试nestloop join,只是使用保证唯一性的成本最低的内表访问路径
                 */
                try_nestloop_path(root,
                                  joinrel,
                                  outerpath,
                                  inner_cheapest_total,
                                  merge_pathkeys,
                                  jointype,
                                  extra);
            }
            else if (nestjoinOK)
            {
                /*
                 * Consider nestloop joins using this outer path and various
                 * available paths for the inner relation.  We consider the
                 * cheapest-total paths for each available parameterization of the
                 * inner relation, including the unparameterized case.
                 * 尝试使用这个外表访问路径和内表的各种可用路径的nestloop连接。
                 * 尝试内部关系的每个可用参数化(包括非参数化用例)的成本最低路径。
                 */
                ListCell   *lc2;
    
                foreach(lc2, innerrel->cheapest_parameterized_paths)//遍历参数化路径
                {
                    Path       *innerpath = (Path *) lfirst(lc2);
    
                    try_nestloop_path(root,
                                      joinrel,
                                      outerpath,
                                      innerpath,
                                      merge_pathkeys,
                                      jointype,
                                      extra);//尝试nestloop join
                }
    
                /* Also consider materialized form of the cheapest inner path */
                if (matpath != NULL)
                    try_nestloop_path(root,
                                      joinrel,
                                      outerpath,
                                      matpath,
                                      merge_pathkeys,
                                      jointype,
                                      extra);//也会尝试成本最低的物化内表访问路径
            }
    
            /* Can't do anything else if outer path needs to be unique'd */
            //如外表需保证唯一,则继续下一循环
            if (save_jointype == JOIN_UNIQUE_OUTER)
                continue;
    
            /* Can't do anything else if inner rel is parameterized by outer */
            //内表是被外表参数化的,继续下一循环
            if (inner_cheapest_total == NULL)
                continue;
    
            /* Generate merge join paths */
            //构造merge join路径
            generate_mergejoin_paths(root, joinrel, innerrel, outerpath,
                                     save_jointype, extra, useallclauses,
                                     inner_cheapest_total, merge_pathkeys,
                                     false);
        }
    
        /*
         * Consider partial nestloop and mergejoin plan if outerrel has any
         * partial path and the joinrel is parallel-safe.  However, we can't
         * handle JOIN_UNIQUE_OUTER, because the outer path will be partial, and
         * therefore we won't be able to properly guarantee uniqueness.  Nor can
         * we handle extra_lateral_rels, since partial paths must not be
         * parameterized. Similarly, we can't handle JOIN_FULL and JOIN_RIGHT,
         * because they can produce false null extended rows.
         * 考虑并行nestloop和合并计划，如果外表有并行访问路径，并且join rel是是并行安全的。
         * 但是，不能处理JOIN_UNIQUE_OUTER，因为外部路径是并行的，因此不能正确地保证惟一性。
         * 同时,也不能处理extra_lateral al_rels，因为部分路径必须不被参数化。
         * 类似地，我们不能处理JOIN_FULL和JOIN_RIGHT，因为它们会产生假空扩展行。
         */
        if (joinrel->consider_parallel &&
            save_jointype != JOIN_UNIQUE_OUTER &&
            save_jointype != JOIN_FULL &&
            save_jointype != JOIN_RIGHT &&
            outerrel->partial_pathlist != NIL &&
            bms_is_empty(joinrel->lateral_relids))
        {
            if (nestjoinOK)
                consider_parallel_nestloop(root, joinrel, outerrel, innerrel,
                                           save_jointype, extra);//尝试并行netstloop join
    
            /*
             * If inner_cheapest_total is NULL or non parallel-safe then find the
             * cheapest total parallel safe path.  If doing JOIN_UNIQUE_INNER, we
             * can't use any alternative inner path.
             * 如果inner_cheapest_total为空或非并行安全，那么可以找到成本最低的总并行安全访问路径。
             * 如果执行JOIN_UNIQUE_INNER，则不能使用任何替代的内表访问路径。
             */
            if (inner_cheapest_total == NULL ||
                !inner_cheapest_total->parallel_safe)
            {
                if (save_jointype == JOIN_UNIQUE_INNER)
                    return;
    
                inner_cheapest_total = get_cheapest_parallel_safe_total_inner(
                                                                              innerrel->pathlist);
            }
    
            if (inner_cheapest_total)
                consider_parallel_mergejoin(root, joinrel, outerrel, innerrel,
                                            save_jointype, extra,
                                            inner_cheapest_total);//尝试并行merge join
        }
    }
    
    //------------------------------------ try_nestloop_path
    
    /*
     * try_nestloop_path
     *    Consider a nestloop join path; if it appears useful, push it into
     *    the joinrel's pathlist via add_path().
     *    尝试nestloop join访问路径;
     *    如果该路径可行,则通过函数add_path把该路径加入到joinrel's pathlist链表中.
     */
    static void
    try_nestloop_path(PlannerInfo *root,
                      RelOptInfo *joinrel,
                      Path *outer_path,
                      Path *inner_path,
                      List *pathkeys,
                      JoinType jointype,
                      JoinPathExtraData *extra)
    {
        Relids      required_outer;
        JoinCostWorkspace workspace;
        RelOptInfo *innerrel = inner_path->parent;
        RelOptInfo *outerrel = outer_path->parent;
        Relids      innerrelids;
        Relids      outerrelids;
        Relids      inner_paramrels = PATH_REQ_OUTER(inner_path);
        Relids      outer_paramrels = PATH_REQ_OUTER(outer_path);
    
        /*
         * Paths are parameterized by top-level parents, so run parameterization
         * tests on the parent relids.
         * 路径由最顶层的父类参数化，因此在父类集上运行参数化测试。
         */
        if (innerrel->top_parent_relids)
            innerrelids = innerrel->top_parent_relids;
        else
            innerrelids = innerrel->relids;
    
        if (outerrel->top_parent_relids)
            outerrelids = outerrel->top_parent_relids;
        else
            outerrelids = outerrel->relids;
    
        /*
         * Check to see if proposed path is still parameterized, and reject if the
         * parameterization wouldn't be sensible --- unless allow_star_schema_join
         * says to allow it anyway.  Also, we must reject if have_dangerous_phv
         * doesn't like the look of it, which could only happen if the nestloop is
         * still parameterized.
         * 检查建议的访问路径是否仍然是参数化的，如果参数化不合理，则丢弃——除非allow_star_schema_join设置为允许。
         * 另外，如果have_dangerousous_phv不允许，则必须丢弃它，而这种情况只有在nestloop仍然被参数化时才会发生。
         */
        required_outer = calc_nestloop_required_outer(outerrelids, outer_paramrels,
                                                      innerrelids, inner_paramrels);
        if (required_outer &&
            ((!bms_overlap(required_outer, extra->param_source_rels) &&
              !allow_star_schema_join(root, outerrelids, inner_paramrels)) ||
             have_dangerous_phv(root, outerrelids, inner_paramrels)))
        {
            /* Waste no memory when we reject a path here */
            bms_free(required_outer);
            return;//退出
        }
    
        /*
         * Do a precheck to quickly eliminate obviously-inferior paths.  We
         * calculate a cheap lower bound on the path's cost and then use
         * add_path_precheck() to see if the path is clearly going to be dominated
         * by some existing path for the joinrel.  If not, do the full pushup with
         * creating a fully valid path structure and submitting it to add_path().
         * The latter two steps are expensive enough to make this two-phase
         * methodology worthwhile.
         * 参见merge join的注释
         */
        initial_cost_nestloop(root, &workspace, jointype,
                              outer_path, inner_path, extra);//初步计算nestloop的成本
    
        if (add_path_precheck(joinrel,
                              workspace.startup_cost, workspace.total_cost,
                              pathkeys, required_outer))//初步检查
        {
            /*
             * If the inner path is parameterized, it is parameterized by the
             * topmost parent of the outer rel, not the outer rel itself.  Fix
             * that.
             * 如果内表访问路径是参数化的，那么它是由外层rel的最上层父元素参数化的，而不是外层rel本身。
             * 需解决这个问题。
             */
            if (PATH_PARAM_BY_PARENT(inner_path, outer_path->parent))
            {
                inner_path = reparameterize_path_by_child(root, inner_path,
                                                          outer_path->parent);
    
                /*
                 * If we could not translate the path, we can't create nest loop
                 * path.
                 * 如果不能处理此情况,则不能创建nest loop访问路径
                 */
                if (!inner_path)
                {
                    bms_free(required_outer);
                    return;
                }
            }
    
            add_path(joinrel, (Path *)
                     create_nestloop_path(root,
                                          joinrel,
                                          jointype,
                                          &workspace,
                                          extra,
                                          outer_path,
                                          inner_path,
                                          extra->restrictlist,
                                          pathkeys,
                                          required_outer));//创建并添加nestloop路径
        }
        else
        {
            /* Waste no memory when we reject a path here */
            bms_free(required_outer);
        }
    }
    
    //----------------------- create_nestloop_path
    
     /*
      * create_nestloop_path
      *    Creates a pathnode corresponding to a nestloop join between two
      *    relations.
      *    创建nestloop 连接路径节点.
      *
      * 'joinrel' is the join relation.
      * 'jointype' is the type of join required
      * 'workspace' is the result from initial_cost_nestloop
      * 'extra' contains various information about the join
      * 'outer_path' is the outer path
      * 'inner_path' is the inner path
      * 'restrict_clauses' are the RestrictInfo nodes to apply at the join
      * 'pathkeys' are the path keys of the new join path
      * 'required_outer' is the set of required outer rels
      *
      * Returns the resulting path node.
      */
     NestPath *
     create_nestloop_path(PlannerInfo *root,//优化器信息
                          RelOptInfo *joinrel,//连接生成的relation
                          JoinType jointype,//连接类型
                          JoinCostWorkspace *workspace,//成本workspace
                          JoinPathExtraData *extra,//额外的信息
                          Path *outer_path,//外表访问路径
                          Path *inner_path,//内部访问路径
                          List *restrict_clauses,//约束条件
                          List *pathkeys,//排序键
                          Relids required_outer)//需依赖的外部relids
     {
         NestPath   *pathnode = makeNode(NestPath);
         Relids      inner_req_outer = PATH_REQ_OUTER(inner_path);
     
         /*
          * If the inner path is parameterized by the outer, we must drop any
          * restrict_clauses that are due to be moved into the inner path.  We have
          * to do this now, rather than postpone the work till createplan time,
          * because the restrict_clauses list can affect the size and cost
          * estimates for this path.
          * 如果内表访问路径是由外部参数化的，那么必须删除任何将移动到内表访问路径的约束条件子句。
          * 现在必须这样做，而不是将此项工作推迟到create plan的时候，因为限制条件链表会影响此路径的大小和成本估计。
          */
         if (bms_overlap(inner_req_outer, outer_path->parent->relids))//内表需依赖的relids与外表父路径的relids存在交集
         {
             Relids      inner_and_outer = bms_union(inner_path->parent->relids,
                                                     inner_req_outer);//合并内部父路径的relids和内部依赖的relids
             List       *jclauses = NIL;
             ListCell   *lc;
     
             foreach(lc, restrict_clauses)
             {
                 RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc);
     
                 if (!join_clause_is_movable_into(rinfo,
                                                  inner_path->parent->relids,
                                                  inner_and_outer))
                     jclauses = lappend(jclauses, rinfo);
             }
             restrict_clauses = jclauses;
         }
         //创建路径
         pathnode->path.pathtype = T_NestLoop;
         pathnode->path.parent = joinrel;
         pathnode->path.pathtarget = joinrel->reltarget;
         pathnode->path.param_info =
             get_joinrel_parampathinfo(root,
                                       joinrel,
                                       outer_path,
                                       inner_path,
                                       extra->sjinfo,
                                       required_outer,
                                       &restrict_clauses);
         pathnode->path.parallel_aware = false;
         pathnode->path.parallel_safe = joinrel->consider_parallel &&
             outer_path->parallel_safe && inner_path->parallel_safe;
         /* This is a foolish way to estimate parallel_workers, but for now... */
         pathnode->path.parallel_workers = outer_path->parallel_workers;
         pathnode->path.pathkeys = pathkeys;
         pathnode->jointype = jointype;
         pathnode->inner_unique = extra->inner_unique;
         pathnode->outerjoinpath = outer_path;
         pathnode->innerjoinpath = inner_path;
         pathnode->joinrestrictinfo = restrict_clauses;
     
         final_cost_nestloop(root, pathnode, workspace, extra);
     
         return pathnode;
     }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain verbose select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# where dwbh = '1001'
    testdb-# order by dw.dwbh;
                                                  QUERY PLAN                                               
    ----------------------------------------------------------------------------------------------     Nested Loop  (cost=0.87..112.00 rows=10 width=47)
       Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
       ->  Nested Loop  (cost=0.58..28.80 rows=10 width=32)
             Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm
             ->  Index Scan using t_dwxx_pkey on public.t_dwxx dw  (cost=0.29..8.30 rows=1 width=20)
                   Output: dw.dwmc, dw.dwbh, dw.dwdz
                   Index Cond: ((dw.dwbh)::text = '1001'::text)
             ->  Index Scan using idx_t_grxx_dwbh on public.t_grxx gr  (cost=0.29..20.40 rows=10 width=16)
                   Output: gr.dwbh, gr.grbh, gr.xm, gr.xb, gr.nl
                   Index Cond: ((gr.dwbh)::text = '1001'::text)
       ->  Index Scan using idx_t_jfxx_grbh on public.t_jfxx jf  (cost=0.29..8.31 rows=1 width=20)
             Output: jf.grbh, jf.ny, jf.je
             Index Cond: ((jf.grbh)::text = (gr.grbh)::text)
    (13 rows)
    

启动gdb,设置断点跟踪

    
    
    (gdb) b match_unsorted_outer
    Breakpoint 1 at 0x7aff2c: file joinpath.c, line 1338.
    (gdb) c
    Continuing.
    
    Breakpoint 1, match_unsorted_outer (root=0x2a987a8, joinrel=0x2aef8a0, outerrel=0x2aa20f0, innerrel=0x2aa3620, 
        jointype=JOIN_INNER, extra=0x7ffc343da930) at joinpath.c:1338
    1338        JoinType    save_jointype = jointype;
    (gdb) 
    

joinrel是1号RTE和3号RTE的连接,即t_dwxx和t_grxx

    
    
    (gdb) p *joinrel->relids->words
    $3 = 10
    

内表成本最低的访问路径是索引扫描

    
    
    (gdb) n
    1341        Path       *inner_cheapest_total = innerrel->cheapest_total_path;
    (gdb) 
    1342        Path       *matpath = NULL;
    (gdb) p *inner_cheapest_total
    $1 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa3620, pathtarget = 0x2aa3858, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.29249999999999998, 
      total_cost = 20.399882614957409, pathkeys = 0x0}
    

连接类型为JOIN_INNER,设置nestjoinOK为T,useallclauses为F

    
    
    (gdb) p save_jointype
    $4 = JOIN_INNER
    (gdb) n
    1352        switch (jointype)
    (gdb) 
    1358                nestjoinOK = true;
    (gdb) 
    1359                useallclauses = false;
    (gdb) 
    1360                break;
    (gdb) 
    

创建物化内表访问路径

    
    
    (gdb) n
    1408            if (enable_material && inner_cheapest_total != NULL &&
    (gdb) 
    1409                !ExecMaterializesOutput(inner_cheapest_total->pathtype))
    (gdb) 
    1408            if (enable_material && inner_cheapest_total != NULL &&
    (gdb) 
    1410                matpath = (Path *)
    (gdb) 
    1414        foreach(lc1, outerrel->pathlist)
    (gdb) p *matpath
    $5 = {type = T_MaterialPath, pathtype = T_Material, parent = 0x2aa3620, pathtarget = 0x2aa3858, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.29249999999999998, 
      total_cost = 20.44988261495741, pathkeys = 0x0}
    

开始遍历外表访问路径

    
    
    (gdb) n
    1416            Path       *outerpath = (Path *) lfirst(lc1);
    

尝试使用这个外表访问路径和内表的各种可用路径构建nestloop连接  
尝试内部关系的每个可用的参数化(包括非参数化用例)成本最低路径

    
    
    ...
    1461            else if (nestjoinOK)
    (gdb) 
    1471                foreach(lc2, innerrel->cheapest_parameterized_paths)
    (gdb) n
    1473                    Path       *innerpath = (Path *) lfirst(lc2);
    (gdb) n
    1475                    try_nestloop_path(root,
    (gdb) p *innerpath
    $6 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa3620, pathtarget = 0x2aa3858, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.29249999999999998, 
      total_cost = 20.399882614957409, pathkeys = 0x0}
    (gdb) n
    1471                foreach(lc2, innerrel->cheapest_parameterized_paths)
    (gdb) 
    1473                    Path       *innerpath = (Path *) lfirst(lc2);
    (gdb) 
    1475                    try_nestloop_path(root,
    (gdb) 
    1471                foreach(lc2, innerrel->cheapest_parameterized_paths)
    (gdb) 
    

同时,也会尝试成本最低的物化内表访问路径

    
    
    1485                if (matpath != NULL)
    (gdb) 
    1486                    try_nestloop_path(root,
    (gdb) 
    1496            if (save_jointype == JOIN_UNIQUE_OUTER)
    

完成match_unsorted_outer的调用

    
    
    1522            save_jointype != JOIN_RIGHT &&
    (gdb) 
    1550    }
    (gdb) 
    add_paths_to_joinrel (root=0x2a987a8, joinrel=0x2aef8a0, outerrel=0x2aa20f0, innerrel=0x2aa3620, jointype=JOIN_INNER, 
        sjinfo=0x7ffc343daa20, restrictlist=0x0) at joinpath.c:306
    306     if (enable_hashjoin || jointype == JOIN_FULL)
    

结果joinrel的访问路径

    
    
    (gdb) p *joinrel->pathlist
    $9 = {type = T_List, length = 1, head = 0x2aefde8, tail = 0x2aefde8}
    (gdb) p *(Node *)joinrel->pathlist->head->data.ptr_value
    $10 = {type = T_NestPath}
    (gdb) p *(NestPath *)joinrel->pathlist->head->data.ptr_value
    $11 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x2aef8a0, pathtarget = 0x2aefad8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.57750000000000001, 
        total_cost = 28.802382614957409, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x2aeb0d8, innerjoinpath = 0x2aeb408, joinrestrictinfo = 0x0}
    

下面考察try_nestloop_path函数的处理逻辑,设置断点,进入try_nestloop_path函数

    
    
    (gdb) b try_nestloop_path
    Breakpoint 1 at 0x7ae950: file joinpath.c, line 373.
    (gdb) c
    Continuing.
    
    Breakpoint 1, try_nestloop_path (root=0x2a987a8, joinrel=0x2aef8a0, outer_path=0x2aeb0d8, inner_path=0x2aeb408, 
        pathkeys=0x0, jointype=JOIN_INNER, extra=0x7ffc343da930) at joinpath.c:373
    373     RelOptInfo *innerrel = inner_path->parent;
    

joinrel初始时,pathlist为NULL

    
    
    (gdb)  p *(Node *)joinrel->pathlist->head->data.ptr_value
    Cannot access memory at address 0x8
    

外表访问路径为索引扫描(t_dwxx),内表访问路径为索引扫描(t_grxx)

    
    
    (gdb) p *outer_path
    $2 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa20f0, pathtarget = 0x2aa2328, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.28500000000000003, 
      total_cost = 8.3025000000000002, pathkeys = 0x0}
    (gdb) p *inner_path
    $3 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa3620, pathtarget = 0x2aa3858, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.29249999999999998, 
      total_cost = 20.399882614957409, pathkeys = 0x0}
    

初步计算nestloop的成本

    
    
    ...
    (gdb) p required_outer
    $4 = (Relids) 0x0
    (gdb) n
    422     initial_cost_nestloop(root, &workspace, jointype,
    

创建nestloop path并添加到joinrel中

    
    
    425     if (add_path_precheck(joinrel,
    (gdb) 
    434         if (PATH_PARAM_BY_PARENT(inner_path, outer_path->parent))
    (gdb) 
    451                  create_nestloop_path(root,
    (gdb) 
    450         add_path(joinrel, (Path *)
    (gdb) 
    

生成的第一条路径

    
    
    (gdb)  p *(NestPath *)joinrel->pathlist->head->data.ptr_value
    $6 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x2aef8a0, pathtarget = 0x2aefad8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.57750000000000001, 
        total_cost = 28.802382614957409, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x2aeb0d8, innerjoinpath = 0x2aeb408, joinrestrictinfo = 0x0}
    

继续执行,尝试第二条路径

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, try_nestloop_path (root=0x2a987a8, joinrel=0x2aef8a0, outer_path=0x2aeb0d8, inner_path=0x2aecf28, 
        pathkeys=0x0, jointype=JOIN_INNER, extra=0x7ffc343da930) at joinpath.c:373
    373     RelOptInfo *innerrel = inner_path->parent;
    

查看inner_path,即内表的cheapest_parameterized_paths链表中的第二个元素

    
    
    (gdb) p *inner_path
    $9 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa3620, pathtarget = 0x2aa3858, param_info = 0x2aed6f8, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.29249999999999998, 
      total_cost = 0.35258, pathkeys = 0x2aeceb0}
    

使用此路径尝试nest loop访问路径,但该路径依赖外部relids(4号RTE),被丢弃

    
    
    403     if (required_outer &&
    (gdb) 
    405           !allow_star_schema_join(root, outerrelids, inner_paramrels)) ||
    (gdb) 
    404         ((!bms_overlap(required_outer, extra->param_source_rels) &&
    (gdb) 
    409         bms_free(required_outer);
    (gdb) p *required_outer
    $12 = {nwords = 1, words = 0x2aefe4c}
    (gdb) p *required_outer->words
    $13 = 16
    (gdb) n
    410         return;
    (gdb) 
    

其他nestloop的访问路径实现,有兴趣的可以自行跟踪.  
initial_cost_nestloop函数和final_cost_nestloop函数在下一节介绍.

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

