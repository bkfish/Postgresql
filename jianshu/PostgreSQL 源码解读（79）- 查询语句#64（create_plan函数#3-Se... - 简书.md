本节介绍了创建计划create_plan函数中扫描计划的实现过程，主要的逻辑在函数create_scan_plan中实现。

### 一、数据结构

**Plan**  
所有计划节点通过将Plan结构作为第一个字段从Plan结构“派生”。这确保了在将节点转换为计划节点时，一切都能正常工作。(在执行器中以通用方式传递时，节点指针经常被转换为Plan
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

create_scan_plan函数创建Scan
Plan.扫描可以分为顺序扫描(全表扫描)/索引扫描/索引快速扫描/TID扫描等多种扫描方式,这里主要介绍常见的顺序扫描和索引扫描,相应的实现函数是create_seqscan_plan和create_indexscan_plan.

    
    
    
    //--------------------------------------------------- create_scan_plan
    /*
     * create_scan_plan
     *   Create a scan plan for the parent relation of 'best_path'.
     *   为relation best_path创建相应的扫描计划
     */
    static Plan *
    create_scan_plan(PlannerInfo *root, Path *best_path, int flags)
    {
        RelOptInfo *rel = best_path->parent;
        List       *scan_clauses;
        List       *gating_clauses;
        List       *tlist;
        Plan       *plan;
    
        /*
         * Extract the relevant restriction clauses from the parent relation. The
         * executor must apply all these restrictions during the scan, except for
         * pseudoconstants which we'll take care of below.
         * 从父关系中提取相关的限制条件。
         * 执行器必须在扫描期间应用所有这些限制条件，除了伪常量，将在下面处理这些限制。
         *
         * If this is a plain indexscan or index-only scan, we need not consider
         * restriction clauses that are implied by the index's predicate, so use
         * indrestrictinfo not baserestrictinfo.  Note that we can't do that for
         * bitmap indexscans, since there's not necessarily a single index
         * involved; but it doesn't matter since create_bitmap_scan_plan() will be
         * able to get rid of such clauses anyway via predicate proof.
         * 如果这是一个普通的indexscan或index-only扫描，
         * 则不需要考虑由索引谓词隐含的限制条款，因此使用indrestrictinfo而不是baserestrictinfo。
         * 注意，对于位图索引扫描，我们不能这样做，因为不需要一个索引;
         * 但是这并不重要，因为create_bitmap_scan_plan()将能够通过谓词证明消除这些子句。
         */
        switch (best_path->pathtype)
        {
            case T_IndexScan:
            case T_IndexOnlyScan://索引扫描,使用索引约束条件
                scan_clauses = castNode(IndexPath, best_path)->indexinfo->indrestrictinfo;
                break;
            default:
                scan_clauses = rel->baserestrictinfo;//默认使用关系中的约束条件
                break;
        }
    
        /*
         * If this is a parameterized scan, we also need to enforce all the join
         * clauses available from the outer relation(s).
         * 如果这是一个参数化扫描,需要从连接外关系中加入所有可用的连接约束条件
         *
         * For paranoia's sake, don't modify the stored baserestrictinfo list.
         * 由于paranoia's sake,不更新baserestrictinfo链表
         */
        if (best_path->param_info)
            scan_clauses = list_concat(list_copy(scan_clauses),
                                       best_path->param_info->ppi_clauses);
    
        /*
         * Detect whether we have any pseudoconstant quals to deal with.  Then, if
         * we'll need a gating Result node, it will be able to project, so there
         * are no requirements on the child's tlist.
         * 检测是否有伪常数函数需要处理。
         * 然后，需要一个能够执行投影的出口结果节点，因此对子tlist没有需求。
         */
        gating_clauses = get_gating_quals(root, scan_clauses);
        if (gating_clauses)
            flags = 0;
    
        /*
         * For table scans, rather than using the relation targetlist (which is
         * only those Vars actually needed by the query), we prefer to generate a
         * tlist containing all Vars in order.  This will allow the executor to
         * optimize away projection of the table tuples, if possible.
         * 对于表扫描，不使用关系targetlist(它只是查询实际需要的Vars)，
         * 我们更喜欢按照顺序生成一个包含所有Vars的tlist。
         * 如果可能的话，这将允许执行程序优化表元组的投影。
         *
         * But if the caller is going to ignore our tlist anyway, then don't
         * bother generating one at all.  We use an exact equality test here, so
         * that this only applies when CP_IGNORE_TLIST is the only flag set.
         * 但是，如果调用者不管怎样都要忽略tlist，那么就根本不用去生成一个。
         * 在这里使用了一个完全相等的测试，因此只有当CP_IGNORE_TLIST是唯一的标志设置时才适用。
         */
        if (flags == CP_IGNORE_TLIST)
        {
            tlist = NULL;//使用CP_IGNORE_TLIST标志,则设置tlist为NULL
        }
        else if (use_physical_tlist(root, best_path, flags))
        {
            if (best_path->pathtype == T_IndexOnlyScan)//索引快速扫描
            {
                /* For index-only scan, the preferred tlist is the index's */
                //对于所有快速扫描,tlist中的列应在索引中
                tlist = copyObject(((IndexPath *) best_path)->indexinfo->indextlist);
    
                /*
                 * Transfer sortgroupref data to the replacement tlist, if
                 * requested (use_physical_tlist checked that this will work).
                 * 如需要,转换sortgroupref数据为tlist(use_physical_tlist检查是否可行)
                 */
                if (flags & CP_LABEL_TLIST)
                    apply_pathtarget_labeling_to_tlist(tlist, best_path->pathtarget);
            }
            else//非索引快速扫描
            {
                tlist = build_physical_tlist(root, rel);//构建物理的tlist
                if (tlist == NIL)
                {
                    /* Failed because of dropped cols, so use regular method */
                    //build_physical_tlist无法构建,则使用常规方法构建
                    tlist = build_path_tlist(root, best_path);
                }
                else
                {
                    /* As above, transfer sortgroupref data to replacement tlist */
                    if (flags & CP_LABEL_TLIST)
                        apply_pathtarget_labeling_to_tlist(tlist, best_path->pathtarget);
                }
            }
        }
        else
        {
            tlist = build_path_tlist(root, best_path);//使用常规方法构建
        }
    
        switch (best_path->pathtype)//根据路径类型进行相应的处理
        {
            case T_SeqScan://顺序扫描
                plan = (Plan *) create_seqscan_plan(root,
                                                    best_path,
                                                    tlist,
                                                    scan_clauses);
                break;
    
            case T_SampleScan:
                plan = (Plan *) create_samplescan_plan(root,
                                                       best_path,
                                                       tlist,
                                                       scan_clauses);
                break;
    
            case T_IndexScan:
                plan = (Plan *) create_indexscan_plan(root,
                                                      (IndexPath *) best_path,
                                                      tlist,
                                                      scan_clauses,
                                                      false);
                break;
    
            case T_IndexOnlyScan:
                plan = (Plan *) create_indexscan_plan(root,
                                                      (IndexPath *) best_path,
                                                      tlist,
                                                      scan_clauses,
                                                      true);
                break;
    
            case T_BitmapHeapScan:
                plan = (Plan *) create_bitmap_scan_plan(root,
                                                        (BitmapHeapPath *) best_path,
                                                        tlist,
                                                        scan_clauses);
                break;
    
            case T_TidScan:
                plan = (Plan *) create_tidscan_plan(root,
                                                    (TidPath *) best_path,
                                                    tlist,
                                                    scan_clauses);
                break;
    
            case T_SubqueryScan:
                plan = (Plan *) create_subqueryscan_plan(root,
                                                         (SubqueryScanPath *) best_path,
                                                         tlist,
                                                         scan_clauses);
                break;
    
            case T_FunctionScan:
                plan = (Plan *) create_functionscan_plan(root,
                                                         best_path,
                                                         tlist,
                                                         scan_clauses);
                break;
    
            case T_TableFuncScan:
                plan = (Plan *) create_tablefuncscan_plan(root,
                                                          best_path,
                                                          tlist,
                                                          scan_clauses);
                break;
    
            case T_ValuesScan:
                plan = (Plan *) create_valuesscan_plan(root,
                                                       best_path,
                                                       tlist,
                                                       scan_clauses);
                break;
    
            case T_CteScan:
                plan = (Plan *) create_ctescan_plan(root,
                                                    best_path,
                                                    tlist,
                                                    scan_clauses);
                break;
    
            case T_NamedTuplestoreScan:
                plan = (Plan *) create_namedtuplestorescan_plan(root,
                                                                best_path,
                                                                tlist,
                                                                scan_clauses);
                break;
    
            case T_WorkTableScan:
                plan = (Plan *) create_worktablescan_plan(root,
                                                          best_path,
                                                          tlist,
                                                          scan_clauses);
                break;
    
            case T_ForeignScan:
                plan = (Plan *) create_foreignscan_plan(root,
                                                        (ForeignPath *) best_path,
                                                        tlist,
                                                        scan_clauses);
                break;
    
            case T_CustomScan:
                plan = (Plan *) create_customscan_plan(root,
                                                       (CustomPath *) best_path,
                                                       tlist,
                                                       scan_clauses);
                break;
    
            default:
                elog(ERROR, "unrecognized node type: %d",
                     (int) best_path->pathtype);
                plan = NULL;        /* keep compiler quiet */
                break;
        }
    
        /*
         * If there are any pseudoconstant clauses attached to this node, insert a
         * gating Result node that evaluates the pseudoconstants as one-time
         * quals.
         * 如果这个节点上附加了伪常量子句，插入一个Result节点，该节点将伪常量计算为一次性的条件quals。
         */
        if (gating_clauses)
            plan = create_gating_plan(root, best_path, plan, gating_clauses);
    
        return plan;
    }
    
    
    //------------------------------------- create_seqscan_plan
    /*
     * create_seqscan_plan
     *   Returns a seqscan plan for the base relation scanned by 'best_path'
     *   with restriction clauses 'scan_clauses' and targetlist 'tlist'.
     *   返回seqscan顺序扫描计划(基于基本关系中的best_path),同时考虑了约束条件scan_clauses和投影列tlist
     */
    static SeqScan *
    create_seqscan_plan(PlannerInfo *root, Path *best_path,
                        List *tlist, List *scan_clauses)
    {
        SeqScan    *scan_plan;
        Index       scan_relid = best_path->parent->relid;
    
        /* it should be a base rel... */
        //基本关系
        Assert(scan_relid > 0);
        Assert(best_path->parent->rtekind == RTE_RELATION);
    
        /* Sort clauses into best execution order */
        //约束条件排序为最佳的执行顺序
        scan_clauses = order_qual_clauses(root, scan_clauses);
    
        /* Reduce RestrictInfo list to bare expressions; ignore pseudoconstants */
        //规约限制条件信息链表到裸表达式,同时忽略pseudoconstants
        scan_clauses = extract_actual_clauses(scan_clauses, false);
    
        /* Replace any outer-relation variables with nestloop params */
        //参数化访问,则使用内嵌循环参数替换外表变量
        if (best_path->param_info)
        {
            scan_clauses = (List *)
                replace_nestloop_params(root, (Node *) scan_clauses);//约束条件
        }
    
        scan_plan = make_seqscan(tlist,
                                 scan_clauses,
                                 scan_relid);//构建扫描计划
    
        copy_generic_path_info(&scan_plan->plan, best_path);
    
        return scan_plan;//返回
    }
    
    //------------------------------------- copy_generic_path_info
    
     /*
      * Copy cost and size info from a Path node to the Plan node created from it.
      * The executor usually won't use this info, but it's needed by EXPLAIN.
      * Also copy the parallel-related flags, which the executor *will* use.
      */
     static void
     copy_generic_path_info(Plan *dest, Path *src)
     {
         dest->startup_cost = src->startup_cost;
         dest->total_cost = src->total_cost;
         dest->plan_rows = src->rows;
         dest->plan_width = src->pathtarget->width;
         dest->parallel_aware = src->parallel_aware;
         dest->parallel_safe = src->parallel_safe;
     }
     
    
    //------------------------------------- make_seqscan
    
    static SeqScan *
    make_seqscan(List *qptlist,
                 List *qpqual,
                 Index scanrelid)
    {
        SeqScan    *node = makeNode(SeqScan);
        Plan       *plan = &node->plan;
    
        plan->targetlist = qptlist;
        plan->qual = qpqual;
        plan->lefttree = NULL;
        plan->righttree = NULL;
        node->scanrelid = scanrelid;
    
        return node;//创建节点
    }
    
    //------------------------------------- create_indexscan_plan
    
    /*
     * create_indexscan_plan
     *    Returns an indexscan plan for the base relation scanned by 'best_path'
     *    with restriction clauses 'scan_clauses' and targetlist 'tlist'.
     *    返回索引扫描计划
     *
     * We use this for both plain IndexScans and IndexOnlyScans, because the
     * qual preprocessing work is the same for both.  Note that the caller tells
     * us which to build --- we don't look at best_path->path.pathtype, because
     * create_bitmap_subplan needs to be able to override the prior decision.
     * 此过程用于普通的indexscan和indexonlyscan，因为两者的条件qual预处理工作是相同的。
     * 注意，调用者明确指示构建哪个——不需要查看best_path->path.pathtype，
     * 因为create_bitmap_subplan需要能够覆盖之前的决定。
     */
    static Scan *
    create_indexscan_plan(PlannerInfo *root,
                          IndexPath *best_path,
                          List *tlist,
                          List *scan_clauses,
                          bool indexonly)
    {
        Scan       *scan_plan;
        List       *indexquals = best_path->indexquals;
        List       *indexorderbys = best_path->indexorderbys;
        Index       baserelid = best_path->path.parent->relid;
        Oid         indexoid = best_path->indexinfo->indexoid;
        List       *qpqual;
        List       *stripped_indexquals;
        List       *fixed_indexquals;
        List       *fixed_indexorderbys;
        List       *indexorderbyops = NIL;
        ListCell   *l;
    
        /* it should be a base rel... */
        //基本关系
        Assert(baserelid > 0);
        Assert(best_path->path.parent->rtekind == RTE_RELATION);
    
        /*
         * Build "stripped" indexquals structure (no RestrictInfos) to pass to
         * executor as indexqualorig
         * 构建"stripped"索引约束条件结构(非RestrictInfos),作为执行器的调用参数indexqualorig
         */
        stripped_indexquals = get_actual_clauses(indexquals);
    
        /*
         * The executor needs a copy with the indexkey on the left of each clause
         * and with index Vars substituted for table ones.
         * 执行器需要在每个子句的左边加上indexkey，并用索引变量替换表变量。
         */
        fixed_indexquals = fix_indexqual_references(root, best_path);
    
        /*
         * Likewise fix up index attr references in the ORDER BY expressions.
         * 同样，修正ORDER BY 表达式的索引属性attr引用。
         */
        fixed_indexorderbys = fix_indexorderby_references(root, best_path);
    
        /*
         * The qpqual list must contain all restrictions not automatically handled
         * by the index, other than pseudoconstant clauses which will be handled
         * by a separate gating plan node.  All the predicates in the indexquals
         * will be checked (either by the index itself, or by nodeIndexscan.c),
         * but if there are any "special" operators involved then they must be
         * included in qpqual.  The upshot is that qpqual must contain
         * scan_clauses minus whatever appears in indexquals.
         * qpqual链表必须包含索引未处理的其他限制条件，伪常量子句除外，伪常量子句将由单独的gating计划节点处理。
         * indexquals中的所有谓词都将被检查(可以通过索引本身检查，也可以通过nodeIndexscan.c检查)，
         * 但是如果涉及到任何“特殊”运算符，那么它们必须包含在qpqual中。
         * 结果是，qpqual必须包含scan_clause，除去indexquals中出现的任何内容。
         * 
         * In normal cases simple pointer equality checks will be enough to spot
         * duplicate RestrictInfos, so we try that first.
         * 在通常情况下，简单的指针相等检查将足以发现双重的RestrictInfos，因此首先执行此检查操作。
         *
         * Another common case is that a scan_clauses entry is generated from the
         * same EquivalenceClass as some indexqual, and is therefore redundant
         * with it, though not equal.  (This happens when indxpath.c prefers a
         * different derived equality than what generate_join_implied_equalities
         * picked for a parameterized scan's ppi_clauses.)
         * 另一种常见的情况是scan_clauses entry是由与indexqual相同的EC生成的，因此尽管不相等但它是多余的。
         * (这发生在indxpath.c与为参数化扫描的ppi_clauses与generate_join_implied_equalities选择的是不同的派生等式时)。
         *
         * In some situations (particularly with OR'd index conditions) we may
         * have scan_clauses that are not equal to, but are logically implied by,
         * the index quals; so we also try a predicate_implied_by() check to see
         * if we can discard quals that way.  (predicate_implied_by assumes its
         * first input contains only immutable functions, so we have to check
         * that.)
         * 在某些情况下(特别是在索引条件下)，可能有scan_clauses，虽然不等价,但逻辑上由索引quals表示;
         * 因此，我们还尝试使用predicate_implied_by()检验是否这样丢弃quals。
         * (predicate_implied_by假设它的第一个输入只包含不可变函数，所以必须检查它。)
         *
         * Note: if you change this bit of code you should also look at
         * extract_nonindex_conditions() in costsize.c.
         * 注意:如果需要这部分代码,需要检查costsize.c中的extract_nonindex_conditions()函数
         */
        qpqual = NIL;
        foreach(l, scan_clauses)//遍历scan_clauses链表
        {
            RestrictInfo *rinfo = lfirst_node(RestrictInfo, l);
    
            if (rinfo->pseudoconstant)
                continue;           /* 不理会pseudoconstants;we may drop pseudoconstants here */
            if (list_member_ptr(indexquals, rinfo))
                continue;           /* 重复不处理;simple duplicate */
            if (is_redundant_derived_clause(rinfo, indexquals))
                continue;           /* 从EC中派生的不处理;derived from same EquivalenceClass */
            if (!contain_mutable_functions((Node *) rinfo->clause) &&
                predicate_implied_by(list_make1(rinfo->clause), indexquals, false))
                continue;           /* 通过indexquals可隐式证明;provably implied by indexquals */
            qpqual = lappend(qpqual, rinfo);
        }
    
        /* Sort clauses into best execution order */
        //条件排序
        qpqual = order_qual_clauses(root, qpqual);
    
        /* Reduce RestrictInfo list to bare expressions; ignore pseudoconstants */
        //规约
        qpqual = extract_actual_clauses(qpqual, false);
    
        /*
         * We have to replace any outer-relation variables with nestloop params in
         * the indexqualorig, qpqual, and indexorderbyorig expressions.  A bit
         * annoying to have to do this separately from the processing in
         * fix_indexqual_references --- rethink this when generalizing the inner
         * indexscan support.  But note we can't really do this earlier because
         * it'd break the comparisons to predicates above ... (or would it?  Those
         * wouldn't have outer refs)
         * 我们必须用indexqualorig、qpqual和indexorderbyorig表达式中的nestloop参数替换任何外关系变量。
         * 与fix_indexqual_references中的处理分开来完成这一工作有点烦人——在泛化内部indexscan支持时重新考虑这一点。
         * 但是请注意，不能提前做这个事情，因为它会打断与上面谓词的比较……(可能发生吗?因为它们没有外部依赖)
         */
        if (best_path->path.param_info)//存在参数信息,使用内嵌循环参数替换
        {
            stripped_indexquals = (List *)
                replace_nestloop_params(root, (Node *) stripped_indexquals);
            qpqual = (List *)
                replace_nestloop_params(root, (Node *) qpqual);
            indexorderbys = (List *)
                replace_nestloop_params(root, (Node *) indexorderbys);
        }
    
        /*
         * If there are ORDER BY expressions, look up the sort operators for their
         * result datatypes.
         * //存在ORDER BY表达式
         */
        if (indexorderbys)
        {
            ListCell   *pathkeyCell,
                       *exprCell;
    
            /*
             * PathKey contains OID of the btree opfamily we're sorting by, but
             * that's not quite enough because we need the expression's datatype
             * to look up the sort operator in the operator family.
             * PathKey已经包含我们正在排序的btree opfamily的OID，
             * 但这还不足够，因为需要表达式的数据类型来查找操作符家族中的sort操作符。
             */
            Assert(list_length(best_path->path.pathkeys) == list_length(indexorderbys));
            forboth(pathkeyCell, best_path->path.pathkeys, exprCell, indexorderbys)
            {
                PathKey    *pathkey = (PathKey *) lfirst(pathkeyCell);
                Node       *expr = (Node *) lfirst(exprCell);
                Oid         exprtype = exprType(expr);
                Oid         sortop;
    
                /* Get sort operator from opfamily */
                sortop = get_opfamily_member(pathkey->pk_opfamily,
                                             exprtype,
                                             exprtype,
                                             pathkey->pk_strategy);
                if (!OidIsValid(sortop))
                    elog(ERROR, "missing operator %d(%u,%u) in opfamily %u",
                         pathkey->pk_strategy, exprtype, exprtype, pathkey->pk_opfamily);
                indexorderbyops = lappend_oid(indexorderbyops, sortop);
            }
        }
    
        /* Finally ready to build the plan node */
        //OK,下面可以构建计划节点了
        if (indexonly)
            scan_plan = (Scan *) make_indexonlyscan(tlist,
                                                    qpqual,
                                                    baserelid,
                                                    indexoid,
                                                    fixed_indexquals,
                                                    fixed_indexorderbys,
                                                    best_path->indexinfo->indextlist,
                                                    best_path->indexscandir);
        else
            scan_plan = (Scan *) make_indexscan(tlist,
                                                qpqual,
                                                baserelid,
                                                indexoid,
                                                fixed_indexquals,
                                                stripped_indexquals,
                                                fixed_indexorderbys,
                                                indexorderbys,
                                                indexorderbyops,
                                                best_path->indexscandir);
    
        copy_generic_path_info(&scan_plan->plan, &best_path->path);
    
        return scan_plan;
    }
    
    
    //------------------------------------- make_indexscan/make_indexonlyscan
    static IndexScan *
    make_indexscan(List *qptlist,
                   List *qpqual,
                   Index scanrelid,
                   Oid indexid,
                   List *indexqual,
                   List *indexqualorig,
                   List *indexorderby,
                   List *indexorderbyorig,
                   List *indexorderbyops,
                   ScanDirection indexscandir)
    {
        IndexScan  *node = makeNode(IndexScan);
        Plan       *plan = &node->scan.plan;
    
        plan->targetlist = qptlist;
        plan->qual = qpqual;
        plan->lefttree = NULL;
        plan->righttree = NULL;
        node->scan.scanrelid = scanrelid;
        node->indexid = indexid;
        node->indexqual = indexqual;
        node->indexqualorig = indexqualorig;
        node->indexorderby = indexorderby;
        node->indexorderbyorig = indexorderbyorig;
        node->indexorderbyops = indexorderbyops;
        node->indexorderdir = indexscandir;
    
        return node;
    }
    
    static IndexOnlyScan *
    make_indexonlyscan(List *qptlist,
                       List *qpqual,
                       Index scanrelid,
                       Oid indexid,
                       List *indexqual,
                       List *indexorderby,
                       List *indextlist,
                       ScanDirection indexscandir)
    {
        IndexOnlyScan *node = makeNode(IndexOnlyScan);
        Plan       *plan = &node->scan.plan;
    
        plan->targetlist = qptlist;
        plan->qual = qpqual;
        plan->lefttree = NULL;
        plan->righttree = NULL;
        node->scan.scanrelid = scanrelid;
        node->indexid = indexid;
        node->indexqual = indexqual;
        node->indexorderby = indexorderby;
        node->indextlist = indextlist;
        node->indexorderdir = indexscandir;
    
        return node;
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
    -------------------------------------------------------------------------------------------------     Sort  (cost=2010.12..2010.17 rows=20 width=47)
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
    (11 rows)
    
    

启动gdb,设置断点,进入函数create_scan_plan

    
    
    (gdb) b create_scan_plan
    Breakpoint 1 at 0x7b7b78: file createplan.c, line 514.
    (gdb) c
    Continuing.
    
    Breakpoint 1, create_scan_plan (root=0x2d36d00, best_path=0x2d57ad0, flags=0) at createplan.c:514
    514     RelOptInfo *rel = best_path->parent;
    

pathtype为T_SeqScan

    
    
    (gdb) p best_path->pathtype
    $1 = T_SeqScan
    

进入相应的分支,约束条件直接使用Relation的限制条件

    
    
    (gdb) n
    532     switch (best_path->pathtype)
    (gdb) 
    539             scan_clauses = rel->baserestrictinfo;
    (gdb) 
    540             break;
    

非索引快速扫描,构建目标投影列

    
    
    (gdb) n
    559     if (gating_clauses)
    (gdb) 
    572     if (flags == CP_IGNORE_TLIST)
    (gdb) 
    576     else if (use_physical_tlist(root, best_path, flags))
    (gdb) 
    578         if (best_path->pathtype == T_IndexOnlyScan)
    (gdb) 
    592             tlist = build_physical_tlist(root, rel);
    (gdb) 
    593             if (tlist == NIL)
    (gdb) 
    601                 if (flags & CP_LABEL_TLIST)
    (gdb) 
    (gdb) p *tlist
    $4 = {type = T_List, length = 5, head = 0x2d89440, tail = 0x2d897d8}
    (gdb) p *(Node *)tlist->head->data.ptr_value
    $5 = {type = T_TargetEntry}
    (gdb) p *(TargetEntry *)tlist->head->data.ptr_value
    $6 = {xpr = {type = T_TargetEntry}, expr = 0x2d89390, resno = 1, resname = 0x0, ressortgroupref = 0, resorigtbl = 0, 
      resorigcol = 0, resjunk = false}
    

根据pathtype进入相应的处理逻辑,调用函数create_seqscan_plan

    
    
    (gdb) n
    614             plan = (Plan *) create_seqscan_plan(root,
    (gdb) step
    create_seqscan_plan (root=0x2d36d00, best_path=0x2d57ad0, tlist=0x2d89468, scan_clauses=0x0) at createplan.c:2444
    2444        Index       scan_relid = best_path->parent->relid;
    

构建扫描条件,为NULL

    
    
    2451        scan_clauses = order_qual_clauses(root, scan_clauses);
    (gdb) 
    2454        scan_clauses = extract_actual_clauses(scan_clauses, false);
    (gdb) 
    2457        if (best_path->param_info)
    (gdb) 
    (gdb) p *scan_clauses
    Cannot access memory at address 0x0
    

生成SeqScan Plan节点,并把启动成本/总成本等相关信息拷贝到该Plan中

    
    
    (gdb) n
    2463        scan_plan = make_seqscan(tlist,
    (gdb) 
    2467        copy_generic_path_info(&scan_plan->plan, best_path);
    

完成创建,返回Plan

    
    
    (gdb) n
    2469        return scan_plan;
    (gdb) p *scan_plan
    $1 = {plan = {type = T_SeqScan, startup_cost = 0, total_cost = 1726, plan_rows = 100000, plan_width = 16, 
        parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2db4e38, qual = 0x0, lefttree = 0x0, 
        righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, scanrelid = 3}
    

下面跟踪分析create_indexscan_plan函数  
设置断点,进入函数

    
    
    (gdb) b create_indexscan_plan
    Breakpoint 2 at 0x7badad: file createplan.c, line 2536.
    (gdb) c
    Continuing.
    
    Breakpoint 1, create_scan_plan (root=0x2d50a00, best_path=0x2da9e98, flags=2) at createplan.c:514
    514     RelOptInfo *rel = best_path->parent;
    (gdb) c
    Continuing.
    
    Breakpoint 2, create_indexscan_plan (root=0x2d50a00, best_path=0x2da9e98, tlist=0x2db5250, scan_clauses=0x2da8998, 
        indexonly=false) at createplan.c:2536
    2536        List       *indexquals = best_path->indexquals;
    (gdb) 
    

赋值并获取相关信息

    
    
    2536        List       *indexquals = best_path->indexquals;
    (gdb) n
    2537        List       *indexorderbys = best_path->indexorderbys;
    (gdb) 
    2538        Index       baserelid = best_path->path.parent->relid;
    (gdb) 
    2539        Oid         indexoid = best_path->indexinfo->indexoid;
    (gdb) 
    2544        List       *indexorderbyops = NIL;
    (gdb) 
    2548        Assert(baserelid > 0);
    (gdb) 
    2549        Assert(best_path->path.parent->rtekind == RTE_RELATION);
    (gdb) 
    2555        stripped_indexquals = get_actual_clauses(indexquals);
    (gdb) n
    2561        fixed_indexquals = fix_indexqual_references(root, best_path);
    (gdb) 
    2566        fixed_indexorderbys = fix_indexorderby_references(root, best_path);
    (gdb) 
    2596        qpqual = NIL;
    (gdb) 
    (gdb) p baserelid
    $3 = 1
    (gdb) p indexoid
    $4 = 16738
    #t_dwxx_pkey
    testdb=# select relname from pg_class where oid=16738;
       relname   
    -------------     t_dwxx_pkey
    (1 row)
    

遍历扫描条件

    
    
    2597        foreach(l, scan_clauses)
    (gdb) p *scan_clauses
    $5 = {type = T_List, length = 1, head = 0x2da8970, tail = 0x2da8970}
    
    

重复的约束条件,不需要处理

    
    
    2603            if (list_member_ptr(indexquals, rinfo))
    (gdb) 
    2604                continue;           /* simple duplicate */
    

对条件进行排序&规约

    
    
    2614        qpqual = order_qual_clauses(root, qpqual);
    (gdb) n
    2617        qpqual = extract_actual_clauses(qpqual, false);
    (gdb) n
    2628        if (best_path->path.param_info)
    (gdb) p *qpqual
    Cannot access memory at address 0x0
    

创建IndexScan节点

    
    
    (gdb) n
    2642        if (indexorderbys)
    (gdb) 
    2673        if (indexonly)
    (gdb) 
    2683            scan_plan = (Scan *) make_indexscan(tlist,
    (gdb) 
    2694        copy_generic_path_info(&scan_plan->plan, &best_path->path);
    (gdb) 
    2696        return scan_plan;
    (gdb) 
    2697    }
    (gdb) p *scan_plan
    $6 = {plan = {type = T_IndexScan, startup_cost = 0.28500000000000003, total_cost = 13.924222117799655, plan_rows = 2, 
        plan_width = 20, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2db5250, qual = 0x0, 
        lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, scanrelid = 1}
    (gdb) 
    

回到create_scan_plan

    
    
    (gdb) n
    create_scan_plan (root=0x2d50a00, best_path=0x2da9e98, flags=2) at createplan.c:633
    633             break;
    (gdb) 
    732     if (gating_clauses)
    (gdb) 
    735     return plan;
    (gdb) 
    736 }
    

调用完成.

### 四、参考资料

[createplan.c](https://doxygen.postgresql.org/createplan_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

