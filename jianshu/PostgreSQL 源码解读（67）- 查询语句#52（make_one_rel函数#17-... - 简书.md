本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel->sort_inner_and_outer中的initial_cost_mergejoin和final_cost_mergejoin函数，这些函数用于计算merge
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

merge join算法实现的伪代码如下:  
READ data_set_1 SORT BY JOIN KEY TO temp_ds1  
READ data_set_2 SORT BY JOIN KEY TO temp_ds2  
READ ds1_row FROM temp_ds1  
READ ds2_row FROM temp_ds2  
WHILE NOT eof ON temp_ds1,temp_ds2 LOOP  
IF ( temp_ds1.key = temp_ds2.key ) OUTPUT JOIN ds1_row,ds2_row  
ELSIF ( temp_ds1.key <= temp_ds2.key ) READ ds1_row FROM temp_ds1  
ELSIF ( temp_ds1.key => temp_ds2.key ) READ ds2_row FROM temp_ds2  
END LOOP

**try_mergejoin_path- >initial_cost_mergejoin**  
该函数用于实现merge join路径成本的初步估计。

    
    
    //----------------------- try_mergejoin_path->initial_cost_mergejoin
     /*
      * initial_cost_mergejoin
      *    Preliminary estimate of the cost of a mergejoin path.
      *    merge join路径成本的初步估计。
      *
      * This must quickly produce lower-bound estimates of the path's startup and
      * total costs.  If we are unable to eliminate the proposed path from
      * consideration using the lower bounds, final_cost_mergejoin will be called
      * to obtain the final estimates.
      * 必须快速的对path启动和总成本做出较低的估算。
      * 如果无法使用下界消除尝试对建议的路径进行比较，那么将调用final_cost_mergejoin来获得最终估计。
      *  
      * The exact division of labor between this function and final_cost_mergejoin
      * is private to them, and represents a tradeoff between speed of the initial
      * estimate and getting a tight lower bound.  We choose to not examine the
      * join quals here, except for obtaining the scan selectivity estimate which
      * is really essential (but fortunately, use of caching keeps the cost of
      * getting that down to something reasonable).
      * We also assume that cost_sort is cheap enough to use here.
      * 这个函数和final_cost_mergejoin之间存在确切的划分,两者是没有相关的，
      * 希望做到的是在初始估计的性能和得到严格的下限之间取得权衡。
      * 在这里,不检查join quals，除非获得非常重要的选择性估计值
      * (幸运的是，使用缓存可以将成本降至合理的水平)。
      * 我们会假定cost_sort参数设置得足够的低.
      *
      * 'workspace' is to be filled with startup_cost, total_cost, and perhaps
      *      other data to be used by final_cost_mergejoin
      * workspace-返回值,用于函数final_cost_mergejoin
      * 'jointype' is the type of join to be performed
      * jointype-连接类型
      * 'mergeclauses' is the list of joinclauses to be used as merge clauses
      * mergeclauses-作为合并条件的连接条件链表
      * 'outer_path' is the outer input to the join
      * 'inner_path' is the inner input to the join
      * 'outersortkeys' is the list of sort keys for the outer path
      * 'innersortkeys' is the list of sort keys for the inner path
      * 'extra' contains miscellaneous information about the join
      *
      * Note: outersortkeys and innersortkeys should be NIL if no explicit
      * sort is needed because the respective source path is already ordered.
      */
     void
     initial_cost_mergejoin(PlannerInfo *root, JoinCostWorkspace *workspace,
                            JoinType jointype,
                            List *mergeclauses,
                            Path *outer_path, Path *inner_path,
                            List *outersortkeys, List *innersortkeys,
                            JoinPathExtraData *extra)
     {
         Cost        startup_cost = 0;
         Cost        run_cost = 0;
         double      outer_path_rows = outer_path->rows;
         double      inner_path_rows = inner_path->rows;
         Cost        inner_run_cost;
         double      outer_rows,
                     inner_rows,
                     outer_skip_rows,
                     inner_skip_rows;
         Selectivity outerstartsel,
                     outerendsel,
                     innerstartsel,
                     innerendsel;
         Path        sort_path;      /* dummy for result of cost_sort */
     
         /* 确保行数大于0,并且不是NaN.Protect some assumptions below that rowcounts aren't zero or NaN */
         if (outer_path_rows <= 0 || isnan(outer_path_rows))
             outer_path_rows = 1;
         if (inner_path_rows <= 0 || isnan(inner_path_rows))
             inner_path_rows = 1;
     
         /*
          * A merge join will stop as soon as it exhausts either input stream
          * (unless it's an outer join, in which case the outer side has to be
          * scanned all the way anyway).  Estimate fraction of the left and right
          * inputs that will actually need to be scanned.  Likewise, we can
          * estimate the number of rows that will be skipped before the first join
          * pair is found, which should be factored into startup cost. We use only
          * the first (most significant) merge clause for this purpose. Since
          * mergejoinscansel() is a fairly expensive computation, we cache the
          * results in the merge clause RestrictInfo.
          * merge join将在耗尽其中任意一个输入流后立即停止
          * (除非它是一个外部连接，在这种情况下无论如何都必须对外部进行扫描)。
        * 估计需要扫描的左右输入的部分。
        * 同样，可以估计在找到第一个连接对之前要跳过的行数，这应该考虑到启动成本。
        * 为此，考虑到mergejoinscansel()是一个相当昂贵的计算，只使用第一个(最重要的)merge子句,
        * 我们将结果缓存到merge子句的RestrictInfo中。
          */
         if (mergeclauses && jointype != JOIN_FULL)//非全外连接
         {
             RestrictInfo *firstclause = (RestrictInfo *) linitial(mergeclauses);
             List       *opathkeys;
             List       *ipathkeys;
             PathKey    *opathkey;
             PathKey    *ipathkey;
             MergeScanSelCache *cache;
     
             /* Get the input pathkeys to determine the sort-order details */
             opathkeys = outersortkeys ? outersortkeys : outer_path->pathkeys;
             ipathkeys = innersortkeys ? innersortkeys : inner_path->pathkeys;
             Assert(opathkeys);
             Assert(ipathkeys);
             opathkey = (PathKey *) linitial(opathkeys);
             ipathkey = (PathKey *) linitial(ipathkeys);
             /* debugging check */
             if (opathkey->pk_opfamily != ipathkey->pk_opfamily ||
                 opathkey->pk_eclass->ec_collation != ipathkey->pk_eclass->ec_collation ||
                 opathkey->pk_strategy != ipathkey->pk_strategy ||
                 opathkey->pk_nulls_first != ipathkey->pk_nulls_first)
                 elog(ERROR, "left and right pathkeys do not match in mergejoin");
     
             /* Get the selectivity with caching */
             cache = cached_scansel(root, firstclause, opathkey);
     
             if (bms_is_subset(firstclause->left_relids,
                               outer_path->parent->relids))
             {
                 /* 子句左侧为outer.left side of clause is outer */
                 outerstartsel = cache->leftstartsel;
                 outerendsel = cache->leftendsel;
                 innerstartsel = cache->rightstartsel;
                 innerendsel = cache->rightendsel;
             }
             else
             {
                 /* 子句左侧为innerer.left side of clause is inner */
                 outerstartsel = cache->rightstartsel;
                 outerendsel = cache->rightendsel;
                 innerstartsel = cache->leftstartsel;
                 innerendsel = cache->leftendsel;
             }
             if (jointype == JOIN_LEFT ||
                 jointype == JOIN_ANTI)
             {
                 outerstartsel = 0.0;
                 outerendsel = 1.0;
             }
             else if (jointype == JOIN_RIGHT)
             {
                 innerstartsel = 0.0;
                 innerendsel = 1.0;
             }
         }
         else
         {
             /* 无条件或者全外连接.cope with clauseless or full mergejoin */
             outerstartsel = innerstartsel = 0.0;
             outerendsel = innerendsel = 1.0;
         }
     
         /*
          * Convert selectivities to row counts.  We force outer_rows and
          * inner_rows to be at least 1, but the skip_rows estimates can be zero.
          * 转换选择率为行数.
          * outer_rows和inner_rows至少为1,但skip_rows可以为0.
          */
         outer_skip_rows = rint(outer_path_rows * outerstartsel);
         inner_skip_rows = rint(inner_path_rows * innerstartsel);
         outer_rows = clamp_row_est(outer_path_rows * outerendsel);
         inner_rows = clamp_row_est(inner_path_rows * innerendsel);
     
         Assert(outer_skip_rows <= outer_rows);
         Assert(inner_skip_rows <= inner_rows);
     
         /*
          * Readjust scan selectivities to account for above rounding.  This is
          * normally an insignificant effect, but when there are only a few rows in
          * the inputs, failing to do this makes for a large percentage error.
          * 重新调整扫描选择性以考虑四舍五入。
          * 通常来说,这是一个无关紧要的事情，但是当输入中只有几行时，如果不这样做，就会造成很大的误差。
          */
         outerstartsel = outer_skip_rows / outer_path_rows;
         innerstartsel = inner_skip_rows / inner_path_rows;
         outerendsel = outer_rows / outer_path_rows;
         innerendsel = inner_rows / inner_path_rows;
     
         Assert(outerstartsel <= outerendsel);
         Assert(innerstartsel <= innerendsel);
     
         /* cost of source data */
     
         if (outersortkeys)          /* do we need to sort outer? */
         {
             cost_sort(&sort_path,
                       root,
                       outersortkeys,
                       outer_path->total_cost,
                       outer_path_rows,
                       outer_path->pathtarget->width,
                       0.0,
                       work_mem,
                       -1.0);//计算outer relation的排序成本
             startup_cost += sort_path.startup_cost;
             startup_cost += (sort_path.total_cost - sort_path.startup_cost)
                 * outerstartsel;
             run_cost += (sort_path.total_cost - sort_path.startup_cost)
                 * (outerendsel - outerstartsel);
         }
         else
         {
             startup_cost += outer_path->startup_cost;
             startup_cost += (outer_path->total_cost - outer_path->startup_cost)
                 * outerstartsel;
             run_cost += (outer_path->total_cost - outer_path->startup_cost)
                 * (outerendsel - outerstartsel);
         }
     
         if (innersortkeys)          /* do we need to sort inner? */
         {
             cost_sort(&sort_path,
                       root,
                       innersortkeys,
                       inner_path->total_cost,
                       inner_path_rows,
                       inner_path->pathtarget->width,
                       0.0,
                       work_mem,
                       -1.0);//计算inner relation的排序成本
             startup_cost += sort_path.startup_cost;
             startup_cost += (sort_path.total_cost - sort_path.startup_cost)
                 * innerstartsel;
             inner_run_cost = (sort_path.total_cost - sort_path.startup_cost)
                 * (innerendsel - innerstartsel);
         }
         else
         {
             startup_cost += inner_path->startup_cost;
             startup_cost += (inner_path->total_cost - inner_path->startup_cost)
                 * innerstartsel;
             inner_run_cost = (inner_path->total_cost - inner_path->startup_cost)
                 * (innerendsel - innerstartsel);
         }
     
         /*
          * We can't yet determine whether rescanning occurs, or whether
          * materialization of the inner input should be done.  The minimum
          * possible inner input cost, regardless of rescan and materialization
          * considerations, is inner_run_cost.  We include that in
          * workspace->total_cost, but not yet in run_cost.
          * 现在还不能确定是否会重新扫描，或者是否应该物化inner relation。
        * 不管是否重新扫描和是否尝试物化，最小的inner relation input成本是inner_run_cost。
        * 将其包含在workspace—>total_cost中，但不包含在run_cost中。
          */
     
         /* CPU costs left for later */
     
         /* Public result fields */
         workspace->startup_cost = startup_cost;
         workspace->total_cost = startup_cost + run_cost + inner_run_cost;
         /* Save private data for final_cost_mergejoin */
         workspace->run_cost = run_cost;
         workspace->inner_run_cost = inner_run_cost;
         workspace->outer_rows = outer_rows;
         workspace->inner_rows = inner_rows;
         workspace->outer_skip_rows = outer_skip_rows;
         workspace->inner_skip_rows = inner_skip_rows;
     } 
    
    

**try_mergejoin_path- >add_path_precheck**  
该函数检查新路径是否可能被接受.

    
    
    //----------------------- try_mergejoin_path->add_path_precheck
     /*
      * add_path_precheck
      *    Check whether a proposed new path could possibly get accepted.
      *    We assume we know the path's pathkeys and parameterization accurately,
      *    and have lower bounds for its costs.
      *    检查新路径是否可能被接受。假设已经准确知道路径的排序键和参数化，并且已知其成本的下限。
      *
      * Note that we do not know the path's rowcount, since getting an estimate for
      * that is too expensive to do before prechecking.  We assume here that paths
      * of a superset parameterization will generate fewer rows; if that holds,
      * then paths with different parameterizations cannot dominate each other
      * and so we can simply ignore existing paths of another parameterization.
      * (In the infrequent cases where that rule of thumb fails, add_path will
      * get rid of the inferior path.)
      * 注意:我们不知道访问路径的行数，因为在进行预检查之前获取它的估计值是非常昂贵的。
      * 只能假设超集参数化的路径会产生更少的行;
      *  如果该假设成立，那么具有不同参数化的路径不能互相控制，因此可以简单地忽略另一个参数化的现有路径。
      * (在这种经验法则失效的情况下-虽然这种情况很少发生-add_path函数将删除相对较差的路径。)
      *
      * At the time this is called, we haven't actually built a Path structure,
      * so the required information has to be passed piecemeal.
      * 在调用这个函数时，实际上还没有构建一个Path结构体，因此需要的信息必须逐段传递。
      */
     bool
     add_path_precheck(RelOptInfo *parent_rel,
                       Cost startup_cost, Cost total_cost,
                       List *pathkeys, Relids required_outer)
     {
         List       *new_path_pathkeys;
         bool        consider_startup;
         ListCell   *p1;
     
         /* Pretend parameterized paths have no pathkeys, per add_path policy */
         new_path_pathkeys = required_outer ? NIL : pathkeys;
     
         /* Decide whether new path's startup cost is interesting */
         consider_startup = required_outer ? parent_rel->consider_param_startup : parent_rel->consider_startup;
     
         foreach(p1, parent_rel->pathlist)
         {
             Path       *old_path = (Path *) lfirst(p1);
             PathKeysComparison keyscmp;
     
             /*
              * We are looking for an old_path with the same parameterization (and
              * by assumption the same rowcount) that dominates the new path on
              * pathkeys as well as both cost metrics.  If we find one, we can
              * reject the new path.
          * 寻找一个old_path，它具有相同的参数化(并假设有相同的行数)，
          * 在PathKey上主导新的路径以及两个成本指标。如果找到一个，就可以丢弃新的路径。
              *
              * Cost comparisons here should match compare_path_costs_fuzzily.
          * 这里的成本比较应该与compare_path_costs_fuzzily匹配。
              */
             if (total_cost > old_path->total_cost * STD_FUZZ_FACTOR)
             {
                 /* new path can win on startup cost only if consider_startup */
                 if (startup_cost > old_path->startup_cost * STD_FUZZ_FACTOR ||
                     !consider_startup)
                 {
                     /* 新路径成本较高,检查排序键.new path loses on cost, so check pathkeys... */
                     List       *old_path_pathkeys;
     
                     old_path_pathkeys = old_path->param_info ? NIL : old_path->pathkeys;
                     keyscmp = compare_pathkeys(new_path_pathkeys,
                                                old_path_pathkeys);
                     if (keyscmp == PATHKEYS_EQUAL ||
                         keyscmp == PATHKEYS_BETTER2)
                     {
                         /* 排序键也不占优,丢弃之.new path does not win on pathkeys... */
                         if (bms_equal(required_outer, PATH_REQ_OUTER(old_path)))
                         {
                             /* Found an old path that dominates the new one */
                             return false;
                         }
                     }
                 }
             }
             else
             {
                 /*
                  * Since the pathlist is sorted by total_cost, we can stop looking
                  * once we reach a path with a total_cost larger than the new
                  * path's.
            * 由于路径链表是按total_cost排序的，所以当发现一个total_cost大于新路径成本的路径时，停止查找。
                  */
                 break;
             }
         }
     
         return true;
     } 
    
    

**create_mergejoin_path- >final_cost_mergejoin**  
该函数确定mergejoin path最终的成本和大小.

    
    
    //-------------- create_mergejoin_path->final_cost_mergejoin
    
     /*
      * final_cost_mergejoin
      *    Final estimate of the cost and result size of a mergejoin path.
      *    确定mergejoin path最终的成本和大小
      *
      * Unlike other costsize functions, this routine makes two actual decisions:
      * whether the executor will need to do mark/restore, and whether we should
      * materialize the inner path.  It would be logically cleaner to build
      * separate paths testing these alternatives, but that would require repeating
      * most of the cost calculations, which are not all that cheap.  Since the
      * choice will not affect output pathkeys or startup cost, only total cost,
      * there is no possibility of wanting to keep more than one path.  So it seems
      * best to make the decisions here and record them in the path's
      * skip_mark_restore and materialize_inner fields.
      * 与其他costsize函数不同，该函数做出了两个实际决策:执行器是否需要执行mark/restore，以及是否应该物化内部访问路径。
      * 在逻辑上，构建单独的路径来测试这些替代方案会更简洁，但这需要重复大部分影响性能的成本计算过程。
      * 由于选择率不会影响输出PathKey或启动成本，只会影响总成本，因此不可能保留多个路径。
      * 因此，最好在这里做出决定，并将其记录在path的skip_mark_restore和materialize_inner字段中。
      *
      * Mark/restore overhead is usually required, but can be skipped if we know
      * that the executor need find only one match per outer tuple, and that the
      * mergeclauses are sufficient to identify a match.
      * 通常需要mark/restore的开销，但是如果知道执行器只需要为每个外部元组找到一个匹配，
      * 并且mergeclauses足以识别匹配，那么可以忽略这个开销。
      *
      * We materialize the inner path if we need mark/restore and either the inner
      * path can't support mark/restore, or it's cheaper to use an interposed
      * Material node to handle mark/restore.
      * 如果需要mark/restore，或者内部路径不支持mark/restore，
      * 或者使用插入的物化节点来处理mark/restore，优化器就会物化inner relation的访问路径。
      *
      * 'path' is already filled in except for the rows and cost fields and
      *      skip_mark_restore and materialize_inner
      * path-MergePath节点
      * 'workspace' is the result from initial_cost_mergejoin
      * workspace-initial_cost_mergejoin函数返回的结果
      * 'extra' contains miscellaneous information about the join
      * extra-额外的信息
      */
     void
     final_cost_mergejoin(PlannerInfo *root, MergePath *path,
                          JoinCostWorkspace *workspace,
                          JoinPathExtraData *extra)
     {
         Path       *outer_path = path->jpath.outerjoinpath;
         Path       *inner_path = path->jpath.innerjoinpath;
         double      inner_path_rows = inner_path->rows;
         List       *mergeclauses = path->path_mergeclauses;
         List       *innersortkeys = path->innersortkeys;
         Cost        startup_cost = workspace->startup_cost;
         Cost        run_cost = workspace->run_cost;
         Cost        inner_run_cost = workspace->inner_run_cost;
         double      outer_rows = workspace->outer_rows;
         double      inner_rows = workspace->inner_rows;
         double      outer_skip_rows = workspace->outer_skip_rows;
         double      inner_skip_rows = workspace->inner_skip_rows;
         Cost        cpu_per_tuple,
                     bare_inner_cost,
                     mat_inner_cost;
         QualCost    merge_qual_cost;
         QualCost    qp_qual_cost;
         double      mergejointuples,
                     rescannedtuples;
         double      rescanratio;
     
         /* 确保行数合法.Protect some assumptions below that rowcounts aren't zero or NaN */
         if (inner_path_rows <= 0 || isnan(inner_path_rows))
             inner_path_rows = 1;
     
         /* 标记访问路径的行数.Mark the path with the correct row estimate */
         if (path->jpath.path.param_info)
             path->jpath.path.rows = path->jpath.path.param_info->ppi_rows;
         else
             path->jpath.path.rows = path->jpath.path.parent->rows;
     
         /* 并行执行,调整行数.For partial paths, scale row estimate. */
         if (path->jpath.path.parallel_workers > 0)
         {
             double      parallel_divisor = get_parallel_divisor(&path->jpath.path);
     
             path->jpath.path.rows =
                 clamp_row_est(path->jpath.path.rows / parallel_divisor);
         }
     
         /*
          * We could include disable_cost in the preliminary estimate, but that
          * would amount to optimizing for the case where the join method is
          * disabled, which doesn't seem like the way to bet.
        * 不允许merge join则设置为高成本
          */
         if (!enable_mergejoin)
             startup_cost += disable_cost;
     
         /*
          * Compute cost of the mergequals and qpquals (other restriction clauses)
          * separately.
        * 分别计算mergequals和qpquals的成本
          */
         cost_qual_eval(&merge_qual_cost, mergeclauses, root);
         cost_qual_eval(&qp_qual_cost, path->jpath.joinrestrictinfo, root);
         qp_qual_cost.startup -= merge_qual_cost.startup;
         qp_qual_cost.per_tuple -= merge_qual_cost.per_tuple;
     
         /*
          * With a SEMI or ANTI join, or if the innerrel is known unique, the
          * executor will stop scanning for matches after the first match.  When
          * all the joinclauses are merge clauses, this means we don't ever need to
          * back up the merge, and so we can skip mark/restore overhead.
        * 使用半连接或反连接，或者inner relation返回的结果唯一，
        * 执行程序将在第一次匹配后停止扫描匹配。
        * 当所有的joinclauses都是merge子句时，意味着不需要备份merge，这样就可以跳过mark/restore步骤。
          */
         if ((path->jpath.jointype == JOIN_SEMI ||
              path->jpath.jointype == JOIN_ANTI ||
              extra->inner_unique) &&
             (list_length(path->jpath.joinrestrictinfo) ==
              list_length(path->path_mergeclauses)))
             path->skip_mark_restore = true;
         else
             path->skip_mark_restore = false;
     
         /*
          * Get approx # tuples passing the mergequals.  We use approx_tuple_count
          * here because we need an estimate done with JOIN_INNER semantics.
        * 通过mergequals条件获得大约的元组数。
          * 这里使用approx_tuple_count，因为需要使用JOIN_INNER语义进行评估。
          */
         mergejointuples = approx_tuple_count(root, &path->jpath, mergeclauses);
     
         /*
          * When there are equal merge keys in the outer relation, the mergejoin
          * must rescan any matching tuples in the inner relation. This means
          * re-fetching inner tuples; we have to estimate how often that happens.
          * 当外部关系中有相等的merge键时，merge join必须重新扫描内部关系中的所有匹配元组。
        * 这意味着需要重新获取内部元组;必须估计这种情况发生的频率。
        *
          * For regular inner and outer joins, the number of re-fetches can be
          * estimated approximately as size of merge join output minus size of
          * inner relation. Assume that the distinct key values are 1, 2, ..., and
          * denote the number of values of each key in the outer relation as m1,
          * m2, ...; in the inner relation, n1, n2, ...  Then we have
          *
          * size of join = m1 * n1 + m2 * n2 + ...
          *
          * number of rescanned tuples = (m1 - 1) * n1 + (m2 - 1) * n2 + ... = m1 *
          * n1 + m2 * n2 + ... - (n1 + n2 + ...) = size of join - size of inner
          * relation
          *
          * 对于常规的内连接和外连接，重新获取的次数可以近似估计为合并连接输出的大小减去内部关系的大小。
        * 假设唯一键值是1, 2…，表示外部关系中每个键的值个数为m1, m2，…;在内部关系中，n1, n2，…
        * 那么会得到:
        * 连接的大小 = m1 * n1 + m2 * n2 + ...
        * 重新扫描的元组数 = (m1 - 1) * n1 + (m2 - 1) * n2 + ... = m1 * n1 + m2 * n2 + ... - (n1 + n2 + ...)
          *                  = size of join - size of inner relation
        *
          * This equation works correctly for outer tuples having no inner match
          * (nk = 0), but not for inner tuples having no outer match (mk = 0); we
          * are effectively subtracting those from the number of rescanned tuples,
          * when we should not.  Can we do better without expensive selectivity
          * computations?
          * 对于没有内部匹配(nk = 0)的外部元组，这个方程是正确的，
        * 但是对于没有外部匹配(mk = 0)的内部元组，这个方程就不正确了;
        * 不这样做时，实际上是在从重新扫描元组的数量中减去这些元组。
        * 如果没有昂贵的选择性计算，能做得更好吗?
          *
          * The whole issue is moot if we are working from a unique-ified outer
          * input, or if we know we don't need to mark/restore at all.
        * 如果使用的是唯一的外部输入，或者知道根本不需要mark/restore，那么探讨该问题是没有意义的。
          */
         if (IsA(outer_path, UniquePath) ||path->skip_mark_restore)
             rescannedtuples = 0;
         else
         {
             rescannedtuples = mergejointuples - inner_path_rows;
             /* Must clamp because of possible underestimate */
             if (rescannedtuples < 0)
                 rescannedtuples = 0;
         }
         /* 对于重复扫描,需要额外增加成本.We'll inflate various costs this much to account for rescanning */
         rescanratio = 1.0 + (rescannedtuples / inner_path_rows);
     
         /*
          * Decide whether we want to materialize the inner input to shield it from
          * mark/restore and performing re-fetches.  Our cost model for regular
          * re-fetches is that a re-fetch costs the same as an original fetch,
          * which is probably an overestimate; but on the other hand we ignore the
          * bookkeeping costs of mark/restore.  Not clear if it's worth developing
          * a more refined model.  So we just need to inflate the inner run cost by
          * rescanratio.
        * 决定是否要物化inner relation的访问路径，以使其不受mark/restore和执行重新获取操作的影响。
        * 对于定期重取的成本模型是，一次重取的成本与一次原始取回的成本相同，当然这可能高估了该成本;
        * 但另一方面，忽略了mark/restore的簿记成本。
        * 目前尚不清楚是否值得开发一个更完善的模型。所以只需要把内部运行成本乘上相应的比例。
          */
         bare_inner_cost = inner_run_cost * rescanratio;
     
         /*
          * When we interpose a Material node the re-fetch cost is assumed to be
          * just cpu_operator_cost per tuple, independently of the underlying
          * plan's cost; and we charge an extra cpu_operator_cost per original
          * fetch as well.  Note that we're assuming the materialize node will
          * never spill to disk, since it only has to remember tuples back to the
          * last mark.  (If there are a huge number of duplicates, our other cost
          * factors will make the path so expensive that it probably won't get
          * chosen anyway.)  So we don't use cost_rescan here.
          * 当插入一个物化节点时，重新取回的成本被假设为cpu_operator_cost/每个元组，该成本独立于底层计划的成本;
        * 对每次原始取回增加额外的cpu_operator_cost。
        * 注意，我们假设物化节点永远不会溢出到磁盘上，因为它只需要记住元组的最后一个标记。
        * (如果有大量的重复项，其他成本因素会使路径变得非常昂贵，可能无论如何都不会被选中。)
        * 因此在这里,我们不调用cost_rescan。
        * 
          * Note: keep this estimate in sync with create_mergejoin_plan's labeling
          * of the generated Material node.
        * 注意:与create_mergejoin_plan函数生成的物化节点标记保持一致。
          */
         mat_inner_cost = inner_run_cost +
             cpu_operator_cost * inner_path_rows * rescanratio;
     
         /*
          * If we don't need mark/restore at all, we don't need materialization.
        * 如果不需要mark/restore,那么也不需要物化
          */
         if (path->skip_mark_restore)
             path->materialize_inner = false;
     
         /*
          * Prefer materializing if it looks cheaper, unless the user has asked to
          * suppress materialization.
        * 如果物化看起来成本更低，那么就选择物化，前提是用户允许物化。
          */
         else if (enable_material && mat_inner_cost < bare_inner_cost)
             path->materialize_inner = true;
     
         /*
          * Even if materializing doesn't look cheaper, we *must* do it if the
          * inner path is to be used directly (without sorting) and it doesn't
          * support mark/restore.
        * 即使物化看起来成本不低，
        * 但如果inner relation的访问路径是直接使用(没有排序)并且不支持mark/restore，我们也必须这样做。
          *
          * Since the inner side must be ordered, and only Sorts and IndexScans can
          * create order to begin with, and they both support mark/restore, you
          * might think there's no problem --- but you'd be wrong.  Nestloop and
          * merge joins can *preserve* the order of their inputs, so they can be
          * selected as the input of a mergejoin, and they don't support
          * mark/restore at present.
          * 因为inner端必须是有序的，并且只有Sorts和IndexScan才能从一开始就创建排序，
        * 而且它们都支持mark/restore，所以可能认为这样处理没有问题——但是错了。
        * Nestloop join和merge join可以“保留”它们的输入顺序，
        * 因此它们可以被选择为merge join的输入，而且它们目前不支持mark/restore。
        * 
          * We don't test the value of enable_material here, because
          * materialization is required for correctness in this case, and turning
          * it off does not entitle us to deliver an invalid plan.
        * 在这里不会理会enable_material的设置，因为在这种情况下，优先保证正确性，
        * 而关闭它会让优化器产生无效的执行计划。
          */
         else if (innersortkeys == NIL &&
                  !ExecSupportsMarkRestore(inner_path))
             path->materialize_inner = true;
     
         /*
          * Also, force materializing if the inner path is to be sorted and the
          * sort is expected to spill to disk.  This is because the final merge
          * pass can be done on-the-fly if it doesn't have to support mark/restore.
          * We don't try to adjust the cost estimates for this consideration,
          * though.
          * 另外，如果要对inner relation访问路径进行排序，且排序可能会溢出到磁盘，则强制实现。
        * 这是因为如果不需要支持mark/restore，最终的merge过程可以在运行中完成。
        * 不过，我们并没有试图为此调整成本估算。
        * 
          * Since materialization is a performance optimization in this case,
          * rather than necessary for correctness, we skip it if enable_material is
          * off.
        * 由于在这种情况下，物化处理是一种性能优化，而不是保证正确的必要条件，
        * 所以如果enable_material关闭了，那么就忽略它。
          */
         else if (enable_material && innersortkeys != NIL &&
                  relation_byte_size(inner_path_rows,
                                     inner_path->pathtarget->width) >
                  (work_mem * 1024L))
             path->materialize_inner = true;
         else
             path->materialize_inner = false;
     
         /* 调整运行期成本.Charge the right incremental cost for the chosen case */
         if (path->materialize_inner)
             run_cost += mat_inner_cost;
         else
             run_cost += bare_inner_cost;
     
         /* CPU 成本计算.CPU costs */
     
         /*
          * The number of tuple comparisons needed is approximately number of outer
          * rows plus number of inner rows plus number of rescanned tuples (can we
          * refine this?).  At each one, we need to evaluate the mergejoin quals.
        * 需要比较的元组数量大约是外部行数加上内部行数加上重扫描元组数(可以改进它吗?)
        * 在每一个点上，需要计算合并后的数量。
          */
         startup_cost += merge_qual_cost.startup;
         startup_cost += merge_qual_cost.per_tuple *
             (outer_skip_rows + inner_skip_rows * rescanratio);
         run_cost += merge_qual_cost.per_tuple *
             ((outer_rows - outer_skip_rows) +
              (inner_rows - inner_skip_rows) * rescanratio);
     
         /*
          * For each tuple that gets through the mergejoin proper, we charge
          * cpu_tuple_cost plus the cost of evaluating additional restriction
          * clauses that are to be applied at the join.  (This is pessimistic since
          * not all of the quals may get evaluated at each tuple.)
          * 对于每个通过合并连接得到的元组，每一个元组的成本是cpu_tuple_cost，加上将在连接上应用的附加限制条件的成本。
        * (当然这是悲观的做法，因为并不是所有的条件都能在每个元组上得到应用。)
        * 
          * Note: we could adjust for SEMI/ANTI joins skipping some qual
          * evaluations here, but it's probably not worth the trouble.
        * 注意:我们可以对半/反连接进行调整，跳过一些条件评估，但这可能并不值得。
          */
         startup_cost += qp_qual_cost.startup;
         cpu_per_tuple = cpu_tuple_cost + qp_qual_cost.per_tuple;
         run_cost += cpu_per_tuple * mergejointuples;
     
         /* 投影列的估算按输出行计算,而不是扫描的元组.tlist eval costs are paid per output row, not per tuple scanned */
         startup_cost += path->jpath.path.pathtarget->cost.startup;
         run_cost += path->jpath.path.pathtarget->cost.per_tuple * path->jpath.path.rows;
     
         path->jpath.path.startup_cost = startup_cost;
         path->jpath.path.total_cost = startup_cost + run_cost;
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

    
    
    (gdb) b try_mergejoin_path
    Breakpoint 1 at 0x7aeeaf: file joinpath.c, line 572.
    (gdb) c
    Continuing.
    
    Breakpoint 1, try_mergejoin_path (root=0x166c880, joinrel=0x16864d0, outer_path=0x167f190, inner_path=0x167f9d0, pathkeys=0x1
        innersortkeys=0x1686c28, jointype=JOIN_INNER, extra=0x7ffea604f500, is_partial=false) at joinpath.c:572
    572   if (is_partial)
    

进入initial_cost_mergejoin函数

    
    
    (gdb) 
    615   initial_cost_mergejoin(root, &workspace, jointype, mergeclauses,
    (gdb) step
    initial_cost_mergejoin (root=0x166c880, workspace=0x7ffea604f360, jointype=JOIN_INNER, mergeclauses=0x1686bc8, outer_path=0x167f190, inner_path=0x167f9d0, outersortkeys=0x1686b68, 
        innersortkeys=0x1686c28, extra=0x7ffea604f500) at costsize.c:2607
    2607    Cost    startup_cost = 0; 
    

初始化参数

    
    
    2607    Cost    startup_cost = 0;
    (gdb) n
    2608    Cost    run_cost = 0;
    ... 
    

存在merge条件,JOIN_INNER连接,进入相应的分支

    
    
    ...
    2639    if (mergeclauses && jointype != JOIN_FULL)
    (gdb) 
    2641      RestrictInfo *firstclause = (RestrictInfo *) linitial(mergeclauses);
    ... 
    

查看约束条件,即t_dwxx.dwbh = t_grxx.dwbh

    
    
    (gdb) set $arg1=(RelabelType *)((OpExpr *)firstclause->clause)->args->head->data.ptr_value
    (gdb) set $arg2=(RelabelType *)((OpExpr *)firstclause->clause)->args->head->next->data.ptr_value
    (gdb) p *(Var *)$arg1->arg
    $9 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 24, varcollid = 100, varlevelsup = 0, varnoold = 1, varoattno = 2, location = 218}
    (gdb) p *(Var *)$arg2->arg
    $10 = {xpr = {type = T_Var}, varno = 3, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, varnoold = 3, varoattno = 1, location = 208}
    

获取连接的outer和inner relation的排序键

    
    
    ...
    (gdb) 
    2649      opathkeys = outersortkeys ? outersortkeys : outer_path->pathkeys;
    (gdb) n
    2650      ipathkeys = innersortkeys ? innersortkeys : inner_path->pathkeys; 
    ...
    

获取缓存的选择率

    
    
    ...
    (gdb) 
    2663      cache = cached_scansel(root, firstclause, opathkey);
    (gdb) 
    2666                outer_path->parent->relids))
    (gdb) p *cache
    $15 = {opfamily = 1994, collation = 100, strategy = 1, nulls_first = false, leftstartsel = 0, leftendsel = 0.99989010989010996, rightstartsel = 0.0091075159436798652, rightendsel = 1}
    

选择率赋值,连接子句左侧为outer relation

    
    
    2665      if (bms_is_subset(firstclause->left_relids,
    (gdb) 
    2669        outerstartsel = cache->leftstartsel;
    (gdb) 
    2670        outerendsel = cache->leftendsel;
    (gdb) 
    2671        innerstartsel = cache->rightstartsel;
    (gdb) 
    2672        innerendsel = cache->rightendsel; 
    

把选择率转换为行数

    
    
    (gdb) 
    2705    outer_skip_rows = rint(outer_path_rows * outerstartsel);
    (gdb) 
    2706    inner_skip_rows = rint(inner_path_rows * innerstartsel);
    (gdb) 
    2707    outer_rows = clamp_row_est(outer_path_rows * outerendsel);
    (gdb) 
    2708    inner_rows = clamp_row_est(inner_path_rows * innerendsel);
    (gdb) p outer_skip_rows
    $16 = 0
    (gdb) p inner_skip_rows
    $17 = 911
    (gdb) p outer_rows
    $18 = 9999
    (gdb) p inner_rows
    $19 = 100000 
    

计算outer relation的排序成本并赋值

    
    
    ...
    (gdb) 
    2728    if (outersortkeys)      /* do we need to sort outer? */
    (gdb) 
    2730      cost_sort(&sort_path, 
    (gdb) n
    2735            outer_path->pathtarget->width,  
    ...
    (gdb) n
    2739      startup_cost += sort_path.startup_cost;
    (gdb) p sort_path
    $24 = {type = T_Invalid, pathtype = T_Invalid, parent = 0x167f9d0, pathtarget = 0x0, param_info = 0x0, parallel_aware = false, parallel_safe = false, parallel_workers = 0, rows = 10000, 
      startup_cost = 828.38561897747286, total_cost = 853.38561897747286, pathkeys = 0x167f9d0}
    

计算inner relation的排序成本并赋值

    
    
    ...
    2754    if (innersortkeys)      /* do we need to sort inner? */
    (gdb) 
    2756      cost_sort(&sort_path, 
    ...
    (gdb) p sort_path
    $25 = {type = T_Invalid, pathtype = T_Invalid, parent = 0x167f9d0, pathtarget = 0x0, param_info = 0x0, parallel_aware = false, parallel_safe = false, parallel_workers = 0, rows = 100000, 
      startup_cost = 10030.82023721841, total_cost = 10280.82023721841, pathkeys = 0x167f9d0}  
    

赋值给workspace,结束initial_cost_mergejoin函数调用.

    
    
    (gdb) 
    2791    workspace->startup_cost = startup_cost;
    (gdb) 
    2792    workspace->total_cost = startup_cost + run_cost + inner_run_cost;
    (gdb) 
    2794    workspace->run_cost = run_cost;
    (gdb) 
    2795    workspace->inner_run_cost = inner_run_cost;
    (gdb) 
    2796    workspace->outer_rows = outer_rows;
    (gdb) 
    2797    workspace->inner_rows = inner_rows;
    (gdb) 
    2798    workspace->outer_skip_rows = outer_skip_rows;
    (gdb) 
    2799    workspace->inner_skip_rows = inner_skip_rows;
    (gdb) 
    2800  } 
    

进入add_path_precheck函数

    
    
    (gdb) n
    try_mergejoin_path (root=0x166c880, joinrel=0x16864d0, outer_path=0x167f190, inner_path=0x167f9d0, pathkeys=0x1686b68, mergeclauses=0x1686bc8, outersortkeys=0x1686b68, innersortkeys=0x1686c28, 
        jointype=JOIN_INNER, extra=0x7ffea604f500, is_partial=false) at joinpath.c:620
    620   if (add_path_precheck(joinrel,
    (gdb) step
    add_path_precheck (parent_rel=0x16864d0, startup_cost=10861.483356195882, total_cost=11134.203356195881, pathkeys=0x1686b68, required_outer=0x0) at pathnode.c:666
    666   new_path_pathkeys = required_outer ? NIL : pathkeys; 
    

parent_rel->pathlist为NULL,结束调用,返回true

    
    
    671   foreach(p1, parent_rel->pathlist)
    (gdb) p *parent_rel->pathlist
    Cannot access memory at address 0x0
    (gdb) n
    719   return true; 
    

进入final_cost_mergejoin函数

    
    
    (gdb) b final_cost_mergejoin
    Breakpoint 2 at 0x7a00c9: file costsize.c, line 2834.
    (gdb) c
    Continuing.
    
    Breakpoint 2, final_cost_mergejoin (root=0x166c880, path=0x1686cb8, workspace=0x7ffea604f360, extra=0x7ffea604f500) at costsize.c:2834
    2834    Path     *outer_path = path->jpath.outerjoinpath; 
    

查看outer relation和inner relation的访问路径

    
    
    2834    Path     *outer_path = path->jpath.outerjoinpath;
    (gdb) n
    2835    Path     *inner_path = path->jpath.innerjoinpath;
    (gdb) 
    2836    double    inner_path_rows = inner_path->rows;
    (gdb) p *outer_path
    $26 = {type = T_Path, pathtype = T_SeqScan, parent = 0x166c2c0, pathtarget = 0x1671670, param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10000, 
      startup_cost = 0, total_cost = 164, pathkeys = 0x0}
    (gdb) p *inner_path
    $27 = {type = T_Path, pathtype = T_SeqScan, parent = 0x166c4d8, pathtarget = 0x1673510, param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
      startup_cost = 0, total_cost = 1726, pathkeys = 0x0} 
    

其他参数的赋值

    
    
    (gdb) n
    2837    List     *mergeclauses = path->path_mergeclauses;
    (gdb) 
    2838    List     *innersortkeys = path->innersortkeys;
    ... 
    

分别计算mergequals和qpquals的成本

    
    
    ...
    (gdb) 
    2886    cost_qual_eval(&merge_qual_cost, mergeclauses, root);
    (gdb) 
    2887    cost_qual_eval(&qp_qual_cost, path->jpath.joinrestrictinfo, root);
    (gdb) 
    2888    qp_qual_cost.startup -= merge_qual_cost.startup;
    (gdb) p merge_qual_cost
    $28 = {startup = 0, per_tuple = 0.0025000000000000001}
    (gdb) p qp_qual_cost
    $29 = {startup = 0, per_tuple = 0.0025000000000000001} 
    

通过mergequals条件获得大约的元组数

    
    
    ...
    (gdb) 
    2910    mergejointuples = approx_tuple_count(root, &path->jpath, mergeclauses);
    (gdb) 
    2938    if (IsA(outer_path, UniquePath) ||path->skip_mark_restore)
    (gdb) p mergejointuples
    $30 = 100000 
    

计算相同元组导致的重复扫描率

    
    
    (gdb) n
    2942      rescannedtuples = mergejointuples - inner_path_rows;
    (gdb) 
    2944      if (rescannedtuples < 0)
    (gdb) 
    2948    rescanratio = 1.0 + (rescannedtuples / inner_path_rows); 
    (gdb) p rescanratio
    $31 = 1
    (gdb) p rescannedtuples
    $32 = 0
    

物化处理,结果是inner relation扫描无需物化:path->materialize_inner = false;

    
    
    (gdb) 
    2974    mat_inner_cost = inner_run_cost +
    (gdb) n
    2980    if (path->skip_mark_restore)
    (gdb) 
    2987    else if (enable_material && mat_inner_cost < bare_inner_cost)
    (gdb) 
    3006    else if (innersortkeys == NIL &&
    (gdb) 
    3021    else if (enable_material && innersortkeys != NIL &&
    (gdb) 
    3023                  inner_path->pathtarget->width) >
    (gdb) p mat_inner_cost
    $34 = 497.72250000000003
    ...
    3021    else if (enable_material && innersortkeys != NIL &&
    (gdb) 
    3027      path->materialize_inner = false; 
    

计算成本

    
    
    (gdb) n
    3043    startup_cost += merge_qual_cost.per_tuple *
    (gdb) 
    3044      (outer_skip_rows + inner_skip_rows * rescanratio);
    (gdb) 
    3043    startup_cost += merge_qual_cost.per_tuple *
    (gdb) 
    3045    run_cost += merge_qual_cost.per_tuple *
    (gdb) 
    3046      ((outer_rows - outer_skip_rows) +
    (gdb) 
    3047       (inner_rows - inner_skip_rows) * rescanratio);
    (gdb) 
    3046      ((outer_rows - outer_skip_rows) +
    (gdb) 
    3045    run_cost += merge_qual_cost.per_tuple *
    (gdb) 
    3058    startup_cost += qp_qual_cost.startup;
    (gdb) 
    3059    cpu_per_tuple = cpu_tuple_cost + qp_qual_cost.per_tuple;
    (gdb) 
    3060    run_cost += cpu_per_tuple * mergejointuples;
    (gdb) 
    3063    startup_cost += path->jpath.path.pathtarget->cost.startup;
    (gdb) 
    3064    run_cost += path->jpath.path.pathtarget->cost.per_tuple * path->jpath.path.rows;
    (gdb) 
    3066    path->jpath.path.startup_cost = startup_cost;
    (gdb) 
    3067    path->jpath.path.total_cost = startup_cost + run_cost;
    (gdb) 
    3068  } 
    

完成调用

    
    
    (gdb) 
    create_mergejoin_path (root=0x166c880, joinrel=0x16864d0, jointype=JOIN_INNER, workspace=0x7ffea604f360, extra=0x7ffea604f500, outer_path=0x167f190, inner_path=0x167f9d0, restrict_clauses=0x16869a8, 
        pathkeys=0x1686b68, required_outer=0x0, mergeclauses=0x1686bc8, outersortkeys=0x1686b68, innersortkeys=0x1686c28) at pathnode.c:2298
    2298    return pathnode; 
    

MergePath节点信息

    
    
    2298    return pathnode;
    (gdb) p *pathnode
    $36 = {jpath = {path = {type = T_MergePath, pathtype = T_MergeJoin, parent = 0x16864d0, pathtarget = 0x1686708, param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, 
          rows = 100000, startup_cost = 10863.760856195882, total_cost = 12409.200856195883, pathkeys = 0x1686b68}, jointype = JOIN_INNER, inner_unique = false, outerjoinpath = 0x167f190, 
        innerjoinpath = 0x167f9d0, joinrestrictinfo = 0x16869a8}, path_mergeclauses = 0x1686bc8, outersortkeys = 0x1686b68, innersortkeys = 0x1686c28, skip_mark_restore = false, 
      materialize_inner = false} 
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

