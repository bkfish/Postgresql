本节回过头来Review subquery_planner函数的实现逻辑，该函数对(子)查询进行执行规划。对于查询树中的每个(子)查询(sub-SELECT)，都会递归执行此处理过程。

### 一、源码解读

subquery_planner函数由函数standard_planner调用,生成最终的结果Relation(成本最低),其输出作为生成实际执行计划的输入,在此函数中会调用grouping_planner执行主要的计划过程

    
    
    /*--------------------     * subquery_planner
     *    Invokes the planner on a subquery.  We recurse to here for each
     *    sub-SELECT found in the query tree.
     *    对子查询进行执行规划。对于查询树中的每个子查询(sub-SELECT)，都会递归此处理过程。    
     *
     * glob is the global state for the current planner run.
     * parse is the querytree produced by the parser & rewriter.
     * parent_root is the immediate parent Query's info (NULL at the top level).
     * hasRecursion is true if this is a recursive WITH query.
     * tuple_fraction is the fraction of tuples we expect will be retrieved.
     * tuple_fraction is interpreted as explained for grouping_planner, below.
     * glob-当前计划器运行的全局状态。
     * parse-由解析器和重写器生成的查询树querytree。
     * parent_root是父查询的信息(如为顶层则为空)。
     * hasRecursion-如果这是一个带查询的递归，值为T。
     * tuple_fraction-扫描元组的比例。tuple_fraction在grouping_planner中详细解释。
     *
     * Basically, this routine does the stuff that should only be done once
     * per Query object.  It then calls grouping_planner.  At one time,
     * grouping_planner could be invoked recursively on the same Query object;
     * that's not currently true, but we keep the separation between the two
     * routines anyway, in case we need it again someday.
     * 基本上，这个函数包含完成了每个Query只需要执行一次的任务。
     * 该函数调用grouping_planner一次。在同一个Query上，每次递归grouping_planner都调用一次;
     * 当然，这不是通常的情况，但我们仍然保持这两个例程（subquery_planner和grouping_planner)之间的分离，
     * 以防有一天我们再次需要它。
     * 
     * subquery_planner will be called recursively to handle sub-Query nodes
     * found within the query's expressions and rangetable.
     * 函数subquery_planner将被递归调用，以处理表达式和RTE中的子查询节点。 
     *
     * Returns the PlannerInfo struct ("root") that contains all data generated
     * while planning the subquery.  In particular, the Path(s) attached to
     * the (UPPERREL_FINAL, NULL) upperrel represent our conclusions about the
     * cheapest way(s) to implement the query.  The top level will select the
     * best Path and pass it through createplan.c to produce a finished Plan.
     * 返回PlannerInfo struct(“root”)，它包含在计划子查询时生成的所有数据。
     * 特别地，访问路径附加到(UPPERREL_FINAL, NULL) 上层关系中,以代表优化器已找到查询成本最低的方法.
     * 顶层将选择最佳路径并将其通过createplan.c传递以制定一个已完成的计划。
     *--------------------     */
    /*
    输入:
        glob-PlannerGlobal
        parse-Query结构体指针
        parent_root-父PlannerInfo Root节点
        hasRecursion-是否递归?
        tuple_fraction-扫描Tuple比例
    输出:
        PlannerInfo指针
    */
    PlannerInfo *
    subquery_planner(PlannerGlobal *glob, Query *parse,
                     PlannerInfo *parent_root,
                     bool hasRecursion, double tuple_fraction)
    {
        PlannerInfo *root;//返回值
        List       *newWithCheckOptions;//
        List       *newHaving;//Having子句
        bool        hasOuterJoins;//是否存在Outer Join?
        RelOptInfo *final_rel;//
        ListCell   *l;//临时变量
    
        /* Create a PlannerInfo data structure for this subquery */
        //创建一个规划器数据结构:PlannerInfo
        root = makeNode(PlannerInfo);//构造返回值
        root->parse = parse;
        root->glob = glob;
        root->query_level = parent_root ? parent_root->query_level + 1 : 1;
        root->parent_root = parent_root;
        root->plan_params = NIL;
        root->outer_params = NULL;
        root->planner_cxt = CurrentMemoryContext;
        root->init_plans = NIL;
        root->cte_plan_ids = NIL;
        root->multiexpr_params = NIL;
        root->eq_classes = NIL;
        root->append_rel_list = NIL;
        root->rowMarks = NIL;
        memset(root->upper_rels, 0, sizeof(root->upper_rels));
        memset(root->upper_targets, 0, sizeof(root->upper_targets));
        root->processed_tlist = NIL;
        root->grouping_map = NULL;
        root->minmax_aggs = NIL;
        root->qual_security_level = 0;
        root->inhTargetKind = INHKIND_NONE;
        root->hasRecursion = hasRecursion;
        if (hasRecursion)
            root->wt_param_id = SS_assign_special_param(root);
        else
            root->wt_param_id = -1;
        root->non_recursive_path = NULL;
        root->partColsUpdated = false;
    
        /*
         * If there is a WITH list, process each WITH query and build an initplan
         * SubPlan structure for it.
         * 如果有一个WITH链表，使用查询处理每个链表，并为其构建一个initplan子计划结构。
         */
        if (parse->cteList)
            SS_process_ctes(root);//处理With 语句
    
        /*
         * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
         * to transform them into joins.  Note that this step does not descend
         * into subqueries; if we pull up any subqueries below, their SubLinks are
         * processed just before pulling them up.
         * 查找WHERE和JOIN/ON子句中的ANY/EXISTS子句，并尝试将它们转换为JOIN。
         * 注意，此步骤不会下降为子查询;如果我们上拉子查询，它们的SubLinks将在调出它们上拉前被处理。
         */
        if (parse->hasSubLinks)
            pull_up_sublinks(root); //上拉子链接
    
        /*
         * Scan the rangetable for set-returning functions, and inline them if
         * possible (producing subqueries that might get pulled up next).
         * Recursion issues here are handled in the same way as for SubLinks.
         * 扫描RTE中的set-returning函数，
         * 如果可能，内联它们(生成下一个可能被上拉的子查询)。
         * 这里递归问题的处理方式与SubLinks相同。
         */
        inline_set_returning_functions(root);//
    
        /*
         * Check to see if any subqueries in the jointree can be merged into this
         * query.
         * 检查连接树中的子查询是否可以合并到该查询中(上拉子查询)
         */
        pull_up_subqueries(root);//上拉子查询
    
        /*
         * If this is a simple UNION ALL query, flatten it into an appendrel. We
         * do this now because it requires applying pull_up_subqueries to the leaf
         * queries of the UNION ALL, which weren't touched above because they
         * weren't referenced by the jointree (they will be after we do this).
         * 如果这是一个简单的UNION ALL查询，则将其ftatten为appendrel结构。
         * 我们现在这样做是因为它需要对UNION ALL的叶子查询应用pull_up_subqueries，
         * 上面没有涉及到这些查询，因为它们没有被jointree引用(在我们这样做之后它们将被引用)。
         */
        if (parse->setOperations)
            flatten_simple_union_all(root);//扁平化处理UNION ALL
    
        /*
         * Detect whether any rangetable entries are RTE_JOIN kind; if not, we can
         * avoid the expense of doing flatten_join_alias_vars().  Also check for
         * outer joins --- if none, we can skip reduce_outer_joins().  And check
         * for LATERAL RTEs, too.  This must be done after we have done
         * pull_up_subqueries(), of course.
         * 检测是否有任何RTE中的元素是RTE_JOIN类型;如果没有，可以避免执行refin_join_alias_vars()的开销。
         * 检查外部连接——如果没有，可以跳过reduce_outer_join()函数。同样的,我们会检查LATERAL RTEs。
         * 当然，这必须在我们完成pull_up_subqueries()调用之后完成。
         */
         //判断RTE中是否存在RTE_JOIN?
        root->hasJoinRTEs = false;
        root->hasLateralRTEs = false;
        hasOuterJoins = false;
        foreach(l, parse->rtable)
        {
            RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
    
            if (rte->rtekind == RTE_JOIN)
            {
                root->hasJoinRTEs = true;
                if (IS_OUTER_JOIN(rte->jointype))
                    hasOuterJoins = true;
            }
            if (rte->lateral)
                root->hasLateralRTEs = true;
        }
    
        /*
         * Preprocess RowMark information.  We need to do this after subquery
         * pullup (so that all non-inherited RTEs are present) and before
         * inheritance expansion (so that the info is available for
         * expand_inherited_tables to examine and modify).
         * 预处理RowMark信息。
         * 我们需要在子查询上拉(以便所有非继承的RTEs都存在)和继承展开之后完成
         * (以便expand_inherited_tables可以使用这个信息来检查和修改)。
         */
         //预处理RowMark信息
        preprocess_rowmarks(root);
    
        /*
         * Expand any rangetable entries that are inheritance sets into "append
         * relations".  This can add entries to the rangetable, but they must be
         * plain base relations not joins, so it's OK (and marginally more
         * efficient) to do it after checking for join RTEs.  We must do it after
         * pulling up subqueries, else we'd fail to handle inherited tables in
         * subqueries.
         * 将继承集的任何可范围条目展开为“append relations”。
         * 将相关的relation添加到RTE中，但它们必须是纯基础关系而不是连接，
         * 因此在检查连接RTEs之后执行它是可以的(而且更有效)。
         * 我们必须在启动子查询后执行，否则我们将无法在子查询中处理继承表。
         */
         //展开继承表
        expand_inherited_tables(root);
    
        /*
         * Set hasHavingQual to remember if HAVING clause is present.  Needed
         * because preprocess_expression will reduce a constant-true condition to
         * an empty qual list ... but "HAVING TRUE" is not a semantic no-op.
         * 如果存在HAVING子句，则务必设置hasHavingQual属性。
         * 因为preprocess_expression将把constant-true条件减少为空的条件qual列表…
         * 但是，“HAVING TRUE”并没有语义错误。
         */
         //是否存在Having表达式
        root->hasHavingQual = (parse->havingQual != NULL);
    
        /* Clear this flag; might get set in distribute_qual_to_rels */
        //清除hasPseudoConstantQuals标记,该标记可能在distribute_qual_to_rels函数中设置
        root->hasPseudoConstantQuals = false;
    
        /*
         * Do expression preprocessing on targetlist and quals, as well as other
         * random expressions in the querytree.  Note that we do not need to
         * handle sort/group expressions explicitly, because they are actually
         * part of the targetlist.
         * 对targetlist和quals以及querytree中的其他随机表达式进行表达式预处理。
         * 注意，我们不需要显式地处理sort/group表达式，因为它们实际上是targetlist的一部分。
         */
         //预处理表达式:targetList(投影列)
        parse->targetList = (List *)
            preprocess_expression(root, (Node *) parse->targetList,
                                  EXPRKIND_TARGET);
    
        /* Constant-folding might have removed all set-returning functions */
        //Constant-folding 可能已经把set-returning函数去掉
        if (parse->hasTargetSRFs)
            parse->hasTargetSRFs = expression_returns_set((Node *) parse->targetList);
    
        newWithCheckOptions = NIL;
        foreach(l, parse->withCheckOptions)//witch Check Options
        {
            WithCheckOption *wco = lfirst_node(WithCheckOption, l);
    
            wco->qual = preprocess_expression(root, wco->qual,
                                              EXPRKIND_QUAL);
            if (wco->qual != NULL)
                newWithCheckOptions = lappend(newWithCheckOptions, wco);
        }
        parse->withCheckOptions = newWithCheckOptions;
         //返回列信息returningList
        parse->returningList = (List *)
            preprocess_expression(root, (Node *) parse->returningList,
                                  EXPRKIND_TARGET);
         //预处理条件表达式
        preprocess_qual_conditions(root, (Node *) parse->jointree);
         //预处理Having表达式
        parse->havingQual = preprocess_expression(root, parse->havingQual,
                                                  EXPRKIND_QUAL);
         //窗口函数
        foreach(l, parse->windowClause)
        {
            WindowClause *wc = lfirst_node(WindowClause, l);
    
            /* partitionClause/orderClause are sort/group expressions */
            wc->startOffset = preprocess_expression(root, wc->startOffset,
                                                    EXPRKIND_LIMIT);
            wc->endOffset = preprocess_expression(root, wc->endOffset,
                                                  EXPRKIND_LIMIT);
        }
         //Limit子句
        parse->limitOffset = preprocess_expression(root, parse->limitOffset,
                                                   EXPRKIND_LIMIT);
        parse->limitCount = preprocess_expression(root, parse->limitCount,
                                                  EXPRKIND_LIMIT);
         //On Conflict子句
        if (parse->onConflict)
        {
            parse->onConflict->arbiterElems = (List *)
                preprocess_expression(root,
                                      (Node *) parse->onConflict->arbiterElems,
                                      EXPRKIND_ARBITER_ELEM);
            parse->onConflict->arbiterWhere =
                preprocess_expression(root,
                                      parse->onConflict->arbiterWhere,
                                      EXPRKIND_QUAL);
            parse->onConflict->onConflictSet = (List *)
                preprocess_expression(root,
                                      (Node *) parse->onConflict->onConflictSet,
                                      EXPRKIND_TARGET);
            parse->onConflict->onConflictWhere =
                preprocess_expression(root,
                                      parse->onConflict->onConflictWhere,
                                      EXPRKIND_QUAL);
            /* exclRelTlist contains only Vars, so no preprocessing needed */
        }
         //集合操作(AppendRelInfo)
        root->append_rel_list = (List *)
            preprocess_expression(root, (Node *) root->append_rel_list,
                                  EXPRKIND_APPINFO);
         //RTE
        /* Also need to preprocess expressions within RTEs */
        foreach(l, parse->rtable)
        {
            RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
            int         kind;
            ListCell   *lcsq;
    
            if (rte->rtekind == RTE_RELATION)
            {
                if (rte->tablesample)
                    rte->tablesample = (TableSampleClause *)
                        preprocess_expression(root,
                                              (Node *) rte->tablesample,
                                              EXPRKIND_TABLESAMPLE);//数据表采样语句
            }
            else if (rte->rtekind == RTE_SUBQUERY)//子查询
            {
                /*
                 * We don't want to do all preprocessing yet on the subquery's
                 * expressions, since that will happen when we plan it.  But if it
                 * contains any join aliases of our level, those have to get
                 * expanded now, because planning of the subquery won't do it.
                 * That's only possible if the subquery is LATERAL.
                 * 我们还不想对子查询的表达式进行预处理，因为这将在计划时发生。
                 * 但是，如果它包含当前级别的任何连接别名，那么现在就必须扩展这些别名，
                 * 因为子查询的计划无法做到这一点。只有在子查询是LATERAL的情况下才有可能。
                 */
                if (rte->lateral && root->hasJoinRTEs)
                    rte->subquery = (Query *)
                        flatten_join_alias_vars(root, (Node *) rte->subquery);
            }
            else if (rte->rtekind == RTE_FUNCTION)//函数
            {
                /* Preprocess the function expression(s) fully */
                //预处理函数表达式
                kind = rte->lateral ? EXPRKIND_RTFUNC_LATERAL : EXPRKIND_RTFUNC;
                rte->functions = (List *)
                    preprocess_expression(root, (Node *) rte->functions, kind);
            }
            else if (rte->rtekind == RTE_TABLEFUNC)//TABLE FUNC
            {
                /* Preprocess the function expression(s) fully */
                kind = rte->lateral ? EXPRKIND_TABLEFUNC_LATERAL : EXPRKIND_TABLEFUNC;
                rte->tablefunc = (TableFunc *)
                    preprocess_expression(root, (Node *) rte->tablefunc, kind);
            }
            else if (rte->rtekind == RTE_VALUES)//VALUES子句
            {
                /* Preprocess the values lists fully */
                kind = rte->lateral ? EXPRKIND_VALUES_LATERAL : EXPRKIND_VALUES;
                rte->values_lists = (List *)
                    preprocess_expression(root, (Node *) rte->values_lists, kind);
            }
    
            /*
             * Process each element of the securityQuals list as if it were a
             * separate qual expression (as indeed it is).  We need to do it this
             * way to get proper canonicalization of AND/OR structure.  Note that
             * this converts each element into an implicit-AND sublist.
             * 处理securityQuals列表的每个元素，就好像它是一个单独的qual表达式(事实也是如此)。
             * 之所以这样做，是因为需要获得适当的规范化AND/OR结构。
             * 注意，这将把每个元素转换为隐含的子列表。
             */
            foreach(lcsq, rte->securityQuals)
            {
                lfirst(lcsq) = preprocess_expression(root,
                                                     (Node *) lfirst(lcsq),
                                                     EXPRKIND_QUAL);
            }
        }
    
        /*
         * Now that we are done preprocessing expressions, and in particular done
         * flattening join alias variables, get rid of the joinaliasvars lists.
         * They no longer match what expressions in the rest of the tree look
         * like, because we have not preprocessed expressions in those lists (and
         * do not want to; for example, expanding a SubLink there would result in
         * a useless unreferenced subplan).  Leaving them in place simply creates
         * a hazard for later scans of the tree.  We could try to prevent that by
         * using QTW_IGNORE_JOINALIASES in every tree scan done after this point,
         * but that doesn't sound very reliable.
         * 现在，已经完成了预处理表达式，特别是扁平化连接别名变量，现在可以去掉joinaliasvars链表了。
         * 它们不再匹配树中其他部分中的表达式，因为我们没有在那些链表中预处理表达式
         * (而且是不希望这样做,例如，在那里展开一个SubLink将导致无用的未引用的子计划)。
         * 把它们放在链表中只会给以后扫描树造成问题。
         * 我们可以在这之后的每一次树扫描中使用QTW_IGNORE_JOINALIASES来防止这种情况，虽然这听起来不太可靠。
         */
        if (root->hasJoinRTEs)
        {
            foreach(l, parse->rtable)
            {
                RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
    
                rte->joinaliasvars = NIL;
            }
        }
    
        /*
         * In some cases we may want to transfer a HAVING clause into WHERE. We
         * cannot do so if the HAVING clause contains aggregates (obviously) or
         * volatile functions (since a HAVING clause is supposed to be executed
         * only once per group).  We also can't do this if there are any nonempty
         * grouping sets; moving such a clause into WHERE would potentially change
         * the results, if any referenced column isn't present in all the grouping
         * sets.  (If there are only empty grouping sets, then the HAVING clause
         * must be degenerate as discussed below.)
         * 在某些情况下，我们可能想把“HAVING”条件转移到WHERE子句中。
         * 如果HAVING子句包含聚合(显式的)或易变volatile函数(因为每个GROUP只执行一次HAVING子句)，就不能这样做。
         * 如果有任何非空GROUPING SET，也不能这样做;
         * 如果在所有GROUPING SET中没有出现任何引用列，将这样的子句移动到WHERE可能会改变结果。
         * (如果只有空的GROUP SET分组集，则可以按照下面讨论的那样简化HAVING子句->WHERE中。)
         *
         * Also, it may be that the clause is so expensive to execute that we're
         * better off doing it only once per group, despite the loss of
         * selectivity.  This is hard to estimate short of doing the entire
         * planning process twice, so we use a heuristic: clauses containing
         * subplans are left in HAVING.  Otherwise, we move or copy the HAVING
         * clause into WHERE, in hopes of eliminating tuples before aggregation
         * instead of after.
         * 而且，执行子句的成本非常高，所以最好每组只执行一次，尽管这样会导致选择性selectivity。
         * 如果不把整个规划过程重复一遍，这是很难估计的，因此我们使用启发式的方法:
         * 包含子计划的条款在HAVING的后面。
         * 否则，我们将把HAVING子句移动到WHERE中，希望在聚合之前而不是聚合之后消除元组。
         * 
         * If the query has explicit grouping then we can simply move such a
         * clause into WHERE; any group that fails the clause will not be in the
         * output because none of its tuples will reach the grouping or
         * aggregation stage.  Otherwise we must have a degenerate (variable-free)
         * HAVING clause, which we put in WHERE so that query_planner() can use it
         * in a gating Result node, but also keep in HAVING to ensure that we
         * don't emit a bogus aggregated row. (This could be done better, but it
         * seems not worth optimizing.)
         * 如果查询有显式分组，那么可以简单地将这样的子句移动到WHERE中;
         * 任何失败的GROUP子句都不会出现在输出中，因为它的元组不会到达分组或聚合阶段。
         * 否则，我们必须有一个退化的(无变量的)HAVING子句，把它放在WHERE中，
         * 以便query_planner()可以在一个控制结果节点中使用它，但同时还要确保不会发出一个伪造的聚合行。
         * (这本来可以做得更好，但似乎不值得继续深入优化。)
         *
         * Note that both havingQual and parse->jointree->quals are in
         * implicitly-ANDed-list form at this point, even though they are declared
         * as Node *.
         * 请注意，现在不管是qual还是parse->jointree->quals，即使它们被声明为节点 *，
         * 但它们在这个点上都是都是隐式的链表形式。
         */
        newHaving = NIL;
        foreach(l, (List *) parse->havingQual)
        {
            Node       *havingclause = (Node *) lfirst(l);
    
            if ((parse->groupClause && parse->groupingSets) ||
                contain_agg_clause(havingclause) ||
                contain_volatile_functions(havingclause) ||
                contain_subplans(havingclause))
            {
                /* keep it in HAVING */
                newHaving = lappend(newHaving, havingclause);
            }
            else if (parse->groupClause && !parse->groupingSets)
            {
                /* move it to WHERE */
                parse->jointree->quals = (Node *)
                    lappend((List *) parse->jointree->quals, havingclause);
            }
            else
            {
                /* put a copy in WHERE, keep it in HAVING */
                parse->jointree->quals = (Node *)
                    lappend((List *) parse->jointree->quals,
                            copyObject(havingclause));
                newHaving = lappend(newHaving, havingclause);
            }
        }
        parse->havingQual = (Node *) newHaving;
    
        /* Remove any redundant GROUP BY columns */
        //移除多余的GROUP BY 列
        remove_useless_groupby_columns(root);
    
        /*
         * If we have any outer joins, try to reduce them to plain inner joins.
         * This step is most easily done after we've done expression
         * preprocessing.
         * 如果存在外连接，则尝试将它们转换为普通的内部连接。
         * 在我们完成表达式预处理之后，这个步骤相对容易完成。
         */
        if (hasOuterJoins)
            reduce_outer_joins(root);
    
        /*
         * Do the main planning.  If we have an inherited target relation, that
         * needs special processing, else go straight to grouping_planner.
         * 执行主要的计划过程。
         * 如果存在继承的目标关系，则需要特殊处理，否则直接执行grouping_planner。
         */
        if (parse->resultRelation &&
            rt_fetch(parse->resultRelation, parse->rtable)->inh)
            inheritance_planner(root);
        else
            grouping_planner(root, false, tuple_fraction);
    
        /*
         * Capture the set of outer-level param IDs we have access to, for use in
         * extParam/allParam calculations later.
         * 获取我们可以访问的outer-level的参数IDs,以便稍后在extParam/allParam计算中使用。
         */
        SS_identify_outer_params(root);
    
        /*
         * If any initPlans were created in this query level, adjust the surviving
         * Paths' costs and parallel-safety flags to account for them.  The
         * initPlans won't actually get attached to the plan tree till
         * create_plan() runs, but we must include their effects now.
         * 如果在此查询级别中创建了initplan，则调整现存的访问路径成本和并行安全标志，以反映这些成本。
         * 在create_plan()运行之前，initPlans实际上不会被附加到计划树中，但是我们现在必须包含它们的效果。
         */
        final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
        SS_charge_for_initplans(root, final_rel);
    
        /*
         * Make sure we've identified the cheapest Path for the final rel.  (By
         * doing this here not in grouping_planner, we include initPlan costs in
         * the decision, though it's unlikely that will change anything.)
         * 确保我们已经为最终的关系确定了成本最低的路径
         * (我们没有在grouping_planner中这样做，而是在最终决定中加入了initPlan的成本，尽管这不太可能改变任何事情)。
         */
        set_cheapest(final_rel);
    
        return root;
    }
     
    

### 二、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

