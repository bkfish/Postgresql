本节简单介绍了PG查询逻辑优化中的子查询(subQuery)上拉优化,包括查询优化中子查询的基本概念，同时介绍了主函数以及使用gdb跟踪分析上拉函数的部分处理逻辑。  
与上拉子链接类似,上拉子查询的目的也是为了使用连接方式提升性能。

### 一、上拉子查询简介

下面是README文件中对上拉子查询的介绍.  
在RTE中出现的子查询在外部查询规划时可以作为黑箱来看待,但这样的处理方式可能会导致性能低下的执行计划,因此对于"简单"的子查询,优化器会尝试上拉子查询,与上层RTE形成连接以提升性能.

    
    
    Pulling Up Subqueries
    ---------------------    
    As we described above, a subquery appearing in the range table is planned
    independently and treated as a "black box" during planning of the outer
    query.  This is necessary when the subquery uses features such as
    aggregates, GROUP, or DISTINCT.  But if the subquery is just a simple
    scan or join, treating the subquery as a black box may produce a poor plan
    compared to considering it as part of the entire plan search space.
    Therefore, at the start of the planning process the planner looks for
    simple subqueries and pulls them up into the main query's jointree.
    
    Pulling up a subquery may result in FROM-list joins appearing below the top
    of the join tree.  Each FROM-list is planned using the dynamic-programming
    search method described above.
    
    If pulling up a subquery produces a FROM-list as a direct child of another
    FROM-list, then we can merge the two FROM-lists together.  Once that's
    done, the subquery is an absolutely integral part of the outer query and
    will not constrain the join tree search space at all.  However, that could
    result in unpleasant growth of planning time, since the dynamic-programming
    search has runtime exponential in the number of FROM-items considered.
    Therefore, we don't merge FROM-lists if the result would have too many
    FROM-items in one list.
    

子查询上拉样例:

    
    
    testdb=# explain analyze select a.*,b.grbh 
    testdb-# from t_dwxx a,(select s.dwbh,s.grbh from t_grxx s where s.grbh > '900') b
    testdb-# where a.dwbh = b.dwbh;
                                                        QUERY PLAN                                                     
    -------------------------------------------------------------------------------------------------------------------     Hash Join  (cost=13.60..31.64 rows=130 width=512) (actual time=0.038..0.042 rows=3 loops=1)
       Hash Cond: ((s.dwbh)::text = (a.dwbh)::text)
       ->  Seq Scan on t_grxx s  (cost=0.00..16.12 rows=163 width=76) (actual time=0.015..0.016 rows=3 loops=1)
             Filter: ((grbh)::text > '900'::text)
       ->  Hash  (cost=11.60..11.60 rows=160 width=474) (actual time=0.009..0.009 rows=3 loops=1)
             Buckets: 1024  Batches: 1  Memory Usage: 9kB
             ->  Seq Scan on t_dwxx a  (cost=0.00..11.60 rows=160 width=474) (actual time=0.003..0.004 rows=3 loops=1)
     Planning Time: 11.114 ms
     Execution Time: 0.126 ms
    (9 rows)
    

子查询上拉形成Hash Join.

### 二、源码解读

本节使用的查询语句见简介中的SQL语句,其相应的查询树如下图所示:

  

![](https://upload-images.jianshu.io/upload_images/8194836-13063115b119f7ce.png)

查询树

子查询上拉在函数pull_up_subqueries中实现,该函数调用pull_up_subqueries_recurse函数递归实现子查询上拉.  
**pull_up_subqueries**

    
    
    /*
      * pull_up_subqueries
      *      Look for subqueries in the rangetable that can be pulled up into
      *      the parent query.  If the subquery has no special features like
      *      grouping/aggregation then we can merge it into the parent's jointree.
      *      Also, subqueries that are simple UNION ALL structures can be
      *      converted into "append relations".
      */
     void
     pull_up_subqueries(PlannerInfo *root)
     {
         /* Top level of jointree must always be a FromExpr */
         Assert(IsA(root->parse->jointree, FromExpr));
         /* Reset flag saying we need a deletion cleanup pass */
         root->hasDeletedRTEs = false;
         /* Recursion starts with no containing join nor appendrel */
         root->parse->jointree = (FromExpr *)
             pull_up_subqueries_recurse(root, (Node *) root->parse->jointree,
                                        NULL, NULL, NULL, false);
         /* Apply cleanup phase if necessary */
         if (root->hasDeletedRTEs)
             root->parse->jointree = (FromExpr *)
                 pull_up_subqueries_cleanup((Node *) root->parse->jointree);
         Assert(IsA(root->parse->jointree, FromExpr));
     }
    
    

**pull_up_subqueries_recurse**

    
    
     
     /*
      * pull_up_subqueries_recurse
      *      Recursive guts of pull_up_subqueries.
      *
      * This recursively processes the jointree and returns a modified jointree.
      * Or, if it's valid to drop the current node from the jointree completely,
      * it returns NULL.
      *
      * If this jointree node is within either side of an outer join, then
      * lowest_outer_join references the lowest such JoinExpr node; otherwise
      * it is NULL.  We use this to constrain the effects of LATERAL subqueries.
      *
      * If this jointree node is within the nullable side of an outer join, then
      * lowest_nulling_outer_join references the lowest such JoinExpr node;
      * otherwise it is NULL.  This forces use of the PlaceHolderVar mechanism for
      * references to non-nullable targetlist items, but only for references above
      * that join.
      *
      * If we are looking at a member subquery of an append relation,
      * containing_appendrel describes that relation; else it is NULL.
      * This forces use of the PlaceHolderVar mechanism for all non-Var targetlist
      * items, and puts some additional restrictions on what can be pulled up.
      *
      * deletion_ok is true if the caller can cope with us returning NULL for a
      * deletable leaf node (for example, a VALUES RTE that could be pulled up).
      * If it's false, we'll avoid pullup in such cases.
      *
      * A tricky aspect of this code is that if we pull up a subquery we have
      * to replace Vars that reference the subquery's outputs throughout the
      * parent query, including quals attached to jointree nodes above the one
      * we are currently processing!  We handle this by being careful not to
      * change the jointree structure while recursing: no nodes other than leaf
      * RangeTblRef entries and entirely-empty FromExprs will be replaced or
      * deleted.  Also, we can't turn pullup_replace_vars loose on the whole
      * jointree, because it'll return a mutated copy of the tree; we have to
      * invoke it just on the quals, instead.  This behavior is what makes it
      * reasonable to pass lowest_outer_join and lowest_nulling_outer_join as
      * pointers rather than some more-indirect way of identifying the lowest
      * OJs.  Likewise, we don't replace append_rel_list members but only their
      * substructure, so the containing_appendrel reference is safe to use.
      *
      * Because of the rule that no jointree nodes with substructure can be
      * replaced, we cannot fully handle the case of deleting nodes from the tree:
      * when we delete one child of a JoinExpr, we need to replace the JoinExpr
      * with a FromExpr, and that can't happen here.  Instead, we set the
      * root->hasDeletedRTEs flag, which tells pull_up_subqueries() that an
      * additional pass over the tree is needed to clean up.
      */
     /*
     输入参数:
        root-计划器相关信息
        jtnode-需要处理的Node(jointree)
        lowest_outer_join-如该节点位于外连接的任意一侧,则该指针指向此节点
        lowest_nulling_outer_join-如该节点位于外连接的可空一侧,,则该指针指向此节点
        containing_appendrel-Append操作中的Relation
        deletion_ok-调用方可处理在可删除的叶子节点的情况下返回NULL,此值为true
     输出参数:
     */
     static Node *
     pull_up_subqueries_recurse(PlannerInfo *root, Node *jtnode,
                                JoinExpr *lowest_outer_join,
                                JoinExpr *lowest_nulling_outer_join,
                                AppendRelInfo *containing_appendrel,
                                bool deletion_ok)
     {
         Assert(jtnode != NULL);
         if (IsA(jtnode, RangeTblRef))//如为RTR
         {
             //获取该RTR相应的RTE
             int         varno = ((RangeTblRef *) jtnode)->rtindex;
             RangeTblEntry *rte = rt_fetch(varno, root->parse->rtable);
     
             /*
              * Is this a subquery RTE, and if so, is the subquery simple enough to
              * pull up?
              *
              * If we are looking at an append-relation member, we can't pull it up
              * unless is_safe_append_member says so.
              */
             if (rte->rtekind == RTE_SUBQUERY &&
                 is_simple_subquery(rte->subquery, rte,
                                    lowest_outer_join, deletion_ok) &&
                 (containing_appendrel == NULL ||
                  is_safe_append_member(rte->subquery)))//简单子查询
                 return pull_up_simple_subquery(root, jtnode, rte,
                                                lowest_outer_join,
                                                lowest_nulling_outer_join,
                                                containing_appendrel,
                                                deletion_ok);
     
             /*
              * Alternatively, is it a simple UNION ALL subquery?  If so, flatten
              * into an "append relation".
              *
              * It's safe to do this regardless of whether this query is itself an
              * appendrel member.  (If you're thinking we should try to flatten the
              * two levels of appendrel together, you're right; but we handle that
              * in set_append_rel_pathlist, not here.)
              */
             if (rte->rtekind == RTE_SUBQUERY &&
                 is_simple_union_all(rte->subquery))//UNION ALL子查询
                 return pull_up_simple_union_all(root, jtnode, rte);
     
             /*
              * Or perhaps it's a simple VALUES RTE?
              *
              * We don't allow VALUES pullup below an outer join nor into an
              * appendrel (such cases are impossible anyway at the moment).
              */
             if (rte->rtekind == RTE_VALUES &&
                 lowest_outer_join == NULL &&
                 containing_appendrel == NULL &&
                 is_simple_values(root, rte, deletion_ok))//VALUES子查询
                 return pull_up_simple_values(root, jtnode, rte);
     
             /* Otherwise, do nothing at this node. */
         }
         else if (IsA(jtnode, FromExpr))//如为FromExpr
         {
             FromExpr   *f = (FromExpr *) jtnode;
             bool        have_undeleted_child = false;
             ListCell   *l;
     
             Assert(containing_appendrel == NULL);
     
             /*
              * If the FromExpr has quals, it's not deletable even if its parent
              * would allow deletion.
              */
             if (f->quals)
                 deletion_ok = false;
     
             foreach(l, f->fromlist)
             {
                 /*
                  * In a non-deletable FromExpr, we can allow deletion of child
                  * nodes so long as at least one child remains; so it's okay
                  * either if any previous child survives, or if there's more to
                  * come.  If all children are deletable in themselves, we'll force
                  * the last one to remain unflattened.
                  *
                  * As a separate matter, we can allow deletion of all children of
                  * the top-level FromExpr in a query, since that's a special case
                  * anyway.
                  */
                 bool        sub_deletion_ok = (deletion_ok ||
                                                have_undeleted_child ||
                                                lnext(l) != NULL ||
                                                f == root->parse->jointree);
     
                 lfirst(l) = pull_up_subqueries_recurse(root, lfirst(l),
                                                        lowest_outer_join,
                                                        lowest_nulling_outer_join,
                                                        NULL,
                                                        sub_deletion_ok);//递归调用
                 if (lfirst(l) != NULL)
                     have_undeleted_child = true;
             }
     
             if (deletion_ok && !have_undeleted_child)
             {
                 /* OK to delete this FromExpr entirely */
                 root->hasDeletedRTEs = true;    /* probably is set already */
                 return NULL;
             }
         }
         else if (IsA(jtnode, JoinExpr))//如为JoinExpr
         {
             JoinExpr   *j = (JoinExpr *) jtnode;
     
             Assert(containing_appendrel == NULL);
             /* Recurse, being careful to tell myself when inside outer join */
             switch (j->jointype)
             {
                 case JOIN_INNER:
     
                     /*
                      * INNER JOIN can allow deletion of either child node, but not
                      * both.  So right child gets permission to delete only if
                      * left child didn't get removed.
                      */
                     j->larg = pull_up_subqueries_recurse(root, j->larg,
                                                          lowest_outer_join,
                                                          lowest_nulling_outer_join,
                                                          NULL,
                                                          true);
                     j->rarg = pull_up_subqueries_recurse(root, j->rarg,
                                                          lowest_outer_join,
                                                          lowest_nulling_outer_join,
                                                          NULL,
                                                          j->larg != NULL);
                     break;
                 case JOIN_LEFT:
                 case JOIN_SEMI:
                 case JOIN_ANTI:
                     j->larg = pull_up_subqueries_recurse(root, j->larg,
                                                          j,
                                                          lowest_nulling_outer_join,
                                                          NULL,
                                                          false);
                     j->rarg = pull_up_subqueries_recurse(root, j->rarg,
                                                          j,
                                                          j,
                                                          NULL,
                                                          false);
                     break;
                 case JOIN_FULL:
                     j->larg = pull_up_subqueries_recurse(root, j->larg,
                                                          j,
                                                          j,
                                                          NULL,
                                                          false);
                     j->rarg = pull_up_subqueries_recurse(root, j->rarg,
                                                          j,
                                                          j,
                                                          NULL,
                                                          false);
                     break;
                 case JOIN_RIGHT:
                     j->larg = pull_up_subqueries_recurse(root, j->larg,
                                                          j,
                                                          j,
                                                          NULL,
                                                          false);
                     j->rarg = pull_up_subqueries_recurse(root, j->rarg,
                                                          j,
                                                          lowest_nulling_outer_join,
                                                          NULL,
                                                          false);
                     break;
                 default:
                     elog(ERROR, "unrecognized join type: %d",
                          (int) j->jointype);
                     break;
             }
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
         return jtnode;
     }
     
    

* * *
    
    
     
     /*
      * pull_up_simple_subquery
      *      Attempt to pull up a single simple subquery.
      *
      * jtnode is a RangeTblRef that has been tentatively identified as a simple
      * subquery by pull_up_subqueries.  We return the replacement jointree node,
      * or NULL if the subquery can be deleted entirely, or jtnode itself if we
      * determine that the subquery can't be pulled up after all.
      *
      * rte is the RangeTblEntry referenced by jtnode.  Remaining parameters are
      * as for pull_up_subqueries_recurse.
      */
     static Node *
     pull_up_simple_subquery(PlannerInfo *root, Node *jtnode, RangeTblEntry *rte,
                             JoinExpr *lowest_outer_join,
                             JoinExpr *lowest_nulling_outer_join,
                             AppendRelInfo *containing_appendrel,
                             bool deletion_ok)
     {
         Query      *parse = root->parse;//查询树
         int         varno = ((RangeTblRef *) jtnode)->rtindex;//RTR中的index,指向rtable中的位置
         Query      *subquery;//子查询
         PlannerInfo *subroot;//子root
         int         rtoffset;//rtable中的偏移
         pullup_replace_vars_context rvcontext;//上下文
         ListCell   *lc;//临时变量
     
         /*
          * Need a modifiable copy of the subquery to hack on.  Even if we didn't
          * sometimes choose not to pull up below, we must do this to avoid
          * problems if the same subquery is referenced from multiple jointree
          * items (which can't happen normally, but might after rule rewriting).
          */
         subquery = copyObject(rte->subquery);//子查询
     
         /*
          * Create a PlannerInfo data structure for this subquery.
          *
          * NOTE: the next few steps should match the first processing in
          * subquery_planner().  Can we refactor to avoid code duplication, or
          * would that just make things uglier?
          */
         //为子查询构建PlannerInfo,尝试对此子查询进行上拉
         subroot = makeNode(PlannerInfo);
         subroot->parse = subquery;
         subroot->glob = root->glob;
         subroot->query_level = root->query_level;
         subroot->parent_root = root->parent_root;
         subroot->plan_params = NIL;
         subroot->outer_params = NULL;
         subroot->planner_cxt = CurrentMemoryContext;
         subroot->init_plans = NIL;
         subroot->cte_plan_ids = NIL;
         subroot->multiexpr_params = NIL;
         subroot->eq_classes = NIL;
         subroot->append_rel_list = NIL;
         subroot->rowMarks = NIL;
         memset(subroot->upper_rels, 0, sizeof(subroot->upper_rels));
         memset(subroot->upper_targets, 0, sizeof(subroot->upper_targets));
         subroot->processed_tlist = NIL;
         subroot->grouping_map = NULL;
         subroot->minmax_aggs = NIL;
         subroot->qual_security_level = 0;
         subroot->inhTargetKind = INHKIND_NONE;
         subroot->hasRecursion = false;
         subroot->wt_param_id = -1;
         subroot->non_recursive_path = NULL;
     
         /* No CTEs to worry about */
         Assert(subquery->cteList == NIL);
     
         /*
          * Pull up any SubLinks within the subquery's quals, so that we don't
          * leave unoptimized SubLinks behind.
          */
         if (subquery->hasSubLinks)//子链接?上拉子链接
             pull_up_sublinks(subroot);
     
         /*
          * Similarly, inline any set-returning functions in its rangetable.
          */
         inline_set_returning_functions(subroot);
     
         /*
          * Recursively pull up the subquery's subqueries, so that
          * pull_up_subqueries' processing is complete for its jointree and
          * rangetable.
          *
          * Note: it's okay that the subquery's recursion starts with NULL for
          * containing-join info, even if we are within an outer join in the upper
          * query; the lower query starts with a clean slate for outer-join
          * semantics.  Likewise, we needn't pass down appendrel state.
          */
         pull_up_subqueries(subroot);//递归上拉子查询中的子查询
     
         /*
          * Now we must recheck whether the subquery is still simple enough to pull
          * up.  If not, abandon processing it.
          *
          * We don't really need to recheck all the conditions involved, but it's
          * easier just to keep this "if" looking the same as the one in
          * pull_up_subqueries_recurse.
          */
         //子查询中子链接&子查询上拉后,再次检查,确保本次上拉没有问题
         if (is_simple_subquery(subquery, rte,
                                lowest_outer_join, deletion_ok) &&
             (containing_appendrel == NULL || is_safe_append_member(subquery)))
         {
             /* good to go */
         }
         else
         {
             /*
              * Give up, return unmodified RangeTblRef.
              *
              * Note: The work we just did will be redone when the subquery gets
              * planned on its own.  Perhaps we could avoid that by storing the
              * modified subquery back into the rangetable, but I'm not gonna risk
              * it now.
              */
             return jtnode;
         }
     
         /*
          * We must flatten any join alias Vars in the subquery's targetlist,
          * because pulling up the subquery's subqueries might have changed their
          * expansions into arbitrary expressions, which could affect
          * pullup_replace_vars' decisions about whether PlaceHolderVar wrappers
          * are needed for tlist entries.  (Likely it'd be better to do
          * flatten_join_alias_vars on the whole query tree at some earlier stage,
          * maybe even in the rewriter; but for now let's just fix this case here.)
          */
         //子查询中的targetList扁平化处理
         subquery->targetList = (List *)
             flatten_join_alias_vars(subroot, (Node *) subquery->targetList);
     
         /*
          * Adjust level-0 varnos in subquery so that we can append its rangetable
          * to upper query's.  We have to fix the subquery's append_rel_list as
          * well.
          */
         //调整Var.varno
         rtoffset = list_length(parse->rtable);
         OffsetVarNodes((Node *) subquery, rtoffset, 0);
         OffsetVarNodes((Node *) subroot->append_rel_list, rtoffset, 0);
     
         /*
          * Upper-level vars in subquery are now one level closer to their parent
          * than before.
          */
         //调整Var.varlevelsup
         IncrementVarSublevelsUp((Node *) subquery, -1, 1);
         IncrementVarSublevelsUp((Node *) subroot->append_rel_list, -1, 1);
     
         /*
          * The subquery's targetlist items are now in the appropriate form to
          * insert into the top query, except that we may need to wrap them in
          * PlaceHolderVars.  Set up required context data for pullup_replace_vars.
          */
         rvcontext.root = root;
         rvcontext.targetlist = subquery->targetList;
         rvcontext.target_rte = rte;
         if (rte->lateral)
             rvcontext.relids = get_relids_in_jointree((Node *) subquery->jointree,
                                                       true);
         else                        /* won't need relids */
             rvcontext.relids = NULL;
         rvcontext.outer_hasSubLinks = &parse->hasSubLinks;
         rvcontext.varno = varno;
         /* these flags will be set below, if needed */
         rvcontext.need_phvs = false;
         rvcontext.wrap_non_vars = false;
         /* initialize cache array with indexes 0 .. length(tlist) */
         rvcontext.rv_cache = palloc0((list_length(subquery->targetList) + 1) *
                                      sizeof(Node *));
     
         /*
          * If we are under an outer join then non-nullable items and lateral
          * references may have to be turned into PlaceHolderVars.
          */
         if (lowest_nulling_outer_join != NULL)
             rvcontext.need_phvs = true;
     
         /*
          * If we are dealing with an appendrel member then anything that's not a
          * simple Var has to be turned into a PlaceHolderVar.  We force this to
          * ensure that what we pull up doesn't get merged into a surrounding
          * expression during later processing and then fail to match the
          * expression actually available from the appendrel.
          */
         if (containing_appendrel != NULL)
         {
             rvcontext.need_phvs = true;
             rvcontext.wrap_non_vars = true;
         }
     
         /*
          * If the parent query uses grouping sets, we need a PlaceHolderVar for
          * anything that's not a simple Var.  Again, this ensures that expressions
          * retain their separate identity so that they will match grouping set
          * columns when appropriate.  (It'd be sufficient to wrap values used in
          * grouping set columns, and do so only in non-aggregated portions of the
          * tlist and havingQual, but that would require a lot of infrastructure
          * that pullup_replace_vars hasn't currently got.)
          */
         if (parse->groupingSets)
         {
             rvcontext.need_phvs = true;
             rvcontext.wrap_non_vars = true;
         }
     
         /*
          * Replace all of the top query's references to the subquery's outputs
          * with copies of the adjusted subtlist items, being careful not to
          * replace any of the jointree structure. (This'd be a lot cleaner if we
          * could use query_tree_mutator.)  We have to use PHVs in the targetList,
          * returningList, and havingQual, since those are certainly above any
          * outer join.  replace_vars_in_jointree tracks its location in the
          * jointree and uses PHVs or not appropriately.
          */
         //处理投影
         parse->targetList = (List *)
             pullup_replace_vars((Node *) parse->targetList, &rvcontext);
         parse->returningList = (List *)
             pullup_replace_vars((Node *) parse->returningList, &rvcontext);
         if (parse->onConflict)
         {
             parse->onConflict->onConflictSet = (List *)
                 pullup_replace_vars((Node *) parse->onConflict->onConflictSet,
                                     &rvcontext);
             parse->onConflict->onConflictWhere =
                 pullup_replace_vars(parse->onConflict->onConflictWhere,
                                     &rvcontext);
     
             /*
              * We assume ON CONFLICT's arbiterElems, arbiterWhere, exclRelTlist
              * can't contain any references to a subquery
              */
         }
         replace_vars_in_jointree((Node *) parse->jointree, &rvcontext,
                                  lowest_nulling_outer_join);
         Assert(parse->setOperations == NULL);
         parse->havingQual = pullup_replace_vars(parse->havingQual, &rvcontext);
     
         /*
          * Replace references in the translated_vars lists of appendrels. When
          * pulling up an appendrel member, we do not need PHVs in the list of the
          * parent appendrel --- there isn't any outer join between. Elsewhere, use
          * PHVs for safety.  (This analysis could be made tighter but it seems
          * unlikely to be worth much trouble.)
          */
         //处理appendrels中的信息
         foreach(lc, root->append_rel_list)
         {
             AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(lc);
             bool        save_need_phvs = rvcontext.need_phvs;
     
             if (appinfo == containing_appendrel)
                 rvcontext.need_phvs = false;
             appinfo->translated_vars = (List *)
                 pullup_replace_vars((Node *) appinfo->translated_vars, &rvcontext);
             rvcontext.need_phvs = save_need_phvs;
         }
     
         /*
          * Replace references in the joinaliasvars lists of join RTEs.
          *
          * You might think that we could avoid using PHVs for alias vars of joins
          * below lowest_nulling_outer_join, but that doesn't work because the
          * alias vars could be referenced above that join; we need the PHVs to be
          * present in such references after the alias vars get flattened.  (It
          * might be worth trying to be smarter here, someday.)
          */
         //处理RTE中类型为RTE_JOIN的节点
         foreach(lc, parse->rtable)
         {
             RangeTblEntry *otherrte = (RangeTblEntry *) lfirst(lc);
     
             if (otherrte->rtekind == RTE_JOIN)
                 otherrte->joinaliasvars = (List *)
                     pullup_replace_vars((Node *) otherrte->joinaliasvars,
                                         &rvcontext);
         }
     
         /*
          * If the subquery had a LATERAL marker, propagate that to any of its
          * child RTEs that could possibly now contain lateral cross-references.
          * The children might or might not contain any actual lateral
          * cross-references, but we have to mark the pulled-up child RTEs so that
          * later planner stages will check for such.
          */
         //LATERAL支持
         if (rte->lateral)
         {
             foreach(lc, subquery->rtable)
             {
                 RangeTblEntry *child_rte = (RangeTblEntry *) lfirst(lc);
     
                 switch (child_rte->rtekind)
                 {
                     case RTE_RELATION:
                         if (child_rte->tablesample)
                             child_rte->lateral = true;
                         break;
                     case RTE_SUBQUERY:
                     case RTE_FUNCTION:
                     case RTE_VALUES:
                     case RTE_TABLEFUNC:
                         child_rte->lateral = true;
                         break;
                     case RTE_JOIN:
                     case RTE_CTE:
                     case RTE_NAMEDTUPLESTORE:
                         /* these can't contain any lateral references */
                         break;
                 }
             }
         }
     
         /*
          * Now append the adjusted rtable entries to upper query. (We hold off
          * until after fixing the upper rtable entries; no point in running that
          * code on the subquery ones too.)
          */
         //子查询中的RTE填充至父查询中
         parse->rtable = list_concat(parse->rtable, subquery->rtable);
     
         /*
          * Pull up any FOR UPDATE/SHARE markers, too.  (OffsetVarNodes already
          * adjusted the marker rtindexes, so just concat the lists.)
          */
         parse->rowMarks = list_concat(parse->rowMarks, subquery->rowMarks);
     
         /*
          * We also have to fix the relid sets of any PlaceHolderVar nodes in the
          * parent query.  (This could perhaps be done by pullup_replace_vars(),
          * but it seems cleaner to use two passes.)  Note in particular that any
          * PlaceHolderVar nodes just created by pullup_replace_vars() will be
          * adjusted, so having created them with the subquery's varno is correct.
          *
          * Likewise, relids appearing in AppendRelInfo nodes have to be fixed. We
          * already checked that this won't require introducing multiple subrelids
          * into the single-slot AppendRelInfo structs.
          */
         if (parse->hasSubLinks || root->glob->lastPHId != 0 ||
             root->append_rel_list)
         {
             Relids      subrelids;
     
             subrelids = get_relids_in_jointree((Node *) subquery->jointree, false);
             substitute_multiple_relids((Node *) parse, varno, subrelids);
             fix_append_rel_relids(root->append_rel_list, varno, subrelids);
         }
     
         /*
          * And now add subquery's AppendRelInfos to our list.
          */
         root->append_rel_list = list_concat(root->append_rel_list,
                                             subroot->append_rel_list);
     
         /*
          * We don't have to do the equivalent bookkeeping for outer-join info,
          * because that hasn't been set up yet.  placeholder_list likewise.
          */
         Assert(root->join_info_list == NIL);
         Assert(subroot->join_info_list == NIL);
         Assert(root->placeholder_list == NIL);
         Assert(subroot->placeholder_list == NIL);
     
         /*
          * Miscellaneous housekeeping.
          *
          * Although replace_rte_variables() faithfully updated parse->hasSubLinks
          * if it copied any SubLinks out of the subquery's targetlist, we still
          * could have SubLinks added to the query in the expressions of FUNCTION
          * and VALUES RTEs copied up from the subquery.  So it's necessary to copy
          * subquery->hasSubLinks anyway.  Perhaps this can be improved someday.
          */
         parse->hasSubLinks |= subquery->hasSubLinks;
     
         /* If subquery had any RLS conditions, now main query does too */
         parse->hasRowSecurity |= subquery->hasRowSecurity;
     
         /*
          * subquery won't be pulled up if it hasAggs, hasWindowFuncs, or
          * hasTargetSRFs, so no work needed on those flags
          */
     
         /*
          * Return the adjusted subquery jointree to replace the RangeTblRef entry
          * in parent's jointree; or, if we're flattening a subquery with empty
          * FROM list, return NULL to signal deletion of the subquery from the
          * parent jointree (and set hasDeletedRTEs to ensure cleanup later).
          */
         if (subquery->jointree->fromlist == NIL)
         {
             Assert(deletion_ok);
             Assert(subquery->jointree->quals == NULL);
             root->hasDeletedRTEs = true;
             return NULL;
         }
     
         return (Node *) subquery->jointree;
     }
    
    

**is_simple_subquery**

    
    
     /*
      * is_simple_subquery
      *    Check a subquery in the range table to see if it's simple enough
      *    to pull up into the parent query.
      *
      * rte is the RTE_SUBQUERY RangeTblEntry that contained the subquery.
      * (Note subquery is not necessarily equal to rte->subquery; it could be a
      * processed copy of that.)
      * lowest_outer_join is the lowest outer join above the subquery, or NULL.
      * deletion_ok is true if it'd be okay to delete the subquery entirely.
      */
     static bool
     is_simple_subquery(Query *subquery, RangeTblEntry *rte,
                        JoinExpr *lowest_outer_join,
                        bool deletion_ok)
     {
         /*
          * Let's just make sure it's a valid subselect ...
          */
         if (!IsA(subquery, Query) ||
             subquery->commandType != CMD_SELECT)
             elog(ERROR, "subquery is bogus");
     
         /*
          * Can't currently pull up a query with setops (unless it's simple UNION
          * ALL, which is handled by a different code path). Maybe after querytree
          * redesign...
          */
         if (subquery->setOperations)
             return false;//存在集合操作
     
         /*
          * Can't pull up a subquery involving grouping, aggregation, SRFs,
          * sorting, limiting, or WITH.  (XXX WITH could possibly be allowed later)
          *
          * We also don't pull up a subquery that has explicit FOR UPDATE/SHARE
          * clauses, because pullup would cause the locking to occur semantically
          * higher than it should.  Implicit FOR UPDATE/SHARE is okay because in
          * that case the locking was originally declared in the upper query
          * anyway.
          */
         if (subquery->hasAggs ||
             subquery->hasWindowFuncs ||
             subquery->hasTargetSRFs ||
             subquery->groupClause ||
             subquery->groupingSets ||
             subquery->havingQual ||
             subquery->sortClause ||
             subquery->distinctClause ||
             subquery->limitOffset ||
             subquery->limitCount ||
             subquery->hasForUpdate ||
             subquery->cteList)
             return false;//存在聚合函数/窗口函数...
     
         /*
          * Don't pull up if the RTE represents a security-barrier view; we
          * couldn't prevent information leakage once the RTE's Vars are scattered
          * about in the upper query.
          */
         if (rte->security_barrier)
             return false;//
     
         /*
          * Don't pull up a subquery with an empty jointree, unless it has no quals
          * and deletion_ok is true and we're not underneath an outer join.
          *
          * query_planner() will correctly generate a Result plan for a jointree
          * that's totally empty, but we can't cope with an empty FromExpr
          * appearing lower down in a jointree: we identify join rels via baserelid
          * sets, so we couldn't distinguish a join containing such a FromExpr from
          * one without it.  We can only handle such cases if the place where the
          * subquery is linked is a FromExpr or inner JOIN that would still be
          * nonempty after removal of the subquery, so that it's still identifiable
          * via its contained baserelids.  Safe contexts are signaled by
          * deletion_ok.
          *
          * But even in a safe context, we must keep the subquery if it has any
          * quals, because it's unclear where to put them in the upper query.
          *
          * Also, we must forbid pullup if such a subquery is underneath an outer
          * join, because then we might need to wrap its output columns with
          * PlaceHolderVars, and the PHVs would then have empty relid sets meaning
          * we couldn't tell where to evaluate them.  (This test is separate from
          * the deletion_ok flag for possible future expansion: deletion_ok tells
          * whether the immediate parent site in the jointree could cope, not
          * whether we'd have PHV issues.  It's possible this restriction could be
          * fixed by letting the PHVs use the relids of the parent jointree item,
          * but that complication is for another day.)
          *
          * Note that deletion of a subquery is also dependent on the check below
          * that its targetlist contains no set-returning functions.  Deletion from
          * a FROM list or inner JOIN is okay only if the subquery must return
          * exactly one row.
          */
         if (subquery->jointree->fromlist == NIL &&
             (subquery->jointree->quals != NULL ||
              !deletion_ok ||
              lowest_outer_join != NULL))
             return false;
     
         /*
          * If the subquery is LATERAL, check for pullup restrictions from that.
          */
         if (rte->lateral)
         {
             bool        restricted;
             Relids      safe_upper_varnos;
     
             /*
              * The subquery's WHERE and JOIN/ON quals mustn't contain any lateral
              * references to rels outside a higher outer join (including the case
              * where the outer join is within the subquery itself).  In such a
              * case, pulling up would result in a situation where we need to
              * postpone quals from below an outer join to above it, which is
              * probably completely wrong and in any case is a complication that
              * doesn't seem worth addressing at the moment.
              */
             if (lowest_outer_join != NULL)
             {
                 restricted = true;
                 safe_upper_varnos = get_relids_in_jointree((Node *) lowest_outer_join,
                                                            true);
             }
             else
             {
                 restricted = false;
                 safe_upper_varnos = NULL;   /* doesn't matter */
             }
     
             if (jointree_contains_lateral_outer_refs((Node *) subquery->jointree,
                                                      restricted, safe_upper_varnos))
                 return false;
     
             /*
              * If there's an outer join above the LATERAL subquery, also disallow
              * pullup if the subquery's targetlist has any references to rels
              * outside the outer join, since these might get pulled into quals
              * above the subquery (but in or below the outer join) and then lead
              * to qual-postponement issues similar to the case checked for above.
              * (We wouldn't need to prevent pullup if no such references appear in
              * outer-query quals, but we don't have enough info here to check
              * that.  Also, maybe this restriction could be removed if we forced
              * such refs to be wrapped in PlaceHolderVars, even when they're below
              * the nearest outer join?  But it's a pretty hokey usage, so not
              * clear this is worth sweating over.)
              */
             if (lowest_outer_join != NULL)
             {
                 Relids      lvarnos = pull_varnos_of_level((Node *) subquery->targetList, 1);
     
                 if (!bms_is_subset(lvarnos, safe_upper_varnos))
                     return false;
             }
         }
     
         /*
          * Don't pull up a subquery that has any volatile functions in its
          * targetlist.  Otherwise we might introduce multiple evaluations of these
          * functions, if they get copied to multiple places in the upper query,
          * leading to surprising results.  (Note: the PlaceHolderVar mechanism
          * doesn't quite guarantee single evaluation; else we could pull up anyway
          * and just wrap such items in PlaceHolderVars ...)
          */
         if (contain_volatile_functions((Node *) subquery->targetList))
             return false;//存在易变函数
     
         return true;
     }
    

**pull_up_subqueries_cleanup**

    
    
     
     /*
      * pull_up_subqueries_cleanup
      *      Recursively fix up jointree after deletion of some subqueries.
      *
      * The jointree now contains some NULL subtrees, which we need to get rid of.
      * In a FromExpr, just rebuild the child-node list with null entries deleted.
      * In an inner JOIN, replace the JoinExpr node with a one-child FromExpr.
      */
     static Node *
     pull_up_subqueries_cleanup(Node *jtnode)
     {
         Assert(jtnode != NULL);
         if (IsA(jtnode, RangeTblRef))
         {
             /* Nothing to do at leaf nodes. */
         }
         else if (IsA(jtnode, FromExpr))
         {
             FromExpr   *f = (FromExpr *) jtnode;
             List       *newfrom = NIL;
             ListCell   *l;
     
             foreach(l, f->fromlist)
             {
                 Node       *child = (Node *) lfirst(l);
     
                 if (child == NULL)
                     continue;
                 child = pull_up_subqueries_cleanup(child);
                 newfrom = lappend(newfrom, child);
             }
             f->fromlist = newfrom;
         }
         else if (IsA(jtnode, JoinExpr))
         {
             JoinExpr   *j = (JoinExpr *) jtnode;
     
             if (j->larg)
                 j->larg = pull_up_subqueries_cleanup(j->larg);
             if (j->rarg)
                 j->rarg = pull_up_subqueries_cleanup(j->rarg);
             if (j->larg == NULL)
             {
                 Assert(j->jointype == JOIN_INNER);
                 Assert(j->rarg != NULL);
                 return (Node *) makeFromExpr(list_make1(j->rarg), j->quals);
             }
             else if (j->rarg == NULL)
             {
                 Assert(j->jointype == JOIN_INNER);
                 return (Node *) makeFromExpr(list_make1(j->larg), j->quals);
             }
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
         return jtnode;
     }
    

### 三、跟踪分析

gdb跟踪分析:

    
    
    (gdb) b pull_up_subqueries
    Breakpoint 1 at 0x77d63b: file prepjointree.c, line 612.
    (gdb) c
    Continuing.
    
    Breakpoint 1, pull_up_subqueries (root=0x1d092d0) at prepjointree.c:612
    612     root->hasDeletedRTEs = false;
    (gdb) 
    #输入参数,root参见上拉子链接中的说明
    #进入pull_up_subqueries_recurse
    (gdb) step
    615         pull_up_subqueries_recurse(root, (Node *) root->parse->jointree,
    (gdb) step
    pull_up_subqueries_recurse (root=0x1d092d0, jtnode=0x1d092a0, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x0, deletion_ok=false) at prepjointree.c:680
    680     if (IsA(jtnode, RangeTblRef))
    (gdb) 
    #输入参数:
    #1.root,同pull_up_subqueries
    #2.jtnode,Query查询树
    #3/4/5.lowest_outer_join/lowest_nulling_outer_join/containing_appendrel均为NULL
    #6.deletion_ok,false
    ...
    (gdb) p *jtnode
    $2 = {type = T_FromExpr}
    #FromExpr,进入相应的分支
    ...
    #递归调用pull_up_subqueries_recurse
    (gdb) 
    763             lfirst(l) = pull_up_subqueries_recurse(root, lfirst(l),
    (gdb) step
    pull_up_subqueries_recurse (root=0x1d092d0, jtnode=0x1c73078, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x0, deletion_ok=true) at prepjointree.c:680
    680     if (IsA(jtnode, RangeTblRef))
    #注意:这时候的jtnode类型为RangeTblRef
    (gdb) n
    682         int         varno = ((RangeTblRef *) jtnode)->rtindex;
    (gdb) 
    683         RangeTblEntry *rte = rt_fetch(varno, root->parse->rtable);
    (gdb) 
    692         if (rte->rtekind == RTE_SUBQUERY &&
    (gdb) p varno
    $4 = 1
    #rtable中第1个RTE是父查询的Relation(即t_dwxx),不是子查询
    (gdb) p *rte
    $5 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16394, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x1c4fd58, eref = 0x1c72c98, 
      lateral = false, inh = true, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x1d07698, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    (gdb) n
    712         if (rte->rtekind == RTE_SUBQUERY &&
    (gdb) 
    722         if (rte->rtekind == RTE_VALUES &&
    (gdb) 
    852     return jtnode;
    (gdb) 
    ...
    #rtable中的第2个元素,类型为RTE_SUBQUERY
    (gdb) step
    pull_up_subqueries_recurse (root=0x1d092d0, jtnode=0x1d07358, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x0, deletion_ok=true) at prepjointree.c:680
    680     if (IsA(jtnode, RangeTblRef))
    (gdb) n
    682         int         varno = ((RangeTblRef *) jtnode)->rtindex;
    (gdb) 
    (gdb) p *rte
    $7 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x1c72968, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, 
      coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x1c50548, eref = 0x1d071a0, 
      lateral = false, inh = false, inFromCl = true, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    ...
    #进入pull_up_simple_subquery
    697             return pull_up_simple_subquery(root, jtnode, rte,
    (gdb) step
    pull_up_simple_subquery (root=0x1d092d0, jtnode=0x1d07358, rte=0x1c72a78, lowest_outer_join=0x0, 
        lowest_nulling_outer_join=0x0, containing_appendrel=0x0, deletion_ok=true) at prepjointree.c:874
    874     Query      *parse = root->parse;
    ...
    1247        return (Node *) subquery->jointree;
    (gdb) 
    1248    }
    (gdb) 
    pull_up_subqueries_recurse (root=0x1d09838, jtnode=0x1c736e0, lowest_outer_join=0x0, lowest_nulling_outer_join=0x0, 
        containing_appendrel=0x0, deletion_ok=true) at prepjointree.c:853
    853 }
    (gdb) 
    

### 四、小结

1、上拉过程：判断子查询是否可以上拉，如可以上拉则调整相关信息包括rtable、quals等；  
2、上拉结果：与上拉子链接类似，把子查询上拉为数据表连接。

