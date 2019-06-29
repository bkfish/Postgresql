本节简单介绍了PG查询优化表达式预处理中的规范化过程。规范化具体的做法一是忽略NULL以及OR中的False,And中的True(实现函数find_duplicate_ors),二是拉平谓词(实现函数:pull_ors/pull_ands),三是清除重复的ORs(实现函数process_duplicate_ors)。这些函数位于文件src/backend/optimizer/prep/prepqual.c中。

### 一、布尔代数基础

规范化处理基于布尔/逻辑代数运算的相关基本定律:  
**幂等律**  
A∪A = A  
A∩A = A  
**交换律**  
A∪B = B∪A  
A∩B = B∩A  
**结合律**  
A∪(B∪С) = (A∪B) ∪ С  
A∩(B∩С) = (A∩B）∩ С  
**吸收律**  
A∪(A∩B) = A  
A∩(A∪B) = A  
**分配律**  
A∪（B∩С）=(A∪B）∩（A∪С）  
A∩（B∪С）=(A∩B）∪（A∩С）  
**幺元律**  
0∪A = A  
1∩A = A  
1∪A = 1  
0∩A = 0  
**补余律**  
A∪A' = 1  
A∩A' = 0

### 二、基本概念

PG源码对规范化表达式的注释如下:

    
    
     /*
      * canonicalize_qual
      *    Convert a qualification expression to the most useful form.
      *    转换为最常规形式的表达式
      *
      * This is primarily intended to be used on top-level WHERE (or JOIN/ON)
      * clauses.  It can also be used on top-level CHECK constraints, for which
      * pass is_check = true.  DO NOT call it on any expression that is not known
      * to be one or the other, as it might apply inappropriate simplifications.
      * 规范化注意用于最高层的Where/Check(输入参数:is_check = true)语句中,如果不是这类语句,不要使用否则会产生不匹配的简化(PG这里加入了DEBUG信息)
      * 
      * The name of this routine is a holdover from a time when it would try to
      * force the expression into canonical AND-of-ORs or OR-of-ANDs form.
      * Eventually, we recognized that that had more theoretical purity than
      * actual usefulness, and so now the transformation doesn't involve any
      * notion of reaching a canonical form.
      *
      * NOTE: we assume the input has already been through eval_const_expressions
      * and therefore possesses AND/OR flatness.  Formerly this function included
      * its own flattening logic, but that requires a useless extra pass over the
      * tree.
      *
      * Returns the modified qualification.
      */
    

**忽略NULL以及OR中的False/AND中的TRUE**  
Where条件语句中的NULL/FALSE/TRUE,如能忽略,忽略之.如:NULL OR FALSE OR dwbh = '1001',则忽略NULL/
FALSE

    
    
    testdb=# explain verbose select * from t_dwxx where NULL OR FALSE OR dwbh = '1001' ;
                               QUERY PLAN                           
    ----------------------------------------------------------------     Seq Scan on public.t_dwxx  (cost=0.00..12.00 rows=1 width=474)
       Output: dwmc, dwbh, dwdz
       Filter: ((t_dwxx.dwbh)::text = '1001'::text)
    (3 rows)
    

**拉平谓词**  
SQL语句中的X1 OR/AND (X2 OR/AND X3),拉平简化为OR/AND(X1,X2,X3)  
X1 OR/AND (X2 OR/AND X3),在查询树中为树状结构,第一层节点是BoolExpr,该Node中的args链表有2个元素,args->1=X1,args->2=BoolExpr,args->2->1=X2,args->2->2=X3,组成树状结构.简化后args->1/2/3=X1/X2/X3,所有条件处于同一个层次上,并不是树状结构.

    
    
    testdb=# explain select * from t_dwxx where dwbh = '1001' OR (dwbh = '1002' OR dwbh = '1003');
                                                     QUERY PLAN                                                  
    -------------------------------------------------------------------------------------------------------------     Seq Scan on t_dwxx  (cost=0.00..12.80 rows=3 width=474)
       Filter: (((dwbh)::text = '1001'::text) OR ((dwbh)::text = '1002'::text) OR ((dwbh)::text = '1003'::text))
    (2 rows)
    

**清除重复ORs**  
清除重复ORs的数学基础是布尔(逻辑)代数:  
(X1 AND X2) OR (X1 AND X3) 应用分配律可以改写为 X1 AND (X2 OR
X3),这样改写的目的是把X1抽取出来,为后续下推谓词X1作准备.  
如(dwbh = '1001' AND dwbh = '1002') OR (dwbh = '1001' AND dwbh =
'1003')条件,会改写为dwbh = '1001' AND (dwbh = '1002' OR dwbh = '1003')

    
    
    testdb=# explain verbose select * from t_dwxx where (dwbh = '1001' AND dwbh = '1002')  OR (dwbh = '1001' AND dwbh = '1003');
                                                                 QUERY PLAN                                                      
            
    -----------------------------------------------------------------------------------------------------------------------------    --------     Seq Scan on public.t_dwxx  (cost=0.00..12.80 rows=1 width=474)
       Output: dwmc, dwbh, dwdz
       Filter: (((t_dwxx.dwbh)::text = '1001'::text) AND (((t_dwxx.dwbh)::text = '1002'::text) OR ((t_dwxx.dwbh)::text = '1003'::
    text)))
    (3 rows)
    

### 三、源码解读

主函数入口:  
**subquery_planner**

    
    
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
          */
         if (parse->cteList)
             SS_process_ctes(root);//处理With 语句
     
         /*
          * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
          * to transform them into joins.  Note that this step does not descend
          * into subqueries; if we pull up any subqueries below, their SubLinks are
          * processed just before pulling them up.
          */
         if (parse->hasSubLinks)
             pull_up_sublinks(root); //上拉子链接
     
         /*
          * Scan the rangetable for set-returning functions, and inline them if
          * possible (producing subqueries that might get pulled up next).
          * Recursion issues here are handled in the same way as for SubLinks.
          */
         inline_set_returning_functions(root);//
     
         /*
          * Check to see if any subqueries in the jointree can be merged into this
          * query.
          */
         pull_up_subqueries(root);//上拉子查询
     
         /*
          * If this is a simple UNION ALL query, flatten it into an appendrel. We
          * do this now because it requires applying pull_up_subqueries to the leaf
          * queries of the UNION ALL, which weren't touched above because they
          * weren't referenced by the jointree (they will be after we do this).
          */
         if (parse->setOperations)
             flatten_simple_union_all(root);//扁平化处理UNION ALL
     
         /*
          * Detect whether any rangetable entries are RTE_JOIN kind; if not, we can
          * avoid the expense of doing flatten_join_alias_vars().  Also check for
          * outer joins --- if none, we can skip reduce_outer_joins().  And check
          * for LATERAL RTEs, too.  This must be done after we have done
          * pull_up_subqueries(), of course.
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
          */
          //展开继承表
         expand_inherited_tables(root);
     
         /*
          * Set hasHavingQual to remember if HAVING clause is present.  Needed
          * because preprocess_expression will reduce a constant-true condition to
          * an empty qual list ... but "HAVING TRUE" is not a semantic no-op.
          */
         //是否存在Having表达式
         root->hasHavingQual = (parse->havingQual != NULL);
     
         /* Clear this flag; might get set in distribute_qual_to_rels */
         root->hasPseudoConstantQuals = false;
     
         /*
          * Do expression preprocessing on targetlist and quals, as well as other
          * random expressions in the querytree.  Note that we do not need to
          * handle sort/group expressions explicitly, because they are actually
          * part of the targetlist.
          */
         //预处理表达式:targetList(投影列)
         parse->targetList = (List *)
             preprocess_expression(root, (Node *) parse->targetList,
                                   EXPRKIND_TARGET);
     
         /* Constant-folding might have removed all set-returning functions */
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
                  */
                 if (rte->lateral && root->hasJoinRTEs)
                     rte->subquery = (Query *)
                         flatten_join_alias_vars(root, (Node *) rte->subquery);
             }
             else if (rte->rtekind == RTE_FUNCTION)//函数
             {
                 /* Preprocess the function expression(s) fully */
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
              */
             foreach(lcsq, rte->securityQuals)
             {
                 lfirst(lcsq) = preprocess_expression(root,
                                                      (Node *) lfirst(lcsq),
                                                      EXPRKIND_QUAL);
             }
         }
     
         ...//其他
         
         return root;
     }
     
    
    

**preprocess_expression**

    
    
     /*
      * preprocess_expression
      *      Do subquery_planner's preprocessing work for an expression,
      *      which can be a targetlist, a WHERE clause (including JOIN/ON
      *      conditions), a HAVING clause, or a few other things.
      */
     static Node *
     preprocess_expression(PlannerInfo *root, Node *expr, int kind)
     {
         /*
          * Fall out quickly if expression is empty.  This occurs often enough to
          * be worth checking.  Note that null->null is the correct conversion for
          * implicit-AND result format, too.
          */
         if (expr == NULL)
             return NULL;
     
         /*
          * If the query has any join RTEs, replace join alias variables with
          * base-relation variables.  We must do this first, since any expressions
          * we may extract from the joinaliasvars lists have not been preprocessed.
          * For example, if we did this after sublink processing, sublinks expanded
          * out from join aliases would not get processed.  But we can skip this in
          * non-lateral RTE functions, VALUES lists, and TABLESAMPLE clauses, since
          * they can't contain any Vars of the current query level.
          */
         if (root->hasJoinRTEs &&
             !(kind == EXPRKIND_RTFUNC ||
               kind == EXPRKIND_VALUES ||
               kind == EXPRKIND_TABLESAMPLE ||
               kind == EXPRKIND_TABLEFUNC))
             expr = flatten_join_alias_vars(root, expr);//扁平化处理joinaliasvars,上节已介绍
     
         /*
          * Simplify constant expressions.
          *
          * Note: an essential effect of this is to convert named-argument function
          * calls to positional notation and insert the current actual values of
          * any default arguments for functions.  To ensure that happens, we *must*
          * process all expressions here.  Previous PG versions sometimes skipped
          * const-simplification if it didn't seem worth the trouble, but we can't
          * do that anymore.
          *
          * Note: this also flattens nested AND and OR expressions into N-argument
          * form.  All processing of a qual expression after this point must be
          * careful to maintain AND/OR flatness --- that is, do not generate a tree
          * with AND directly under AND, nor OR directly under OR.
          */
         expr = eval_const_expressions(root, expr);//简化常量表达式
     
         /*
          * If it's a qual or havingQual, canonicalize it.
          */
         if (kind == EXPRKIND_QUAL)
         {
             expr = (Node *) canonicalize_qual((Expr *) expr, false);//表达式规范化
     
     #ifdef OPTIMIZER_DEBUG
             printf("After canonicalize_qual()\n");
             pprint(expr);
     #endif
         }
     
         /* Expand SubLinks to SubPlans */
         if (root->parse->hasSubLinks)//扩展子链接为子计划
             expr = SS_process_sublinks(root, expr, (kind == EXPRKIND_QUAL));
     
         /*
          * XXX do not insert anything here unless you have grokked the comments in
          * SS_replace_correlation_vars ...
          */
     
         /* Replace uplevel vars with Param nodes (this IS possible in VALUES) */
         if (root->query_level > 1)
             expr = SS_replace_correlation_vars(root, expr);//使用Param节点替换上层的Vars
     
         /*
          * If it's a qual or havingQual, convert it to implicit-AND format. (We
          * don't want to do this before eval_const_expressions, since the latter
          * would be unable to simplify a top-level AND correctly. Also,
          * SS_process_sublinks expects explicit-AND format.)
          */
         if (kind == EXPRKIND_QUAL)
             expr = (Node *) make_ands_implicit((Expr *) expr);
     
         return expr;
     }
    

**canonicalize_qual**

    
    
     /*
      * canonicalize_qual
      *    Convert a qualification expression to the most useful form.
      *
      * This is primarily intended to be used on top-level WHERE (or JOIN/ON)
      * clauses.  It can also be used on top-level CHECK constraints, for which
      * pass is_check = true.  DO NOT call it on any expression that is not known
      * to be one or the other, as it might apply inappropriate simplifications.
      *
      * The name of this routine is a holdover from a time when it would try to
      * force the expression into canonical AND-of-ORs or OR-of-ANDs form.
      * Eventually, we recognized that that had more theoretical purity than
      * actual usefulness, and so now the transformation doesn't involve any
      * notion of reaching a canonical form.
      *
      * NOTE: we assume the input has already been through eval_const_expressions
      * and therefore possesses AND/OR flatness.  Formerly this function included
      * its own flattening logic, but that requires a useless extra pass over the
      * tree.
      *
      * Returns the modified qualification.
      */
     Expr *
     canonicalize_qual(Expr *qual, bool is_check)//规范化表达式
     {
         Expr       *newqual;
     
         /* Quick exit for empty qual */
         if (qual == NULL)
             return NULL;
     
         /* This should not be invoked on quals in implicit-AND format */
         Assert(!IsA(qual, List));
     
         /*
          * Pull up redundant subclauses in OR-of-AND trees.  We do this only
          * within the top-level AND/OR structure; there's no point in looking
          * deeper.  Also remove any NULL constants in the top-level structure.
          */
         newqual = find_duplicate_ors(qual, is_check);//执行实际处理逻辑
     
         return newqual;
     }
    
    

**find_duplicate_ors**

    
    
     /*
      * find_duplicate_ors
      *    Given a qualification tree with the NOTs pushed down, search for
      *    OR clauses to which the inverse OR distributive law might apply.
      *    Only the top-level AND/OR structure is searched.
      *
      * While at it, we remove any NULL constants within the top-level AND/OR
      * structure, eg in a WHERE clause, "x OR NULL::boolean" is reduced to "x".
      * In general that would change the result, so eval_const_expressions can't
      * do it; but at top level of WHERE, we don't need to distinguish between
      * FALSE and NULL results, so it's valid to treat NULL::boolean the same
      * as FALSE and then simplify AND/OR accordingly.  Conversely, in a top-level
      * CHECK constraint, we may treat a NULL the same as TRUE.
      *
      * Returns the modified qualification.  AND/OR flatness is preserved.
      */
     static Expr *
     find_duplicate_ors(Expr *qual, bool is_check)
     {
         if (or_clause((Node *) qual))//OR语句
         {
             List       *orlist = NIL;
             ListCell   *temp;
     
             /* Recurse */
             foreach(temp, ((BoolExpr *) qual)->args)//遍历args链表
             {
                 Expr       *arg = (Expr *) lfirst(temp);//获取链表中的元素
     
                 arg = find_duplicate_ors(arg, is_check);//递归调用
     
                 /* Get rid of any constant inputs */
                 if (arg && IsA(arg, Const))//arg为常量
                 {
                     Const      *carg = (Const *) arg;
     
                     if (is_check)//Check语句
                     {
                         /* Within OR in CHECK, drop constant FALSE */
                         if (!carg->constisnull && !DatumGetBool(carg->constvalue))
                             continue;//不为NULL而且为FALSE,继续循环
                         //arg为NULL或者TURE,即NULL OR TRUE,简化为TRUE
                         /* Constant TRUE or NULL, so OR reduces to TRUE */
                         return (Expr *) makeBoolConst(true, false);
                     }
                     else//Where条件语句
                     {
                         /* Within OR in WHERE, drop constant FALSE or NULL */
                         if (carg->constisnull || !DatumGetBool(carg->constvalue))
                             continue;//arg为NULL或者FALSE,继续循环
                         //arg不为NULL而且为TRUE,即TRUE常量,直接返回arg
                         /* Constant TRUE, so OR reduces to TRUE */
                         return arg;
                     }
                 }
                 //加入结果链表
                 orlist = lappend(orlist, arg);
             }
     
             /* Flatten any ORs pulled up to just below here */
             orlist = pull_ors(orlist);//扁平化ORs
     
             /* Now we can look for duplicate ORs */
             return process_duplicate_ors(orlist);//处理重复的ORs
         }
         else if (and_clause((Node *) qual))//AND语句
         {
             List       *andlist = NIL;
             ListCell   *temp;
     
             /* Recurse */
             foreach(temp, ((BoolExpr *) qual)->args)//遍历链表
             {
                 Expr       *arg = (Expr *) lfirst(temp);
     
                 arg = find_duplicate_ors(arg, is_check);
     
                 /* Get rid of any constant inputs */
                 if (arg && IsA(arg, Const))
                 {
                     Const      *carg = (Const *) arg;
     
                     if (is_check)
                     {
                         /* Within AND in CHECK, drop constant TRUE or NULL */
                         if (carg->constisnull || DatumGetBool(carg->constvalue))
                             continue;//CHECK语句
                         /* Constant FALSE, so AND reduces to FALSE */
                         //不为空且值为FALSE,返回FALSE
                         return arg;
                     }
                     else
                     {
                         /* Within AND in WHERE, drop constant TRUE */
                         if (!carg->constisnull && DatumGetBool(carg->constvalue))
                             continue;
                         /* Constant FALSE or NULL, so AND reduces to FALSE */
                         //NULL OR FALSE,返回FALSE
                         return (Expr *) makeBoolConst(false, false);
                     }
                 }
     
                 andlist = lappend(andlist, arg);//加入到结果列表
             }
     
             /* Flatten any ANDs introduced just below here */
             andlist = pull_ands(andlist);//扁平化处理AND
     
             /* AND of no inputs reduces to TRUE */
             if (andlist == NIL)//为空指针
                 return (Expr *) makeBoolConst(true, false);//返回TRUE
     
             /* Single-expression AND just reduces to that expression */
             if (list_length(andlist) == 1)//单个表达式
                 return (Expr *) linitial(andlist);//返回此表达式链表
     
             /* Else we still need an AND node */
             return make_andclause(andlist);//否则返回结果链表
         }
         else
             return qual;//非AND/OR语句,直接返回结果
     }
    

**process_duplicate_ors**

    
    
     /*
      * process_duplicate_ors
      *    Given a list of exprs which are ORed together, try to apply
      *    the inverse OR distributive law.
      *
      * Returns the resulting expression (could be an AND clause, an OR
      * clause, or maybe even a single subexpression).
      */
     static Expr *
     process_duplicate_ors(List *orlist)
     {
         List       *reference = NIL;
         int         num_subclauses = 0;
         List       *winners;
         List       *neworlist;
         ListCell   *temp;
     
         /* OR of no inputs reduces to FALSE */
         if (orlist == NIL)
             return (Expr *) makeBoolConst(false, false);
     
         /* Single-expression OR just reduces to that expression */
         if (list_length(orlist) == 1)
             return (Expr *) linitial(orlist);
     
         /*
          * Choose the shortest AND clause as the reference list --- obviously, any
          * subclause not in this clause isn't in all the clauses. If we find a
          * clause that's not an AND, we can treat it as a one-element AND clause,
          * which necessarily wins as shortest.
          */
         //遍历OR链表,找到AND语句中约束条件最少的那个表达式
         //与求解最小公约数同理,公共的谓词只可能在此最小的表达式中产生 
         foreach(temp, orlist)
         {
             Expr       *clause = (Expr *) lfirst(temp);
     
             if (and_clause((Node *) clause))//AND语句
             {
                 List       *subclauses = ((BoolExpr *) clause)->args;
                 int         nclauses = list_length(subclauses);
     
                 if (reference == NIL || nclauses < num_subclauses)
                 {
                     reference = subclauses;
                     num_subclauses = nclauses;
                 }
             }
             else//单个约束条件或者带有表达式的约束条件,如(X1 AND X2) OR X3等
             {
                 reference = list_make1(clause);
                 break;
             }
         }
     
         /*
          * Just in case, eliminate any duplicates in the reference list.
          */
         reference = list_union(NIL, reference);//去掉重复的谓词
     
         /*
          * Check each element of the reference list to see if it's in all the OR
          * clauses.  Build a new list of winning clauses.
          */
         winners = NIL;
         foreach(temp, reference)//遍历链表
         {
             Expr       *refclause = (Expr *) lfirst(temp);
             bool        win = true;
             ListCell   *temp2;
     
             foreach(temp2, orlist)
             {
                 Expr       *clause = (Expr *) lfirst(temp2);
     
                 if (and_clause((Node *) clause))//该谓词是否在链表中存在?
                 {
                     if (!list_member(((BoolExpr *) clause)->args, refclause))
                     {
                         win = false;
                         break;
                     }
                 }
                 else//该谓词是否与单个条件表达式等价?
                 {
                     if (!equal(refclause, clause))
                     {
                         win = false;
                         break;
                     }
                 }
             }
     
             if (win)//找到了公共的谓词
                 winners = lappend(winners, refclause);//加入到结果中
         }
     
         /*
          * If no winners, we can't transform the OR
          */
         if (winners == NIL)
             return make_orclause(orlist);//如果找到,原样返回
     
         /*
          * Generate new OR list consisting of the remaining sub-clauses.
          *
          * 找到了,产生新的条件
          *
          * If any clause degenerates to empty, then we have a situation like (A
          * AND B) OR (A), which can be reduced to just A --- that is, the
          * additional conditions in other arms of the OR are irrelevant.
          *
          * Note that because we use list_difference, any multiple occurrences of a
          * winning clause in an AND sub-clause will be removed automatically.
          */
         neworlist = NIL;
         foreach(temp, orlist)//遍历OR链表
         {
             Expr       *clause = (Expr *) lfirst(temp);
     
             if (and_clause((Node *) clause))//AND语句
             {
                 List       *subclauses = ((BoolExpr *) clause)->args;//获取条件语句参数
     
                 subclauses = list_difference(subclauses, winners);//剔除相同部分
                 if (subclauses != NIL)//成功剔除,产生新的AND语句
                 {
                     if (list_length(subclauses) == 1)
                         neworlist = lappend(neworlist, linitial(subclauses));
                     else
                         neworlist = lappend(neworlist, make_andclause(subclauses));
                 }
                 else
                 {
                     neworlist = NIL;    /* degenerate case, see above */
                     break;
                 }
             }
             else//不是AND语句
             {
                 if (!list_member(winners, clause))//单个条件语句,直接添加条件
                     neworlist = lappend(neworlist, clause);
                 else
                 //OR语句?公共部分已提出,无需加入其他条件,直接返回
                 //根据吸收律,X AND (X OR B) 等价于X
                 {
                     neworlist = NIL;    /* degenerate case, see above */
                     break;
                 }
             }
         }
     
         /*
          * Append reduced OR to the winners list, if it's not degenerate, handling
          * the special case of one element correctly (can that really happen?).
          * Also be careful to maintain AND/OR flatness in case we pulled up a
          * sub-sub-OR-clause.
          */
         if (neworlist != NIL)//新产生的链表
         {
             if (list_length(neworlist) == 1)
                 winners = lappend(winners, linitial(neworlist));
             else
                 winners = lappend(winners, make_orclause(pull_ors(neworlist)));//拉平OR
         }
     
         /*
          * And return the constructed AND clause, again being wary of a single
          * element and AND/OR flatness.
          */
         //返回结果 
         if (list_length(winners) == 1)
             return (Expr *) linitial(winners);
         else
             return make_andclause(pull_ands(winners));//拉平AND
     }
    

**pull_ors**

    
    
     /*
      * pull_ors
      *    Recursively flatten nested OR clauses into a single or-clause list.
      *
      * Input is the arglist of an OR clause.
      * Returns the rebuilt arglist (note original list structure is not touched).
      */
     static List *
     pull_ors(List *orlist)
     {
         List       *out_list = NIL;
         ListCell   *arg;
     
         foreach(arg, orlist)
         {
             Node       *subexpr = (Node *) lfirst(arg);
     
             /*
              * Note: we can destructively concat the subexpression's arglist
              * because we know the recursive invocation of pull_ors will have
              * built a new arglist not shared with any other expr. Otherwise we'd
              * need a list_copy here.
              */
             if (or_clause(subexpr))
                 out_list = list_concat(out_list,
                                        pull_ors(((BoolExpr *) subexpr)->args));//递归拉平
             else
                 out_list = lappend(out_list, subexpr);
         }
         return out_list;
     }
    

**pull_ands**

    
    
     /*
      * pull_ands
      *    Recursively flatten nested AND clauses into a single and-clause list.
      *
      * Input is the arglist of an AND clause.
      * Returns the rebuilt arglist (note original list structure is not touched).
      */
     static List *
     pull_ands(List *andlist)
     {
         List       *out_list = NIL;
         ListCell   *arg;
     
         foreach(arg, andlist)
         {
             Node       *subexpr = (Node *) lfirst(arg);
     
             /*
              * Note: we can destructively concat the subexpression's arglist
              * because we know the recursive invocation of pull_ands will have
              * built a new arglist not shared with any other expr. Otherwise we'd
              * need a list_copy here.
              */
             if (and_clause(subexpr))
                 out_list = list_concat(out_list,
                                        pull_ands(((BoolExpr *) subexpr)->args));//递归拉平
             else
                 out_list = lappend(out_list, subexpr);
         }
         return out_list;
     }
    
    

### 四、跟踪分析

测试脚本:

    
    
    select * from t_dwxx 
    where (FALSE OR dwbh = '1001') 
           AND ((dwbh = '1001' AND dwbh = '1002')  
                 OR (dwbh = '1001' AND dwbh = '1003'));
    
    

该语句经规范化后与以下SQL语句无异:

    
    
    select * from t_dwxx 
    where  dwbh = '1001' AND (dwbh='1002' OR dwbh='1003');
    
    -- 执行计划
    testdb=# explain verbose select * from t_dwxx 
    where (FALSE OR dwbh = '1001') 
           AND ((dwbh = '1001' AND dwbh = '1002')  
                 OR (dwbh = '1001' AND dwbh = '1003'));
                                                                 QUERY PLAN                                                      
            
    -----------------------------------------------------------------------------------------------------------------------------    --------     Seq Scan on public.t_dwxx  (cost=0.00..12.80 rows=1 width=474)
       Output: dwmc, dwbh, dwdz
       Filter: (((t_dwxx.dwbh)::text = '1001'::text) AND (((t_dwxx.dwbh)::text = '1002'::text) OR ((t_dwxx.dwbh)::text = '1003'::
    text)))
    (3 rows)
    
    

gdb跟踪:

    
    
    (gdb) b find_duplicate_ors
    Breakpoint 1 at 0x7811b4: file prepqual.c, line 418.
    (gdb) c
    Continuing.
    
    Breakpoint 1, find_duplicate_ors (qual=0x2727528, is_check=false) at prepqual.c:418
    418   if (or_clause((Node *) qual))
    #输入参数,BoolExpr
    #args链表有两个元素,一个是(FALSE OR dwbh = '1001'),
    #另外一个是((dwbh = '1001' AND dwbh = '1002')  
                 OR (dwbh = '1001' AND dwbh = '1003'))
    (gdb) p *(BoolExpr *)qual
    $2 = {xpr = {type = T_BoolExpr}, boolop = AND_EXPR, args = 0x2726c88, location = -1}
    (gdb) p *((BoolExpr *)qual)->args
    $3 = {type = T_List, length = 2, head = 0x2726c68, tail = 0x2727508}
    ...
    #进入AND分支
    462   else if (and_clause((Node *) qual))
    ...
    #获得链表第一个元素
    470       Expr     *arg = (Expr *) lfirst(temp);
    #再次进入find_duplicate_ors
    #FALSE OR dwbh='1001'在此前已被简化为dwbh='1001',所以在这里直接返回
    Breakpoint 1, find_duplicate_ors (qual=0x2726bc8, is_check=false) at prepqual.c:418
    418   if (or_clause((Node *) qual))
    (gdb) n
    462   else if (and_clause((Node *) qual))
    (gdb) 
    #
    515     return qual;
    (gdb) n
    516 }
    (gdb) 
    475       if (arg && IsA(arg, Const))
    ...
    497       andlist = lappend(andlist, arg);
    (gdb) n
    468     foreach(temp, ((BoolExpr *) qual)->args)
    (gdb) n
    470       Expr     *arg = (Expr *) lfirst(temp);
    (gdb) 
    472       arg = find_duplicate_ors(arg, is_check);
    #链表的下一个元素,即
    #((dwbh = '1001' AND dwbh = '1002')  
                 OR (dwbh = '1001' AND dwbh = '1003'))
    #由两个BoolExpr组成
    (gdb) p *(BoolExpr *)arg
    $23 = {xpr = {type = T_BoolExpr}, boolop = OR_EXPR, args = 0x27270d8, location = -1}
    (gdb) p *((BoolExpr *)arg)->args
    $24 = {type = T_List, length = 2, head = 0x27270b8, tail = 0x27274b8}
    (gdb) p *(Node *)((BoolExpr *)arg)->args->head->data.ptr_value
    $25 = {type = T_BoolExpr}
    (gdb) p *(Node *)((BoolExpr *)arg)->args->head->next->data.ptr_value
    $26 = {type = T_BoolExpr}
    ...
    #args左右两边的的arg处理完毕
    424     foreach(temp, ((BoolExpr *) qual)->args)
    (gdb) 
    457     orlist = pull_ors(orlist);
    ...
    #进入process_duplicate_ors
    460     return process_duplicate_ors(orlist);
    (gdb) step
    process_duplicate_ors (orlist=0x2727858) at prepqual.c:529
    529   List     *reference = NIL;
    ...
    #获得最少谓词的链表
    (gdb) p *reference
    $36 = {type = T_List, length = 2, head = 0x27278a8, tail = 0x27278f8}
    ...
    #获得了公共的谓词winner!即dwbh = '1001'
    (gdb) p *winners
    $40 = {type = T_List, length = 1, head = 0x2727918, tail = 0x2727918}
    ...
    (gdb) 
    canonicalize_qual (qual=0x2727528, is_check=false) at prepqual.c:309
    309   return newqual;
    (gdb) p *newqual
    $43 = {type = T_BoolExpr}
    (gdb) p *(BoolExpr *)newqual
    $45 = {xpr = {type = T_BoolExpr}, boolop = AND_EXPR, args = 0x2727c18, location = -1}
    #DONE!
    

### 五、小结

1、优化的数学基础：布尔代数以及相关的定律；  
2、表达式规范化的过程：表达式扁平化处理以及公共谓词提取等的处理逻辑。

### 六、参考资料

[布尔代数](https://zh.wikipedia.org/wiki/%E5%B8%83%E5%B0%94%E4%BB%A3%E6%95%B0)  
[prepqual.c](https://doxygen.postgresql.org/prepqual_8c_source.html#l00291)

