本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel函数中的hash_inner_and_outer函数，该函数尝试构造hash
join访问路径。

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

hash join的算法实现伪代码如下:  
_Step 1_  
FOR small_table_row IN (SELECT * FROM small_table)  
LOOP  
slot := HASH(small_table_row.join_key);  
INSERT_HASH_TABLE(slot,small_table_row);  
END LOOP;

_Step 2_  
FOR large_table_row IN (SELECT * FROM large_table) LOOP  
slot := HASH(large_table_row.join_key);  
small_table_row = LOOKUP_HASH_TABLE(slot,large_table_row.join_key);  
IF small_table_row FOUND THEN  
output small_table_row + large_table_row;  
END IF;  
END LOOP;

**hash_inner_and_outer**  
该函数创建hash join访问路径。

    
    
    //------------------------------------------------ hash_inner_and_outer
    
    /*
     * hash_inner_and_outer
     *    Create hashjoin join paths by explicitly hashing both the outer and
     *    inner keys of each available hash clause.
     *    通过显式对外表和内表(应用每个可用的hash条件)进行hash操作,创建hash join访问路径
     *
     * 'joinrel' is the join relation
     * 'outerrel' is the outer join relation
     * 'innerrel' is the inner join relation
     * 'jointype' is the type of join to do
     * 'extra' contains additional input values
     */
    static void
    hash_inner_and_outer(PlannerInfo *root,
                         RelOptInfo *joinrel,
                         RelOptInfo *outerrel,
                         RelOptInfo *innerrel,
                         JoinType jointype,
                         JoinPathExtraData *extra)
    {
        JoinType    save_jointype = jointype;
        bool        isouterjoin = IS_OUTER_JOIN(jointype);
        List       *hashclauses;
        ListCell   *l;
    
        /*
         * We need to build only one hashclauses list for any given pair of outer
         * and inner relations; all of the hashable clauses will be used as keys.
         * 只需要为给定的外表和内表对构建一个hashclauses条件链表;所有的hashable子句将用作hash键。
         *
         * Scan the join's restrictinfo list to find hashjoinable clauses that are
         * usable with this pair of sub-relations.
         * 扫描连接的约束条件restrictinfo链表，找到可用于这对子关系的hash连接hashjoinable子句。
         */
        hashclauses = NIL;
        foreach(l, extra->restrictlist)
        {
            RestrictInfo *restrictinfo = (RestrictInfo *) lfirst(l);
    
            /*
             * If processing an outer join, only use its own join clauses for
             * hashing.  For inner joins we need not be so picky.
             * 如果处理外连接，则仅使用其自己的连接子句进行哈希操作。对于内连接，则无需如此操作。
             */
            if (isouterjoin && RINFO_IS_PUSHED_DOWN(restrictinfo, joinrel->relids))
                continue;
    
            if (!restrictinfo->can_join ||
                restrictinfo->hashjoinoperator == InvalidOid)
                continue;           /* 不能被hash.not hashjoinable */
    
            /*
             * Check if clause has the form "outer op inner" or "inner op outer".
             * 检查条件是否有形如outer op inner或者inner op outer的形式
             */
            if (!clause_sides_match_join(restrictinfo, outerrel, innerrel))
                continue;           /* no good for these input relations */
    
            hashclauses = lappend(hashclauses, restrictinfo);//加入到hash条件中
        }
    
        /* If we found any usable hashclauses, make paths */
        //如发现可用于hash连接的条件,则构建hash连接访问路径,如无则无法构建
        if (hashclauses)
        {
            /*
             * We consider both the cheapest-total-cost and cheapest-startup-cost
             * outer paths.  There's no need to consider any but the
             * cheapest-total-cost inner path, however.
             * 外表:既考虑了成本最低的总成本，也考虑了外表启动成本最低的访问路径。
             * 内表:除了成本最低的内部路径之外，不需要考虑任何其他路径。
             */
            Path       *cheapest_startup_outer = outerrel->cheapest_startup_path;
            Path       *cheapest_total_outer = outerrel->cheapest_total_path;
            Path       *cheapest_total_inner = innerrel->cheapest_total_path;
    
            /*
             * If either cheapest-total path is parameterized by the other rel, we
             * can't use a hashjoin.  (There's no use looking for alternative
             * input paths, since these should already be the least-parameterized
             * available paths.)
             * 如果其中一个关系参数化了其中一个成本最低的访问路径，那么不能使用hash join。
             * (没有必要寻找替代的输入路径，因为这些路径应该已经是参数化最少的可用路径了。)
             */
            if (PATH_PARAM_BY_REL(cheapest_total_outer, innerrel) ||
                PATH_PARAM_BY_REL(cheapest_total_inner, outerrel))
                return;//直接退出
    
            /* Unique-ify if need be; we ignore parameterized possibilities */
            //如果需要保证唯一性,丢弃参数化
            if (jointype == JOIN_UNIQUE_OUTER)
            {
                cheapest_total_outer = (Path *)
                    create_unique_path(root, outerrel,
                                       cheapest_total_outer, extra->sjinfo);
                Assert(cheapest_total_outer);
                jointype = JOIN_INNER;
                try_hashjoin_path(root,
                                  joinrel,
                                  cheapest_total_outer,
                                  cheapest_total_inner,
                                  hashclauses,
                                  jointype,
                                  extra);
                /* no possibility of cheap startup here */
            }
            else if (jointype == JOIN_UNIQUE_INNER)
            {
                cheapest_total_inner = (Path *)
                    create_unique_path(root, innerrel,
                                       cheapest_total_inner, extra->sjinfo);
                Assert(cheapest_total_inner);
                jointype = JOIN_INNER;
                try_hashjoin_path(root,
                                  joinrel,
                                  cheapest_total_outer,
                                  cheapest_total_inner,
                                  hashclauses,
                                  jointype,
                                  extra);
                if (cheapest_startup_outer != NULL &&
                    cheapest_startup_outer != cheapest_total_outer)
                    try_hashjoin_path(root,
                                      joinrel,
                                      cheapest_startup_outer,
                                      cheapest_total_inner,
                                      hashclauses,
                                      jointype,
                                      extra);
            }
            else//其他连接类型
            {
                /*
                 * For other jointypes, we consider the cheapest startup outer
                 * together with the cheapest total inner, and then consider
                 * pairings of cheapest-total paths including parameterized ones.
                 * There is no use in generating parameterized paths on the basis
                 * of possibly cheap startup cost, so this is sufficient.
                 * 对于其他连接类型，我们考虑成本最低的的外表启动和内表启动访问路径，
                 * 然后考虑包括参数化路径在内的成本最低的访问路径对。
                 * 在基于可能较低的启动成本的基础上生成参数化路径是没有用的，上面的做法就足够了。
                 */
                ListCell   *lc1;
                ListCell   *lc2;
    
                if (cheapest_startup_outer != NULL)//启动成本最低的外表访问路径
                    try_hashjoin_path(root,
                                      joinrel,
                                      cheapest_startup_outer,
                                      cheapest_total_inner,
                                      hashclauses,
                                      jointype,
                                      extra);//构建hash join访问路径
    
                foreach(lc1, outerrel->cheapest_parameterized_paths)//遍历外表参数化路径
                {
                    Path       *outerpath = (Path *) lfirst(lc1);
    
                    /*
                     * We cannot use an outer path that is parameterized by the
                     * inner rel.
                     * 不能使用被内表参数化使用的外表访问路径
                     */
                    if (PATH_PARAM_BY_REL(outerpath, innerrel))
                        continue;
    
                    foreach(lc2, innerrel->cheapest_parameterized_paths)//遍历内表参数化路径
                    {
                        Path       *innerpath = (Path *) lfirst(lc2);
    
                        /*
                         * We cannot use an inner path that is parameterized by
                         * the outer rel, either.
                         * 同样的,不能使用被外表参数化使用的内表访问路径
                         */
                        if (PATH_PARAM_BY_REL(innerpath, outerrel))
                            continue;
    
                        if (outerpath == cheapest_startup_outer &&
                            innerpath == cheapest_total_inner)
                            continue;   /* already tried it */
    
                        try_hashjoin_path(root,
                                          joinrel,
                                          outerpath,
                                          innerpath,
                                          hashclauses,
                                          jointype,
                                          extra);//构建hash连接访问路径
                    }
                }
            }
    
            /*
             * If the joinrel is parallel-safe, we may be able to consider a
             * partial hash join.  However, we can't handle JOIN_UNIQUE_OUTER,
             * because the outer path will be partial, and therefore we won't be
             * able to properly guarantee uniqueness.  Similarly, we can't handle
             * JOIN_FULL and JOIN_RIGHT, because they can produce false null
             * extended rows.  Also, the resulting path must not be parameterized.
             * We would be able to support JOIN_FULL and JOIN_RIGHT for Parallel
             * Hash, since in that case we're back to a single hash table with a
             * single set of match bits for each batch, but that will require
             * figuring out a deadlock-free way to wait for the probe to finish.
             * 如果连接是并行安全的，可以考虑并行哈希连接。
             * 但是，我们不能处理JOIN_UNIQUE_OUTER，因为外部路径是部分的，因此我们不能正确地保证惟一性。
             * 类似地，我们不能处理JOIN_FULL和JOIN_RIGHT，因为它们会产生假空扩展行。
             * 此外，生成的路径不能被参数化。
             * 我们将能够支持JOIN_FULL和JOIN_RIGHT用于并行哈希，
             * 因为在这种情况下，我们将返回到一个哈希表，每个批处理只有一组匹配位，
             * 但这需要找到一种没有死锁的方式来等待探测完成。
             */
            if (joinrel->consider_parallel &&
                save_jointype != JOIN_UNIQUE_OUTER &&
                save_jointype != JOIN_FULL &&
                save_jointype != JOIN_RIGHT &&
                outerrel->partial_pathlist != NIL &&
                bms_is_empty(joinrel->lateral_relids))
            {
                Path       *cheapest_partial_outer;
                Path       *cheapest_partial_inner = NULL;
                Path       *cheapest_safe_inner = NULL;
    
                cheapest_partial_outer =
                    (Path *) linitial(outerrel->partial_pathlist);
    
                /*
                 * Can we use a partial inner plan too, so that we can build a
                 * shared hash table in parallel?
                 * 我们是否也可以使用部分内表访问路径，以便并行构建共享哈希表?
                 */
                if (innerrel->partial_pathlist != NIL && enable_parallel_hash)
                {
                    cheapest_partial_inner =
                        (Path *) linitial(innerrel->partial_pathlist);
                    try_partial_hashjoin_path(root, joinrel,
                                              cheapest_partial_outer,
                                              cheapest_partial_inner,
                                              hashclauses, jointype, extra,
                                              true /* parallel_hash */ );
                }
    
                /*
                 * Normally, given that the joinrel is parallel-safe, the cheapest
                 * total inner path will also be parallel-safe, but if not, we'll
                 * have to search for the cheapest safe, unparameterized inner
                 * path.  If doing JOIN_UNIQUE_INNER, we can't use any alternative
                 * inner path.
                 * 通常，假设连接是并行安全的，最便宜的总内表访问路径也是并行安全的，
                 * 但如果不是，我们将不得不寻找成本最低的安全的、非参数化的内表访问路径。
                 * 如果执行JOIN_UNIQUE_INNER，则不能使用任何替代的内表访问路径。
                 */
                if (cheapest_total_inner->parallel_safe)
                    cheapest_safe_inner = cheapest_total_inner;
                else if (save_jointype != JOIN_UNIQUE_INNER)
                    cheapest_safe_inner =
                        get_cheapest_parallel_safe_total_inner(innerrel->pathlist);
    
                if (cheapest_safe_inner != NULL)
                    try_partial_hashjoin_path(root, joinrel,
                                              cheapest_partial_outer,
                                              cheapest_safe_inner,
                                              hashclauses, jointype, extra,
                                              false /* parallel_hash */ );
            }
        }
    }
     
    
    //----------------------------- try_hashjoin_path
    
     /*
      * try_hashjoin_path
      *    Consider a hash join path; if it appears useful, push it into
      *    the joinrel's pathlist via add_path().
      *    尝试构造hash join访问路径.
      *    如果该访问路径可用,通过add_path函数添加到连接新生成的关系joinrel中的pathlist链表中
      */
     static void
     try_hashjoin_path(PlannerInfo *root,
                       RelOptInfo *joinrel,
                       Path *outer_path,
                       Path *inner_path,
                       List *hashclauses,
                       JoinType jointype,
                       JoinPathExtraData *extra)
     {
         Relids      required_outer;
         JoinCostWorkspace workspace;
     
         /*
          * Check to see if proposed path is still parameterized, and reject if the
          * parameterization wouldn't be sensible.
          * 检查建议的路径是否仍然是参数化的，如果参数化不合理，则拒绝。
          * 
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
          * See comments in try_nestloop_path().  Also note that hashjoin paths
          * never have any output pathkeys, per comments in create_hashjoin_path.
          * 参见try_nestloop_path()中的注释。
          * 还要注意，hash join访问路径从来没有任何输出路径键，参见create_hashjoin_path中的注释.
          */
         initial_cost_hashjoin(root, &workspace, jointype, hashclauses,
                               outer_path, inner_path, extra, false);//初步估算成本
     
         if (add_path_precheck(joinrel,
                               workspace.startup_cost, workspace.total_cost,
                               NIL, required_outer))//初始判断
         {
             add_path(joinrel, (Path *)
                      create_hashjoin_path(root,
                                           joinrel,
                                           jointype,
                                           &workspace,
                                           extra,
                                           outer_path,
                                           inner_path,
                                           false,    /* parallel_hash */
                                           extra->restrictlist,
                                           required_outer,
                                           hashclauses));//创建hash join访问路径,并添加
         }
         else
         {
             /* Waste no memory when we reject a path here */
             bms_free(required_outer);
         }
     }
     
    
    //------------------ create_hashjoin_path
    
     /*
      * create_hashjoin_path
      *    Creates a pathnode corresponding to a hash join between two relations.
      *    创建hash join访问路径Node
      *
      * 'joinrel' is the join relation
      * 'jointype' is the type of join required
      * 'workspace' is the result from initial_cost_hashjoin
      * 'extra' contains various information about the join
      * 'outer_path' is the cheapest outer path
      * 'inner_path' is the cheapest inner path
      * 'parallel_hash' to select Parallel Hash of inner path (shared hash table)
      * 'restrict_clauses' are the RestrictInfo nodes to apply at the join
      * 'required_outer' is the set of required outer rels
      * 'hashclauses' are the RestrictInfo nodes to use as hash clauses
      *      (this should be a subset of the restrict_clauses list)
      */
     HashPath *
     create_hashjoin_path(PlannerInfo *root,
                          RelOptInfo *joinrel,
                          JoinType jointype,
                          JoinCostWorkspace *workspace,
                          JoinPathExtraData *extra,
                          Path *outer_path,
                          Path *inner_path,
                          bool parallel_hash,
                          List *restrict_clauses,
                          Relids required_outer,
                          List *hashclauses)
     {
         HashPath   *pathnode = makeNode(HashPath);
     
         pathnode->jpath.path.pathtype = T_HashJoin;
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
         pathnode->jpath.path.parallel_aware =
             joinrel->consider_parallel && parallel_hash;
         pathnode->jpath.path.parallel_safe = joinrel->consider_parallel &&
             outer_path->parallel_safe && inner_path->parallel_safe;
         /* This is a foolish way to estimate parallel_workers, but for now... */
         pathnode->jpath.path.parallel_workers = outer_path->parallel_workers;
     
         /*
          * A hashjoin never has pathkeys, since its output ordering is
          * unpredictable due to possible batching.  XXX If the inner relation is
          * small enough, we could instruct the executor that it must not batch,
          * and then we could assume that the output inherits the outer relation's
          * ordering, which might save a sort step.  However there is considerable
          * downside if our estimate of the inner relation size is badly off. For
          * the moment we don't risk it.  (Note also that if we wanted to take this
          * seriously, joinpath.c would have to consider many more paths for the
          * outer rel than it does now.)
          * hashjoin从来没有路径键，因为由于可能的批处理，其输出顺序不可预测。
          * 如果内部关系足够小，可以指示执行器它不执行批处理，然后可以假设输出继承外部关系的顺序，这样可以节省排序步骤。
          * 然而，如果对内部关系大小的估计严重不足，就会有相当大的负面影响。
          * (还要注意，如果我们想认真对待这个问题，那就是joinpath.c将不得不考虑比现在更多的外表访问路径。)
          */
         pathnode->jpath.path.pathkeys = NIL;
         pathnode->jpath.jointype = jointype;
         pathnode->jpath.inner_unique = extra->inner_unique;
         pathnode->jpath.outerjoinpath = outer_path;
         pathnode->jpath.innerjoinpath = inner_path;
         pathnode->jpath.joinrestrictinfo = restrict_clauses;
         pathnode->path_hashclauses = hashclauses;
         /* final_cost_hashjoin will fill in pathnode->num_batches */
     
         final_cost_hashjoin(root, pathnode, workspace, extra);//最终的成本估算
     
         return pathnode;
     }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain verbose select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# order by dw.dwbh;
                                               QUERY PLAN                                            
    -------------------------------------------------------------------------------------------------     Sort  (cost=20070.93..20320.93 rows=100000 width=47)
       Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
       Sort Key: dw.dwbh
       ->  Hash Join  (cost=3754.00..8689.61 rows=100000 width=47)
             Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
             Inner Unique: true
             Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
             ->  Hash Join  (cost=3465.00..8138.00 rows=100000 width=31)
                   Output: gr.grbh, gr.xm, gr.dwbh, jf.ny, jf.je
                   Hash Cond: ((jf.grbh)::text = (gr.grbh)::text)
                   ->  Seq Scan on public.t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                         Output: jf.ny, jf.je, jf.grbh
                   ->  Hash  (cost=1726.00..1726.00 rows=100000 width=16)
                         Output: gr.grbh, gr.xm, gr.dwbh
                         ->  Seq Scan on public.t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
                               Output: gr.grbh, gr.xm, gr.dwbh
             ->  Hash  (cost=164.00..164.00 rows=10000 width=20)
                   Output: dw.dwmc, dw.dwbh, dw.dwdz
                   ->  Seq Scan on public.t_dwxx dw  (cost=0.00..164.00 rows=10000 width=20)
                         Output: dw.dwmc, dw.dwbh, dw.dwdz
    (20 rows)
    

启动gdb,设置断点跟踪

    
    
    (gdb) b hash_inner_and_outer
    Breakpoint 1 at 0x7b066b: file joinpath.c, line 1684.
    (gdb) c
    Continuing.
    
    Breakpoint 1, hash_inner_and_outer (root=0x2676078, joinrel=0x26d2bc0, outerrel=0x26814e0, innerrel=0x2682a10, 
        jointype=JOIN_INNER, extra=0x7ffd6ea6b9d0) at joinpath.c:1684
    1684        JoinType    save_jointype = jointype;
    

连接类型为JOIN_INNER

    
    
    (gdb) p jointype
    $1 = JOIN_INNER
    

1号和3号RTE的连接(即t_dwxx和t_grxx)

    
    
    (gdb) p *joinrel->relids->words
    $3 = 10
    

开始遍历连接条件,获取hash连接条件

    
    
    1697        foreach(l, extra->restrictlist)
    (gdb) 
    1699            RestrictInfo *restrictinfo = (RestrictInfo *) lfirst(l);
    

成功获取,t_dwxx.dwbh = t_grxx.dwbh

    
    
    (gdb) 
    1697        foreach(l, extra->restrictlist)
    (gdb) 
    1722        if (hashclauses)
    (gdb) p *hashclauses
    $4 = {type = T_List, length = 1, head = 0x26d4068, tail = 0x26d4068}
    

获取成本最低的外表启动路径/成本最低的外表访问路径/成本最低的内部访问路径  
分别是外表顺序扫描/外表顺序扫描/内部顺序扫描

    
    
    (gdb) n
    1729            Path       *cheapest_startup_outer = outerrel->cheapest_startup_path;
    (gdb) 
    1730            Path       *cheapest_total_outer = outerrel->cheapest_total_path;
    (gdb) 
    1731            Path       *cheapest_total_inner = innerrel->cheapest_total_path;
    (gdb) p *cheapest_startup_outer
    $5 = {type = T_Path, pathtype = T_SeqScan, parent = 0x26814e0, pathtarget = 0x2681718, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10000, startup_cost = 0, total_cost = 164, 
      pathkeys = 0x0}
    (gdb) p *cheapest_total_outer
    $6 = {type = T_Path, pathtype = T_SeqScan, parent = 0x26814e0, pathtarget = 0x2681718, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10000, startup_cost = 0, total_cost = 164, 
      pathkeys = 0x0}
    (gdb) p *cheapest_total_inner
    $7 = {type = T_Path, pathtype = T_SeqScan, parent = 0x2682a10, pathtarget = 0x2682c48, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, startup_cost = 0, total_cost = 1726, 
      pathkeys = 0x0}
    

如外表成本最低的启动路径不为NULL,则尝试hash连接

    
    
    (gdb) n
    1740                PATH_PARAM_BY_REL(cheapest_total_inner, outerrel))
    (gdb) 
    1739            if (PATH_PARAM_BY_REL(cheapest_total_outer, innerrel) ||
    (gdb) 
    1744            if (jointype == JOIN_UNIQUE_OUTER)
    (gdb) 
    1760            else if (jointype == JOIN_UNIQUE_INNER)
    (gdb) 
    1796                if (cheapest_startup_outer != NULL)
    (gdb) 
    1797                    try_hashjoin_path(root,
    

进入try_hashjoin_path

    
    
    (gdb) step
    try_hashjoin_path (root=0x2676078, joinrel=0x26d2bc0, outer_path=0x26853b8, inner_path=0x26cf610, hashclauses=0x26d4090, 
        jointype=JOIN_INNER, extra=0x7ffd6ea6b9d0) at joinpath.c:737
    737     required_outer = calc_non_nestloop_required_outer(outer_path,
    

try_hashjoin_path->初步估算成本

    
    
    ...
    751     initial_cost_hashjoin(root, &workspace, jointype, hashclauses,
    (gdb) p workspace
    $9 = {startup_cost = 3465, total_cost = 4261, run_cost = 796, inner_run_cost = 0, 
      inner_rescan_run_cost = 6.9528109284473596e-310, outer_rows = 3.7882102964330281e-317, 
      inner_rows = 2.0115578425988515e-316, outer_skip_rows = 2.0115578425988515e-316, 
      inner_skip_rows = 6.9528109284331305e-310, numbuckets = 131072, numbatches = 2, inner_rows_total = 100000}    
    

try_hashjoin_path->进入函数create_hashjoin_path

    
    
    (gdb) n
    759                  create_hashjoin_path(root,
    (gdb) step
    create_hashjoin_path (root=0x2676078, joinrel=0x26d2bc0, jointype=JOIN_INNER, workspace=0x7ffd6ea6b850, 
        extra=0x7ffd6ea6b9d0, outer_path=0x26853b8, inner_path=0x26cf610, parallel_hash=false, restrict_clauses=0x26d3098, 
        required_outer=0x0, hashclauses=0x26d4090) at pathnode.c:2330
    2330        HashPath   *pathnode = makeNode(HashPath);
    

try_hashjoin_path->create_hashjoin_path->计算成本并返回

    
    
    (gdb) 
    2370        final_cost_hashjoin(root, pathnode, workspace, extra);
    (gdb) 
    2372        return pathnode;
    (gdb) 
    2373    }
    (gdb) p *pathnode
    $10 = {jpath = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x26d2bc0, pathtarget = 0x26d2df8, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 3465, total_cost = 5386, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
        outerjoinpath = 0x26853b8, innerjoinpath = 0x26cf610, joinrestrictinfo = 0x26d3098}, path_hashclauses = 0x26d4090, 
      num_batches = 2, inner_rows_total = 100000}
    

try_hashjoin_path->添加路径

    
    
    (gdb) n
    try_hashjoin_path (root=0x2676078, joinrel=0x26d2bc0, outer_path=0x26853b8, inner_path=0x26cf610, hashclauses=0x26d4090, 
        jointype=JOIN_INNER, extra=0x7ffd6ea6b9d0) at joinpath.c:758
    758         add_path(joinrel, (Path *)
    (gdb) 
    776 }
    (gdb) 
    

回到hash_inner_and_outer,继续循环

    
    
    (gdb) 
    hash_inner_and_outer (root=0x2676078, joinrel=0x26d2bc0, outerrel=0x26814e0, innerrel=0x2682a10, jointype=JOIN_INNER, 
        extra=0x7ffd6ea6b9d0) at joinpath.c:1805
    1805                foreach(lc1, outerrel->cheapest_parameterized_paths)
    

结束函数调用

    
    
    1904    }
    (gdb) 
    add_paths_to_joinrel (root=0x2676078, joinrel=0x26d2bc0, outerrel=0x26814e0, innerrel=0x2682a10, jointype=JOIN_INNER, 
        sjinfo=0x7ffd6ea6bac0, restrictlist=0x26d3098) at joinpath.c:315
    315     if (joinrel->fdwroutine &&
    (gdb) p *joinrel->pathlist
    $11 = {type = T_List, length = 2, head = 0x26d4160, tail = 0x26d3e30}
    

查看joinrel的路径链表

    
    
    (gdb) p *(Node *)joinrel->pathlist->head->data.ptr_value
    $12 = {type = T_HashPath}
    (gdb) p *(Node *)joinrel->pathlist->head->next->data.ptr_value
    $13 = {type = T_MergePath}
    (gdb) p *(HashPath *)joinrel->pathlist->head->data.ptr_value
    $14 = {jpath = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x26d2bc0, pathtarget = 0x26d2df8, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 3465, total_cost = 5386, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
        outerjoinpath = 0x26853b8, innerjoinpath = 0x26cf610, joinrestrictinfo = 0x26d3098}, path_hashclauses = 0x26d4090, 
      num_batches = 2, inner_rows_total = 100000}
    (gdb) p *(MergePath *)joinrel->pathlist->head->next->data.ptr_value
    $15 = {jpath = {path = {type = T_MergePath, pathtype = T_MergeJoin, parent = 0x26d2bc0, pathtarget = 0x26d2df8, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 10035.66023721841, total_cost = 11955.396048959938, pathkeys = 0x2685928}, jointype = JOIN_INNER, 
        inner_unique = false, outerjoinpath = 0x26ce070, innerjoinpath = 0x26cf610, joinrestrictinfo = 0x26d3098}, 
      path_mergeclauses = 0x26d3eb8, outersortkeys = 0x0, innersortkeys = 0x26d3f18, skip_mark_restore = false, 
      materialize_inner = false}
    

DONE!  
函数initial_cost_hashjoin和final_cost_hashjoin在下一小节介绍.

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

