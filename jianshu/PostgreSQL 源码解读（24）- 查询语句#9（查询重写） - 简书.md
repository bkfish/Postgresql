本节简单介绍了PG查询语句优化部分查询重写相关的内容。查询重写在pg_rewrite_query函数中实现，该函数位于文件postgres.c中。

### 一、测试脚本

    
    
    testdb=# CREATE VIEW vw_dwxx AS SELECT * FROM t_dwxx;
    CREATE VIEW
    testdb=# select dw.dwmc,gr.grbh,gr.xm
    from vw_dwxx dw inner join t_grxx gr 
    on dw.dwbh = gr.dwbh
    where dw.dwbh = '1001';
    

### 二、查询重写

可以把查询重写系统当作某种宏展开的机制。  
创建视图的SQL命令:

    
    
    CREATE VIEW vw_dwxx AS SELECT * FROM t_dwxx;
    

与下面两个命令相比没有什么不同:

    
    
    CREATE TABLE vw_dwxx (same column list as t_dwxx);
    CREATE RULE "_RETURN" AS ON SELECT TO vw_dwxx DO INSTEAD
        SELECT * FROM t_dwxx;
    

查询重写前的Query结构:

  

![](https://upload-images.jianshu.io/upload_images/8194836-bec605c058dc0d66.png)

重写前的Query结构

查询重写后的Query结构:

  

![](https://upload-images.jianshu.io/upload_images/8194836-95847550977a4093.png)

重写后的Query结构

### 三、源码解读

**pg_rewrite_query**

    
    
     /*
      * Perform rewriting of a query produced by parse analysis.
      *
      * Note: query must just have come from the parser, because we do not do
      * AcquireRewriteLocks() on it.
      */
     static List *
     pg_rewrite_query(Query *query) //输入查询树
     {
         List       *querytree_list;
     
         if (Debug_print_parse) //debug模式,输出parse tree树
             elog_node_display(LOG, "parse tree", query,
                               Debug_pretty_print);
     
         if (log_parser_stats)
             ResetUsage();
     
         if (query->commandType == CMD_UTILITY) //工具类语句
         {
             /* don't rewrite utilities, just dump 'em into result list */
             querytree_list = list_make1(query);
         }
         else//非工具类语句
         {
             /* rewrite regular queries */
             querytree_list = QueryRewrite(query); //进入查询重写
         }
     
         if (log_parser_stats)
             ShowUsage("REWRITER STATISTICS");
     
     #ifdef COPY_PARSE_PLAN_TREES
         /* Optional debugging check: pass querytree output through copyObject() */
         {
             List       *new_list;
     
             new_list = copyObject(querytree_list);
             /* This checks both copyObject() and the equal() routines... */
             if (!equal(new_list, querytree_list))
                 elog(WARNING, "copyObject() failed to produce equal parse tree");
             else
                 querytree_list = new_list;
         }
     #endif
     
         if (Debug_print_rewritten)
             elog_node_display(LOG, "rewritten parse tree", querytree_list,
                               Debug_pretty_print);
     
         return querytree_list;
     }
    
    

**QueryRewrite**

    
    
    /*
      * QueryRewrite -      *    Primary entry point to the query rewriter.
      *    Rewrite one query via query rewrite system, possibly returning 0
      *    or many queries.
      *
      * NOTE: the parsetree must either have come straight from the parser,
      * or have been scanned by AcquireRewriteLocks to acquire suitable locks.
      */
     List *
     QueryRewrite(Query *parsetree) //输入查询树
     {
         uint64      input_query_id = parsetree->queryId;//查询id
         List       *querylist;//查询树,中间结果
         List       *results;//最终结果
         ListCell   *l;//临时变量
         CmdType     origCmdType;//命令类型
         bool        foundOriginalQuery;
         Query      *lastInstead;
     
         /*
          * This function is only applied to top-level original queries
          */
         Assert(parsetree->querySource == QSRC_ORIGINAL);
         Assert(parsetree->canSetTag);
     
         /*
          * Step 1
          *
          * Apply all non-SELECT rules possibly getting 0 or many queries
          * 第1步,应用所有非SELECT规则(UPDATE/INSERT等),查询不需要执行
          */
         querylist = RewriteQuery(parsetree, NIL);
     
         /*
          * Step 2
          *
          * Apply all the RIR rules on each query
          *
          * This is also a handy place to mark each query with the original queryId
          * 第2步,应用所有RIR规则
          * RIR是Retrieve-Instead-Retrieve的缩写,The name is based on history.RETRIEVE was the PostQUEL keyword 
          * for what you know as SELECT. A rule fired on a RETRIEVE event, that is an unconditional INSTEAD rule 
          * with exactly one RETRIEVE action is called RIR-rule.
          */
         results = NIL;
         foreach(l, querylist) //循环
         {
             Query      *query = (Query *) lfirst(l);//获取Query
     
             query = fireRIRrules(query, NIL);//应用RIR规则
     
             query->queryId = input_query_id;//设置查询id
     
             results = lappend(results, query);//加入返回结果列表中
         }
     
         /*
          * Step 3
          *
          * 第3步,确定哪一个Query设置命令结果标签,并更新canSetTag字段
          * Determine which, if any, of the resulting queries is supposed to set
          * the command-result tag; and update the canSetTag fields accordingly.
          * 
          *
          * If the original query is still in the list, it sets the command tag.
          * Otherwise, the last INSTEAD query of the same kind as the original is
          * allowed to set the tag.  (Note these rules can leave us with no query
          * setting the tag.  The tcop code has to cope with this by setting up a
          * default tag based on the original un-rewritten query.)
          *
          * The Asserts verify that at most one query in the result list is marked
          * canSetTag.  If we aren't checking asserts, we can fall out of the loop
          * as soon as we find the original query.
          */
         origCmdType = parsetree->commandType;
         foundOriginalQuery = false;
         lastInstead = NULL;
     
         foreach(l, results)
         {
             Query      *query = (Query *) lfirst(l);
     
             if (query->querySource == QSRC_ORIGINAL)
             {
                 Assert(query->canSetTag);
                 Assert(!foundOriginalQuery);
                 foundOriginalQuery = true;
     #ifndef USE_ASSERT_CHECKING
                 break;
     #endif
             }
             else
             {
                 Assert(!query->canSetTag);
                 if (query->commandType == origCmdType &&
                     (query->querySource == QSRC_INSTEAD_RULE ||
                      query->querySource == QSRC_QUAL_INSTEAD_RULE))
                     lastInstead = query;
             }
         }
     
         if (!foundOriginalQuery && lastInstead != NULL)
             lastInstead->canSetTag = true;
     
         return results;
     }
    

**fireRIRrules**

    
    
     /*
      * fireRIRrules -      *  Apply all RIR rules on each rangetable entry in the given query
      *  在每一个RTE上应用所有的RIR规则
      *
      * activeRIRs is a list of the OIDs of views we're already processing RIR
      * rules for, used to detect/reject recursion.
      */
     static Query *
     fireRIRrules(Query *parsetree, List *activeRIRs)
     {
         int         origResultRelation = parsetree->resultRelation;//结果Relation
         int         rt_index;//RTE的index
         ListCell   *lc;//临时变量
     
         /*
          * don't try to convert this into a foreach loop, because rtable list can
          * get changed each time through...
          */
         rt_index = 0;
         while (rt_index < list_length(parsetree->rtable)) //循环
         {
             RangeTblEntry *rte;//RTE
             Relation    rel;//关系
             List       *locks;//锁列表
             RuleLock   *rules;//规则锁
             RewriteRule *rule;//重写规则
             int         i;//临时比那里
     
             ++rt_index;//索引+1
     
             rte = rt_fetch(rt_index, parsetree->rtable);//获取RTE
     
             /*
              * A subquery RTE can't have associated rules, so there's nothing to
              * do to this level of the query, but we must recurse into the
              * subquery to expand any rule references in it.
              */
             if (rte->rtekind == RTE_SUBQUERY)//子查询
             {
                 rte->subquery = fireRIRrules(rte->subquery, activeRIRs);//递归处理
                 continue;
             }
     
             /*
              * Joins and other non-relation RTEs can be ignored completely.
              */
             if (rte->rtekind != RTE_RELATION)//非RTE_RELATION无需处理
                 continue;
     
             /*
              * Always ignore RIR rules for materialized views referenced in
              * queries.  (This does not prevent refreshing MVs, since they aren't
              * referenced in their own query definitions.)
              *
              * Note: in the future we might want to allow MVs to be conditionally
              * expanded as if they were regular views, if they are not scannable.
              * In that case this test would need to be postponed till after we've
              * opened the rel, so that we could check its state.
              */
             if (rte->relkind == RELKIND_MATVIEW)//物化视图类的Relation无需处理
                 continue;
     
             /*
              * In INSERT ... ON CONFLICT, ignore the EXCLUDED pseudo-relation;
              * even if it points to a view, we needn't expand it, and should not
              * because we want the RTE to remain of RTE_RELATION type.  Otherwise,
              * it would get changed to RTE_SUBQUERY type, which is an
              * untested/unsupported situation.
              */
             if (parsetree->onConflict &&
                 rt_index == parsetree->onConflict->exclRelIndex)//INSERT ... ON CONFLICT 无需处理
                 continue;
     
             /*
              * If the table is not referenced in the query, then we ignore it.
              * This prevents infinite expansion loop due to new rtable entries
              * inserted by expansion of a rule. A table is referenced if it is
              * part of the join set (a source table), or is referenced by any Var
              * nodes, or is the result table.
              */
             if (rt_index != parsetree->resultRelation &&
                 !rangeTableEntry_used((Node *) parsetree, rt_index, 0))//相应的RTE为NULL,无需处理
                 continue;
     
             /*
              * Also, if this is a new result relation introduced by
              * ApplyRetrieveRule, we don't want to do anything more with it.
              */
             if (rt_index == parsetree->resultRelation &&
                 rt_index != origResultRelation)//结果关系
                 continue;
     
             /*
              * We can use NoLock here since either the parser or
              * AcquireRewriteLocks should have locked the rel already.
              */
             rel = heap_open(rte->relid, NoLock);//根据relid获取"关系"
     
             /*
              * Collect the RIR rules that we must apply
              */
             rules = rel->rd_rules;//获取关系上的规则
             if (rules != NULL)
             {
                 locks = NIL;
                 for (i = 0; i < rules->numLocks; i++)
                 {
                     rule = rules->rules[i];//获取规则
                     if (rule->event != CMD_SELECT)
                         continue;//非SELECT类型,继续下一个规则
     
                     locks = lappend(locks, rule);//添加到列表中
                 }
     
                 /*
                  * If we found any, apply them --- but first check for recursion!
                  */
                 if (locks != NIL)
                 {
                     ListCell   *l;
     
                     if (list_member_oid(activeRIRs, RelationGetRelid(rel)))//检查是否存在递归
                         ereport(ERROR,
                                 (errcode(ERRCODE_INVALID_OBJECT_DEFINITION),
                                  errmsg("infinite recursion detected in rules for relation \"%s\"",
                                         RelationGetRelationName(rel))));
                     activeRIRs = lcons_oid(RelationGetRelid(rel), activeRIRs);//
     
                     foreach(l, locks)//循环
                     {
                         rule = lfirst(l);
     
                         parsetree = ApplyRetrieveRule(parsetree,
                                                       rule,
                                                       rt_index,
                                                       rel,
                                                       activeRIRs);//应用规则
                     }
     
                     activeRIRs = list_delete_first(activeRIRs);//删除已应用的规则
                 }
             }
     
             heap_close(rel, NoLock);//释放资源
         }
     
         /* Recurse into subqueries in WITH */
         foreach(lc, parsetree->cteList) //WITH 语句处理
         {
             CommonTableExpr *cte = (CommonTableExpr *) lfirst(lc);
     
             cte->ctequery = (Node *)
                 fireRIRrules((Query *) cte->ctequery, activeRIRs);
         }
     
         /*
          * Recurse into sublink subqueries, too.  But we already did the ones in
          * the rtable and cteList.
          */
         if (parsetree->hasSubLinks) //存在子链接
             query_tree_walker(parsetree, fireRIRonSubLink, (void *) activeRIRs,
                               QTW_IGNORE_RC_SUBQUERIES);
     
         /*
          * Apply any row level security policies.  We do this last because it
          * requires special recursion detection if the new quals have sublink
          * subqueries, and if we did it in the loop above query_tree_walker would
          * then recurse into those quals a second time.
          */
         rt_index = 0;
         foreach(lc, parsetree->rtable)//应用行级安全策略
         {
             RangeTblEntry *rte = (RangeTblEntry *) lfirst(lc);
             Relation    rel;
             List       *securityQuals;
             List       *withCheckOptions;
             bool        hasRowSecurity;
             bool        hasSubLinks;
     
             ++rt_index;
     
             /* Only normal relations can have RLS policies */
             if (rte->rtekind != RTE_RELATION ||
                 (rte->relkind != RELKIND_RELATION &&
                  rte->relkind != RELKIND_PARTITIONED_TABLE))
                 continue;
     
             rel = heap_open(rte->relid, NoLock);
     
             /*
              * Fetch any new security quals that must be applied to this RTE.
              */
             get_row_security_policies(parsetree, rte, rt_index,
                                       &securityQuals, &withCheckOptions,
                                       &hasRowSecurity, &hasSubLinks);
     
             if (securityQuals != NIL || withCheckOptions != NIL)
             {
                 if (hasSubLinks)
                 {
                     acquireLocksOnSubLinks_context context;
     
                     /*
                      * Recursively process the new quals, checking for infinite
                      * recursion.
                      */
                     if (list_member_oid(activeRIRs, RelationGetRelid(rel)))
                         ereport(ERROR,
                                 (errcode(ERRCODE_INVALID_OBJECT_DEFINITION),
                                  errmsg("infinite recursion detected in policy for relation \"%s\"",
                                         RelationGetRelationName(rel))));
     
                     activeRIRs = lcons_oid(RelationGetRelid(rel), activeRIRs);
     
                     /*
                      * get_row_security_policies just passed back securityQuals
                      * and/or withCheckOptions, and there were SubLinks, make sure
                      * we lock any relations which are referenced.
                      *
                      * These locks would normally be acquired by the parser, but
                      * securityQuals and withCheckOptions are added post-parsing.
                      */
                     context.for_execute = true;
                     (void) acquireLocksOnSubLinks((Node *) securityQuals, &context);
                     (void) acquireLocksOnSubLinks((Node *) withCheckOptions,
                                                   &context);
     
                     /*
                      * Now that we have the locks on anything added by
                      * get_row_security_policies, fire any RIR rules for them.
                      */
                     expression_tree_walker((Node *) securityQuals,
                                            fireRIRonSubLink, (void *) activeRIRs);
     
                     expression_tree_walker((Node *) withCheckOptions,
                                            fireRIRonSubLink, (void *) activeRIRs);
     
                     activeRIRs = list_delete_first(activeRIRs);
                 }
     
                 /*
                  * Add the new security barrier quals to the start of the RTE's
                  * list so that they get applied before any existing barrier quals
                  * (which would have come from a security-barrier view, and should
                  * get lower priority than RLS conditions on the table itself).
                  */
                 rte->securityQuals = list_concat(securityQuals,
                                                  rte->securityQuals);
     
                 parsetree->withCheckOptions = list_concat(withCheckOptions,
                                                           parsetree->withCheckOptions);
             }
     
             /*
              * Make sure the query is marked correctly if row level security
              * applies, or if the new quals had sublinks.
              */
             if (hasRowSecurity)
                 parsetree->hasRowSecurity = true;
             if (hasSubLinks)
                 parsetree->hasSubLinks = true;
     
             heap_close(rel, NoLock);
         }
     
         return parsetree;//返回
     }
    

**ApplyRetrieveRule**

    
    
      * ApplyRetrieveRule - expand an ON SELECT rule
      */
     static Query *
     ApplyRetrieveRule(Query *parsetree,//查询树
                       RewriteRule *rule,//重写规则
                       int rt_index,//RTE index
                       Relation relation,//关系
                       List *activeRIRs)//RIR链表
     {
         Query      *rule_action;
         RangeTblEntry *rte,
                    *subrte;//RTE
         RowMarkClause *rc;
     
         if (list_length(rule->actions) != 1)
             elog(ERROR, "expected just one rule action");
         if (rule->qual != NULL)
             elog(ERROR, "cannot handle qualified ON SELECT rule");
     
         if (rt_index == parsetree->resultRelation) //目标关系?
         {
             /*
              * We have a view as the result relation of the query, and it wasn't
              * rewritten by any rule.  This case is supported if there is an
              * INSTEAD OF trigger that will trap attempts to insert/update/delete
              * view rows.  The executor will check that; for the moment just plow
              * ahead.  We have two cases:
              *
              * For INSERT, we needn't do anything.  The unmodified RTE will serve
              * fine as the result relation.
              *
              * For UPDATE/DELETE, we need to expand the view so as to have source
              * data for the operation.  But we also need an unmodified RTE to
              * serve as the target.  So, copy the RTE and add the copy to the
              * rangetable.  Note that the copy does not get added to the jointree.
              * Also note that there's a hack in fireRIRrules to avoid calling this
              * function again when it arrives at the copied RTE.
              */
             if (parsetree->commandType == CMD_INSERT)
                 return parsetree;
             else if (parsetree->commandType == CMD_UPDATE ||
                      parsetree->commandType == CMD_DELETE)
             {
                 RangeTblEntry *newrte;
                 Var        *var;
                 TargetEntry *tle;
     
                 rte = rt_fetch(rt_index, parsetree->rtable);
                 newrte = copyObject(rte);
                 parsetree->rtable = lappend(parsetree->rtable, newrte);
                 parsetree->resultRelation = list_length(parsetree->rtable);
     
                 /*
                  * There's no need to do permissions checks twice, so wipe out the
                  * permissions info for the original RTE (we prefer to keep the
                  * bits set on the result RTE).
                  */
                 rte->requiredPerms = 0;
                 rte->checkAsUser = InvalidOid;
                 rte->selectedCols = NULL;
                 rte->insertedCols = NULL;
                 rte->updatedCols = NULL;
     
                 /*
                  * For the most part, Vars referencing the view should remain as
                  * they are, meaning that they implicitly represent OLD values.
                  * But in the RETURNING list if any, we want such Vars to
                  * represent NEW values, so change them to reference the new RTE.
                  *
                  * Since ChangeVarNodes scribbles on the tree in-place, copy the
                  * RETURNING list first for safety.
                  */
                 parsetree->returningList = copyObject(parsetree->returningList);
                 ChangeVarNodes((Node *) parsetree->returningList, rt_index,
                                parsetree->resultRelation, 0);
     
                 /*
                  * To allow the executor to compute the original view row to pass
                  * to the INSTEAD OF trigger, we add a resjunk whole-row Var
                  * referencing the original RTE.  This will later get expanded
                  * into a RowExpr computing all the OLD values of the view row.
                  */
                 var = makeWholeRowVar(rte, rt_index, 0, false);
                 tle = makeTargetEntry((Expr *) var,
                                       list_length(parsetree->targetList) + 1,
                                       pstrdup("wholerow"),
                                       true);
     
                 parsetree->targetList = lappend(parsetree->targetList, tle);
     
                 /* Now, continue with expanding the original view RTE */
             }
             else
                 elog(ERROR, "unrecognized commandType: %d",
                      (int) parsetree->commandType);
         }
        
         /*
          * Check if there's a FOR [KEY] UPDATE/SHARE clause applying to this view.
          * 检查是否有FOR UPDATE/SHARE语句
          * Note: we needn't explicitly consider any such clauses appearing in
          * ancestor query levels; their effects have already been pushed down to
          * here by markQueryForLocking, and will be reflected in "rc".
          */
         rc = get_parse_rowmark(parsetree, rt_index);
     
         /*
          * Make a modifiable copy of the view query, and acquire needed locks on
          * the relations it mentions.  Force at least RowShareLock for all such
          * rels if there's a FOR [KEY] UPDATE/SHARE clause affecting this view.
          */
         //copy规则,上锁
         rule_action = copyObject(linitial(rule->actions));
     
         AcquireRewriteLocks(rule_action, true, (rc != NULL));
     
         /*
          * If FOR [KEY] UPDATE/SHARE of view, mark all the contained tables as
          * implicit FOR [KEY] UPDATE/SHARE, the same as the parser would have done
          * if the view's subquery had been written out explicitly.
          */
         if (rc != NULL)
             markQueryForLocking(rule_action, (Node *) rule_action->jointree,
                                 rc->strength, rc->waitPolicy, true);
     
         /*
          * Recursively expand any view references inside the view.
          * 递归展开
          * Note: this must happen after markQueryForLocking.  That way, any UPDATE
          * permission bits needed for sub-views are initially applied to their
          * RTE_RELATION RTEs by markQueryForLocking, and then transferred to their
          * OLD rangetable entries by the action below (in a recursive call of this
          * routine).
          */
         rule_action = fireRIRrules(rule_action, activeRIRs);
     
         /*
          * Now, plug the view query in as a subselect, replacing the relation's
          * original RTE.
          */
         rte = rt_fetch(rt_index, parsetree->rtable);//获取原RTE
     
         rte->rtekind = RTE_SUBQUERY;//转换为子查询
         rte->relid = InvalidOid;//设置为0
         rte->security_barrier = RelationIsSecurityView(relation);
         rte->subquery = rule_action;//子查询设置为刚才构造的Query
         rte->inh = false;           /* must not be set for a subquery */
     
         /*
          * We move the view's permission check data down to its rangetable. The
          * checks will actually be done against the OLD entry therein.
          */
         subrte = rt_fetch(PRS2_OLD_VARNO, rule_action->rtable);//OLD RTE仍需要检查权限
         Assert(subrte->relid == relation->rd_id);
         subrte->requiredPerms = rte->requiredPerms;
         subrte->checkAsUser = rte->checkAsUser;
         subrte->selectedCols = rte->selectedCols;
         subrte->insertedCols = rte->insertedCols;
         subrte->updatedCols = rte->updatedCols;
     
         rte->requiredPerms = 0;     /* no permission check on subquery itself */
         rte->checkAsUser = InvalidOid;
         rte->selectedCols = NULL;
         rte->insertedCols = NULL;
         rte->updatedCols = NULL;
     
         return parsetree;//返回结果
     }
     
    

### 四、跟踪分析

SQL语句:

    
    
    testdb=# select dw.dwmc,gr.grbh,gr.xm
    from vw_dwxx dw inner join t_grxx gr 
    on dw.dwbh = gr.dwbh
    where dw.dwbh = '1001';
    

启动gdb，跟踪调试：

    
    
    (gdb) b QueryRewrite
    Breakpoint 1 at 0x80a85e: file rewriteHandler.c, line 3571.
    (gdb) c
    Continuing.
    
    Breakpoint 1, QueryRewrite (parsetree=0x2868820) at rewriteHandler.c:3571
    3571        uint64      input_query_id = parsetree->queryId;
    (gdb) n
    3590        querylist = RewriteQuery(parsetree, NIL);
    #parsetree查询树
    #rtable有3个元素(RTE),查询重写注意处理这3个RTE
    (gdb) p *parsetree
    $6 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x2868be0, jointree = 0x2962180, 
      targetList = 0x2961db8, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 110}
    (gdb) p *parsetree->rtable
    $7 = {type = T_List, length = 3, head = 0x2868bc0, tail = 0x2961bb8}
    (gdb) 
    3604            query = fireRIRrules(query, NIL);
    (gdb) step
    fireRIRrules (parsetree=0x2868820, activeRIRs=0x0) at rewriteHandler.c:1721
    1721        int         origResultRelation = parsetree->resultRelation;
    (gdb) n
    ...
    1729        rt_index = 0;
    (gdb) 
    1730        while (rt_index < list_length(parsetree->rtable))
    1741            rte = rt_fetch(rt_index, parsetree->rtable);
    (gdb) 
    1748            if (rte->rtekind == RTE_SUBQUERY)
    #第1个RTE,视图vw_dwxx
    #rtekind = RTE_RELATION, relid = 16403, relkind = 118 'v'
    (gdb) p *rte
    $12 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16403, relkind = 118 'v', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x2867f08, eref = 0x2868a40, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x29614e8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    1796            rel = heap_open(rte->relid, NoLock);
    (gdb) 
    1801            rules = rel->rd_rules;
    #关系Relation
    (gdb) p *rel
    $13 = {rd_node = {spcNode = 1663, dbNode = 16384, relNode = 16403}, rd_smgr = 0x0, rd_refcnt = 1, 
      rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 0 '\000', 
      rd_statvalid = false, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f57a6b90df8, 
      rd_att = 0x7f57a6b90f08, rd_id = 16403, rd_lockInfo = {lockRelId = {relId = 16403, dbId = 16384}}, 
      rd_rules = 0x2945bc0, rd_rulescxt = 0x2947140, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, 
      rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, rd_partdesc = 0x0, 
      rd_partcheck = 0x0, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, rd_replidindex = 0, 
      rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, 
      rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, 
      rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, rd_opfamily = 0x0, 
      rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, 
      rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, rd_exclstrats = 0x0, rd_amcache = 0x0, 
      rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x0}
    #rules
    (gdb) p *rel->rd_rules #rules是指向RewriteRule数组的指针,元素只有一个(numLocks)
    $15 = {numLocks = 1, rules = 0x2947268}
    (gdb) p *(RewriteRule *)rel->rd_rules->rules[0]
    $28 = {ruleId = 16406, event = CMD_SELECT, qual = 0x0, actions = 0x2945b90, enabled = 79 'O', 
      isInstead = true}
    #查看rules中的actions链表
    (gdb) p *rel->rd_rules->rules[0]->actions
    $31 = {type = T_List, length = 1, head = 0x2945b70, tail = 0x2945b70}
    (gdb) p *(Node *)rel->rd_rules->rules[0]->actions->head->data.ptr_value
    $32 = {type = T_Query}
    (gdb) set $action=(Query *)rel->rd_rules->rules[0]->actions->head->data.ptr_value
    #重写规则中的action为Query!
    (gdb) p $action
    $34 = (Query *) 0x29472c8
    (gdb) p *$action
    $35 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x2947708, jointree = 0x2945820, 
      targetList = 0x2945990, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = -1, stmt_len = -1}
    (gdb) p *$action->rtable
    $36 = {type = T_List, length = 3, head = 0x29476e8, tail = 0x2945800}
    (gdb) p *(Node *)$action->rtable->head->data.ptr_value
    $37 = {type = T_RangeTblEntry}
    (gdb) p *(RangeTblEntry *)$action->rtable->head->data.ptr_value
    $38 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16403, relkind = 118 'v', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x29474e8, eref = 0x2947588, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 0, checkAsUser = 10, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$action->rtable->head->data.ptr_value)->eref
    $42 = {type = T_Alias, aliasname = 0x29475b8 "old", colnames = 0x2947608}
    (gdb) p *((RangeTblEntry *)$action->rtable->head->next->data.ptr_value)->eref
    $43 = {type = T_Alias, aliasname = 0x29478c0 "new", colnames = 0x2947930}
    (gdb) p *((RangeTblEntry *)$action->rtable->head->next->next->data.ptr_value)->eref
    $44 = {type = T_Alias, aliasname = 0x2945698 "t_dwxx", colnames = 0x2945708}
    ...
    1826                    activeRIRs = lcons_oid(RelationGetRelid(rel), activeRIRs);
    (gdb) 
    1828                    foreach(l, locks)
    (gdb) p *activeRIRs
    $46 = {type = T_OidList, length = 1, head = 0x29320c8, tail = 0x29320c8}
    (gdb) p activeRIRs->head->data.oid_value
    $47 = 16403
    #进入ApplyRetrieveRule
    (gdb) n
    1830                        rule = lfirst(l);
    (gdb) 
    1832                        parsetree = ApplyRetrieveRule(parsetree,
    (gdb) step
    ApplyRetrieveRule (parsetree=0x2868820, rule=0x2947298, rt_index=1, relation=0x7f57a6b90be8, 
        activeRIRs=0x29320e8) at rewriteHandler.c:1462
    ...
    #进入ApplyRetrieveRule调用fireRIRrules
    1581        rule_action = fireRIRrules(rule_action, activeRIRs);
    (gdb) step
    fireRIRrules (parsetree=0x2970118, activeRIRs=0x29320e8) at rewriteHandler.c:1721
    1721        int         origResultRelation = parsetree->resultRelation;
    ...
    (gdb) finish #结束ApplyRetrieveRule->fireRIRrules的调用
    Run till exit from #0  fireRIRrules (parsetree=0x2970118, activeRIRs=0x29320e8) at rewriteHandler.c:1788
    0x00000000008079cf in ApplyRetrieveRule (parsetree=0x2868820, rule=0x2947298, rt_index=1, 
        relation=0x7f57a6b90be8, activeRIRs=0x29320e8) at rewriteHandler.c:1581
    1581        rule_action = fireRIRrules(rule_action, activeRIRs);
    Value returned is $56 = (Query *) 0x2970118
    ...
    #重写为子查询
    (gdb) n
    1589        rte->rtekind = RTE_SUBQUERY;
    (gdb) p *rte
    $58 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16403, relkind = 118 'v', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x2867f08, eref = 0x2868a40, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x29614e8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) n
    1590        rte->relid = InvalidOid;
    (gdb) 
    1591        rte->security_barrier = RelationIsSecurityView(relation);
    (gdb) 
    1592        rte->subquery = rule_action;
    (gdb) 
    1593        rte->inh = false;           /* must not be set for a subquery */
    (gdb) 
    1599        subrte = rt_fetch(PRS2_OLD_VARNO, rule_action->rtable);
    (gdb) p *rte
    $59 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 118 'v', tablesample = 0x0, 
      subquery = 0x2970118, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x2867f08, eref = 0x2868a40, lateral = false, inh = false, 
      inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x29614e8, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    ...
    #结束ApplyRetrieveRule调用,返回上层的fireRIRrules
    (gdb) finish
    Run till exit from #0  ApplyRetrieveRule (parsetree=0x2868820, rule=0x2947298, rt_index=1, 
        relation=0x7f57a6b90be8, activeRIRs=0x29320e8) at rewriteHandler.c:1601
    0x0000000000807fe3 in fireRIRrules (parsetree=0x2868820, activeRIRs=0x29320e8) at rewriteHandler.c:1832
    1832                        parsetree = ApplyRetrieveRule(parsetree,
    Value returned is $60 = (Query *) 0x2868820
    (gdb) n
    1828                    foreach(l, locks)
    (gdb) 
    1839                    activeRIRs = list_delete_first(activeRIRs);
    (gdb) 
    1843            heap_close(rel, NoLock);
    #RIR处理完毕
    (gdb) p activeRIRs
    $61 = (List *) 0x0
    (gdb) finish
    Run till exit from #0  fireRIRrules (parsetree=0x2868820, activeRIRs=0x0) at rewriteHandler.c:1843
    0x000000000080a8b5 in QueryRewrite (parsetree=0x2868820) at rewriteHandler.c:3604
    3604            query = fireRIRrules(query, NIL);
    Value returned is $62 = (Query *) 0x2868820
    (gdb) finish
    Run till exit from #0  0x000000000080a8b5 in QueryRewrite (parsetree=0x2868820) at rewriteHandler.c:3604
    0x000000000084c945 in pg_rewrite_query (query=0x2868820) at postgres.c:759
    759         querytree_list = QueryRewrite(query);
    Value returned is $63 = (List *) 0x29320e8
    (gdb) c
    Continuing.
    #DONE!
    
    

### 五、数据结构

**RewriteRule**

    
    
     /*
      * RuleLock -      *    all rules that apply to a particular relation. Even though we only
      *    have the rewrite rule system left and these are not really "locks",
      *    the name is kept for historical reasons.
      */
     typedef struct RuleLock //rd_rules
     {
         int         numLocks;
         RewriteRule **rules;
     } RuleLock;
    
     /*
      * RewriteRule -      *    holds an info for a rewrite rule
      *
      */
     typedef struct RewriteRule
     {
         Oid         ruleId;
         CmdType     event;
         Node       *qual;
         List       *actions;
         char        enabled;
         bool        isInstead;
     } RewriteRule;
    
    

### 六、小结

1、查询重写：简单介绍了查询重写的基本概念以及改写前后的异同；  
2、重写实现：通过跟踪分析，PG利用系统定义的规则进行重写，定义的规则Actions为Query结构，重写后在Query中把视图转变为子查询。

