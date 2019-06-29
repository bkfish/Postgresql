本节简单介绍了PG查询优化表达式预处理中常量的简化过程。表达式预处理主要的函数主要有preprocess_expression和preprocess_qual_conditions（调用preprocess_expression），在文件src/backend/optimizer/plan/planner.c中。preprocess_expression调用了eval_const_expressions，该函数调用了mutator函数通过遍历的方式对表达式进行处理。

### 一、基本概念

PG源码对简化表达式的注释如下:

    
    
     /*--------------------      * eval_const_expressions
      *
      * Reduce any recognizably constant subexpressions of the given
      * expression tree, for example "2 + 2" => "4".  More interestingly,
      * we can reduce certain boolean expressions even when they contain
      * non-constant subexpressions: "x OR true" => "true" no matter what
      * the subexpression x is.  (XXX We assume that no such subexpression
      * will have important side-effects, which is not necessarily a good
      * assumption in the presence of user-defined functions; do we need a
      * pg_proc flag that prevents discarding the execution of a function?)
      *
      * We do understand that certain functions may deliver non-constant
      * results even with constant inputs, "nextval()" being the classic
      * example.  Functions that are not marked "immutable" in pg_proc
      * will not be pre-evaluated here, although we will reduce their
      * arguments as far as possible.
      *
      * Whenever a function is eliminated from the expression by means of
      * constant-expression evaluation or inlining, we add the function to
      * root->glob->invalItems.  This ensures the plan is known to depend on
      * such functions, even though they aren't referenced anymore.
      *
      * We assume that the tree has already been type-checked and contains
      * only operators and functions that are reasonable to try to execute.
      *
      * NOTE: "root" can be passed as NULL if the caller never wants to do any
      * Param substitutions nor receive info about inlined functions.
      *
      * NOTE: the planner assumes that this will always flatten nested AND and
      * OR clauses into N-argument form.  See comments in prepqual.c.
      *
      * NOTE: another critical effect is that any function calls that require
      * default arguments will be expanded, and named-argument calls will be
      * converted to positional notation.  The executor won't handle either.
      *--------------------      */
    

比如表达式1 + 2,直接求解得到3;x OR true,直接求解得到true而无需理会x的值,类似的x AND
false直接求解得到false而无需理会x的值.  
不过,这里的简化只是执行了基础分析,并没有做深入分析:

    
    
    testdb=# explain verbose select max(a.dwbh::int+(1+2)) from t_dwxx a;
                                   QUERY PLAN                                
    -------------------------------------------------------------------------     Aggregate  (cost=13.20..13.21 rows=1 width=4)
       Output: max(((dwbh)::integer + 3))
       ->  Seq Scan on public.t_dwxx a  (cost=0.00..11.60 rows=160 width=38)
             Output: dwmc, dwbh, dwdz
    (4 rows)
    
    testdb=# explain verbose select max(a.dwbh::int+1+2) from t_dwxx a;
                                   QUERY PLAN                                
    -------------------------------------------------------------------------     Aggregate  (cost=13.60..13.61 rows=1 width=4)
       Output: max((((dwbh)::integer + 1) + 2))
       ->  Seq Scan on public.t_dwxx a  (cost=0.00..11.60 rows=160 width=38)
             Output: dwmc, dwbh, dwdz
    (4 rows)
    

见上测试脚本,如(1+2),把括号去掉,a.dwbh先跟1运算,再跟2运算,没有执行简化.

### 二、源码解读

主函数入口:  
**subquery_planner**

    
    
     /*--------------------      * subquery_planner
      *    Invokes the planner on a subquery.  We recurse to here for each
      *    sub-SELECT found in the query tree.
      *
      * glob is the global state for the current planner run.
      * parse is the querytree produced by the parser & rewriter.
      * parent_root is the immediate parent Query's info (NULL at the top level).
      * hasRecursion is true if this is a recursive WITH query.
      * tuple_fraction is the fraction of tuples we expect will be retrieved.
      * tuple_fraction is interpreted as explained for grouping_planner, below.
      *
      * Basically, this routine does the stuff that should only be done once
      * per Query object.  It then calls grouping_planner.  At one time,
      * grouping_planner could be invoked recursively on the same Query object;
      * that's not currently true, but we keep the separation between the two
      * routines anyway, in case we need it again someday.
      *
      * subquery_planner will be called recursively to handle sub-Query nodes
      * found within the query's expressions and rangetable.
      *
      * Returns the PlannerInfo struct ("root") that contains all data generated
      * while planning the subquery.  In particular, the Path(s) attached to
      * the (UPPERREL_FINAL, NULL) upperrel represent our conclusions about the
      * cheapest way(s) to implement the query.  The top level will select the
      * best Path and pass it through createplan.c to produce a finished Plan.
      *--------------------      */
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
             expr = (Node *) canonicalize_qual((Expr *) expr, false);//表达式规约,下节介绍
     
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
    

**preprocess_qual_conditions**

    
    
     /*
      * preprocess_qual_conditions
      *      Recursively scan the query's jointree and do subquery_planner's
      *      preprocessing work on each qual condition found therein.
      */
     static void
     preprocess_qual_conditions(PlannerInfo *root, Node *jtnode)
     {
         if (jtnode == NULL)
             return;
         if (IsA(jtnode, RangeTblRef))
         {
             /* nothing to do here */
         }
         else if (IsA(jtnode, FromExpr))
         {
             FromExpr   *f = (FromExpr *) jtnode;
             ListCell   *l;
     
             foreach(l, f->fromlist)
                 preprocess_qual_conditions(root, lfirst(l));//递归调用
     
             f->quals = preprocess_expression(root, f->quals, EXPRKIND_QUAL);
         }
         else if (IsA(jtnode, JoinExpr))
         {
             JoinExpr   *j = (JoinExpr *) jtnode;
     
             preprocess_qual_conditions(root, j->larg);//递归调用
             preprocess_qual_conditions(root, j->rarg);//递归调用
     
             j->quals = preprocess_expression(root, j->quals, EXPRKIND_QUAL);
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
     }
    

**eval_const_expressions**

    
    
     Node *
     eval_const_expressions(PlannerInfo *root, Node *node)
     {
         eval_const_expressions_context context;
     
         if (root)
             context.boundParams = root->glob->boundParams;  /* bound Params */
         else
             context.boundParams = NULL;
         context.root = root;        /* for inlined-function dependencies */
         context.active_fns = NIL;   /* nothing being recursively simplified */
         context.case_val = NULL;    /* no CASE being examined */
         context.estimate = false;   /* safe transformations only */
         //调用XX_mutator函数遍历处理
         return eval_const_expressions_mutator(node, &context);
     }
    

**eval_const_expressions_mutator**

    
    
    /*
      * Recursive guts of eval_const_expressions/estimate_expression_value
      */
     static Node *
     eval_const_expressions_mutator(Node *node,
                                    eval_const_expressions_context *context)
     {
         if (node == NULL)
             return NULL;
         switch (nodeTag(node))
         {
             case T_Param:
                 {
                     Param      *param = (Param *) node;
                     ParamListInfo paramLI = context->boundParams;
     
                     /* Look to see if we've been given a value for this Param */
                     if (param->paramkind == PARAM_EXTERN &&
                         paramLI != NULL &&
                         param->paramid > 0 &&
                         param->paramid <= paramLI->numParams)
                     {
                         ParamExternData *prm;
                         ParamExternData prmdata;
     
                         /*
                          * Give hook a chance in case parameter is dynamic.  Tell
                          * it that this fetch is speculative, so it should avoid
                          * erroring out if parameter is unavailable.
                          */
                         if (paramLI->paramFetch != NULL)
                             prm = paramLI->paramFetch(paramLI, param->paramid,
                                                       true, &prmdata);
                         else
                             prm = &paramLI->params[param->paramid - 1];
     
                         /*
                          * We don't just check OidIsValid, but insist that the
                          * fetched type match the Param, just in case the hook did
                          * something unexpected.  No need to throw an error here
                          * though; leave that for runtime.
                          */
                         if (OidIsValid(prm->ptype) &&
                             prm->ptype == param->paramtype)
                         {
                             /* OK to substitute parameter value? */
                             if (context->estimate ||
                                 (prm->pflags & PARAM_FLAG_CONST))
                             {
                                 /*
                                  * Return a Const representing the param value.
                                  * Must copy pass-by-ref datatypes, since the
                                  * Param might be in a memory context
                                  * shorter-lived than our output plan should be.
                                  */
                                 int16       typLen;
                                 bool        typByVal;
                                 Datum       pval;
     
                                 get_typlenbyval(param->paramtype,
                                                 &typLen, &typByVal);
                                 if (prm->isnull || typByVal)
                                     pval = prm->value;
                                 else
                                     pval = datumCopy(prm->value, typByVal, typLen);
                                 return (Node *) makeConst(param->paramtype,
                                                           param->paramtypmod,
                                                           param->paramcollid,
                                                           (int) typLen,
                                                           pval,
                                                           prm->isnull,
                                                           typByVal);
                             }
                         }
                     }
     
                     /*
                      * Not replaceable, so just copy the Param (no need to
                      * recurse)
                      */
                     return (Node *) copyObject(param);
                 }
             case T_WindowFunc:
                 {
                     WindowFunc *expr = (WindowFunc *) node;
                     Oid         funcid = expr->winfnoid;
                     List       *args;
                     Expr       *aggfilter;
                     HeapTuple   func_tuple;
                     WindowFunc *newexpr;
     
                     /*
                      * We can't really simplify a WindowFunc node, but we mustn't
                      * just fall through to the default processing, because we
                      * have to apply expand_function_arguments to its argument
                      * list.  That takes care of inserting default arguments and
                      * expanding named-argument notation.
                      */
                     func_tuple = SearchSysCache1(PROCOID, ObjectIdGetDatum(funcid));
                     if (!HeapTupleIsValid(func_tuple))
                         elog(ERROR, "cache lookup failed for function %u", funcid);
     
                     args = expand_function_arguments(expr->args, expr->wintype,
                                                      func_tuple);
     
                     ReleaseSysCache(func_tuple);
     
                     /* Now, recursively simplify the args (which are a List) */
                     args = (List *)
                         expression_tree_mutator((Node *) args,
                                                 eval_const_expressions_mutator,
                                                 (void *) context);
                     /* ... and the filter expression, which isn't */
                     aggfilter = (Expr *)
                         eval_const_expressions_mutator((Node *) expr->aggfilter,
                                                        context);
     
                     /* And build the replacement WindowFunc node */
                     newexpr = makeNode(WindowFunc);
                     newexpr->winfnoid = expr->winfnoid;
                     newexpr->wintype = expr->wintype;
                     newexpr->wincollid = expr->wincollid;
                     newexpr->inputcollid = expr->inputcollid;
                     newexpr->args = args;
                     newexpr->aggfilter = aggfilter;
                     newexpr->winref = expr->winref;
                     newexpr->winstar = expr->winstar;
                     newexpr->winagg = expr->winagg;
                     newexpr->location = expr->location;
     
                     return (Node *) newexpr;
                 }
             case T_FuncExpr:
                 {
                     FuncExpr   *expr = (FuncExpr *) node;
                     List       *args = expr->args;
                     Expr       *simple;
                     FuncExpr   *newexpr;
     
                     /*
                      * Code for op/func reduction is pretty bulky, so split it out
                      * as a separate function.  Note: exprTypmod normally returns
                      * -1 for a FuncExpr, but not when the node is recognizably a
                      * length coercion; we want to preserve the typmod in the
                      * eventual Const if so.
                      */
                     simple = simplify_function(expr->funcid,
                                                expr->funcresulttype,
                                                exprTypmod(node),
                                                expr->funccollid,
                                                expr->inputcollid,
                                                &args,
                                                expr->funcvariadic,
                                                true,
                                                true,
                                                context);
                     if (simple)     /* successfully simplified it */
                         return (Node *) simple;
     
                     /*
                      * The expression cannot be simplified any further, so build
                      * and return a replacement FuncExpr node using the
                      * possibly-simplified arguments.  Note that we have also
                      * converted the argument list to positional notation.
                      */
                     newexpr = makeNode(FuncExpr);
                     newexpr->funcid = expr->funcid;
                     newexpr->funcresulttype = expr->funcresulttype;
                     newexpr->funcretset = expr->funcretset;
                     newexpr->funcvariadic = expr->funcvariadic;
                     newexpr->funcformat = expr->funcformat;
                     newexpr->funccollid = expr->funccollid;
                     newexpr->inputcollid = expr->inputcollid;
                     newexpr->args = args;
                     newexpr->location = expr->location;
                     return (Node *) newexpr;
                 }
             case T_OpExpr://操作(运算)表达式
                 {
                     OpExpr     *expr = (OpExpr *) node;
                     List       *args = expr->args;
                     Expr       *simple;
                     OpExpr     *newexpr;
     
                     /*
                      * Need to get OID of underlying function.  Okay to scribble
                      * on input to this extent.
                      */
                     set_opfuncid(expr);
     
                     /*
                      * Code for op/func reduction is pretty bulky, so split it out
                      * as a separate function.
                      */
                     simple = simplify_function(expr->opfuncid,
                                                expr->opresulttype, -1,
                                                expr->opcollid,
                                                expr->inputcollid,
                                                &args,
                                                false,
                                                true,
                                                true,
                                                context);
                     if (simple)     /* successfully simplified it */
                         return (Node *) simple;
     
                     /*
                      * If the operator is boolean equality or inequality, we know
                      * how to simplify cases involving one constant and one
                      * non-constant argument.
                      */
                     if (expr->opno == BooleanEqualOperator ||
                         expr->opno == BooleanNotEqualOperator)
                     {
                         simple = (Expr *) simplify_boolean_equality(expr->opno,
                                                                     args);
                         if (simple) /* successfully simplified it */
                             return (Node *) simple;
                     }
     
                     /*
                      * The expression cannot be simplified any further, so build
                      * and return a replacement OpExpr node using the
                      * possibly-simplified arguments.
                      */
                     newexpr = makeNode(OpExpr);
                     newexpr->opno = expr->opno;
                     newexpr->opfuncid = expr->opfuncid;
                     newexpr->opresulttype = expr->opresulttype;
                     newexpr->opretset = expr->opretset;
                     newexpr->opcollid = expr->opcollid;
                     newexpr->inputcollid = expr->inputcollid;
                     newexpr->args = args;
                     newexpr->location = expr->location;
                     return (Node *) newexpr;
                 }
             case T_DistinctExpr:
                 {
                     DistinctExpr *expr = (DistinctExpr *) node;
                     List       *args;
                     ListCell   *arg;
                     bool        has_null_input = false;
                     bool        all_null_input = true;
                     bool        has_nonconst_input = false;
                     Expr       *simple;
                     DistinctExpr *newexpr;
     
                     /*
                      * Reduce constants in the DistinctExpr's arguments.  We know
                      * args is either NIL or a List node, so we can call
                      * expression_tree_mutator directly rather than recursing to
                      * self.
                      */
                     args = (List *) expression_tree_mutator((Node *) expr->args,
                                                             eval_const_expressions_mutator,
                                                             (void *) context);
     
                     /*
                      * We must do our own check for NULLs because DistinctExpr has
                      * different results for NULL input than the underlying
                      * operator does.
                      */
                     foreach(arg, args)
                     {
                         if (IsA(lfirst(arg), Const))
                         {
                             has_null_input |= ((Const *) lfirst(arg))->constisnull;
                             all_null_input &= ((Const *) lfirst(arg))->constisnull;
                         }
                         else
                             has_nonconst_input = true;
                     }
     
                     /* all constants? then can optimize this out */
                     if (!has_nonconst_input)
                     {
                         /* all nulls? then not distinct */
                         if (all_null_input)
                             return makeBoolConst(false, false);
     
                         /* one null? then distinct */
                         if (has_null_input)
                             return makeBoolConst(true, false);
     
                         /* otherwise try to evaluate the '=' operator */
                         /* (NOT okay to try to inline it, though!) */
     
                         /*
                          * Need to get OID of underlying function.  Okay to
                          * scribble on input to this extent.
                          */
                         set_opfuncid((OpExpr *) expr);  /* rely on struct
                                                          * equivalence */
     
                         /*
                          * Code for op/func reduction is pretty bulky, so split it
                          * out as a separate function.
                          */
                         simple = simplify_function(expr->opfuncid,
                                                    expr->opresulttype, -1,
                                                    expr->opcollid,
                                                    expr->inputcollid,
                                                    &args,
                                                    false,
                                                    false,
                                                    false,
                                                    context);
                         if (simple) /* successfully simplified it */
                         {
                             /*
                              * Since the underlying operator is "=", must negate
                              * its result
                              */
                             Const      *csimple = castNode(Const, simple);
     
                             csimple->constvalue =
                                 BoolGetDatum(!DatumGetBool(csimple->constvalue));
                             return (Node *) csimple;
                         }
                     }
     
                     /*
                      * The expression cannot be simplified any further, so build
                      * and return a replacement DistinctExpr node using the
                      * possibly-simplified arguments.
                      */
                     newexpr = makeNode(DistinctExpr);
                     newexpr->opno = expr->opno;
                     newexpr->opfuncid = expr->opfuncid;
                     newexpr->opresulttype = expr->opresulttype;
                     newexpr->opretset = expr->opretset;
                     newexpr->opcollid = expr->opcollid;
                     newexpr->inputcollid = expr->inputcollid;
                     newexpr->args = args;
                     newexpr->location = expr->location;
                     return (Node *) newexpr;
                 }
             case T_ScalarArrayOpExpr:
                 {
                     ScalarArrayOpExpr *saop;
     
                     /* Copy the node and const-simplify its arguments */
                     saop = (ScalarArrayOpExpr *) ece_generic_processing(node);
     
                     /* Make sure we know underlying function */
                     set_sa_opfuncid(saop);
     
                     /*
                      * If all arguments are Consts, and it's a safe function, we
                      * can fold to a constant
                      */
                     if (ece_all_arguments_const(saop) &&
                         ece_function_is_safe(saop->opfuncid, context))
                         return ece_evaluate_expr(saop);
                     return (Node *) saop;
                 }
             case T_BoolExpr:
                 {
                     BoolExpr   *expr = (BoolExpr *) node;
     
                     switch (expr->boolop)
                     {
                         case OR_EXPR:
                             {
                                 List       *newargs;
                                 bool        haveNull = false;
                                 bool        forceTrue = false;
     
                                 newargs = simplify_or_arguments(expr->args,
                                                                 context,
                                                                 &haveNull,
                                                                 &forceTrue);
                                 if (forceTrue)
                                     return makeBoolConst(true, false);
                                 if (haveNull)
                                     newargs = lappend(newargs,
                                                       makeBoolConst(false, true));
                                 /* If all the inputs are FALSE, result is FALSE */
                                 if (newargs == NIL)
                                     return makeBoolConst(false, false);
     
                                 /*
                                  * If only one nonconst-or-NULL input, it's the
                                  * result
                                  */
                                 if (list_length(newargs) == 1)
                                     return (Node *) linitial(newargs);
                                 /* Else we still need an OR node */
                                 return (Node *) make_orclause(newargs);
                             }
                         case AND_EXPR:
                             {
                                 List       *newargs;
                                 bool        haveNull = false;
                                 bool        forceFalse = false;
     
                                 newargs = simplify_and_arguments(expr->args,
                                                                  context,
                                                                  &haveNull,
                                                                  &forceFalse);
                                 if (forceFalse)
                                     return makeBoolConst(false, false);
                                 if (haveNull)
                                     newargs = lappend(newargs,
                                                       makeBoolConst(false, true));
                                 /* If all the inputs are TRUE, result is TRUE */
                                 if (newargs == NIL)
                                     return makeBoolConst(true, false);
     
                                 /*
                                  * If only one nonconst-or-NULL input, it's the
                                  * result
                                  */
                                 if (list_length(newargs) == 1)
                                     return (Node *) linitial(newargs);
                                 /* Else we still need an AND node */
                                 return (Node *) make_andclause(newargs);
                             }
                         case NOT_EXPR:
                             {
                                 Node       *arg;
     
                                 Assert(list_length(expr->args) == 1);
                                 arg = eval_const_expressions_mutator(linitial(expr->args),
                                                                      context);
     
                                 /*
                                  * Use negate_clause() to see if we can simplify
                                  * away the NOT.
                                  */
                                 return negate_clause(arg);
                             }
                         default:
                             elog(ERROR, "unrecognized boolop: %d",
                                  (int) expr->boolop);
                             break;
                     }
                     break;
                 }
             case T_SubPlan:
             case T_AlternativeSubPlan:
     
                 /*
                  * Return a SubPlan unchanged --- too late to do anything with it.
                  *
                  * XXX should we ereport() here instead?  Probably this routine
                  * should never be invoked after SubPlan creation.
                  */
                 return node;
             case T_RelabelType:
                 {
                     /*
                      * If we can simplify the input to a constant, then we don't
                      * need the RelabelType node anymore: just change the type
                      * field of the Const node.  Otherwise, must copy the
                      * RelabelType node.
                      */
                     RelabelType *relabel = (RelabelType *) node;
                     Node       *arg;
     
                     arg = eval_const_expressions_mutator((Node *) relabel->arg,
                                                          context);
     
                     /*
                      * If we find stacked RelabelTypes (eg, from foo :: int ::
                      * oid) we can discard all but the top one.
                      */
                     while (arg && IsA(arg, RelabelType))
                         arg = (Node *) ((RelabelType *) arg)->arg;
     
                     if (arg && IsA(arg, Const))
                     {
                         Const      *con = (Const *) arg;
     
                         con->consttype = relabel->resulttype;
                         con->consttypmod = relabel->resulttypmod;
                         con->constcollid = relabel->resultcollid;
                         return (Node *) con;
                     }
                     else
                     {
                         RelabelType *newrelabel = makeNode(RelabelType);
     
                         newrelabel->arg = (Expr *) arg;
                         newrelabel->resulttype = relabel->resulttype;
                         newrelabel->resulttypmod = relabel->resulttypmod;
                         newrelabel->resultcollid = relabel->resultcollid;
                         newrelabel->relabelformat = relabel->relabelformat;
                         newrelabel->location = relabel->location;
                         return (Node *) newrelabel;
                     }
                 }
             case T_CoerceViaIO:
                 {
                     CoerceViaIO *expr = (CoerceViaIO *) node;
                     List       *args;
                     Oid         outfunc;
                     bool        outtypisvarlena;
                     Oid         infunc;
                     Oid         intypioparam;
                     Expr       *simple;
                     CoerceViaIO *newexpr;
     
                     /* Make a List so we can use simplify_function */
                     args = list_make1(expr->arg);
     
                     /*
                      * CoerceViaIO represents calling the source type's output
                      * function then the result type's input function.  So, try to
                      * simplify it as though it were a stack of two such function
                      * calls.  First we need to know what the functions are.
                      *
                      * Note that the coercion functions are assumed not to care
                      * about input collation, so we just pass InvalidOid for that.
                      */
                     getTypeOutputInfo(exprType((Node *) expr->arg),
                                       &outfunc, &outtypisvarlena);
                     getTypeInputInfo(expr->resulttype,
                                      &infunc, &intypioparam);
     
                     simple = simplify_function(outfunc,
                                                CSTRINGOID, -1,
                                                InvalidOid,
                                                InvalidOid,
                                                &args,
                                                false,
                                                true,
                                                true,
                                                context);
                     if (simple)     /* successfully simplified output fn */
                     {
                         /*
                          * Input functions may want 1 to 3 arguments.  We always
                          * supply all three, trusting that nothing downstream will
                          * complain.
                          */
                         args = list_make3(simple,
                                           makeConst(OIDOID,
                                                     -1,
                                                     InvalidOid,
                                                     sizeof(Oid),
                                                     ObjectIdGetDatum(intypioparam),
                                                     false,
                                                     true),
                                           makeConst(INT4OID,
                                                     -1,
                                                     InvalidOid,
                                                     sizeof(int32),
                                                     Int32GetDatum(-1),
                                                     false,
                                                     true));
     
                         simple = simplify_function(infunc,
                                                    expr->resulttype, -1,
                                                    expr->resultcollid,
                                                    InvalidOid,
                                                    &args,
                                                    false,
                                                    false,
                                                    true,
                                                    context);
                         if (simple) /* successfully simplified input fn */
                             return (Node *) simple;
                     }
     
                     /*
                      * The expression cannot be simplified any further, so build
                      * and return a replacement CoerceViaIO node using the
                      * possibly-simplified argument.
                      */
                     newexpr = makeNode(CoerceViaIO);
                     newexpr->arg = (Expr *) linitial(args);
                     newexpr->resulttype = expr->resulttype;
                     newexpr->resultcollid = expr->resultcollid;
                     newexpr->coerceformat = expr->coerceformat;
                     newexpr->location = expr->location;
                     return (Node *) newexpr;
                 }
             case T_ArrayCoerceExpr:
                 {
                     ArrayCoerceExpr *ac;
     
                     /* Copy the node and const-simplify its arguments */
                     ac = (ArrayCoerceExpr *) ece_generic_processing(node);
     
                     /*
                      * If constant argument and the per-element expression is
                      * immutable, we can simplify the whole thing to a constant.
                      * Exception: although contain_mutable_functions considers
                      * CoerceToDomain immutable for historical reasons, let's not
                      * do so here; this ensures coercion to an array-over-domain
                      * does not apply the domain's constraints until runtime.
                      */
                     if (ac->arg && IsA(ac->arg, Const) &&
                         ac->elemexpr && !IsA(ac->elemexpr, CoerceToDomain) &&
                         !contain_mutable_functions((Node *) ac->elemexpr))
                         return ece_evaluate_expr(ac);
                     return (Node *) ac;
                 }
             case T_CollateExpr:
                 {
                     /*
                      * If we can simplify the input to a constant, then we don't
                      * need the CollateExpr node at all: just change the
                      * constcollid field of the Const node.  Otherwise, replace
                      * the CollateExpr with a RelabelType. (We do that so as to
                      * improve uniformity of expression representation and thus
                      * simplify comparison of expressions.)
                      */
                     CollateExpr *collate = (CollateExpr *) node;
                     Node       *arg;
     
                     arg = eval_const_expressions_mutator((Node *) collate->arg,
                                                          context);
     
                     if (arg && IsA(arg, Const))
                     {
                         Const      *con = (Const *) arg;
     
                         con->constcollid = collate->collOid;
                         return (Node *) con;
                     }
                     else if (collate->collOid == exprCollation(arg))
                     {
                         /* Don't need a RelabelType either... */
                         return arg;
                     }
                     else
                     {
                         RelabelType *relabel = makeNode(RelabelType);
     
                         relabel->resulttype = exprType(arg);
                         relabel->resulttypmod = exprTypmod(arg);
                         relabel->resultcollid = collate->collOid;
                         relabel->relabelformat = COERCE_IMPLICIT_CAST;
                         relabel->location = collate->location;
     
                         /* Don't create stacked RelabelTypes */
                         while (arg && IsA(arg, RelabelType))
                             arg = (Node *) ((RelabelType *) arg)->arg;
                         relabel->arg = (Expr *) arg;
     
                         return (Node *) relabel;
                     }
                 }
             case T_CaseExpr:
                 {
                     /*----------                      * CASE expressions can be simplified if there are constant
                      * condition clauses:
                      *      FALSE (or NULL): drop the alternative
                      *      TRUE: drop all remaining alternatives
                      * If the first non-FALSE alternative is a constant TRUE,
                      * we can simplify the entire CASE to that alternative's
                      * expression.  If there are no non-FALSE alternatives,
                      * we simplify the entire CASE to the default result (ELSE).
                      *
                      * If we have a simple-form CASE with constant test
                      * expression, we substitute the constant value for contained
                      * CaseTestExpr placeholder nodes, so that we have the
                      * opportunity to reduce constant test conditions.  For
                      * example this allows
                      *      CASE 0 WHEN 0 THEN 1 ELSE 1/0 END
                      * to reduce to 1 rather than drawing a divide-by-0 error.
                      * Note that when the test expression is constant, we don't
                      * have to include it in the resulting CASE; for example
                      *      CASE 0 WHEN x THEN y ELSE z END
                      * is transformed by the parser to
                      *      CASE 0 WHEN CaseTestExpr = x THEN y ELSE z END
                      * which we can simplify to
                      *      CASE WHEN 0 = x THEN y ELSE z END
                      * It is not necessary for the executor to evaluate the "arg"
                      * expression when executing the CASE, since any contained
                      * CaseTestExprs that might have referred to it will have been
                      * replaced by the constant.
                      *----------                      */
                     CaseExpr   *caseexpr = (CaseExpr *) node;
                     CaseExpr   *newcase;
                     Node       *save_case_val;
                     Node       *newarg;
                     List       *newargs;
                     bool        const_true_cond;
                     Node       *defresult = NULL;
                     ListCell   *arg;
     
                     /* Simplify the test expression, if any */
                     newarg = eval_const_expressions_mutator((Node *) caseexpr->arg,
                                                             context);
     
                     /* Set up for contained CaseTestExpr nodes */
                     save_case_val = context->case_val;
                     if (newarg && IsA(newarg, Const))
                     {
                         context->case_val = newarg;
                         newarg = NULL;  /* not needed anymore, see above */
                     }
                     else
                         context->case_val = NULL;
     
                     /* Simplify the WHEN clauses */
                     newargs = NIL;
                     const_true_cond = false;
                     foreach(arg, caseexpr->args)
                     {
                         CaseWhen   *oldcasewhen = lfirst_node(CaseWhen, arg);
                         Node       *casecond;
                         Node       *caseresult;
     
                         /* Simplify this alternative's test condition */
                         casecond = eval_const_expressions_mutator((Node *) oldcasewhen->expr,
                                                                   context);
     
                         /*
                          * If the test condition is constant FALSE (or NULL), then
                          * drop this WHEN clause completely, without processing
                          * the result.
                          */
                         if (casecond && IsA(casecond, Const))
                         {
                             Const      *const_input = (Const *) casecond;
     
                             if (const_input->constisnull ||
                                 !DatumGetBool(const_input->constvalue))
                                 continue;   /* drop alternative with FALSE cond */
                             /* Else it's constant TRUE */
                             const_true_cond = true;
                         }
     
                         /* Simplify this alternative's result value */
                         caseresult = eval_const_expressions_mutator((Node *) oldcasewhen->result,
                                                                     context);
     
                         /* If non-constant test condition, emit a new WHEN node */
                         if (!const_true_cond)
                         {
                             CaseWhen   *newcasewhen = makeNode(CaseWhen);
     
                             newcasewhen->expr = (Expr *) casecond;
                             newcasewhen->result = (Expr *) caseresult;
                             newcasewhen->location = oldcasewhen->location;
                             newargs = lappend(newargs, newcasewhen);
                             continue;
                         }
     
                         /*
                          * Found a TRUE condition, so none of the remaining
                          * alternatives can be reached.  We treat the result as
                          * the default result.
                          */
                         defresult = caseresult;
                         break;
                     }
     
                     /* Simplify the default result, unless we replaced it above */
                     if (!const_true_cond)
                         defresult = eval_const_expressions_mutator((Node *) caseexpr->defresult,
                                                                    context);
     
                     context->case_val = save_case_val;
     
                     /*
                      * If no non-FALSE alternatives, CASE reduces to the default
                      * result
                      */
                     if (newargs == NIL)
                         return defresult;
                     /* Otherwise we need a new CASE node */
                     newcase = makeNode(CaseExpr);
                     newcase->casetype = caseexpr->casetype;
                     newcase->casecollid = caseexpr->casecollid;
                     newcase->arg = (Expr *) newarg;
                     newcase->args = newargs;
                     newcase->defresult = (Expr *) defresult;
                     newcase->location = caseexpr->location;
                     return (Node *) newcase;
                 }
             case T_CaseTestExpr:
                 {
                     /*
                      * If we know a constant test value for the current CASE
                      * construct, substitute it for the placeholder.  Else just
                      * return the placeholder as-is.
                      */
                     if (context->case_val)
                         return copyObject(context->case_val);
                     else
                         return copyObject(node);
                 }
             case T_ArrayRef:
             case T_ArrayExpr:
             case T_RowExpr:
                 {
                     /*
                      * Generic handling for node types whose own processing is
                      * known to be immutable, and for which we need no smarts
                      * beyond "simplify if all inputs are constants".
                      */
     
                     /* Copy the node and const-simplify its arguments */
                     node = ece_generic_processing(node);
                     /* If all arguments are Consts, we can fold to a constant */
                     if (ece_all_arguments_const(node))
                         return ece_evaluate_expr(node);
                     return node;
                 }
             case T_CoalesceExpr:
                 {
                     CoalesceExpr *coalesceexpr = (CoalesceExpr *) node;
                     CoalesceExpr *newcoalesce;
                     List       *newargs;
                     ListCell   *arg;
     
                     newargs = NIL;
                     foreach(arg, coalesceexpr->args)
                     {
                         Node       *e;
     
                         e = eval_const_expressions_mutator((Node *) lfirst(arg),
                                                            context);
     
                         /*
                          * We can remove null constants from the list. For a
                          * non-null constant, if it has not been preceded by any
                          * other non-null-constant expressions then it is the
                          * result. Otherwise, it's the next argument, but we can
                          * drop following arguments since they will never be
                          * reached.
                          */
                         if (IsA(e, Const))
                         {
                             if (((Const *) e)->constisnull)
                                 continue;   /* drop null constant */
                             if (newargs == NIL)
                                 return e;   /* first expr */
                             newargs = lappend(newargs, e);
                             break;
                         }
                         newargs = lappend(newargs, e);
                     }
     
                     /*
                      * If all the arguments were constant null, the result is just
                      * null
                      */
                     if (newargs == NIL)
                         return (Node *) makeNullConst(coalesceexpr->coalescetype,
                                                       -1,
                                                       coalesceexpr->coalescecollid);
     
                     newcoalesce = makeNode(CoalesceExpr);
                     newcoalesce->coalescetype = coalesceexpr->coalescetype;
                     newcoalesce->coalescecollid = coalesceexpr->coalescecollid;
                     newcoalesce->args = newargs;
                     newcoalesce->location = coalesceexpr->location;
                     return (Node *) newcoalesce;
                 }
             case T_SQLValueFunction:
                 {
                     /*
                      * All variants of SQLValueFunction are stable, so if we are
                      * estimating the expression's value, we should evaluate the
                      * current function value.  Otherwise just copy.
                      */
                     SQLValueFunction *svf = (SQLValueFunction *) node;
     
                     if (context->estimate)
                         return (Node *) evaluate_expr((Expr *) svf,
                                                       svf->type,
                                                       svf->typmod,
                                                       InvalidOid);
                     else
                         return copyObject((Node *) svf);
                 }
             case T_FieldSelect:
                 {
                     /*
                      * We can optimize field selection from a whole-row Var into a
                      * simple Var.  (This case won't be generated directly by the
                      * parser, because ParseComplexProjection short-circuits it.
                      * But it can arise while simplifying functions.)  Also, we
                      * can optimize field selection from a RowExpr construct, or
                      * of course from a constant.
                      *
                      * However, replacing a whole-row Var in this way has a
                      * pitfall: if we've already built the rel targetlist for the
                      * source relation, then the whole-row Var is scheduled to be
                      * produced by the relation scan, but the simple Var probably
                      * isn't, which will lead to a failure in setrefs.c.  This is
                      * not a problem when handling simple single-level queries, in
                      * which expression simplification always happens first.  It
                      * is a risk for lateral references from subqueries, though.
                      * To avoid such failures, don't optimize uplevel references.
                      *
                      * We must also check that the declared type of the field is
                      * still the same as when the FieldSelect was created --- this
                      * can change if someone did ALTER COLUMN TYPE on the rowtype.
                      * If it isn't, we skip the optimization; the case will
                      * probably fail at runtime, but that's not our problem here.
                      */
                     FieldSelect *fselect = (FieldSelect *) node;
                     FieldSelect *newfselect;
                     Node       *arg;
     
                     arg = eval_const_expressions_mutator((Node *) fselect->arg,
                                                          context);
                     if (arg && IsA(arg, Var) &&
                         ((Var *) arg)->varattno == InvalidAttrNumber &&
                         ((Var *) arg)->varlevelsup == 0)
                     {
                         if (rowtype_field_matches(((Var *) arg)->vartype,
                                                   fselect->fieldnum,
                                                   fselect->resulttype,
                                                   fselect->resulttypmod,
                                                   fselect->resultcollid))
                             return (Node *) makeVar(((Var *) arg)->varno,
                                                     fselect->fieldnum,
                                                     fselect->resulttype,
                                                     fselect->resulttypmod,
                                                     fselect->resultcollid,
                                                     ((Var *) arg)->varlevelsup);
                     }
                     if (arg && IsA(arg, RowExpr))
                     {
                         RowExpr    *rowexpr = (RowExpr *) arg;
     
                         if (fselect->fieldnum > 0 &&
                             fselect->fieldnum <= list_length(rowexpr->args))
                         {
                             Node       *fld = (Node *) list_nth(rowexpr->args,
                                                                 fselect->fieldnum - 1);
     
                             if (rowtype_field_matches(rowexpr->row_typeid,
                                                       fselect->fieldnum,
                                                       fselect->resulttype,
                                                       fselect->resulttypmod,
                                                       fselect->resultcollid) &&
                                 fselect->resulttype == exprType(fld) &&
                                 fselect->resulttypmod == exprTypmod(fld) &&
                                 fselect->resultcollid == exprCollation(fld))
                                 return fld;
                         }
                     }
                     newfselect = makeNode(FieldSelect);
                     newfselect->arg = (Expr *) arg;
                     newfselect->fieldnum = fselect->fieldnum;
                     newfselect->resulttype = fselect->resulttype;
                     newfselect->resulttypmod = fselect->resulttypmod;
                     newfselect->resultcollid = fselect->resultcollid;
                     if (arg && IsA(arg, Const))
                     {
                         Const      *con = (Const *) arg;
     
                         if (rowtype_field_matches(con->consttype,
                                                   newfselect->fieldnum,
                                                   newfselect->resulttype,
                                                   newfselect->resulttypmod,
                                                   newfselect->resultcollid))
                             return ece_evaluate_expr(newfselect);
                     }
                     return (Node *) newfselect;
                 }
             case T_NullTest:
                 {
                     NullTest   *ntest = (NullTest *) node;
                     NullTest   *newntest;
                     Node       *arg;
     
                     arg = eval_const_expressions_mutator((Node *) ntest->arg,
                                                          context);
                     if (ntest->argisrow && arg && IsA(arg, RowExpr))
                     {
                         /*
                          * We break ROW(...) IS [NOT] NULL into separate tests on
                          * its component fields.  This form is usually more
                          * efficient to evaluate, as well as being more amenable
                          * to optimization.
                          */
                         RowExpr    *rarg = (RowExpr *) arg;
                         List       *newargs = NIL;
                         ListCell   *l;
     
                         foreach(l, rarg->args)
                         {
                             Node       *relem = (Node *) lfirst(l);
     
                             /*
                              * A constant field refutes the whole NullTest if it's
                              * of the wrong nullness; else we can discard it.
                              */
                             if (relem && IsA(relem, Const))
                             {
                                 Const      *carg = (Const *) relem;
     
                                 if (carg->constisnull ?
                                     (ntest->nulltesttype == IS_NOT_NULL) :
                                     (ntest->nulltesttype == IS_NULL))
                                     return makeBoolConst(false, false);
                                 continue;
                             }
     
                             /*
                              * Else, make a scalar (argisrow == false) NullTest
                              * for this field.  Scalar semantics are required
                              * because IS [NOT] NULL doesn't recurse; see comments
                              * in ExecEvalRowNullInt().
                              */
                             newntest = makeNode(NullTest);
                             newntest->arg = (Expr *) relem;
                             newntest->nulltesttype = ntest->nulltesttype;
                             newntest->argisrow = false;
                             newntest->location = ntest->location;
                             newargs = lappend(newargs, newntest);
                         }
                         /* If all the inputs were constants, result is TRUE */
                         if (newargs == NIL)
                             return makeBoolConst(true, false);
                         /* If only one nonconst input, it's the result */
                         if (list_length(newargs) == 1)
                             return (Node *) linitial(newargs);
                         /* Else we need an AND node */
                         return (Node *) make_andclause(newargs);
                     }
                     if (!ntest->argisrow && arg && IsA(arg, Const))
                     {
                         Const      *carg = (Const *) arg;
                         bool        result;
     
                         switch (ntest->nulltesttype)
                         {
                             case IS_NULL:
                                 result = carg->constisnull;
                                 break;
                             case IS_NOT_NULL:
                                 result = !carg->constisnull;
                                 break;
                             default:
                                 elog(ERROR, "unrecognized nulltesttype: %d",
                                      (int) ntest->nulltesttype);
                                 result = false; /* keep compiler quiet */
                                 break;
                         }
     
                         return makeBoolConst(result, false);
                     }
     
                     newntest = makeNode(NullTest);
                     newntest->arg = (Expr *) arg;
                     newntest->nulltesttype = ntest->nulltesttype;
                     newntest->argisrow = ntest->argisrow;
                     newntest->location = ntest->location;
                     return (Node *) newntest;
                 }
             case T_BooleanTest:
                 {
                     /*
                      * This case could be folded into the generic handling used
                      * for ArrayRef etc.  But because the simplification logic is
                      * so trivial, applying evaluate_expr() to perform it would be
                      * a heavy overhead.  BooleanTest is probably common enough to
                      * justify keeping this bespoke implementation.
                      */
                     BooleanTest *btest = (BooleanTest *) node;
                     BooleanTest *newbtest;
                     Node       *arg;
     
                     arg = eval_const_expressions_mutator((Node *) btest->arg,
                                                          context);
                     if (arg && IsA(arg, Const))
                     {
                         Const      *carg = (Const *) arg;
                         bool        result;
     
                         switch (btest->booltesttype)
                         {
                             case IS_TRUE:
                                 result = (!carg->constisnull &&
                                           DatumGetBool(carg->constvalue));
                                 break;
                             case IS_NOT_TRUE:
                                 result = (carg->constisnull ||
                                           !DatumGetBool(carg->constvalue));
                                 break;
                             case IS_FALSE:
                                 result = (!carg->constisnull &&
                                           !DatumGetBool(carg->constvalue));
                                 break;
                             case IS_NOT_FALSE:
                                 result = (carg->constisnull ||
                                           DatumGetBool(carg->constvalue));
                                 break;
                             case IS_UNKNOWN:
                                 result = carg->constisnull;
                                 break;
                             case IS_NOT_UNKNOWN:
                                 result = !carg->constisnull;
                                 break;
                             default:
                                 elog(ERROR, "unrecognized booltesttype: %d",
                                      (int) btest->booltesttype);
                                 result = false; /* keep compiler quiet */
                                 break;
                         }
     
                         return makeBoolConst(result, false);
                     }
     
                     newbtest = makeNode(BooleanTest);
                     newbtest->arg = (Expr *) arg;
                     newbtest->booltesttype = btest->booltesttype;
                     newbtest->location = btest->location;
                     return (Node *) newbtest;
                 }
             case T_PlaceHolderVar:
     
                 /*
                  * In estimation mode, just strip the PlaceHolderVar node
                  * altogether; this amounts to estimating that the contained value
                  * won't be forced to null by an outer join.  In regular mode we
                  * just use the default behavior (ie, simplify the expression but
                  * leave the PlaceHolderVar node intact).
                  */
                 if (context->estimate)
                 {
                     PlaceHolderVar *phv = (PlaceHolderVar *) node;
     
                     return eval_const_expressions_mutator((Node *) phv->phexpr,
                                                           context);
                 }
                 break;
             default:
                 break;
         }
     
         /*
          * For any node type not handled above, copy the node unchanged but
          * const-simplify its subexpressions.  This is the correct thing for node
          * types whose behavior might change between planning and execution, such
          * as CoerceToDomain.  It's also a safe default for new node types not
          * known to this routine.
          */
         return ece_generic_processing(node);
     }
    

**simplify_function**

    
    
     /*
      * Subroutine for eval_const_expressions: try to simplify a function call
      * (which might originally have been an operator; we don't care)
      *
      * Inputs are the function OID, actual result type OID (which is needed for
      * polymorphic functions), result typmod, result collation, the input
      * collation to use for the function, the original argument list (not
      * const-simplified yet, unless process_args is false), and some flags;
      * also the context data for eval_const_expressions.
      *
      * Returns a simplified expression if successful, or NULL if cannot
      * simplify the function call.
      *
      * This function is also responsible for converting named-notation argument
      * lists into positional notation and/or adding any needed default argument
      * expressions; which is a bit grotty, but it avoids extra fetches of the
      * function's pg_proc tuple.  For this reason, the args list is
      * pass-by-reference.  Conversion and const-simplification of the args list
      * will be done even if simplification of the function call itself is not
      * possible.
      */
     static Expr *
     simplify_function(Oid funcid, Oid result_type, int32 result_typmod,
                       Oid result_collid, Oid input_collid, List **args_p,
                       bool funcvariadic, bool process_args, bool allow_non_const,
                       eval_const_expressions_context *context)
     {
         List       *args = *args_p;
         HeapTuple   func_tuple;
         Form_pg_proc func_form;
         Expr       *newexpr;
     
         /*
          * We have three strategies for simplification: execute the function to
          * deliver a constant result, use a transform function to generate a
          * substitute node tree, or expand in-line the body of the function
          * definition (which only works for simple SQL-language functions, but
          * that is a common case).  Each case needs access to the function's
          * pg_proc tuple, so fetch it just once.
          *
          * Note: the allow_non_const flag suppresses both the second and third
          * strategies; so if !allow_non_const, simplify_function can only return a
          * Const or NULL.  Argument-list rewriting happens anyway, though.
          */
         //查询proc(视为Tuple) 
         func_tuple = SearchSysCache1(PROCOID, ObjectIdGetDatum(funcid));
         if (!HeapTupleIsValid(func_tuple))
             elog(ERROR, "cache lookup failed for function %u", funcid);
         //从Tuple中分解得到函数体
         func_form = (Form_pg_proc) GETSTRUCT(func_tuple);
     
         /*
          * Process the function arguments, unless the caller did it already.
          *
          * Here we must deal with named or defaulted arguments, and then
          * recursively apply eval_const_expressions to the whole argument list.
          */
         if (process_args)//参数不为空
         {
             args = expand_function_arguments(args, result_type, func_tuple);//展开参数
             args = (List *) expression_tree_mutator((Node *) args,
                                                     eval_const_expressions_mutator,
                                                     (void *) context);//递归处理
             /* Argument processing done, give it back to the caller */
             *args_p = args;//重新赋值
         }
     
         /* Now attempt simplification of the function call proper. */
     
         newexpr = evaluate_function(funcid, result_type, result_typmod,
                                     result_collid, input_collid,
                                     args, funcvariadic,
                                     func_tuple, context);//对函数进行预求解
         //求解成功并且允许非Const值并且(func_form->protransform是合法的Oid
         if (!newexpr && allow_non_const && OidIsValid(func_form->protransform))
         {
             /*
              * Build a dummy FuncExpr node containing the simplified arg list.  We
              * use this approach to present a uniform interface to the transform
              * function regardless of how the function is actually being invoked.
              */
             FuncExpr    fexpr;
     
             fexpr.xpr.type = T_FuncExpr;
             fexpr.funcid = funcid;
             fexpr.funcresulttype = result_type;
             fexpr.funcretset = func_form->proretset;
             fexpr.funcvariadic = funcvariadic;
             fexpr.funcformat = COERCE_EXPLICIT_CALL;
             fexpr.funccollid = result_collid;
             fexpr.inputcollid = input_collid;
             fexpr.args = args;
             fexpr.location = -1;
     
             newexpr = (Expr *)
                 DatumGetPointer(OidFunctionCall1(func_form->protransform,
                                                  PointerGetDatum(&fexpr)));
         }
     
         if (!newexpr && allow_non_const)
             newexpr = inline_function(funcid, result_type, result_collid,
                                       input_collid, args, funcvariadic,
                                       func_tuple, context);
     
         ReleaseSysCache(func_tuple);
     
         return newexpr;
     }
     
    

* * *
    
    
     /*
      * evaluate_expr: pre-evaluate a constant expression
      *
      * We use the executor's routine ExecEvalExpr() to avoid duplication of
      * code and ensure we get the same result as the executor would get.
      */
     static Expr *
     evaluate_expr(Expr *expr, Oid result_type, int32 result_typmod,
                   Oid result_collation)
     {
         EState     *estate;
         ExprState  *exprstate;
         MemoryContext oldcontext;
         Datum       const_val;
         bool        const_is_null;
         int16       resultTypLen;
         bool        resultTypByVal;
     
         /*
          * To use the executor, we need an EState.
          */
         estate = CreateExecutorState();
     
         /* We can use the estate's working context to avoid memory leaks. */
         oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
     
         /* Make sure any opfuncids are filled in. */
         fix_opfuncids((Node *) expr);
     
         /*
          * Prepare expr for execution.  (Note: we can't use ExecPrepareExpr
          * because it'd result in recursively invoking eval_const_expressions.)
          */
         //初始化表达式,为执行作准备
         //把函数放在exprstate->evalfunc中
         exprstate = ExecInitExpr(expr, NULL);
     
         /*
          * And evaluate it.
          *
          * It is OK to use a default econtext because none of the ExecEvalExpr()
          * code used in this situation will use econtext.  That might seem
          * fortuitous, but it's not so unreasonable --- a constant expression does
          * not depend on context, by definition, n'est ce pas?
          */
         const_val = ExecEvalExprSwitchContext(exprstate,
                                               GetPerTupleExprContext(estate),
                                               &const_is_null);//执行表达式求解
     
         /* Get info needed about result datatype */
         get_typlenbyval(result_type, &resultTypLen, &resultTypByVal);
     
         /* Get back to outer memory context */
         MemoryContextSwitchTo(oldcontext);
     
         /*
          * Must copy result out of sub-context used by expression eval.
          *
          * Also, if it's varlena, forcibly detoast it.  This protects us against
          * storing TOAST pointers into plans that might outlive the referenced
          * data.  (makeConst would handle detoasting anyway, but it's worth a few
          * extra lines here so that we can do the copy and detoast in one step.)
          */
         if (!const_is_null)
         {
             if (resultTypLen == -1)
                 const_val = PointerGetDatum(PG_DETOAST_DATUM_COPY(const_val));
             else
                 const_val = datumCopy(const_val, resultTypByVal, resultTypLen);
         }
     
         /* Release all the junk we just created */
         FreeExecutorState(estate);
     
         /*
          * Make the constant result node.
          */
         return (Expr *) makeConst(result_type, result_typmod, result_collation,
                                   resultTypLen,
                                   const_val, const_is_null,
                                   resultTypByVal);
     }
    
     /*
      * ExecEvalExprSwitchContext
      *
      * Same as ExecEvalExpr, but get into the right allocation context explicitly.
      */
     #ifndef FRONTEND
     static inline Datum
     ExecEvalExprSwitchContext(ExprState *state,
                               ExprContext *econtext,
                               bool *isNull)
     {
         Datum       retDatum;
         MemoryContext oldContext;
     
         oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);
         retDatum = state->evalfunc(state, econtext, isNull);
         MemoryContextSwitchTo(oldContext);
         return retDatum;
     }
     #endif
    
    

### 三、跟踪分析

测试脚本,表达式位于targetList中:

    
    
    select max(a.dwbh::int+(1+2)) 
    from t_dwxx a;
    

gdb跟踪:

    
    
    Breakpoint 1, preprocess_expression (root=0x133eca8, expr=0x13441a8, kind=1) at planner.c:1007
    1007        if (expr == NULL)
    ...
    (gdb) p *(TargetEntry *)((List *)expr)->head->data.ptr_value
    $6 = {xpr = {type = T_TargetEntry}, expr = 0x1343fb8, resno = 1, resname = 0x124bb38 "max", ressortgroupref = 0, 
      resorigtbl = 0, resorigcol = 0, resjunk = false}
    ...
    #OpExpr,参数args链表,第1个参数是1,第2个参数是2
    (gdb) p *(Const *)$opexpr->args->head->data.ptr_value
    $25 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 1, 
      constisnull = false, constbyval = true, location = 24}
    (gdb) p *(Const *)$opexpr->args->tail->data.ptr_value
    $26 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 2, 
      constisnull = false, constbyval = true, location = 26}
    (gdb) 
    #调整断点
    (gdb) info break
    Num     Type           Disp Enb Address            What
    1       breakpoint     keep y   0x000000000076ac6f in preprocess_expression at planner.c:1007
        breakpoint already hit 8 times
    (gdb) del 1
    (gdb) b clauses.c:2713
    Breakpoint 2 at 0x78952a: file clauses.c, line 2713.
    (gdb) c
    Continuing.
    
    Breakpoint 2, eval_const_expressions_mutator (node=0x124cbf0, context=0x7ffebc48f630) at clauses.c:2716
    2716                    set_opfuncid(expr);
    #这个表达式是a.dwbh::int+(1+2)
    (gdb) p *((OpExpr *)node)->args
    $29 = {type = T_List, length = 2, head = 0x124cbd0, tail = 0x124cb80}
    (gdb) p *(Node *)((OpExpr *)node)->args->head->data.ptr_value
    $30 = {type = T_CoerceViaIO}
    (gdb) c
    Continuing.
    
    Breakpoint 2, eval_const_expressions_mutator (node=0x124cb30, context=0x7ffebc48f630) at clauses.c:2716
    2716                    set_opfuncid(expr);
    #这个表达式是1+2,对此表达式进行求解
    (gdb) p *(Node *)((OpExpr *)node)->args->head->data.ptr_value
    $34 = {type = T_Const}
    (gdb) p *(Const *)((OpExpr *)node)->args->head->data.ptr_value
    $35 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 1, 
      constisnull = false, constbyval = true, location = 24}
    #进入simplify_function
    (gdb) step
    simplify_function (funcid=177, result_type=23, result_typmod=-1, result_collid=0, input_collid=0, args_p=0x7ffebc48c838, 
        funcvariadic=false, process_args=true, allow_non_const=true, context=0x7ffebc48f630) at clauses.c:4022
    4022        List       *args = *args_p; 
    ...
    #函数是int4pl
    (gdb) p *func_form
    $38 = {proname = {data = "int4pl", '\000' <repeats 57 times>}, pronamespace = 11, proowner = 10, prolang = 12, procost = 1, 
      prorows = 0, provariadic = 0, protransform = 0, prokind = 102 'f', prosecdef = false, proleakproof = false, 
      proisstrict = true, proretset = false, provolatile = 105 'i', proparallel = 115 's', pronargs = 2, pronargdefaults = 0, 
      prorettype = 23, proargtypes = {vl_len_ = 128, ndim = 1, dataoffset = 0, elemtype = 26, dim1 = 2, lbound1 = 0, 
        values = 0x7fd820a599a4}}
    ...
    #求解,得到结果为3
    (gdb) p const_val
    $48 = 3
    (gdb) 
    evaluate_function (funcid=177, result_type=23, result_typmod=-1, result_collid=0, input_collid=0, args=0x13135c8, 
        funcvariadic=false, func_tuple=0x7fd820a598d8, context=0x7ffebc48f630) at clauses.c:4424
    4424    }
    (gdb) 
    simplify_function (funcid=177, result_type=23, result_typmod=-1, result_collid=0, input_collid=0, args_p=0x7ffebc48c838, 
        funcvariadic=false, process_args=true, allow_non_const=true, context=0x7ffebc48f630) at clauses.c:4067
    4067        if (!newexpr && allow_non_const && OidIsValid(func_form->protransform))
    (gdb) p *newexpr
    $50 = {type = T_Const}
    (gdb) p *(Const *)newexpr
    $51 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 3, 
      constisnull = false, constbyval = true, location = -1}
    ...
    #DONE!
    #把1+2的T_OpExpr变换为T_Const
    

### 四、小结

1、简化过程：通过eval_const_expressions_mutator函数遍历相关节点，根据函数信息读取pg_proc中的函数并通过这些函数对表达式逐个处理；  
2、表达式求解：通过调用evaluate_expr进而调用内置函数进行求解。

### 五、参考资料

[planner.c](https://doxygen.postgresql.org/planner_8c_source.html#l00999)  
[clauses.c](https://doxygen.postgresql.org/clauses_8c_source.html#l02460)

