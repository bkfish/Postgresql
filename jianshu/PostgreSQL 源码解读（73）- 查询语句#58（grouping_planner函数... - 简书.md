先前花了二十多个小节介绍query_planner及其子函数make_one_rel，已基本介绍完毕。本节回过头来Review
query_planner函数的调用方-查询优化实现中的grouping_planner函数，该函数执行与与分组/聚集相关的"规划步骤"。

### 一、源码解读

分组/聚集等操作是在一个Relation上叠加分组/聚集运算,grouping_planner函数首先通过query_planner函数生成一个新的关系,然后在此关系上attached分组/聚集等操作。

    
    
    /*--------------------     * grouping_planner
     *    Perform planning steps related to grouping, aggregation, etc.
     *    执行与与分组/聚集相关的"规划步骤".
     *    分组/聚集等操作是在一个Relation上叠加分组/聚集运算,
     *    PG首先通过query_planner函数生成一个新的关系,然后在此关系上attached分组/聚集等操作
     *
     * This function adds all required top-level processing to the scan/join
     * Path(s) produced by query_planner.
     *
     * 该函数还处理了所有需要在顶层处理的扫描/连接路径(通过query_planner函数生成)
     *
     * If inheritance_update is true, we're being called from inheritance_planner
     * and should not include a ModifyTable step in the resulting Path(s).
     * (inheritance_planner will create a single ModifyTable node covering all the
     * target tables.)
     *
     * 如果标志inheritance_update为true,这个函数的调用者是inheritance_planner,在结果路径中
     * 不应包含ModifyTable步骤(inheritance_planner会创建一个单独的覆盖所有目标表的ModifyTable节点).
     *
     * tuple_fraction is the fraction of tuples we expect will be retrieved.
     * tuple_fraction is interpreted as follows:
     *    0: expect all tuples to be retrieved (normal case)
     *    0 < tuple_fraction < 1: expect the given fraction of tuples available
     *      from the plan to be retrieved
     *    tuple_fraction >= 1: tuple_fraction is the absolute number of tuples
     *      expected to be retrieved (ie, a LIMIT specification)
     *
     * tuple_fraction是我们希望搜索的元组比例:
     * 0:正常情况下,期望扫描所有的元组
     * 大于0小于1:按给定的比例扫描
     * 大于等于1:扫描的元组数量(比如通过LIMIT语句指定)
     *
     * Returns nothing; the useful output is in the Paths we attach to the
     * (UPPERREL_FINAL, NULL) upperrel in *root.  In addition,
     * root->processed_tlist contains the final processed targetlist.
     *
     * 该函数没有返回值,有用的输出是root->upperrel->Paths,另外,root->processed_tlist中存储最终的投影列
     *
     * Note that we have not done set_cheapest() on the final rel; it's convenient
     * to leave this to the caller.
     *--------------------     */
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
        //如果存在LIMIT/OFFSET子句,调整tuple_fraction
        if (parse->limitCount || parse->limitOffset)//存在LIMIT/OFFSET语句
        {
            tuple_fraction = preprocess_limit(root, tuple_fraction,
                                              &offset_est, &count_est);//获取元组数量
    
            /*
             * If we have a known LIMIT, and don't have an unknown OFFSET, we can
             * estimate the effects of using a bounded sort.
             * 如果我们有一个已知LIMIT，并且没有未知的OFFSET，我们可以估算使用有界排序的效果。
             */
            if (count_est > 0 && offset_est >= 0)
                limit_tuples = (double) count_est + (double) offset_est;//
        }
    
        /* Make tuple_fraction accessible to lower-level routines */
        //使tuple_fraction可被低级别的处理过程访问(在优化器信息中设置)
        root->tuple_fraction = tuple_fraction;//设置值
    
        if (parse->setOperations)//集合操作,如UNION等
        {
            /*
             * If there's a top-level ORDER BY, assume we have to fetch all the
             * tuples.  This might be too simplistic given all the hackery below
             * to possibly avoid the sort; but the odds of accurate estimates here
             * are pretty low anyway.  XXX try to get rid of this in favor of
             * letting plan_set_operations generate both fast-start and
             * cheapest-total paths.
             * 如果语句的最外层(顶级)存在ORDER BY子句，假设我们必须获取所有元组。
             * 这可能过于简单，但无论如何，准确估计的几率是相当低的。
             * XXX试图摆脱这种情况，让plan_set_operations同时生成快速启动和最便宜的路径。
             */
            if (parse->sortClause)
                root->tuple_fraction = 0.0;//存在排序操作,需扫描所有的元组
    
            /*
             * Construct Paths for set operations.  The results will not need any
             * work except perhaps a top-level sort and/or LIMIT.  Note that any
             * special work for recursive unions is the responsibility of
             * plan_set_operations.
             * 为集合操作构造路径。
             * 除了最外层的SORT/LIMIT操作外不需要作其他操作。
    注意，递归联合的任何特殊工作都是plan_set_operations负责。
             */
            current_rel = plan_set_operations(root);//调用集合操作的"规划"函数
    
            /*
             * We should not need to call preprocess_targetlist, since we must be
             * in a SELECT query node.  Instead, use the targetlist returned by
             * plan_set_operations (since this tells whether it returned any
             * resjunk columns!), and transfer any sort key information from the
             * original tlist.
             * 我们不需要调用preprocess_targetlist函数，因为执行这些操作必须在SELECT查询NODE中。
             * 相反，使用plan_set_operations函数返回的targetlist(因为这告诉它是否返回了所有的resjunk列)，
             * 并从原始投影列链表tlist中传输所有的排序sort键信息。
             */
            Assert(parse->commandType == CMD_SELECT);
    
            tlist = root->processed_tlist;  /* 从plan_set_operations函数的返回结果中获取;from plan_set_operations */
    
            /* for safety, copy processed_tlist instead of modifying in-place */
            //为了安全起见，复制processed_tlist，而不是就地修改
            tlist = postprocess_setop_tlist(copyObject(tlist), parse->targetList);
    
            /* Save aside the final decorated tlist */
            //
            root->processed_tlist = tlist;
    
            /* Also extract the PathTarget form of the setop result tlist */
            //从集合操作结果投影列中获取PathTarget格式的结果列
            final_target = current_rel->cheapest_total_path->pathtarget;
    
            /* And check whether it's parallel safe */
            //检查是否并行安全
            final_target_parallel_safe =
                is_parallel_safe(root, (Node *) final_target->exprs);
    
            /* The setop result tlist couldn't contain any SRFs */
            //集合操作结果投影列不能包含任何的SRFs
            Assert(!parse->hasTargetSRFs);
            final_targets = final_targets_contain_srfs = NIL;
    
            /*
             * Can't handle FOR [KEY] UPDATE/SHARE here (parser should have
             * checked already, but let's make sure).
             * 无法在这里处理[KEY]更新/共享(解析器应该已经检查过了，但需要确认)。
             */
            if (parse->rowMarks)
                ereport(ERROR,
                        (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                /*------                  translator: %s is a SQL row locking clause such as FOR UPDATE */
                         errmsg("%s is not allowed with UNION/INTERSECT/EXCEPT",
                                LCS_asString(linitial_node(RowMarkClause,
                                                           parse->rowMarks)->strength))));
    
            /*
             * Calculate pathkeys that represent result ordering requirements
             * 计算表示结果排序需求的pathkeys
             */
            Assert(parse->distinctClause == NIL);
            root->sort_pathkeys = make_pathkeys_for_sortclauses(root,
                                                                parse->sortClause,
                                                                tlist);
        }
        else//非集合操作
        {
            /* No set operations, do regular planning */
            //没有集合操作,执行常规的规划过程
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
            //递归查询应包含集合操作,检查!
            Assert(!root->hasRecursion);//检查
    
            /* Preprocess grouping sets and GROUP BY clause, if any */
            //预处理grouping sets语句和GROUP BY 子句
            if (parse->groupingSets)//
            {
                gset_data = preprocess_grouping_sets(root);//预处理grouping sets语句
            }
            else
            {
                /* Preprocess regular GROUP BY clause, if any */
                //如处理常规的GROUP BY 子句
                if (parse->groupClause)
                    parse->groupClause = preprocess_groupclause(root, NIL);//处理普通的Group By语句
            }
    
            /* Preprocess targetlist */
            //预处理投影列
            tlist = preprocess_targetlist(root);//处理投影列
    
            /*
             * We are now done hacking up the query's targetlist.  Most of the
             * remaining planning work will be done with the PathTarget
             * representation of tlists, but save aside the full representation so
             * that we can transfer its decoration (resnames etc) to the topmost
             * tlist of the finished Plan.
             * 现在已经完成了对查询语句targetlist的hacking工作。
             * 剩下的大部分规划工作将使用tlists的PathTarget来完成，
             * 但是需要保留完整的信息，这样我们就可以将它的修饰信息(如resname等)转移到完成计划的最顶层tlist中。
             */
            root->processed_tlist = tlist;//赋值
    
            /*
             * Collect statistics about aggregates for estimating costs, and mark
             * all the aggregates with resolved aggtranstypes.  We must do this
             * before slicing and dicing the tlist into various pathtargets, else
             * some copies of the Aggref nodes might escape being marked with the
             * correct transtypes.
             * 收集关于聚集操作的统计数据以估计成本，并在所有聚集操作上标上已解决的aggtranstypes。
             * 必须在将tlist切割成各种PathKeys之前完成这项工作，
             * 否则一些Aggref节点的副本中正确transtypes可能会被替换。
             * 
             * Note: currently, we do not detect duplicate aggregates here.  This
             * may result in somewhat-overestimated cost, which is fine for our
             * purposes since all Paths will get charged the same.  But at some
             * point we might wish to do that detection in the planner, rather
             * than during executor startup.
             * 注意:目前，我们没有检测到重复的聚合。
             * 这可能会导致一些过高估算的成本，这对于我们的目的来说是好的，因为所有的Path都会耗费相同的成本。
             * 但在某些时候，可能希望在计划器中进行检测，而不是在执行器executor启动期间。
             */
            MemSet(&agg_costs, 0, sizeof(AggClauseCosts));
            if (parse->hasAggs)//存在聚合函数
            {
                get_agg_clause_costs(root, (Node *) tlist, AGGSPLIT_SIMPLE,
                                     &agg_costs);//收集用于估算成本的统计信息
                get_agg_clause_costs(root, parse->havingQual, AGGSPLIT_SIMPLE,
                                     &agg_costs);//收集用于估算成本的统计信息
            }
    
            /*
             * Locate any window functions in the tlist.  (We don't need to look
             * anywhere else, since expressions used in ORDER BY will be in there
             * too.)  Note that they could all have been eliminated by constant
             * folding, in which case we don't need to do any more work.
             * 在tlist中找到所有的窗口函数。
             * (我们不需要在其他地方查找，因为ORDER BY中使用的表达式也在那里。)
             * 注意，它们可以通过不断折叠来消除，在这种情况下，我们不需要做更多的工作。
             */
            if (parse->hasWindowFuncs)//窗口函数
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
             * 重新处理MAX/MIN聚集操作，如果有的话。
             * 注意:在这里和query_planner()调用之间添加逻辑时要小心。
             * 在MIN/MAX优化情况下需要的所有东西都必须在plan .c中重复。
             */
            if (parse->hasAggs)//预处理最大最小聚合
                preprocess_minmax_aggregates(root, tlist);
    
            /*
             * Figure out whether there's a hard limit on the number of rows that
             * query_planner's result subplan needs to return.  Even if we know a
             * hard limit overall, it doesn't apply if the query has any
             * grouping/aggregation operations, or SRFs in the tlist.
             * 计算query_planner结果子计划需要返回的行数是否有硬性限制。
             * 即使我们知道总的强制限制，如果查询在tlist中有任何分组/聚合操作或SRFs，它也不适用。
             */
            if (parse->groupClause ||
                parse->groupingSets ||
                parse->distinctClause ||
                parse->hasAggs ||
                parse->hasWindowFuncs ||
                parse->hasTargetSRFs ||
                root->hasHavingQual)//存在Group By/Grouping Set等语句,则limit_tuples设置为-1
                root->limit_tuples = -1.0;
            else
                root->limit_tuples = limit_tuples;//否则,正常赋值
    
            /* Set up data needed by standard_qp_callback */
            //配置standard_qp_callback函数需要的相关数据
            qp_extra.tlist = tlist;//赋值
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
             * 为这个查询的扫描/连接部分(即FROM/WHERE子句表示的处理)生成最好的未排序和预排序路径。
             * (注意，可能没有任何预先设置的路径。)
             * 我们还生成(在standard_qp_callback中)查询语句的sort子句和distinct子句对应的PathKey。
             */
             //为查询中的扫描/连接部分生成最优的未排序/预排序路径(如FROM/WHERE语句表示的处理过程)
            current_rel = query_planner(root, tlist,
                                        standard_qp_callback, &qp_extra);
    
            /*
             * Convert the query's result tlist into PathTarget format.
             * 转换查询结果为PathTarget格式
             *
             * Note: it's desirable to not do this till after query_planner(),
             * because the target width estimates can use per-Var width numbers
             * that were obtained within query_planner().
             * 注意:在query_planner()之后才需要这样做，因为目标列的宽度估算可以使用在query_planner()中获得的每个VAR信息。
             */
            final_target = create_pathtarget(root, tlist);
            final_target_parallel_safe =
                is_parallel_safe(root, (Node *) final_target->exprs);
    
            /*
             * If ORDER BY was given, consider whether we should use a post-sort
             * projection, and compute the adjusted target for preceding steps if
             * so.
             * 如果存在ORDER BY子句，考虑是否使用post-sort投影，如使用则计算前面已调整过的步骤目标列。
             */
            if (parse->sortClause)//存在sort语句?
            {
                sort_input_target = make_sort_input_target(root,
                                                           final_target,
                                                           &have_postponed_srfs);
                sort_input_target_parallel_safe =
                    is_parallel_safe(root, (Node *) sort_input_target->exprs);
            }
            else
            {
                sort_input_target = final_target;//不存在,则直接赋值
                sort_input_target_parallel_safe = final_target_parallel_safe;
            }
    
            /*
             * If we have window functions to deal with, the output from any
             * grouping step needs to be what the window functions want;
             * otherwise, it should be sort_input_target.
             * 如果要处理窗口函数，任何分组步骤的输出都需要满足窗口函数的要求;
             * 否则，它应该是sort_input_target。
             */
            if (activeWindows)//存在窗口函数?
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
             * 如果要进行分组或聚合，最外层的扫描/连接计划节点必须发出分组步骤需要的内容;
             * 否则，它应该设置grouping_target。
             */
            have_grouping = (parse->groupClause || parse->groupingSets ||
                             parse->hasAggs || root->hasHavingQual);
            if (have_grouping)
            {//存在group等分组语句
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
             * 如果targetlist中有任何SRFs，我们必须将这些PathKeys分别划分为SRF-computing和SRF-free 目标列。
             * 用一个没有SRF的版本替换每个指定的目标，并记住后面需要添加的其他投影步骤链表。
             */
            if (parse->hasTargetSRFs)//存在SRFs
            {
                /* final_target doesn't recompute any SRFs in sort_input_target */
                //在sort_input_target中不需要重复计算SRFs
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
                //初始化链表
                final_targets = final_targets_contain_srfs = NIL;
                sort_input_targets = sort_input_targets_contain_srfs = NIL;
                grouping_targets = grouping_targets_contain_srfs = NIL;
                scanjoin_targets = list_make1(scanjoin_target);
                scanjoin_targets_contain_srfs = NIL;
            }
    
            /* Apply scan/join target. */
             //应用扫描/连接target
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
             * 保存刚刚计算的各种upper- >upper_targets[]信息。
             * 核心代码不使用这个功能，但是它为扩展提供了一个方便的地方来获取信息。
             * 为了保持一致性，我们保存了所有的中间目标列，即使这个查询可能不需要一些相应的上层关系。
             */
             //赋值
            root->upper_targets[UPPERREL_FINAL] = final_target;
            root->upper_targets[UPPERREL_WINDOW] = sort_input_target;
            root->upper_targets[UPPERREL_GROUP_AGG] = grouping_target;
    
            /*
             * If we have grouping and/or aggregation, consider ways to implement
             * that.  We build a new upperrel representing the output of this
             * phase.
             * 如果我们有分组和/或聚合，考虑如何实现它。需要构建一个表示此阶段输出的上层关系。
             */
            if (have_grouping)//存在分组操作
            {
                current_rel = create_grouping_paths(root,
                                                    current_rel,
                                                    grouping_target,
                                                    grouping_target_parallel_safe,
                                                    &agg_costs,
                                                    gset_data);//创建分组访问路径
                /* Fix things up if grouping_target contains SRFs */
                if (parse->hasTargetSRFs)
                    adjust_paths_for_srfs(root, current_rel,
                                          grouping_targets,
                                          grouping_targets_contain_srfs);
            }
    
            /*
             * If we have window functions, consider ways to implement those.  We
             * build a new upperrel representing the output of this phase.
             * 如果有窗口函数，考虑如何实现这些函数。
             * 我们建立一个新的上层关系表示这个阶段的输出。
             */
            if (activeWindows)//存在窗口函数
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
             * 如果有一个DISTINCT子句，考虑如何实现它。构建一个表示此阶段输出的上层关系。
             */
            if (parse->distinctClause)//存在distinct?
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
         * 如果指定了ORDER BY，考虑实现它的方法，并生成一个仅包含ORDER和final_target的Path的上层关系。
         * 我们可以在排序成本中应用初始的limit_tuples限制，但前提是没有延迟的SRFs。
         */
        if (parse->sortClause)//存在sort语句?
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
         * 可以构建最终的关系了!
         */
        final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);//获取最终的RelOptInfo(用于替换RTE)
    
        /*
         * If the input rel is marked consider_parallel and there's nothing that's
         * not parallel-safe in the LIMIT clause, then the final_rel can be marked
         * consider_parallel as well.  Note that if the query has rowMarks or is
         * not a SELECT, consider_parallel will be false for every relation in the
         * query.
         * 如果关系被标记为consider_parallel，并且在LIMIT子句中没有任何非并行安全的地方，
         * 那么final_rel也可以被标记为consider_parallel。
         * 请注意，如果查询有rowMarks或不是SELECT语句，则认为对查询中的每个关系consider_parallel都为false。
         */
        if (current_rel->consider_parallel &&
            is_parallel_safe(root, parse->limitOffset) &&
            is_parallel_safe(root, parse->limitCount))
            final_rel->consider_parallel = true;//并行
    
        /*
         * If the current_rel belongs to a single FDW, so does the final_rel.
         * 如current_rel属于某个单独的FDW,设置final_rel信息
         */
        final_rel->serverid = current_rel->serverid;
        final_rel->userid = current_rel->userid;
        final_rel->useridiscurrent = current_rel->useridiscurrent;
        final_rel->fdwroutine = current_rel->fdwroutine;
    
        /*
         * Generate paths for the final_rel.  Insert all surviving paths, with
         * LockRows, Limit, and/or ModifyTable steps added if needed.
         * 为final_rel生成访问路径.
         * 插入所有筛选后的访问路径,包含需添加的LockRows/Limit/ModifyTable步骤
         */
        foreach(lc, current_rel->pathlist)//逐一遍历访问路径
        {
            Path       *path = (Path *) lfirst(lc);
    
            /*
             * If there is a FOR [KEY] UPDATE/SHARE clause, add the LockRows node.
             * (Note: we intentionally test parse->rowMarks not root->rowMarks
             * here.  If there are only non-locking rowmarks, they should be
             * handled by the ModifyTable node instead.  However, root->rowMarks
             * is what goes into the LockRows node.)
             * 如果存在FOR [KEY] UPDATE/SHARE子句，则添加LockRows节点。
             * (注意:我们在这里有意测试的是parse->rowMarks，而不是root->rowMarks。
             * 如果只有非锁定行标记，则应该由ModifyTable节点处理。
             * 但是，root->rowMarks是进入LockRows节点的行标记。
             */
            if (parse->rowMarks)
            {
                path = (Path *) create_lockrows_path(root, final_rel, path,
                                                     root->rowMarks,
                                                     SS_assign_special_param(root));
            }
    
            /*
             * If there is a LIMIT/OFFSET clause, add the LIMIT node.
             * 如果存在LIMIT/OFFSET子句,添加LIMIT节点
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
             * 如为INSERT/UPDATE/DELETE,而且不是从inheritance_planner函数中调用,则添加ModifyTable节点
             */
            if (parse->commandType != CMD_SELECT && !inheritance_update)//非查询语句
            {
                List       *withCheckOptionLists;
                List       *returningLists;
                List       *rowMarks;
    
                /*
                 * Set up the WITH CHECK OPTION and RETURNING lists-of-lists, if
                 * needed.
                 * 如需要,添加WITH CHECK OPTION and RETURNING信息
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
                 * 如果存在FOR [KEY] UPDATE/SHARE子句，那么LockRows节点将处理获取非带锁标记的行，
                 * 否则我们需要使用ModifyTable来完成。
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
            //添加到final_rel中
            add_path(final_rel, path);
        }
    
        /*
         * Generate partial paths for final_rel, too,xxwssssssssssssssssss if outer query levels might
         * be able to make use of them.
         * 并行执行访问路径
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
         * 如查询中存在FDW,添加ForeignPaths
         */
        if (final_rel->fdwroutine &&
            final_rel->fdwroutine->GetForeignUpperPaths)
            final_rel->fdwroutine->GetForeignUpperPaths(root, UPPERREL_FINAL,
                                                        current_rel, final_rel,
                                                        NULL);
    
        /* Let extensions possibly add some more paths */
        //通过扩展添加访问路径
        if (create_upper_paths_hook)
            (*create_upper_paths_hook) (root, UPPERREL_FINAL,
                                        current_rel, final_rel, NULL);
    
        /* Note: currently, we leave it to callers to do set_cheapest() */
        //注意:目前的做法是让调用放来执行set_cheap()函数
    }
    
    

### 二、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

