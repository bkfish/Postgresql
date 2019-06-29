在主函数subquery_planner完成外连接消除后,接下来调用grouping_planner函数,本节简单介绍了此函数的主体逻辑。

### 一、源码解读

grouping_planner函数源码：

    
    
    /*--------------------      * grouping_planner
      *    Perform planning steps related to grouping, aggregation, etc.
      *
      * 生成与分组/聚合相关的"规划步骤"
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
         if (parse->limitCount || parse->limitOffset)//存在LIMIT/OFFSET语句
         {
             tuple_fraction = preprocess_limit(root, tuple_fraction,
                                               &offset_est, &count_est);//获取元组数量
     
             /*
              * If we have a known LIMIT, and don't have an unknown OFFSET, we can
              * estimate the effects of using a bounded sort.
              */
             if (count_est > 0 && offset_est >= 0)
                 limit_tuples = (double) count_est + (double) offset_est;//
         }
     
         /* Make tuple_fraction accessible to lower-level routines */
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
              */
             if (parse->sortClause)
                 root->tuple_fraction = 0.0;//存在排序操作,需扫描所有的元组
     
             /*
              * Construct Paths for set operations.  The results will not need any
              * work except perhaps a top-level sort and/or LIMIT.  Note that any
              * special work for recursive unions is the responsibility of
              * plan_set_operations.
              */
             current_rel = plan_set_operations(root);//调用集合操作的"规划"函数
     
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
         else//非集合操作
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
             Assert(!root->hasRecursion);//检查
     
             /* Preprocess grouping sets and GROUP BY clause, if any */
             if (parse->groupingSets)//
             {
                 gset_data = preprocess_grouping_sets(root);//预处理grouping sets语句
             }
             else
             {
                 /* Preprocess regular GROUP BY clause, if any */
                 if (parse->groupClause)
                     parse->groupClause = preprocess_groupclause(root, NIL);//处理普通的Group By语句
             }
     
             /* Preprocess targetlist */
             tlist = preprocess_targetlist(root);//处理投影列
     
             /*
              * We are now done hacking up the query's targetlist.  Most of the
              * remaining planning work will be done with the PathTarget
              * representation of tlists, but save aside the full representation so
              * that we can transfer its decoration (resnames etc) to the topmost
              * tlist of the finished Plan.
              */
             root->processed_tlist = tlist;//赋值
     
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
              */
             if (parse->hasAggs)//预处理最大最小聚合
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
                 root->hasHavingQual)//存在Group By/Grouping Set等语句,则limit_tuples设置为-1
                 root->limit_tuples = -1.0;
             else
                 root->limit_tuples = limit_tuples;//否则,正常赋值
     
             /* Set up data needed by standard_qp_callback */
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
              */
             final_target = create_pathtarget(root, tlist);
             final_target_parallel_safe =
                 is_parallel_safe(root, (Node *) final_target->exprs);
     
             /*
              * If ORDER BY was given, consider whether we should use a post-sort
              * projection, and compute the adjusted target for preceding steps if
              * so.
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
              */
             if (parse->hasTargetSRFs)//存在SRFs
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
              */
         //赋值
             root->upper_targets[UPPERREL_FINAL] = final_target;
             root->upper_targets[UPPERREL_WINDOW] = sort_input_target;
             root->upper_targets[UPPERREL_GROUP_AGG] = grouping_target;
     
             /*
              * If we have grouping and/or aggregation, consider ways to implement
              * that.  We build a new upperrel representing the output of this
              * phase.
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
          */
         final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);//获取最终的RelOptInfo(用于替换RTE)
     
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
             final_rel->consider_parallel = true;//并行
     
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
         foreach(lc, current_rel->pathlist)//逐一遍历访问路径
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
             if (parse->commandType != CMD_SELECT && !inheritance_update)//非查询语句
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
          * Generate partial paths for final_rel, too,if outer query levels might
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
    
    

### 二、参考资料

[planner.c](https://doxygen.postgresql.org/planner_8c_source.html)

