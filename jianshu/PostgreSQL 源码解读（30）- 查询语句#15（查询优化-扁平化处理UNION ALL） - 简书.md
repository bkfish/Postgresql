本文简单介绍了PG查询优化中对顶层UNION ALL语句进行的扁平化(flatten)处理过程。扁平化处理的目的是把UNION
ALL的集合操作转换为Append Relation以进行优化处理。

### 一、测试脚本

测试脚本：

    
    
    testdb=# select a.dwbh 
    from t_dwxx a
    union all
    select b.dwbh
    from t_grxx b
    union all
    select c.grbh
    from t_jfxx c;
    

查询树如下图所示：

  

![](https://upload-images.jianshu.io/upload_images/8194836-900ad7f3b377dd66.png)

查询树

### 二、源码解读

    
    
    /*
     * flatten_simple_union_all
     *      Try to optimize top-level UNION ALL structure into an appendrel
     *
     * If a query's setOperations tree consists entirely of simple UNION ALL
     * operations, flatten it into an append relation, which we can process more
     * intelligently than the general setops case.  Otherwise, do nothing.
     *
     * In most cases, this can succeed only for a top-level query, because for a
     * subquery in FROM, the parent query's invocation of pull_up_subqueries would
     * already have flattened the UNION via pull_up_simple_union_all.  But there
     * are a few cases we can support here but not in that code path, for example
     * when the subquery also contains ORDER BY.
     */
    void
    flatten_simple_union_all(PlannerInfo *root)
    {
        Query      *parse = root->parse;//查询树
        SetOperationStmt *topop;//集合操作语句
        Node       *leftmostjtnode;//最左边节点
        int         leftmostRTI;//最左边节点对应的RTI(rtable中的位置,即rtindex)
        RangeTblEntry *leftmostRTE;//最左边节点对应的RTE
        int         childRTI;//child的RTI
        RangeTblEntry *childRTE;//子RTE
        RangeTblRef *rtr;//RTR
    
        /* Shouldn't be called unless query has setops */
        topop = castNode(SetOperationStmt, parse->setOperations);//获取集合操作语句
        Assert(topop);
    
        /* Can't optimize away a recursive UNION */
        if (root->hasRecursion)
            return;//不支持递归UNION
    
        /*
         * Recursively check the tree of set operations.  If not all UNION ALL
         * with identical column types, punt.
         */
        if (!is_simple_union_all_recurse((Node *) topop, parse, topop->colTypes))
            return;//不是简单的UNION ALL,返回
    
        /*
         * Locate the leftmost leaf query in the setops tree.  The upper query's
         * Vars all refer to this RTE (see transformSetOperationStmt).
         */
        leftmostjtnode = topop->larg;//左边的child
        while (leftmostjtnode && IsA(leftmostjtnode, SetOperationStmt))
            leftmostjtnode = ((SetOperationStmt *) leftmostjtnode)->larg;//获取最左边的叶子节点(非SetOperationStmt)
        Assert(leftmostjtnode && IsA(leftmostjtnode, RangeTblRef));
        leftmostRTI = ((RangeTblRef *) leftmostjtnode)->rtindex;//最左边节点的RTI
        leftmostRTE = rt_fetch(leftmostRTI, parse->rtable);//从rtable中获取leftmostRTI指向的RTE
        Assert(leftmostRTE->rtekind == RTE_SUBQUERY);//该RTE一定是子查询
    
        /*
         * Make a copy of the leftmost RTE and add it to the rtable.  This copy
         * will represent the leftmost leaf query in its capacity as a member of
         * the appendrel.  The original will represent the appendrel as a whole.
         * (We must do things this way because the upper query's Vars have to be
         * seen as referring to the whole appendrel.)
         */
        childRTE = copyObject(leftmostRTE);//Make a copy
        parse->rtable = lappend(parse->rtable, childRTE);//把RTE追加到上层rtable
        childRTI = list_length(parse->rtable);//相应的rtindex
    
        /* Modify the setops tree to reference the child copy */
        ((RangeTblRef *) leftmostjtnode)->rtindex = childRTI;//调整RTR的rtindex
    
        /* Modify the formerly-leftmost RTE to mark it as an appendrel parent */
        leftmostRTE->inh = true;
    
        /*
         * Form a RangeTblRef for the appendrel, and insert it into FROM.  The top
         * Query of a setops tree should have had an empty FromClause initially.
         */
        rtr = makeNode(RangeTblRef);//构造一个RTR
        rtr->rtindex = leftmostRTI;//rtindex=最左边节点的rtindex
        Assert(parse->jointree->fromlist == NIL);//Query的jointree(FromExpr)->fromlist应为NULL
        parse->jointree->fromlist = list_make1(rtr);//把RTR放到jointree的fromlist中
    
        /*
         * Now pretend the query has no setops.  We must do this before trying to
         * do subquery pullup, because of Assert in pull_up_simple_subquery.
         */
        parse->setOperations = NULL;//集合操作设置为NULL
    
        /*
         * Build AppendRelInfo information, and apply pull_up_subqueries to the
         * leaf queries of the UNION ALL.  (We must do that now because they
         * weren't previously referenced by the jointree, and so were missed by
         * the main invocation of pull_up_subqueries.)
         */
        pull_up_union_leaf_queries((Node *) topop, root, leftmostRTI, parse, 0);
    }
     
    /*
     * pull_up_union_leaf_queries -- recursive guts of pull_up_simple_union_all
     *
     * Build an AppendRelInfo for each leaf query in the setop tree, and then
     * apply pull_up_subqueries to the leaf query.
     *
     * Note that setOpQuery is the Query containing the setOp node, whose tlist
     * contains references to all the setop output columns.  When called from
     * pull_up_simple_union_all, this is *not* the same as root->parse, which is
     * the parent Query we are pulling up into.
     *
     * parentRTindex is the appendrel parent's index in root->parse->rtable.
     *
     * The child RTEs have already been copied to the parent.  childRToffset
     * tells us where in the parent's range table they were copied.  When called
     * from flatten_simple_union_all, childRToffset is 0 since the child RTEs
     * were already in root->parse->rtable and no RT index adjustment is needed.
     */
    static void
    pull_up_union_leaf_queries(Node *setOp, PlannerInfo *root, int parentRTindex,
                               Query *setOpQuery, int childRToffset)
    {
        if (IsA(setOp, RangeTblRef))//RTR
        {
            RangeTblRef *rtr = (RangeTblRef *) setOp;
            int         childRTindex;
            AppendRelInfo *appinfo;
    
            /*
             * Calculate the index in the parent's range table
             */
            childRTindex = childRToffset + rtr->rtindex;
    
            /*
             * Build a suitable AppendRelInfo, and attach to parent's list.
             */
            appinfo = makeNode(AppendRelInfo);//构造AppendRelInfo
            appinfo->parent_relid = parentRTindex;
            appinfo->child_relid = childRTindex;
            appinfo->parent_reltype = InvalidOid;
            appinfo->child_reltype = InvalidOid;
            make_setop_translation_list(setOpQuery, childRTindex,
                                        &appinfo->translated_vars);//把相关的Vars添加到AppendRelInfo中
            appinfo->parent_reloid = InvalidOid;
            root->append_rel_list = lappend(root->append_rel_list, appinfo);//添加到PlannerInfo中
    
            /*
             * Recursively apply pull_up_subqueries to the new child RTE.  (We
             * must build the AppendRelInfo first, because this will modify it.)
             * Note that we can pass NULL for containing-join info even if we're
             * actually under an outer join, because the child's expressions
             * aren't going to propagate up to the join.  Also, we ignore the
             * possibility that pull_up_subqueries_recurse() returns a different
             * jointree node than what we pass it; if it does, the important thing
             * is that it replaced the child relid in the AppendRelInfo node.
             */
            rtr = makeNode(RangeTblRef);
            rtr->rtindex = childRTindex;
            (void) pull_up_subqueries_recurse(root, (Node *) rtr,
                                              NULL, NULL, appinfo, false);//提升子查询
        }
        else if (IsA(setOp, SetOperationStmt))//类型为SetOperationStmt
        {
            SetOperationStmt *op = (SetOperationStmt *) setOp;
    
            /* Recurse to reach leaf queries */
            pull_up_union_leaf_queries(op->larg, root, parentRTindex, setOpQuery,
                                       childRToffset);//左Child
            pull_up_union_leaf_queries(op->rarg, root, parentRTindex, setOpQuery,
                                       childRToffset);//右Child
        }
        else
        {
            elog(ERROR, "unrecognized node type: %d",
                 (int) nodeTag(setOp));
        }
    }
    

### 三、基础信息

**数据结构/宏定义**  
_1、SetOperationStmt_

    
    
     /* ----------------------      *      Set Operation node for post-analysis query trees
      *
      * After parse analysis, a SELECT with set operations is represented by a
      * top-level Query node containing the leaf SELECTs as subqueries in its
      * range table.  Its setOperations field shows the tree of set operations,
      * with leaf SelectStmt nodes replaced by RangeTblRef nodes, and internal
      * nodes replaced by SetOperationStmt nodes.  Information about the output
      * column types is added, too.  (Note that the child nodes do not necessarily
      * produce these types directly, but we've checked that their output types
      * can be coerced to the output column type.)  Also, if it's not UNION ALL,
      * information about the types' sort/group semantics is provided in the form
      * of a SortGroupClause list (same representation as, eg, DISTINCT).
      * The resolved common column collations are provided too; but note that if
      * it's not UNION ALL, it's okay for a column to not have a common collation,
      * so a member of the colCollations list could be InvalidOid even though the
      * column has a collatable type.
      * ----------------------      */
     typedef enum SetOperation
     {
         SETOP_NONE = 0,
         SETOP_UNION,
         SETOP_INTERSECT,
         SETOP_EXCEPT
     } SetOperation;
    
     typedef struct SetOperationStmt
     {
         NodeTag     type;//节点类型
         SetOperation op;            /* 操作类型,type of set op */
         bool        all;            /* 是否UNION ALL,ALL specified? */
         Node       *larg;           /* left child,左Child,类型为RTR或SetOperationStmt */
         Node       *rarg;           /* right child,右Child,类型为RTR或SetOperationStmt */
         /* Eventually add fields for CORRESPONDING spec here */
     
         /* Fields derived during parse analysis: */
         List       *colTypes;       /* OID list of output column type OIDs,数据列类型 */
         List       *colTypmods;     /* integer list of output column typmods,精度/刻度信息 */
         List       *colCollations;  /* OID list of output column collation OIDs,collation信息*/
         List       *groupClauses;   /* a list of SortGroupClause's,SortGroup语句*/
         /* groupClauses is NIL if UNION ALL, but must be set otherwise */
     } SetOperationStmt;
     
    

**依赖的函数**  
_1、make_setop_translation_list_

    
    
     /*
      * make_setop_translation_list
      *    Build the list of translations from parent Vars to child Vars for
      *    a UNION ALL member.  (At this point it's just a simple list of
      *    referencing Vars, but if we succeed in pulling up the member
      *    subquery, the Vars will get replaced by pulled-up expressions.)
      */
     static void
     make_setop_translation_list(Query *query, Index newvarno,
                                 List **translated_vars)
     {
         List       *vars = NIL;
         ListCell   *l;
     
         foreach(l, query->targetList)
         {
             TargetEntry *tle = (TargetEntry *) lfirst(l);
     
             if (tle->resjunk)
                 continue;
     
             vars = lappend(vars, makeVarFromTargetEntry(newvarno, tle));
         }
     
         *translated_vars = vars;
     }
    

### 四、跟踪分析

测试脚本见上,启动gdb跟踪:

    
    
    (gdb) b flatten_simple_union_all
    Breakpoint 1 at 0x77f964: file prepjointree.c, line 2355.
    (gdb) c
    Continuing.
    
    Breakpoint 1, flatten_simple_union_all (root=0x16a6338) at prepjointree.c:2355
    2355        Query      *parse = root->parse;
    #输入参数
    #1.root,见前面章节,root->parse见先前小节的图(查询树)
    ...
    #获取最左边的叶子节点(类型为RTR)
    2384        while (leftmostjtnode && IsA(leftmostjtnode, SetOperationStmt))
    (gdb) 
    2387        leftmostRTI = ((RangeTblRef *) leftmostjtnode)->rtindex;
    (gdb) p *leftmostjtnode
    $9 = {type = T_RangeTblRef}
    ...
    #添加到root->rtable中
    (gdb) n
    2399        parse->rtable = lappend(parse->rtable, childRTE);
    (gdb) 
    2400        childRTI = list_length(parse->rtable);
    #原来的rtable为3个子查询,在最后面添加1个
    (gdb) p *parse->rtable
    $12 = {type = T_List, length = 4, head = 0x15e9dc8, tail = 0x16b09d0}
    (gdb) p leftmostRTI
    $19 = 1
    #copy了一份"子查询",放在rtable的最后面
    (gdb) p *((RangeTblEntry *)parse->rtable->head->next->next->next->data.ptr_value)->alias
    $20 = {type = T_Alias, aliasname = 0x16b08d8 "*SELECT* 1", colnames = 0x0}
    (gdb) p *((RangeTblEntry *)parse->rtable->head->data.ptr_value)->alias
    $21 = {type = T_Alias, aliasname = 0x15e9cd0 "*SELECT* 1", colnames = 0x0}
    (gdb) 
    #构造RTR(rtindex=1),放在jointree的fromlist中
    2406        leftmostRTE->inh = true;
    (gdb) 
    2412        rtr = makeNode(RangeTblRef);
    (gdb) 
    2413        rtr->rtindex = leftmostRTI;
    (gdb) 
    2415        parse->jointree->fromlist = list_make1(rtr);
    (gdb) p *rtr
    $23 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) p *(RangeTblRef *) leftmostjtnode
    $24 = {type = T_RangeTblRef, rtindex = 4}
    (gdb) p *parse->jointree->fromlist
    $26 = {type = T_List, length = 1, head = 0x16b0a08, tail = 0x16b0a08}
    #进入pull_up_union_leaf_queries
    (gdb) n
    2429        pull_up_union_leaf_queries((Node *) topop, root, leftmostRTI, parse, 0);
    (gdb) step
    pull_up_union_leaf_queries (setOp=0x15c59c8, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1344
    1344        if (IsA(setOp, RangeTblRef))
    #输入参数
    #1.setOp,SetOperationStmt
    #2.root,同flatten_simple_union_all
    #3.leftmostRTI=1,最左边节点的rtindex
    #4.parse,查询树
    #5.childRToffset,RT的初始偏移量
    1344        if (IsA(setOp, RangeTblRef))
    #setOp为SetOperationStmt类型,递归调用SetOperationStmt->larg->larg
    (gdb) n
    1383        else if (IsA(setOp, SetOperationStmt))
    ...
    1388            pull_up_union_leaf_queries(op->larg, root, parentRTindex, setOpQuery,
    (gdb) step
    pull_up_union_leaf_queries (setOp=0x15c5a18, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1344
    1344        if (IsA(setOp, RangeTblRef))
    1344        if (IsA(setOp, RangeTblRef))
    (gdb) n
    1383        else if (IsA(setOp, SetOperationStmt))
    (gdb) 
    1385            SetOperationStmt *op = (SetOperationStmt *) setOp;
    (gdb) 
    1388            pull_up_union_leaf_queries(op->larg, root, parentRTindex, setOpQuery,
    (gdb) step
    pull_up_union_leaf_queries (setOp=0x15e9e18, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1344
    1344        if (IsA(setOp, RangeTblRef))
    ...
    1346            RangeTblRef *rtr = (RangeTblRef *) setOp;
    (gdb) 
    1353            childRTindex = childRToffset + rtr->rtindex;
    (gdb) 
    1358            appinfo = makeNode(AppendRelInfo);
    #最左边的RTR在flatten_simple_union_all中已调整为4
    (gdb) p *rtr
    $27 = {type = T_RangeTblRef, rtindex = 4}
    (gdb) p childRTindex
    $28 = 4
    #构造AppendRelInfo,插入到root->append_rel_list中
    (gdb) n
    1359            appinfo->parent_relid = parentRTindex;
    ...
    #进入pull_up_subqueries_recurse
    (gdb) step
    pull_up_subqueries_recurse (root=0x16a6338, jtnode=0x16b0b98, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x16b0a58, deletion_ok=false) at prepjointree.c:680
    680     if (IsA(jtnode, RangeTblRef))
    #查询树的rtable中增加RTE(RTE_RELATION)
    (gdb) p *parse->rtable
    $47 = {type = T_List, length = 5, head = 0x15e9dc8, tail = 0x16b0cf0}
    (gdb) p *(RangeTblEntry *)parse->rtable->tail->data.ptr_value
    $49 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16394, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x16b0e20, eref = 0x16b0e68, 
      lateral = false, inh = true, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16b0fe8, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    (gdb) n
    pull_up_subqueries_recurse (root=0x16a6338, jtnode=0x16b0b98, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x16b0a58, deletion_ok=false) at prepjointree.c:853
    853 }
    (gdb) 
    pull_up_union_leaf_queries (setOp=0x15e9e18, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1398
    1398    }
    (gdb) n
    #处理SetOperationStmt->larg(SetOperationStmt)->rarg
    pull_up_union_leaf_queries (setOp=0x15c5a18, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1390
    1390            pull_up_union_leaf_queries(op->rarg, root, parentRTindex, setOpQuery,
    (gdb) 
    1398    }
    (gdb)
    #处理SetOperationStmt->rarg 
    pull_up_union_leaf_queries (setOp=0x15c59c8, root=0x16a6338, parentRTindex=1, setOpQuery=0x15c58b8, childRToffset=0)
        at prepjointree.c:1390
    1390            pull_up_union_leaf_queries(op->rarg, root, parentRTindex, setOpQuery,
    (gdb) 
    1398    }
    (gdb) 
    flatten_simple_union_all (root=0x16a6338) at prepjointree.c:2430
    2430    }
    #
    (gdb) p *root
    $54 = {type = T_PlannerInfo, parse = 0x15c58b8, glob = 0x15eb8b0, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x0, simple_rel_array_size = 0, simple_rte_array = 0x0, all_baserels = 0x0, 
      nullable_baserels = 0x0, join_rel_list = 0x0, join_rel_hash = 0x0, join_rel_level = 0x0, join_cur_level = 0, 
      init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, eq_classes = 0x0, canon_pathkeys = 0x0, 
      left_join_clauses = 0x0, right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, 
      append_rel_list = 0x16b0b68, rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, 
      group_pathkeys = 0x0, window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 
        0x0}, processed_tlist = 0x0, grouping_map = 0x0, minmax_aggs = 0x0, planner_cxt = 0x15c3e90, total_table_pages = 0, 
      tuple_fraction = 0, limit_tuples = 0, qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = false, 
      hasLateralRTEs = false, hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, 
      hasRecursion = false, wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    (gdb) p *root->append_rel_list
    $55 = {type = T_List, length = 3, head = 0x16b0b48, tail = 0x16b2790}
    (gdb) p root->parse
    $56 = (Query *) 0x15c58b8
    (gdb) p *root->parse
    $57 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, hasForUpdate = false, 
      hasRowSecurity = false, cteList = 0x0, rtable = 0x15e9de8, jointree = 0x15eb880, targetList = 0x16b3348, 
      override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, groupClause = 0x0, groupingSets = 0x0, 
      havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, 
      rowMarks = 0x0, setOperations = 0x0, constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 104}
    #rtable中已有7个元素,其中第4个是子查询,第5-7个是实际的RTE_RELATION(relid分别是16394/16397/16400)
    (gdb) p *root->parse->rtable
    $58 = {type = T_List, length = 7, head = 0x15e9dc8, tail = 0x16b2908}
    (gdb) 
    #第4个元素
    (gdb) p *((RangeTblEntry *)root->parse->rtable->head->next->next->next->data.ptr_value)
    $80 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x15c57a8, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, 
      coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x16b08a8, eref = 0x16b08f8, 
      lateral = false, inh = false, inFromCl = false, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)root->parse->rtable->head->next->next->next->data.ptr_value)->eref
    $81 = {type = T_Alias, aliasname = 0x16b0928 "*SELECT* 1", colnames = 0x16b0948}
    #第5个元素
    (gdb) p *((RangeTblEntry *)root->parse->rtable->head->next->next->next->next->data.ptr_value)
    $82 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16394, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x16b0e20, eref = 0x16b0e68, 
      lateral = false, inh = true, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16b0fe8, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)root->parse->rtable->head->next->next->next->next->data.ptr_value)->eref
    $83 = {type = T_Alias, aliasname = 0x16b0e98 "a", colnames = 0x16b0eb0}
    
    #root->append_rel_list中有3个元素,分别对应rtable中的5-7
    (gdb) p *root->append_rel_list
    $69 = {type = T_List, length = 3, head = 0x16b0b48, tail = 0x16b2790}
    (gdb) p *(Node *)root->append_rel_list->head->data.ptr_value
    $70 = {type = T_AppendRelInfo}
    (gdb) p *(AppendRelInfo *)root->append_rel_list->head->data.ptr_value
    $71 = {type = T_AppendRelInfo, parent_relid = 1, child_relid = 5, parent_reltype = 0, child_reltype = 0, 
      translated_vars = 0x16b33e8, parent_reloid = 0}
    #查询树的jointree
    (gdb) p *root->parse->jointree
    $87 = {type = T_FromExpr, fromlist = 0x16b0a28, quals = 0x0}
    (gdb) p *root->parse->jointree->fromlist
    $88 = {type = T_List, length = 1, head = 0x16b0a08, tail = 0x16b0a08}
    (gdb) p *(RangeTblRef *)root->parse->jointree->fromlist->head->data.ptr_value
    $90 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) n
    subquery_planner (glob=0x15eb8b0, parse=0x15c58b8, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:680
    680     root->hasJoinRTEs = false;
    #DONE!
    
    

优化后的查询树如下图所示:

  

![](https://upload-images.jianshu.io/upload_images/8194836-42372216e676e0a7.png)

查询树(优化后)

### 五、小结

UNION ALL优化过程：把集合操作扁平化为AppendRelInfo，通过增加AppendRelInfo、修改rtable等方式实现。

参考资料：  
[prepjointree.c](https://doxygen.postgresql.org/prepjointree_8c_source.html#l02353)  
[list.c](https://doxygen.postgresql.org/prepjointree_8c_source.html#l01408)

