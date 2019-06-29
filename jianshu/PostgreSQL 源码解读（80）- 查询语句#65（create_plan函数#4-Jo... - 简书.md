本节介绍了创建计划create_plan函数中连接(join)计划的实现过程，主要的逻辑在函数create_join_plan中实现。

### 一、数据结构

**Plan**  
所有计划节点通过将Plan结构作为第一个字段从Plan结构“派生”。这确保了在将节点转换为计划节点时能正常工作。(在执行器中以通用方式传递时，节点指针经常被转换为Plan
*)

    
    
    /* ----------------     *      Plan node
     *
     * All plan nodes "derive" from the Plan structure by having the
     * Plan structure as the first field.  This ensures that everything works
     * when nodes are cast to Plan's.  (node pointers are frequently cast to Plan*
     * when passed around generically in the executor)
     * 所有计划节点通过将Plan结构作为第一个字段从Plan结构“派生”。
     * 这确保了在将节点转换为计划节点时，一切都能正常工作。
     * (在执行器中以通用方式传递时，节点指针经常被转换为Plan *)
     *
     * We never actually instantiate any Plan nodes; this is just the common
     * abstract superclass for all Plan-type nodes.
     * 从未实例化任何Plan节点;这只是所有Plan-type节点的通用抽象超类。
     * ----------------     */
    typedef struct Plan
    {
        NodeTag     type;//节点类型
    
        /*
         * 成本估算信息;estimated execution costs for plan (see costsize.c for more info)
         */
        Cost        startup_cost;   /* 启动成本;cost expended before fetching any tuples */
        Cost        total_cost;     /* 总成本;total cost (assuming all tuples fetched) */
    
        /*
         * 优化器估算信息;planner's estimate of result size of this plan step
         */
        double      plan_rows;      /* 行数;number of rows plan is expected to emit */
        int         plan_width;     /* 平均行大小(Byte为单位);average row width in bytes */
    
        /*
         * 并行执行相关的信息;information needed for parallel query
         */
        bool        parallel_aware; /* 是否参与并行执行逻辑?engage parallel-aware logic? */
        bool        parallel_safe;  /* 是否并行安全;OK to use as part of parallel plan? */
    
        /*
         * Plan类型节点通用的信息.Common structural data for all Plan types.
         */
        int         plan_node_id;   /* unique across entire final plan tree */
        List       *targetlist;     /* target list to be computed at this node */
        List       *qual;           /* implicitly-ANDed qual conditions */
        struct Plan *lefttree;      /* input plan tree(s) */
        struct Plan *righttree;
        List       *initPlan;       /* Init Plan nodes (un-correlated expr
                                     * subselects) */
    
        /*
         * Information for management of parameter-change-driven rescanning
         * parameter-change-driven重扫描的管理信息.
         * 
         * extParam includes the paramIDs of all external PARAM_EXEC params
         * affecting this plan node or its children.  setParam params from the
         * node's initPlans are not included, but their extParams are.
         *
         * allParam includes all the extParam paramIDs, plus the IDs of local
         * params that affect the node (i.e., the setParams of its initplans).
         * These are _all_ the PARAM_EXEC params that affect this node.
         */
        Bitmapset  *extParam;
        Bitmapset  *allParam;
    } Plan;
    
    

### 二、源码解读

create_join_plan函数创建Join Plan节点.Join可以分为Merge Join/Hash Join/NestLoop
Join三种,相应的实现函数是create_nestloop_plan/create_mergejoin_plan/create_hashjoin_plan.

    
    
    //------------------------------------------------------------------ create_join_plan
    /*
     * create_join_plan
     *    Create a join plan for 'best_path' and (recursively) plans for its
     *    inner and outer paths.
     *    创建连接计划Plan节点.
     */
    static Plan *
    create_join_plan(PlannerInfo *root, JoinPath *best_path)
    {
        Plan       *plan;
        List       *gating_clauses;
    
        switch (best_path->path.pathtype)
        {
            case T_MergeJoin://Merge Join
                plan = (Plan *) create_mergejoin_plan(root,
                                                      (MergePath *) best_path);
                break;
            case T_HashJoin://Hash Join
                plan = (Plan *) create_hashjoin_plan(root,
                                                     (HashPath *) best_path);
                break;
            case T_NestLoop://NestLoop Join
                plan = (Plan *) create_nestloop_plan(root,
                                                     (NestPath *) best_path);
                break;
            default://目前仅支持上述三种
                elog(ERROR, "unrecognized node type: %d",
                     (int) best_path->path.pathtype);
                plan = NULL;        /* keep compiler quiet */
                break;
        }
    
        /*
         * If there are any pseudoconstant clauses attached to this node, insert a
         * gating Result node that evaluates the pseudoconstants as one-time
         * quals.
         * 如果这个节点上附加了伪常量子句，插入一个gating Result节点，该节点将伪常量计算为一次性条件quals。
         */
        gating_clauses = get_gating_quals(root, best_path->joinrestrictinfo);
        if (gating_clauses)
            plan = create_gating_plan(root, (Path *) best_path, plan,
                                      gating_clauses);
    
    #ifdef NOT_USED
    
        /*
         * * Expensive function pullups may have pulled local predicates * into
         * this path node.  Put them in the qpqual of the plan node. * JMH,
         * 6/15/92
         * pullups函数可能已经把本地谓词上拉到该访问路径节点中,把这些信息放在Plan节点的qpqual中
         */
        if (get_loc_restrictinfo(best_path) != NIL)
            set_qpqual((Plan) plan,
                       list_concat(get_qpqual((Plan) plan),
                                   get_actual_clauses(get_loc_restrictinfo(best_path))));
    #endif
    
        return plan;
    }
    
    //------------------------------------------ create_nestloop_plan
    
    static NestLoop *
    create_nestloop_plan(PlannerInfo *root,
                         NestPath *best_path)
    {
        NestLoop   *join_plan;
        Plan       *outer_plan;
        Plan       *inner_plan;
        List       *tlist = build_path_tlist(root, &best_path->path);
        List       *joinrestrictclauses = best_path->joinrestrictinfo;
        List       *joinclauses;
        List       *otherclauses;
        Relids      outerrelids;
        List       *nestParams;
        Relids      saveOuterRels = root->curOuterRels;
        ListCell   *cell;
        ListCell   *prev;
        ListCell   *next;
    
        /* NestLoop can project, so no need to be picky about child tlists */
        //NestLoop可以执行投影操作,所以不需要关心子计划的tlists
        //递归调用生成外表计划
        outer_plan = create_plan_recurse(root, best_path->outerjoinpath, 0);
    
        /* For a nestloop, include outer relids in curOuterRels for inner side */
        //对于nestloop,对应内侧的curOuterRels中需要包含外表的relids
        root->curOuterRels = bms_union(root->curOuterRels,
                                       best_path->outerjoinpath->parent->relids);
        //递归调用生成内表计划
        inner_plan = create_plan_recurse(root, best_path->innerjoinpath, 0);
    
        /* Restore curOuterRels */
        //恢复curOuterRels
        bms_free(root->curOuterRels);
        root->curOuterRels = saveOuterRels;
    
        /* Sort join qual clauses into best execution order */
        //排序连接条件
        joinrestrictclauses = order_qual_clauses(root, joinrestrictclauses);
    
        /* Get the join qual clauses (in plain expression form) */
        /* Any pseudoconstant clauses are ignored here */
        //获取连接条件子句,在这里,会忽略伪常量
        if (IS_OUTER_JOIN(best_path->jointype))
        {
            extract_actual_join_clauses(joinrestrictclauses,
                                        best_path->path.parent->relids,
                                        &joinclauses, &otherclauses);//外连接
        }
        else
        {
            /* We can treat all clauses alike for an inner join */
            //内连接
            joinclauses = extract_actual_clauses(joinrestrictclauses, false);
            otherclauses = NIL;
        }
    
        /* Replace any outer-relation variables with nestloop params */
        //使用nestloop参数替代外表变量
        if (best_path->path.param_info)
        {
            joinclauses = (List *)
                replace_nestloop_params(root, (Node *) joinclauses);
            otherclauses = (List *)
                replace_nestloop_params(root, (Node *) otherclauses);
        }
    
        /*
         * Identify any nestloop parameters that should be supplied by this join
         * node, and move them from root->curOuterParams to the nestParams list.
         * 确定这个连接节点应该提供的所有nestloop连接参数，
         * 并将它们从root->curOuterParams移动到nestParams链表中。
         */
        outerrelids = best_path->outerjoinpath->parent->relids;
        nestParams = NIL;
        prev = NULL;
        for (cell = list_head(root->curOuterParams); cell; cell = next)//遍历curOuterParams
        {
            NestLoopParam *nlp = (NestLoopParam *) lfirst(cell);//获取参数
    
            next = lnext(cell);
            if (IsA(nlp->paramval, Var) &&
                bms_is_member(nlp->paramval->varno, outerrelids))//Var变量,而且是外层的relids
            {
                root->curOuterParams = list_delete_cell(root->curOuterParams,
                                                        cell, prev);
                nestParams = lappend(nestParams, nlp);
            }
            else if (IsA(nlp->paramval, PlaceHolderVar) &&//PHV
                     bms_overlap(((PlaceHolderVar *) nlp->paramval)->phrels,
                                 outerrelids) &&
                     bms_is_subset(find_placeholder_info(root,
                                                         (PlaceHolderVar *) nlp->paramval,
                                                         false)->ph_eval_at,
                                   outerrelids))
            {
                root->curOuterParams = list_delete_cell(root->curOuterParams,
                                                        cell, prev);
                nestParams = lappend(nestParams, nlp);
            }
            else
                prev = cell;//直接赋值
        }
    
        join_plan = make_nestloop(tlist,
                                  joinclauses,
                                  otherclauses,
                                  nestParams,
                                  outer_plan,
                                  inner_plan,
                                  best_path->jointype,
                                  best_path->inner_unique);//构造nestloop访问节点
    
        copy_generic_path_info(&join_plan->join.plan, &best_path->path);
    
        return join_plan;
    }
    
    //------------------------------------------ create_mergejoin_plan
    
    static MergeJoin *
    create_mergejoin_plan(PlannerInfo *root,
                          MergePath *best_path)
    {
        MergeJoin  *join_plan;
        Plan       *outer_plan;
        Plan       *inner_plan;
        List       *tlist = build_path_tlist(root, &best_path->jpath.path);
        List       *joinclauses;
        List       *otherclauses;
        List       *mergeclauses;
        List       *outerpathkeys;
        List       *innerpathkeys;
        int         nClauses;
        Oid        *mergefamilies;
        Oid        *mergecollations;
        int        *mergestrategies;
        bool       *mergenullsfirst;
        PathKey    *opathkey;
        EquivalenceClass *opeclass;
        int         i;
        ListCell   *lc;
        ListCell   *lop;
        ListCell   *lip;
        Path       *outer_path = best_path->jpath.outerjoinpath;
        Path       *inner_path = best_path->jpath.innerjoinpath;
    
        /*
         * MergeJoin can project, so we don't have to demand exact tlists from the
         * inputs.  However, if we're intending to sort an input's result, it's
         * best to request a small tlist so we aren't sorting more data than
         * necessary.
         * MergeJoin可以进行投影运算，因此不必从输入中要求精确的tlist。
        * 然而，如果打算对输入的结果进行排序，最好是请求一个小的tlist，这样就不会对多余的数据进行排序。
         */
        //对外表生成计划Plan
        outer_plan = create_plan_recurse(root, best_path->jpath.outerjoinpath,
                                         (best_path->outersortkeys != NIL) ? CP_SMALL_TLIST : 0);
        //对内部生成计划Plan
        inner_plan = create_plan_recurse(root, best_path->jpath.innerjoinpath,
                                         (best_path->innersortkeys != NIL) ? CP_SMALL_TLIST : 0);
    
        /* Sort join qual clauses into best execution order */
        /* NB: do NOT reorder the mergeclauses */
        //排序连接条件
        joinclauses = order_qual_clauses(root, best_path->jpath.joinrestrictinfo);
    
        /* Get the join qual clauses (in plain expression form) */
        /* Any pseudoconstant clauses are ignored here */
        //获取连接约束条件子句(以扁平化的形式)
        if (IS_OUTER_JOIN(best_path->jpath.jointype))
        {
            extract_actual_join_clauses(joinclauses,
                                        best_path->jpath.path.parent->relids,
                                        &joinclauses, &otherclauses);
        }
        else
        {
            /* We can treat all clauses alike for an inner join */
            //以内连接的方式处理所有条件子句
            joinclauses = extract_actual_clauses(joinclauses, false);
            otherclauses = NIL;
        }
    
        /*
         * Remove the mergeclauses from the list of join qual clauses, leaving the
         * list of quals that must be checked as qpquals.
         * 从join qual子句链表中删除mergeclauses，将必须检查为qpquals的quals链表保留下来。
         */
        mergeclauses = get_actual_clauses(best_path->path_mergeclauses);
        joinclauses = list_difference(joinclauses, mergeclauses);
    
        /*
         * Replace any outer-relation variables with nestloop params.  There
         * should not be any in the mergeclauses.
         * 使用nestloop参数替代外表变量.这些变量不应在mergeclauses中出现.
         */
        if (best_path->jpath.path.param_info)
        {
            joinclauses = (List *)
                replace_nestloop_params(root, (Node *) joinclauses);//连接条件
            otherclauses = (List *)
                replace_nestloop_params(root, (Node *) otherclauses);//其他条件
        }
    
        /*
         * Rearrange mergeclauses, if needed, so that the outer variable is always
         * on the left; mark the mergeclause restrictinfos with correct
         * outer_is_left status.
         * 如果需要，重新安排mergeclauses，使外部变量总是在左边;
         * 用正确的outer_is_left状态标记mergeclause restrictinfos。
         */
        mergeclauses = get_switched_clauses(best_path->path_mergeclauses,
                                            best_path->jpath.outerjoinpath->parent->relids);
    
        /*
         * Create explicit sort nodes for the outer and inner paths if necessary.
         * 如需要创建显式的Sort节点
         */
        if (best_path->outersortkeys)
        {
            Relids      outer_relids = outer_path->parent->relids;
            Sort       *sort = make_sort_from_pathkeys(outer_plan,
                                                       best_path->outersortkeys,
                                                       outer_relids);
    
            label_sort_with_costsize(root, sort, -1.0);
            outer_plan = (Plan *) sort;
            outerpathkeys = best_path->outersortkeys;
        }
        else
            outerpathkeys = best_path->jpath.outerjoinpath->pathkeys;
    
        if (best_path->innersortkeys)
        {
            Relids      inner_relids = inner_path->parent->relids;
            Sort       *sort = make_sort_from_pathkeys(inner_plan,
                                                       best_path->innersortkeys,
                                                       inner_relids);
    
            label_sort_with_costsize(root, sort, -1.0);
            inner_plan = (Plan *) sort;
            innerpathkeys = best_path->innersortkeys;
        }
        else
            innerpathkeys = best_path->jpath.innerjoinpath->pathkeys;
    
        /*
         * If specified, add a materialize node to shield the inner plan from the
         * need to handle mark/restore.
         * 如指定物化,则添加物化节点
         */
        if (best_path->materialize_inner)
        {
            Plan       *matplan = (Plan *) make_material(inner_plan);
    
            /*
             * We assume the materialize will not spill to disk, and therefore
             * charge just cpu_operator_cost per tuple.  (Keep this estimate in
             * sync with final_cost_mergejoin.)
             * 假设materialize不会溢出到磁盘，因此每个元组的成本为cpu_operator_cost。
             * (让这个估计与final_cost_mergejoin保持同步。)
             */
            copy_plan_costsize(matplan, inner_plan);
            matplan->total_cost += cpu_operator_cost * matplan->plan_rows;
    
            inner_plan = matplan;
        }
    
        /*
         * Compute the opfamily/collation/strategy/nullsfirst arrays needed by the
         * executor.  The information is in the pathkeys for the two inputs, but
         * we need to be careful about the possibility of mergeclauses sharing a
         * pathkey, as well as the possibility that the inner pathkeys are not in
         * an order matching the mergeclauses.
         * 计算执行器需要的opfamily/collation/strategy/nullsfirst数组。
         * 信息在这两个输入的pathkeys中，但是需要注意mergeclauses共享一个pathkey的可能性，
         * 以及内表路径键不符合mergeclauses顺序的可能性。
         */
        nClauses = list_length(mergeclauses);
        Assert(nClauses == list_length(best_path->path_mergeclauses));
        mergefamilies = (Oid *) palloc(nClauses * sizeof(Oid));//申请内存
        mergecollations = (Oid *) palloc(nClauses * sizeof(Oid));
        mergestrategies = (int *) palloc(nClauses * sizeof(int));
        mergenullsfirst = (bool *) palloc(nClauses * sizeof(bool));
    
        opathkey = NULL;
        opeclass = NULL;
        lop = list_head(outerpathkeys);
        lip = list_head(innerpathkeys);
        i = 0;
        foreach(lc, best_path->path_mergeclauses)//遍历条件
        {
            RestrictInfo *rinfo = lfirst_node(RestrictInfo, lc);
            EquivalenceClass *oeclass;
            EquivalenceClass *ieclass;
            PathKey    *ipathkey = NULL;
            EquivalenceClass *ipeclass = NULL;
            bool        first_inner_match = false;
    
            /* fetch outer/inner eclass from mergeclause */
            //从mergeclause中获取outer/inner等价类
            if (rinfo->outer_is_left)
            {
                oeclass = rinfo->left_ec;
                ieclass = rinfo->right_ec;
            }
            else
            {
                oeclass = rinfo->right_ec;
                ieclass = rinfo->left_ec;
            }
            Assert(oeclass != NULL);
            Assert(ieclass != NULL);
    
            /*
             * We must identify the pathkey elements associated with this clause
             * by matching the eclasses (which should give a unique match, since
             * the pathkey lists should be canonical).  In typical cases the merge
             * clauses are one-to-one with the pathkeys, but when dealing with
             * partially redundant query conditions, things are more complicated.
             * 必须通过匹配等价类eclasses来标识与此子句关联的pathkey元素
             * (它应该提供唯一的匹配，因为pathkey链表应该是规范的)。
             * 在典型的情况下，merge子句与pathkey是一对一的，但是在处理部分冗余查询条件时，事情就有些复杂了。
             * 
             * lop and lip reference the first as-yet-unmatched pathkey elements.
             * If they're NULL then all pathkey elements have been matched.
             * lop和lip引用第一个尚未匹配的pathkey元素。如果它们为空，那么所有的pathkey元素都已匹配。
             *
             * The ordering of the outer pathkeys should match the mergeclauses,
             * by construction (see find_mergeclauses_for_outer_pathkeys()). There
             * could be more than one mergeclause for the same outer pathkey, but
             * no pathkey may be entirely skipped over.
             * 通过处理，外表pathkey顺序应该与mergeclauses匹配(参见find_mergeclauses_for_outer_pathkeys()函数)。
             * 同一个外表pathkey可以有多个mergeclause，但是不能完全跳过所有pathkey。
             */
            if (oeclass != opeclass)    /* multiple matches are not interesting */
            {
                /* doesn't match the current opathkey, so must match the next */
                //与当前的opathkey不匹配,那么必须与接下来的匹配
                if (lop == NULL)
                    elog(ERROR, "outer pathkeys do not match mergeclauses");
                opathkey = (PathKey *) lfirst(lop);
                opeclass = opathkey->pk_eclass;
                lop = lnext(lop);
                if (oeclass != opeclass)
                    elog(ERROR, "outer pathkeys do not match mergeclauses");
            }
    
            /*
             * The inner pathkeys likewise should not have skipped-over keys, but
             * it's possible for a mergeclause to reference some earlier inner
             * pathkey if we had redundant pathkeys.  For example we might have
             * mergeclauses like "o.a = i.x AND o.b = i.y AND o.c = i.x".  The
             * implied inner ordering is then "ORDER BY x, y, x", but the pathkey
             * mechanism drops the second sort by x as redundant, and this code
             * must cope.
             * 同样，内表pathkey也不应该有skipped-over keys，但是如果我们有冗余的路径键，
             * mergeclause可以引用一些早期的内部路径键。
             * 例如，我们可能存在下面的mergeclauses，比如"o.a = i.x AND o.b = i.y AND o.c = i.x"。
             * 隐含的内部排序是“x, y, x的排序”，但是pathkey机制将按x排序视为多余并删除，在这里必须处理这种情况。
             *
             * It's also possible for the implied inner-rel ordering to be like
             * "ORDER BY x, y, x DESC".  We still drop the second instance of x as
             * redundant; but this means that the sort ordering of a redundant
             * inner pathkey should not be considered significant.  So we must
             * detect whether this is the first clause matching an inner pathkey.
             * 对于隐含的内表排序，也有可能是“ORDER BY x, y, x DESC”。
             * 仍然将x的第二个实例视为冗余并删除;
             * 但是这意味着冗余的内表pathkey的排序顺序不应该被认为是重要的。
             * 因此，我们必须检测这是否是与内表pathkey匹配的第一个子句。
             * 
             */
            if (lip)
            {
                ipathkey = (PathKey *) lfirst(lip);
                ipeclass = ipathkey->pk_eclass;
                if (ieclass == ipeclass)
                {
                    /* successful first match to this inner pathkey */
                    //成功匹配
                    lip = lnext(lip);
                    first_inner_match = true;
                }
            }
            if (!first_inner_match)
            {
                /* redundant clause ... must match something before lip */
                //多余的条件子句,必须在lip前匹配某些pathkey
                ListCell   *l2;
    
                foreach(l2, innerpathkeys)
                {
                    if (l2 == lip)
                        break;
                    ipathkey = (PathKey *) lfirst(l2);
                    ipeclass = ipathkey->pk_eclass;
                    if (ieclass == ipeclass)
                        break;
                }
                if (ieclass != ipeclass)
                    elog(ERROR, "inner pathkeys do not match mergeclauses");
            }
    
            /*
             * The pathkeys should always match each other as to opfamily and
             * collation (which affect equality), but if we're considering a
             * redundant inner pathkey, its sort ordering might not match.  In
             * such cases we may ignore the inner pathkey's sort ordering and use
             * the outer's.  (In effect, we're lying to the executor about the
             * sort direction of this inner column, but it does not matter since
             * the run-time row comparisons would only reach this column when
             * there's equality for the earlier column containing the same eclass.
             * There could be only one value in this column for the range of inner
             * rows having a given value in the earlier column, so it does not
             * matter which way we imagine this column to be ordered.)  But a
             * non-redundant inner pathkey had better match outer's ordering too.
             * 对于opfamily和collation(这会影响等式)，pathkey应该总是匹配的，
             * 但是如果我们考虑一个冗余的内表pathkey，它的排序顺序可能不匹配。
             * 在这种情况下，我们可以忽略内表pathkey的排序顺序，而使用外表访问路径。
             * (实际上，是在内表列的排序方向上欺骗执行器，但这无关紧要，
             *  因为运行时行比较只在包含相同eclass的前一列相等时才会到达这一列。
             *  对于在前一列中具有给定值的内部行范围，在此列中只能有一个值，
             *  因此我们认为该列的顺序如何并不重要。)
             * 而一个非冗余的内部路径也最好与外部的排序匹配。
             */
            if (opathkey->pk_opfamily != ipathkey->pk_opfamily ||
                opathkey->pk_eclass->ec_collation != ipathkey->pk_eclass->ec_collation)
                elog(ERROR, "left and right pathkeys do not match in mergejoin");
            if (first_inner_match &&
                (opathkey->pk_strategy != ipathkey->pk_strategy ||
                 opathkey->pk_nulls_first != ipathkey->pk_nulls_first))
                elog(ERROR, "left and right pathkeys do not match in mergejoin");
    
            /* OK, save info for executor */
            mergefamilies[i] = opathkey->pk_opfamily;
            mergecollations[i] = opathkey->pk_eclass->ec_collation;
            mergestrategies[i] = opathkey->pk_strategy;
            mergenullsfirst[i] = opathkey->pk_nulls_first;
            i++;
        }
    
        /*
         * Note: it is not an error if we have additional pathkey elements (i.e.,
         * lop or lip isn't NULL here).  The input paths might be better-sorted
         * than we need for the current mergejoin.
         * 注意:如果有额外的pathkey元素(例如， lop或lip在这里不是空的)。
         * 输入路径可能比当前合并连接所需的排序更好。
         */
    
        /*
         * Now we can build the mergejoin node.
         * 创建mergejoin节点
         */
        join_plan = make_mergejoin(tlist,
                                   joinclauses,
                                   otherclauses,
                                   mergeclauses,
                                   mergefamilies,
                                   mergecollations,
                                   mergestrategies,
                                   mergenullsfirst,
                                   outer_plan,
                                   inner_plan,
                                   best_path->jpath.jointype,
                                   best_path->jpath.inner_unique,
                                   best_path->skip_mark_restore);
    
        /* Costs of sort and material steps are included in path cost already */
        //排序和物化步骤一包含在访问路径的成本中
        copy_generic_path_info(&join_plan->join.plan, &best_path->jpath.path);
    
        return join_plan;
    }
    
    //------------------------------------------ create_hashjoin_plan
    
    static HashJoin *
    create_hashjoin_plan(PlannerInfo *root,
                         HashPath *best_path)
    {
        HashJoin   *join_plan;
        Hash       *hash_plan;
        Plan       *outer_plan;
        Plan       *inner_plan;
        List       *tlist = build_path_tlist(root, &best_path->jpath.path);
        List       *joinclauses;
        List       *otherclauses;
        List       *hashclauses;
        Oid         skewTable = InvalidOid;
        AttrNumber  skewColumn = InvalidAttrNumber;
        bool        skewInherit = false;
    
        /*
         * HashJoin can project, so we don't have to demand exact tlists from the
         * inputs.  However, it's best to request a small tlist from the inner
         * side, so that we aren't storing more data than necessary.  Likewise, if
         * we anticipate batching, request a small tlist from the outer side so
         * that we don't put extra data in the outer batch files.
         * HashJoin可以进行投影运算，因此我们不必从输入中要求精确的tlist。
         * 但是，最好从内部请求一个小tlist，这样就不需要存储过多的数据。
         * 同样，如果我们进行预批处理，从外部请求一个小tlist，这样就不会在外部批处理文件中添加额外的数据。
         */
        outer_plan = create_plan_recurse(root, best_path->jpath.outerjoinpath,
                                         (best_path->num_batches > 1) ? CP_SMALL_TLIST : 0);
    
        inner_plan = create_plan_recurse(root, best_path->jpath.innerjoinpath,
                                         CP_SMALL_TLIST);
    
        /* Sort join qual clauses into best execution order */
        joinclauses = order_qual_clauses(root, best_path->jpath.joinrestrictinfo);
        /* There's no point in sorting the hash clauses ... */
    
        /* Get the join qual clauses (in plain expression form) */
        /* Any pseudoconstant clauses are ignored here */
        if (IS_OUTER_JOIN(best_path->jpath.jointype))
        {
            extract_actual_join_clauses(joinclauses,
                                        best_path->jpath.path.parent->relids,
                                        &joinclauses, &otherclauses);
        }
        else
        {
            /* We can treat all clauses alike for an inner join */
            joinclauses = extract_actual_clauses(joinclauses, false);
            otherclauses = NIL;
        }
    
        /*
         * Remove the hashclauses from the list of join qual clauses, leaving the
         * list of quals that must be checked as qpquals.
         * 从join qual子句列表中删除hashclause，将必须检查为qpquals的quals列表保留下来。
         */
        hashclauses = get_actual_clauses(best_path->path_hashclauses);
        joinclauses = list_difference(joinclauses, hashclauses);
    
        /*
         * Replace any outer-relation variables with nestloop params.  There
         * should not be any in the hashclauses.
         * 用nestloop参数替换任何外部关系变量。而且不应在hashclauses中出现。
         */
        if (best_path->jpath.path.param_info)
        {
            joinclauses = (List *)
                replace_nestloop_params(root, (Node *) joinclauses);
            otherclauses = (List *)
                replace_nestloop_params(root, (Node *) otherclauses);
        }
    
        /*
         * Rearrange hashclauses, if needed, so that the outer variable is always
         * on the left.
         * 重新安排hashclausees,以便外表的Var出现在左侧
         */
        hashclauses = get_switched_clauses(best_path->path_hashclauses,
                                           best_path->jpath.outerjoinpath->parent->relids);
    
        /*
         * If there is a single join clause and we can identify the outer variable
         * as a simple column reference, supply its identity for possible use in
         * skew optimization.  (Note: in principle we could do skew optimization
         * with multiple join clauses, but we'd have to be able to determine the
         * most common combinations of outer values, which we don't currently have
         * enough stats for.)
         * 如果有一个连接条件子句，并且可以将外表变量标识为一个简单的列引用，
         * 那么可以通过提供它的标识以表在列数据倾斜优化中使用。
         * (注意:原则上可以使用多个连接子句进行倾斜优化，
         * 但我们必须能够确定最常见的外部值组合，目前我们还没有足够的统计数据。)
         */
        if (list_length(hashclauses) == 1)
        {
            OpExpr     *clause = (OpExpr *) linitial(hashclauses);
            Node       *node;
    
            Assert(is_opclause(clause));
            node = (Node *) linitial(clause->args);
            if (IsA(node, RelabelType))
                node = (Node *) ((RelabelType *) node)->arg;
            if (IsA(node, Var))
            {
                Var        *var = (Var *) node;
                RangeTblEntry *rte;
    
                rte = root->simple_rte_array[var->varno];
                if (rte->rtekind == RTE_RELATION)
                {
                    skewTable = rte->relid;
                    skewColumn = var->varattno;
                    skewInherit = rte->inh;
                }
            }
        }
    
        /*
         * Build the hash node and hash join node.
         * 创建hash节点和hash join节点
         */
        hash_plan = make_hash(inner_plan,
                              skewTable,
                              skewColumn,
                              skewInherit);//为内表创建hash表
    
        /*
         * Set Hash node's startup & total costs equal to total cost of input
         * plan; this only affects EXPLAIN display not decisions.
         * 设置哈希节点的启动和总成本等于输入的计划总成本;
         * 这只影响解释显示而不是决策。
         */
        copy_plan_costsize(&hash_plan->plan, inner_plan);
        hash_plan->plan.startup_cost = hash_plan->plan.total_cost;
    
        /*
         * If parallel-aware, the executor will also need an estimate of the total
         * number of rows expected from all participants so that it can size the
         * shared hash table.
         * 如果需要并行，执行器还需要估计所有参与者预期的行数，以便对共享哈希表进行大小计算。
         */
        if (best_path->jpath.path.parallel_aware)
        {
            hash_plan->plan.parallel_aware = true;
            hash_plan->rows_total = best_path->inner_rows_total;
        }
    
        join_plan = make_hashjoin(tlist,
                                  joinclauses,
                                  otherclauses,
                                  hashclauses,
                                  outer_plan,
                                  (Plan *) hash_plan,
                                  best_path->jpath.jointype,
                                  best_path->jpath.inner_unique);//创建hash join节点
    
        copy_generic_path_info(&join_plan->join.plan, &best_path->jpath.path);
    
        return join_plan;
    }
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# where dw.dwbh in ('1001','1002')
    testdb-# order by dw.dwbh;
                                         QUERY PLAN                                               
    --------------------------------------------------------------------------------------------------     Sort  (cost=2010.12..2010.17 rows=20 width=47)
       Sort Key: dw.dwbh
       ->  Nested Loop  (cost=14.24..2009.69 rows=20 width=47)
             ->  Hash Join  (cost=13.95..2002.56 rows=20 width=32)
                   Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
                   ->  Seq Scan on t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
                   ->  Hash  (cost=13.92..13.92 rows=2 width=20)
                         ->  Index Scan using t_dwxx_pkey on t_dwxx dw  (cost=0.29..13.92 rows=2 width=20)
                               Index Cond: ((dwbh)::text = ANY ('{1001,1002}'::text[]))
             ->  Index Scan using idx_t_jfxx_grbh on t_jfxx jf  (cost=0.29..0.35 rows=1 width=20)
                   Index Cond: ((grbh)::text = (gr.grbh)::text)
    
    

启动gdb,设置断点,进入create_join_plan函数

    
    
    (gdb) b create_join_plan
    Breakpoint 1 at 0x7b8426: file createplan.c, line 973.
    (gdb) c
    Continuing.
    
    Breakpoint 1, create_join_plan (root=0x2ef8a00, best_path=0x2f5ad40) at createplan.c:973
    973     switch (best_path->path.pathtype)
    

查看输入参数,pathtype为T_NestLoop

    
    
    (gdb) p *best_path
    $3 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x2f5a570, pathtarget = 0x2f5a788, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 20, startup_cost = 14.241722117799656, 
        total_cost = 2009.6908721177995, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x2f58bb0, innerjoinpath = 0x2f56080, joinrestrictinfo = 0x0}
    

进入create_nestloop_plan

    
    
    973     switch (best_path->path.pathtype)
    (gdb) n
    984             plan = (Plan *) create_nestloop_plan(root,
    (gdb) step
    create_nestloop_plan (root=0x2f49180, best_path=0x2f5ad40) at createplan.c:3678
    3678        List       *tlist = build_path_tlist(root, &best_path->path);    
    

nestloop join->创建tlist,获取连接条件等

    
    
    3678        List       *tlist = build_path_tlist(root, &best_path->path);
    (gdb) n
    3679        List       *joinrestrictclauses = best_path->joinrestrictinfo;
    (gdb) 
    3684        Relids      saveOuterRels = root->curOuterRels;
    (gdb) p root->curOuterRels
    $1 = (Relids) 0x0
    

nestloop join->调用create_plan_recurse创建outer_plan

    
    
    (gdb) n
    3690        outer_plan = create_plan_recurse(root, best_path->outerjoinpath, 0);
    (gdb) 
    
    Breakpoint 1, create_join_plan (root=0x2f49180, best_path=0x2f58bb0) at createplan.c:973
    973     switch (best_path->path.pathtype)
    

nestloop join->外表对应的outer_plan为T_HashJoin

    
    
    (gdb) p *best_path
    $2 = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x2f572d0, pathtarget = 0x2f57508, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 20, startup_cost = 13.949222117799655, 
        total_cost = 2002.5604721177997, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = true, 
      outerjoinpath = 0x2f512f0, innerjoinpath = 0x2f51e98, joinrestrictinfo = 0x2f577a8}
    (gdb) 
    

nestloop join->进入create_hashjoin_plan

    
    
    (gdb) n
    980             plan = (Plan *) create_hashjoin_plan(root,
    (gdb) step
    create_hashjoin_plan (root=0x2f49180, best_path=0x2f58bb0) at createplan.c:4093
    4093        List       *tlist = build_path_tlist(root, &best_path->jpath.path);
    

hash join->创建outer plan

    
    
    (gdb) 
    4108        outer_plan = create_plan_recurse(root, best_path->jpath.outerjoinpath,
    (gdb) p *best_path->jpath.outerjoinpath
    $4 = {type = T_Path, pathtype = T_SeqScan, parent = 0x2f06090, pathtarget = 0x2f062c8, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, startup_cost = 0, total_cost = 1726, 
      pathkeys = 0x0}
    

hash join->创建inner plan

    
    
    (gdb) p *best_path->jpath.innerjoinpath
    $5 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2f04b60, pathtarget = 0x2f04d98, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 2, startup_cost = 0.28500000000000003, 
      total_cost = 13.924222117799655, pathkeys = 0x2f51e20}
    

hash join->获取连接条件

    
    
    (gdb) n
    4115        joinclauses = order_qual_clauses(root, best_path->jpath.joinrestrictinfo);
    (gdb) 
    4120        if (IS_OUTER_JOIN(best_path->jpath.jointype))
    (gdb) p *joinclauses
    $6 = {type = T_List, length = 1, head = 0x2f57780, tail = 0x2f57780}
    

hash join->处理连接条件&hash条件

    
    
    (gdb) n
    4137        hashclauses = get_actual_clauses(best_path->path_hashclauses);
    (gdb) 
    4138        joinclauses = list_difference(joinclauses, hashclauses);
    (gdb) 
    4144        if (best_path->jpath.path.param_info)
    (gdb) p *hashclauses
    $8 = {type = T_List, length = 1, head = 0x2f5d690, tail = 0x2f5d690}
    (gdb) p *joinclauses
    Cannot access memory at address 0x0
    

hash join->变换位置,把外表的Var放在左侧

    
    
    (gdb) n
    4156        hashclauses = get_switched_clauses(best_path->path_hashclauses,
    (gdb) 
    

hash join->Hash连接条件只有一个,进行数据倾斜优化

    
    
    (gdb) 
    4167        if (list_length(hashclauses) == 1)
    (gdb) n
    4169            OpExpr     *clause = (OpExpr *) linitial(hashclauses);
    (gdb) n
    4172            Assert(is_opclause(clause));
    (gdb) 
    4173            node = (Node *) linitial(clause->args);
    (gdb) 
    4174            if (IsA(node, RelabelType))
    (gdb) 
    4175                node = (Node *) ((RelabelType *) node)->arg;
    (gdb) 
    4176            if (IsA(node, Var))
    (gdb) 
    4178                Var        *var = (Var *) node;
    (gdb) 
    4181                rte = root->simple_rte_array[var->varno];
    (gdb) p *node
    $9 = {type = T_Var}
    (gdb) p *(Var *)node
    $10 = {xpr = {type = T_Var}, varno = 3, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 1, location = 208}
    (gdb) n
    4182                if (rte->rtekind == RTE_RELATION)
    (gdb) 
    4184                    skewTable = rte->relid;
    (gdb) 
    4185                    skewColumn = var->varattno;
    (gdb) 
    4186                    skewInherit = rte->inh;
    (gdb) 
    

hash join->开始创建创建hash节点和hash join节点  
创建hash节点(构建Hash表)

    
    
    4194        hash_plan = make_hash(inner_plan,
    (gdb) n
    4203        copy_plan_costsize(&hash_plan->plan, inner_plan);
    (gdb) 
    4204        hash_plan->plan.startup_cost = hash_plan->plan.total_cost;
    (gdb) p *hash_plan
    $11 = {plan = {type = T_Hash, startup_cost = 0.28500000000000003, total_cost = 13.924222117799655, plan_rows = 2, 
        plan_width = 20, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2f5d250, qual = 0x0, 
        lefttree = 0x2f58428, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, skewTable = 16742, 
      skewColumn = 1, skewInherit = false, rows_total = 0}
    

hash join->创建hash join节点

    
    
    (gdb) n
    4217        join_plan = make_hashjoin(tlist,
    (gdb) 
    4226        copy_generic_path_info(&join_plan->join.plan, &best_path->jpath.path);
    (gdb) 
    4228        return join_plan;
    (gdb) p *join_plan
    $13 = {join = {plan = {type = T_HashJoin, startup_cost = 13.949222117799655, total_cost = 2002.5604721177997, 
          plan_rows = 20, plan_width = 32, parallel_aware = false, parallel_safe = true, plan_node_id = 0, 
          targetlist = 0x2f5cb28, qual = 0x0, lefttree = 0x2f5ae98, righttree = 0x2f5d830, initPlan = 0x0, extParam = 0x0, 
          allParam = 0x0}, jointype = JOIN_INNER, inner_unique = true, joinqual = 0x0}, hashclauses = 0x2f5d7f8}
    

hash join->回到create_nestloop_plan

    
    
    (gdb) n
    create_nestloop_plan (root=0x2f49180, best_path=0x2f5ad40) at createplan.c:3694
    3694                                       best_path->outerjoinpath->parent->relids);
    (gdb) n
    3693        root->curOuterRels = bms_union(root->curOuterRels,        
    

nestloop join->创建内表Plan

    
    
    (gdb) n
    3696        inner_plan = create_plan_recurse(root, best_path->innerjoinpath, 0);
    (gdb) p *best_path->innerjoinpath
    $16 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2f06858, pathtarget = 0x2f06a70, param_info = 0x2f56910, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.29249999999999998, 
      total_cost = 0.34651999999999999, pathkeys = 0x2f56608}
    

nestloop join->获取连接条件子句

    
    
    (gdb) n
    3699        bms_free(root->curOuterRels);
    (gdb) 
    3700        root->curOuterRels = saveOuterRels;
    (gdb) 
    3703        joinrestrictclauses = order_qual_clauses(root, joinrestrictclauses);
    (gdb) 
    3707        if (IS_OUTER_JOIN(best_path->jointype))
    (gdb) p *joinrestrictclauses
    Cannot access memory at address 0x0
    

nestloop join->获取连接条件&参数化处理(相关值为NULL)

    
    
    (gdb) n
    3716            joinclauses = extract_actual_clauses(joinrestrictclauses, false);
    (gdb) 
    3717            otherclauses = NIL;
    (gdb) 
    3721        if (best_path->path.param_info)
    (gdb) p *joinclauses
    Cannot access memory at address 0x0
    (gdb) p *best_path->path.param_info
    Cannot access memory at address 0x0
    

nestloop join->获取外表的relids(外表为1和3号RTE的连接)

    
    
    (gdb) n
    3733        outerrelids = best_path->outerjoinpath->parent->relids;
    (gdb) 
    3734        nestParams = NIL;
    (gdb) p *outerrelids
    $17 = {nwords = 1, words = 0x2f574ec}
    (gdb) p *outerrelids->words
    $18 = 10
    

nestloop join->遍历当前的外表参数链表

    
    
    (gdb) n
    3735        prev = NULL;
    (gdb) 
    3736        for (cell = list_head(root->curOuterParams); cell; cell = next)
    (gdb) p *root->curOuterParams
    $19 = {type = T_List, length = 1, head = 0x2f5df98, tail = 0x2f5df98}
    

nestloop join->查看该参数信息,3号RTE编号为2的字段(即grbh)

    
    
    (gdb) n
    3738            NestLoopParam *nlp = (NestLoopParam *) lfirst(cell);
    (gdb) 
    3740            next = lnext(cell);
    (gdb) p *(NestLoopParam *)nlp
    $21 = {type = T_NestLoopParam, paramno = 0, paramval = 0x2f54e50}
    (gdb) p *nlp->paramval
    $22 = {xpr = {type = T_Var}, varno = 3, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 2, location = 273}
    

nestloop join->把条件从root->curOuterParams移动到nestParams链表中

    
    
    (gdb) n
    3741            if (IsA(nlp->paramval, Var) &&
    (gdb) n
    3742                bms_is_member(nlp->paramval->varno, outerrelids))
    (gdb) 
    3741            if (IsA(nlp->paramval, Var) &&
    (gdb) 
    3744                root->curOuterParams = list_delete_cell(root->curOuterParams,
    (gdb) 
    3746                nestParams = lappend(nestParams, nlp);
    (gdb) 
    3736        for (cell = list_head(root->curOuterParams); cell; cell = next)
    (gdb) p *nestParams
    $23 = {type = T_List, length = 1, head = 0x2f5df98, tail = 0x2f5df98}
    (gdb) p *(Node *)nestParams->head->data.ptr_value
    $24 = {type = T_NestLoopParam}
    (gdb) p *(NestLoopParam *)nestParams->head->data.ptr_value
    $25 = {type = T_NestLoopParam, paramno = 0, paramval = 0x2f54e50}
    (gdb) set $nlp=(NestLoopParam *)nestParams->head->data.ptr_value
    (gdb) p $nlp->paramval
    $26 = (Var *) 0x2f54e50
    (gdb) p *$nlp->paramval
    $27 = {xpr = {type = T_Var}, varno = 3, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 2, location = 273}
    (gdb) 
    

nestloop join->创建nestloop join节点

    
    
    (gdb) n
    3771                                  best_path->inner_unique);
    (gdb) 
    3764        join_plan = make_nestloop(tlist,
    (gdb) 
    3773        copy_generic_path_info(&join_plan->join.plan, &best_path->path);
    (gdb) 
    3775        return join_plan;
    (gdb) p *join_plan
    $28 = {join = {plan = {type = T_NestLoop, startup_cost = 14.241722117799656, total_cost = 2009.6908721177995, 
          plan_rows = 20, plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, 
          targetlist = 0x2f5c770, qual = 0x0, lefttree = 0x2f5d8c8, righttree = 0x2f59ed0, initPlan = 0x0, extParam = 0x0, 
          allParam = 0x0}, jointype = JOIN_INNER, inner_unique = false, joinqual = 0x0}, nestParams = 0x2f5dfc0}
    (gdb) 
    

DONE!

### 四、参考资料

[createplan.c](https://doxygen.postgresql.org/createplan_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

