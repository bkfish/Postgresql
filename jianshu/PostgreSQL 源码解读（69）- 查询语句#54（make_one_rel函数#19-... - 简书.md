本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel->match_unsorted_outer中的initial_cost_nestloop和final_cost_nestloop函数，这些函数用于计算nestloop
join的Cost。

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

**initial_cost_nestloop**  
该函数预估nestloop join访问路径的成本

    
    
    //---------------------------------------------------------------------- initial_cost_nestloop
    
    /*
     * initial_cost_nestloop
     *    Preliminary estimate of the cost of a nestloop join path.
     *    预估nestloop join访问路径的成本
     *
     * This must quickly produce lower-bound estimates of the path's startup and
     * total costs.  If we are unable to eliminate the proposed path from
     * consideration using the lower bounds, final_cost_nestloop will be called
     * to obtain the final estimates.
     *
     * The exact division of labor between this function and final_cost_nestloop
     * is private to them, and represents a tradeoff between speed of the initial
     * estimate and getting a tight lower bound.  We choose to not examine the
     * join quals here, since that's by far the most expensive part of the
     * calculations.  The end result is that CPU-cost considerations must be
     * left for the second phase; and for SEMI/ANTI joins, we must also postpone
     * incorporation of the inner path's run cost.
     *
     * 'workspace' is to be filled with startup_cost, total_cost, and perhaps
     *      other data to be used by final_cost_nestloop
     * 'jointype' is the type of join to be performed
     * 'outer_path' is the outer input to the join
     * 'inner_path' is the inner input to the join
     * 'extra' contains miscellaneous information about the join
     */
    void
    initial_cost_nestloop(PlannerInfo *root, JoinCostWorkspace *workspace,
                          JoinType jointype,
                          Path *outer_path, Path *inner_path,
                          JoinPathExtraData *extra)
    {
        Cost        startup_cost = 0;
        Cost        run_cost = 0;
        double      outer_path_rows = outer_path->rows;
        Cost        inner_rescan_start_cost;
        Cost        inner_rescan_total_cost;
        Cost        inner_run_cost;
        Cost        inner_rescan_run_cost;
    
        /* 估算重新扫描内表的成本.estimate costs to rescan the inner relation */
        cost_rescan(root, inner_path,
                    &inner_rescan_start_cost,
                    &inner_rescan_total_cost);
    
        /* cost of source data */
    
        /*
         * NOTE: clearly, we must pay both outer and inner paths' startup_cost
         * before we can start returning tuples, so the join's startup cost is
         * their sum.  We'll also pay the inner path's rescan startup cost
         * multiple times.
       * 注意:显然，在开始返回元组之前，必须耗费外表访问路径和内表访问路径的启动成本startup_cost，
       * 因此连接的启动成本是它们的总和。我们还需要多次计算内表访问路径的重新扫描启动成本。
         */
        startup_cost += outer_path->startup_cost + inner_path->startup_cost;
        run_cost += outer_path->total_cost - outer_path->startup_cost;
        if (outer_path_rows > 1)
            run_cost += (outer_path_rows - 1) * inner_rescan_start_cost;
    
        inner_run_cost = inner_path->total_cost - inner_path->startup_cost;
        inner_rescan_run_cost = inner_rescan_total_cost - inner_rescan_start_cost;
    
        if (jointype == JOIN_SEMI || jointype == JOIN_ANTI ||
            extra->inner_unique)
        {
            /*
             * With a SEMI or ANTI join, or if the innerrel is known unique, the
             * executor will stop after the first match.
             * 对于半连接或反连接，或者如果内表已知是唯一的，执行器将在第一次匹配后停止。
         *
             * Getting decent estimates requires inspection of the join quals,
             * which we choose to postpone to final_cost_nestloop.
         * 获得适当的估算需要检查join quals，我们选择将其推迟到final_cost_nestloop中实现。
             */
    
            /* Save private data for final_cost_nestloop */
            workspace->inner_run_cost = inner_run_cost;
            workspace->inner_rescan_run_cost = inner_rescan_run_cost;
        }
        else
        {
            /* Normal case; we'll scan whole input rel for each outer row */
        //常规的情况,对于每一个外表行,将扫描整个内表
            run_cost += inner_run_cost;
            if (outer_path_rows > 1)
                run_cost += (outer_path_rows - 1) * inner_rescan_run_cost;
        }
    
        /* CPU costs left for later */
    
        /* Public result fields */
      //结果赋值
        workspace->startup_cost = startup_cost;
        workspace->total_cost = startup_cost + run_cost;
        /* Save private data for final_cost_nestloop */
        workspace->run_cost = run_cost;
    }
    
    //-------------------------------------- cost_rescan
    /*
     * cost_rescan
     *      Given a finished Path, estimate the costs of rescanning it after
     *      having done so the first time.  For some Path types a rescan is
     *      cheaper than an original scan (if no parameters change), and this
     *      function embodies knowledge about that.  The default is to return
     *      the same costs stored in the Path.  (Note that the cost estimates
     *      actually stored in Paths are always for first scans.)
     *    给定一个已完成的访问路径，估算在第一次完成之后重新扫描它的成本。
     *      对于某些访问路径，重新扫描比原始扫描成本更低(如果没有参数更改)，
     *    该函数体现了这方面的信息。
     *    默认值是返回存储在路径中的相同成本。 
     *    (请注意，实际存储在路径中的成本估算总是用于首次扫描。)
     *
     * This function is not currently intended to model effects such as rescans
     * being cheaper due to disk block caching; what we are concerned with is
     * plan types wherein the executor caches results explicitly, or doesn't
     * redo startup calculations, etc.
     * 该函数目前并不打算对诸如由于磁盘块缓存而使后续的扫描成本更低等效果进行建模;
     * 我们关心的是计划类型，其中执行器显式缓存结果，或不重做启动计算，等等。
     */
    static void
    cost_rescan(PlannerInfo *root, Path *path,
                Cost *rescan_startup_cost,  /* output parameters */
                Cost *rescan_total_cost)
    {
        switch (path->pathtype)//路径类型
        {
            case T_FunctionScan:
    
                /*
                 * Currently, nodeFunctionscan.c always executes the function to
                 * completion before returning any rows, and caches the results in
                 * a tuplestore.  So the function eval cost is all startup cost
                 * and isn't paid over again on rescans. However, all run costs
                 * will be paid over again.
           * 目前nodeFunctionscan.c在在返回数据行之前完成执行,同时会在tuplestore中缓存结果.
           * 因此,函数估算成本是启动成本,同时不需要考虑重新扫描.
           * 但是所有的运行成本在重新扫描时是需要的.
                 */
                *rescan_startup_cost = 0;
                *rescan_total_cost = path->total_cost - path->startup_cost;
                break;
            case T_HashJoin://Hash Join
    
                /*
                 * If it's a single-batch join, we don't need to rebuild the hash
                 * table during a rescan.
           * 如果是单批连接，则不需要在重新扫描期间重新构建哈希表。
                 */
                if (((HashPath *) path)->num_batches == 1)
                {
                    /* Startup cost is exactly the cost of hash table building */
                    *rescan_startup_cost = 0;//启动成本为0(构建hash表)
                    *rescan_total_cost = path->total_cost - path->startup_cost;
                }
                else
                {
                    /* Otherwise, no special treatment */
                    *rescan_startup_cost = path->startup_cost;
                    *rescan_total_cost = path->total_cost;
                }
                break;
            case T_CteScan:
            case T_WorkTableScan:
                {
                    /*
                     * These plan types materialize their final result in a
                     * tuplestore or tuplesort object.  So the rescan cost is only
                     * cpu_tuple_cost per tuple, unless the result is large enough
                     * to spill to disk.
             * 这些计划类型在tuplestore或tuplesort对象中物化它们的最终结果。
             * 因此，重新扫描的成本只是每个元组的cpu_tuple_cost，除非结果大到足以溢出到磁盘。
                     */
                    Cost        run_cost = cpu_tuple_cost * path->rows;
                    double      nbytes = relation_byte_size(path->rows,
                                                            path->pathtarget->width);
                    long        work_mem_bytes = work_mem * 1024L;
    
                    if (nbytes > work_mem_bytes)
                    {
                        /* It will spill, so account for re-read cost */
              //如果溢出到磁盘,那么需考虑重新读取的成本
                        double      npages = ceil(nbytes / BLCKSZ);
    
                        run_cost += seq_page_cost * npages;
                    }
                    *rescan_startup_cost = 0;
                    *rescan_total_cost = run_cost;
                }
                break;
            case T_Material:
            case T_Sort:
                {
                    /*
                     * These plan types not only materialize their results, but do
                     * not implement qual filtering or projection.  So they are
                     * even cheaper to rescan than the ones above.  We charge only
                     * cpu_operator_cost per tuple.  (Note: keep that in sync with
                     * the run_cost charge in cost_sort, and also see comments in
                     * cost_material before you change it.)
             * 这些计划类型不仅物化了它们的结果，而且没有物化qual条件子句过滤或投影。
             * 所以重新扫描比上面的路径成本更低。每个元组耗费的成本只有cpu_operator_cost。
             * (注意:请与cost_sort中的run_cost成本保持一致，并在更改cost_material之前查看注释)。
                     */
                    Cost        run_cost = cpu_operator_cost * path->rows;
                    double      nbytes = relation_byte_size(path->rows,
                                                            path->pathtarget->width);
                    long        work_mem_bytes = work_mem * 1024L;
    
                    if (nbytes > work_mem_bytes)
                    {
                        /* It will spill, so account for re-read cost */
              //如果溢出到磁盘,那么需考虑重新读取的成本
                        double      npages = ceil(nbytes / BLCKSZ);
    
                        run_cost += seq_page_cost * npages;
                    }
                    *rescan_startup_cost = 0;
                    *rescan_total_cost = run_cost;
                }
                break;
            default:
                *rescan_startup_cost = path->startup_cost;
                *rescan_total_cost = path->total_cost;
                break;
        }
    }
    
    

**final_cost_nestloop**  
该函数实现nestloop join访问路径的成本和结果大小的最终估算。

    
    
    //---------------------------------------------------------------------- final_cost_nestloop
    
    /*
     * final_cost_nestloop
     *    Final estimate of the cost and result size of a nestloop join path.
     *    nestloop join访问路径的成本和结果大小的最终估算。
     *
     * 'path' is already filled in except for the rows and cost fields
     * 'workspace' is the result from initial_cost_nestloop
     * 'extra' contains miscellaneous information about the join
     */
    void
    final_cost_nestloop(PlannerInfo *root, NestPath *path,//NL访问路径
                        JoinCostWorkspace *workspace,//initial_cost_nestloop返回的结果
                        JoinPathExtraData *extra)//额外的信息
    {
        Path       *outer_path = path->outerjoinpath;//外表访问路径
        Path       *inner_path = path->innerjoinpath;//内部访问路径
        double      outer_path_rows = outer_path->rows;//外表访问路径行数
        double      inner_path_rows = inner_path->rows;//内部访问路径行数
        Cost        startup_cost = workspace->startup_cost;//启动成本
        Cost        run_cost = workspace->run_cost;//运行成本
        Cost        cpu_per_tuple;//处理每个tuple的CPU成本
        QualCost    restrict_qual_cost;//表达式处理成本
        double      ntuples;//元组数目
    
        /* Protect some assumptions below that rowcounts aren't zero or NaN */
      //确保参数正确
        if (outer_path_rows <= 0 || isnan(outer_path_rows))
            outer_path_rows = 1;
        if (inner_path_rows <= 0 || isnan(inner_path_rows))
            inner_path_rows = 1;
    
        /* Mark the path with the correct row estimate */
      //修正行数估算
        if (path->path.param_info)
            path->path.rows = path->path.param_info->ppi_rows;
        else
            path->path.rows = path->path.parent->rows;
    
        /* For partial paths, scale row estimate. */
      //调整并行执行的行数估算
        if (path->path.parallel_workers > 0)
        {
            double      parallel_divisor = get_parallel_divisor(&path->path);
    
            path->path.rows =
                clamp_row_est(path->path.rows / parallel_divisor);
        }
    
        /*
         * We could include disable_cost in the preliminary estimate, but that
         * would amount to optimizing for the case where the join method is
         * disabled, which doesn't seem like the way to bet.
       * 我们可以将disable_cost包含在初步估算中，但是这相当于为禁用join方法的情况进行了优化。
         */
        if (!enable_nestloop)
            startup_cost += disable_cost;
    
        /* cost of inner-relation source data (we already dealt with outer rel) */
      // 内部源数据的成本
        if (path->jointype == JOIN_SEMI || path->jointype == JOIN_ANTI ||
            extra->inner_unique)//半连接/反连接或者内部返回唯一值
        {
            /*
             * With a SEMI or ANTI join, or if the innerrel is known unique, the
             * executor will stop after the first match.
         * 执行器在第一次匹配后立即停止.
             */
            Cost        inner_run_cost = workspace->inner_run_cost;
            Cost        inner_rescan_run_cost = workspace->inner_rescan_run_cost;
            double      outer_matched_rows;
            double      outer_unmatched_rows;
            Selectivity inner_scan_frac;
    
            /*
             * For an outer-rel row that has at least one match, we can expect the
             * inner scan to stop after a fraction 1/(match_count+1) of the inner
             * rows, if the matches are evenly distributed.  Since they probably
             * aren't quite evenly distributed, we apply a fuzz factor of 2.0 to
             * that fraction.  (If we used a larger fuzz factor, we'd have to
             * clamp inner_scan_frac to at most 1.0; but since match_count is at
             * least 1, no such clamp is needed now.)
         * 对于至少有一个匹配的外表行，如果匹配是均匀分布的，
         * 那么可以任务内表扫描在内部行的:1/(match_count+1)之后停止。
           * 因为它们可能不是均匀分布的，所以我们对这个分数乘上2.0的系数。
         * (如果我们使用更大的因子，我们将不得不将inner_scan_frac限制为1.0;
         * 但是因为match_count至少为1，所以现在不需要这样的处理了。)
             */
            outer_matched_rows = rint(outer_path_rows * extra->semifactors.outer_match_frac);
            outer_unmatched_rows = outer_path_rows - outer_matched_rows;
            inner_scan_frac = 2.0 / (extra->semifactors.match_count + 1.0);
    
            /*
             * Compute number of tuples processed (not number emitted!).  First,
             * account for successfully-matched outer rows.
         * 计算处理的元组的数量(而不是发出的数量!)
         * 首先，计算成功匹配的外表行。
             */
            ntuples = outer_matched_rows * inner_path_rows * inner_scan_frac;
    
            /*
             * Now we need to estimate the actual costs of scanning the inner
             * relation, which may be quite a bit less than N times inner_run_cost
             * due to early scan stops.  We consider two cases.  If the inner path
             * is an indexscan using all the joinquals as indexquals, then an
             * unmatched outer row results in an indexscan returning no rows,
             * which is probably quite cheap.  Otherwise, the executor will have
             * to scan the whole inner rel for an unmatched row; not so cheap.
         * 现在需要估算扫描内部关系的实际成本，由于先前的扫描停止，这可能会比inner_run_cost小很多。
         * 需要考虑两种情况。如果内表访问路径是使用所有的joinquals连接条件作为indexquals的索引扫描indexscan，
         * 那么一个不匹配的外部行将导致indexscan不返回任何行，这个成本可能相当低。
         * 否则，执行器将必须扫描整个内表以寻找一个不匹配的行;这个成本就非常的高了。
             */
            if (has_indexed_join_quals(path))//连接条件上存在索引
            {
                /*
                 * Successfully-matched outer rows will only require scanning
                 * inner_scan_frac of the inner relation.  In this case, we don't
                 * need to charge the full inner_run_cost even when that's more
                 * than inner_rescan_run_cost, because we can assume that none of
                 * the inner scans ever scan the whole inner relation.  So it's
                 * okay to assume that all the inner scan executions can be
                 * fractions of the full cost, even if materialization is reducing
                 * the rescan cost.  At this writing, it's impossible to get here
                 * for a materialized inner scan, so inner_run_cost and
                 * inner_rescan_run_cost will be the same anyway; but just in
                 * case, use inner_run_cost for the first matched tuple and
                 * inner_rescan_run_cost for additional ones.
           * 成功匹配的外部行只需要扫描内部关系的一部分(比例是inner_scan_frac)。
             * 在这种情况下，不需要增加整个内表运行成本inner_run_cost，即使这比inner_rescan_run_cost还要高，
           * 因为可以假设任何内表扫描都不会扫描整个内表。
             * 因此，可以假设所有内部扫描执行都可以是全部成本的一部分，即使物化降低了重新扫描成本。
           * 在写这篇文章时，不可能在这里进行物化的内部扫描，因此inner_run_cost和inner_rescan_run_cost是相同的;
           * 但是为了以防万一，对第一个匹配的元组使用inner_run_cost，对其他元组使用inner_rescan_run_cost。
                 */
                run_cost += inner_run_cost * inner_scan_frac;
                if (outer_matched_rows > 1)
                    run_cost += (outer_matched_rows - 1) * inner_rescan_run_cost * inner_scan_frac;
    
                /*
                 * Add the cost of inner-scan executions for unmatched outer rows.
                 * We estimate this as the same cost as returning the first tuple
                 * of a nonempty scan.  We consider that these are all rescans,
                 * since we used inner_run_cost once already.
           * 为不匹配的外部行添加内部扫描执行的成本。
           * 这与返回非空扫描的第一个元组的成本相同。
           * 我们认为这些都是rescans，因为已经使用了inner_run_cost一次。
                 */
                run_cost += outer_unmatched_rows *
                    inner_rescan_run_cost / inner_path_rows;
    
                /*
                 * We won't be evaluating any quals at all for unmatched rows, so
                 * don't add them to ntuples.
           * 对于所有不匹配的行,不会解析约束条件,因此不需要增加这些成本.
                 */
            }
            else
            {
                /*
                 * Here, a complicating factor is that rescans may be cheaper than
                 * first scans.  If we never scan all the way to the end of the
                 * inner rel, it might be (depending on the plan type) that we'd
                 * never pay the whole inner first-scan run cost.  However it is
                 * difficult to estimate whether that will happen (and it could
                 * not happen if there are any unmatched outer rows!), so be
                 * conservative and always charge the whole first-scan cost once.
                 * We consider this charge to correspond to the first unmatched
                 * outer row, unless there isn't one in our estimate, in which
                 * case blame it on the first matched row.
           * 在这里，一个复杂的因素是重新扫描可能比第一次扫描成本更低。
           * 如果我们从来没有扫描到内表的末尾，那么(取决于计划类型)可能永远不会耗费整个内表first-scan运行成本。
           * 然而，很难估计这种情况是否会发生(如果存在无法匹配的外部行，这是不可能发生的!)
                 */
    
                /* First, count all unmatched join tuples as being processed */
          //首先,统计所有处理过程中不匹配的行数
                ntuples += outer_unmatched_rows * inner_path_rows;
    
                /* Now add the forced full scan, and decrement appropriate count */
          //现在强制添加全表扫描,并减少相应的计数
                run_cost += inner_run_cost;
                if (outer_unmatched_rows >= 1)
                    outer_unmatched_rows -= 1;
                else
                    outer_matched_rows -= 1;
    
                /* Add inner run cost for additional outer tuples having matches */
          //对于已经匹配的外表行,增加内表运行成本
                if (outer_matched_rows > 0)
                    run_cost += outer_matched_rows * inner_rescan_run_cost * inner_scan_frac;
    
                /* Add inner run cost for additional unmatched outer tuples */
          //对于未匹配的外表行,增加内表运行成本
                if (outer_unmatched_rows > 0)
                    run_cost += outer_unmatched_rows * inner_rescan_run_cost;
            }
        }
        else//普通连接
        {
            /* Normal-case source costs were included in preliminary estimate */
        //正常情况下,源成本已在预估计算过程中统计
            /* Compute number of tuples processed (not number emitted!) */
        //计算处理的元组数量
            ntuples = outer_path_rows * inner_path_rows;
        }
    
        /* CPU成本.CPU costs */
        cost_qual_eval(&restrict_qual_cost, path->joinrestrictinfo, root);
        startup_cost += restrict_qual_cost.startup;
        cpu_per_tuple = cpu_tuple_cost + restrict_qual_cost.per_tuple;
        run_cost += cpu_per_tuple * ntuples;
    
        /* tlist eval costs are paid per output row, not per tuple scanned */
        startup_cost += path->path.pathtarget->cost.startup;
        run_cost += path->path.pathtarget->cost.per_tuple * path->path.rows;
    
        path->path.startup_cost = startup_cost;
        path->path.total_cost = startup_cost + run_cost;
    }
     
    

### 三、跟踪分析

测试脚本如下

    
    
    select a.*,b.grbh,b.je 
    from t_dwxx a,
        lateral (select t1.dwbh,t1.grbh,t2.je 
         from t_grxx t1 
              inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    where a.dwbh = '1001'
    order by b.dwbh;
    
    

启动gdb,设置断点

    
    
    (gdb) b try_nestloop_path
    Breakpoint 1 at 0x7ae950: file joinpath.c, line 373.
    (gdb) c
    Continuing.
    
    Breakpoint 1, try_nestloop_path (root=0x2fb3b30, joinrel=0x2fc6d28, outer_path=0x2fc2540, inner_path=0x2fc1280, 
        pathkeys=0x0, jointype=JOIN_INNER, extra=0x7ffec5f496e0) at joinpath.c:373
    373   RelOptInfo *innerrel = inner_path->parent;
    

进入函数initial_cost_nestloop

    
    
    (gdb) 
    422   initial_cost_nestloop(root, &workspace, jointype,
    (gdb) step
    initial_cost_nestloop (root=0x2fb3b30, workspace=0x7ffec5f49540, jointype=JOIN_INNER, outer_path=0x2fc2540, 
        inner_path=0x2fc1280, extra=0x7ffec5f496e0) at costsize.c:2323
    2323    Cost    startup_cost = 0;
    

进入initial_cost_nestloop->cost_rescan函数

    
    
    (gdb) 
    2332    cost_rescan(root, inner_path,
    (gdb) step
    cost_rescan (root=0x2fb3b30, path=0x2fc1280, rescan_startup_cost=0x7ffec5f494a0, rescan_total_cost=0x7ffec5f49498)
        at costsize.c:3613
    3613    switch (path->pathtype)
    

路径类型为T_SeqScan(在执行该SQL语句前,删除了t_grxx.dwbh上的索引)

    
    
    (gdb) p path->pathtype
    $1 = T_SeqScan
    

进入相应的处理逻辑,直接复制,启动成本&总成本与T_SeqScan一样

    
    
    (gdb) n
    3699        *rescan_startup_cost = path->startup_cost;
    (gdb) n
    3700        *rescan_total_cost = path->total_cost;
    (gdb) 
    3701        break;
    (gdb) 
    

回到initial_cost_nestloop,执行完成,最终结果  
外表存在约束条件dwbh='1001',只有一行,内表在dwbh上没有索引,使用了顺序全表扫描

    
    
    ...
    (gdb) 
    2381    workspace->run_cost = run_cost;
    (gdb) 
    2382  }
    (gdb) p *workspace
    $4 = {startup_cost = 0.28500000000000003, total_cost = 1984.3025, run_cost = 1984.0174999999999, 
      inner_run_cost = 2.4712728827210812e-316, inner_rescan_run_cost = 6.9530954948263344e-310, 
      outer_rows = 3.9937697668447996e-317, inner_rows = 2.4712728827210812e-316, outer_skip_rows = 6.9530954948287059e-310, 
      inner_skip_rows = 6.9443062041807458e-310, numbuckets = 50092024, numbatches = 0, 
      inner_rows_total = 2.4751428001118265e-316}
    

回到try_nestloop_path

    
    
    (gdb) n
    try_nestloop_path (root=0x2fb3b30, joinrel=0x2fc6d28, outer_path=0x2fc2540, inner_path=0x2fc1280, pathkeys=0x0, 
        jointype=JOIN_INNER, extra=0x7ffec5f496e0) at joinpath.c:425
    425   if (add_path_precheck(joinrel,
    

设置断点,进入final_cost_nestloop

    
    
    (gdb) b final_cost_nestloop
    Breakpoint 2 at 0x79f3ff: file costsize.c, line 2397.
    (gdb) c
    Continuing.
    
    Breakpoint 2, final_cost_nestloop (root=0x2fb3b30, path=0x2fc2ef0, workspace=0x7ffec5f49540, extra=0x7ffec5f496e0)
        at costsize.c:2397
    2397    Path     *outer_path = path->outerjoinpath;
    

外表访问路径是索引扫描(t_dwxx),内表访问路径是全表顺序扫描

    
    
    (gdb) p *outer_path
    $6 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2fb3570, pathtarget = 0x2fb8ee0, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.28500000000000003, 
      total_cost = 8.3025000000000002, pathkeys = 0x0}
    

内表行数为10,PG通过统计信息准确计算了该值

    
    
    (gdb) n
    2400    double    inner_path_rows = inner_path->rows;
    (gdb) 
    2401    Cost    startup_cost = workspace->startup_cost;
    (gdb) p inner_path_rows
    $8 = 10
    

计算成本

    
    
    (gdb) 
    2556    cost_qual_eval(&restrict_qual_cost, path->joinrestrictinfo, root);
    (gdb) 
    2557    startup_cost += restrict_qual_cost.startup;
    (gdb) p restrict_qual_cost
    $10 = {startup = 0, per_tuple = 0}
    

最终结果,T_NestPath,总成本total_cost = 1984.4024999999999,启动成本startup_cost =
0.28500000000000003

    
    
    2567  }
    (gdb) p *path
    $11 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x2fc6d28, pathtarget = 0x2fc6f60, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.28500000000000003, 
        total_cost = 1984.4024999999999, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x2fc2540, innerjoinpath = 0x2fc1280, joinrestrictinfo = 0x0}
    

完成调用

    
    
    (gdb) n
    create_nestloop_path (root=0x2fb3b30, joinrel=0x2fc6d28, jointype=JOIN_INNER, workspace=0x7ffec5f49540, 
        extra=0x7ffec5f496e0, outer_path=0x2fc2540, inner_path=0x2fc1280, restrict_clauses=0x0, pathkeys=0x0, 
        required_outer=0x0) at pathnode.c:2229
    2229    return pathnode;
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

