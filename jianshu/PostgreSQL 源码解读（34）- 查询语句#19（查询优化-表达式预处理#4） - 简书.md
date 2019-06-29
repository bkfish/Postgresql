本节简单介绍了PG查询优化表达式预处理中的生成子链接执行计划、使用Param替换上层Vars以及转换表达式为隐式AND格式（implicit-AND
format）。

### 一、主函数

主函数preprocess_expression先前章节也介绍过，在此函数中调用了生成子链接执行计划、使用Param替换上层Vars以及转换表达式为隐式AND格式（implicit-AND format）等相关子函数。  
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
         //...
     
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
         if (kind == EXPRKIND_QUAL)//转换为隐式AND格式
             expr = (Node *) make_ands_implicit((Expr *) expr);
     
         return expr;
     }
    

### 二、生成子链接执行计划

先前的章节已介绍了上拉子链接的相关处理过程,对于不能上拉的子链接,PG会生成子执行计划.对于会生成常量的子链接,则会把生成的常量记录在Param中,在需要的时候由父查询使用.  
例1:以下的子链接,PG会生成子计划,并把子链接的结果物化(Materialize)提升整体性能.

    
    
    testdb=# explain verbose select * from t_dwxx where dwbh > all (select b.dwbh from t_grxx b);
                                       QUERY PLAN                                    
    ---------------------------------------------------------------------------------     Seq Scan on public.t_dwxx  (cost=0.00..1498.00 rows=80 width=474)
       Output: t_dwxx.dwmc, t_dwxx.dwbh, t_dwxx.dwdz
       Filter: (SubPlan 1)
       SubPlan 1
         ->  Materialize  (cost=0.00..17.35 rows=490 width=38)
               Output: b.dwbh
               ->  Seq Scan on public.t_grxx b  (cost=0.00..14.90 rows=490 width=38)
                     Output: b.dwbh
    (8 rows)
    

例2:以下的子链接,PG会把生成的常量记录在Param中(注意生成的参数:$0)

    
    
    testdb=# explain verbose select * from t_dwxx a where exists (select  max(b.dwbh) from t_grxx b);
                                       QUERY PLAN                                    
    ---------------------------------------------------------------------------------     Result  (cost=16.14..27.73 rows=160 width=474)
       Output: a.dwmc, a.dwbh, a.dwdz
       One-Time Filter: $0
       InitPlan 1 (returns $0)
         ->  Aggregate  (cost=16.12..16.14 rows=1 width=32)
               Output: max((b.dwbh)::text)
               ->  Seq Scan on public.t_grxx b  (cost=0.00..14.90 rows=490 width=38)
                     Output: b.dwbh, b.grbh, b.xm, b.nl
       ->  Seq Scan on public.t_dwxx a  (cost=16.14..27.73 rows=160 width=474)
             Output: a.dwmc, a.dwbh, a.dwdz
    (10 rows)
    

源代码如下:  
**SS_process_sublinks**

    
    
    /*
      * Expand SubLinks to SubPlans in the given expression.
      *
      * The isQual argument tells whether or not this expression is a WHERE/HAVING
      * qualifier expression.  If it is, any sublinks appearing at top level need
      * not distinguish FALSE from UNKNOWN return values.
      */
     Node *
     SS_process_sublinks(PlannerInfo *root, Node *expr, bool isQual)
     {
         process_sublinks_context context;
     
         context.root = root;
         context.isTopQual = isQual;
         return process_sublinks_mutator(expr, &context);//调用XX_mutator函数遍历并处理
     }
     
     static Node *
     process_sublinks_mutator(Node *node, process_sublinks_context *context)
     {
         process_sublinks_context locContext;
     
         locContext.root = context->root;
     
         if (node == NULL)
             return NULL;
         if (IsA(node, SubLink))//子链接
         {
             SubLink    *sublink = (SubLink *) node;
             Node       *testexpr;
     
             /*
              * First, recursively process the lefthand-side expressions, if any.
              * They're not top-level anymore.
              */
             locContext.isTopQual = false;
             testexpr = process_sublinks_mutator(sublink->testexpr, &locContext);
     
             /*
              * Now build the SubPlan node and make the expr to return.
              */
             return make_subplan(context->root,
                                 (Query *) sublink->subselect,
                                 sublink->subLinkType,
                                 sublink->subLinkId,
                                 testexpr,
                                 context->isTopQual);//生成子执行计划,与整体的执行计划类似
         }
     
         /*
          * Don't recurse into the arguments of an outer PHV or aggregate here. Any
          * SubLinks in the arguments have to be dealt with at the outer query
          * level; they'll be handled when build_subplan collects the PHV or Aggref
          * into the arguments to be passed down to the current subplan.
          */
         if (IsA(node, PlaceHolderVar))
         {
             if (((PlaceHolderVar *) node)->phlevelsup > 0)
                 return node;
         }
         else if (IsA(node, Aggref))
         {
             if (((Aggref *) node)->agglevelsup > 0)
                 return node;
         }
     
         /*
          * We should never see a SubPlan expression in the input (since this is
          * the very routine that creates 'em to begin with).  We shouldn't find
          * ourselves invoked directly on a Query, either.
          */
         Assert(!IsA(node, SubPlan));
         Assert(!IsA(node, AlternativeSubPlan));
         Assert(!IsA(node, Query));
     
         /*
          * Because make_subplan() could return an AND or OR clause, we have to
          * take steps to preserve AND/OR flatness of a qual.  We assume the input
          * has been AND/OR flattened and so we need no recursion here.
          *
          * (Due to the coding here, we will not get called on the List subnodes of
          * an AND; and the input is *not* yet in implicit-AND format.  So no check
          * is needed for a bare List.)
          *
          * Anywhere within the top-level AND/OR clause structure, we can tell
          * make_subplan() that NULL and FALSE are interchangeable.  So isTopQual
          * propagates down in both cases.  (Note that this is unlike the meaning
          * of "top level qual" used in most other places in Postgres.)
          */
         if (and_clause(node))//AND语句
         {
             List       *newargs = NIL;
             ListCell   *l;
     
             /* Still at qual top-level */
             locContext.isTopQual = context->isTopQual;
     
             foreach(l, ((BoolExpr *) node)->args)
             {
                 Node       *newarg;
     
                 newarg = process_sublinks_mutator(lfirst(l), &locContext);
                 if (and_clause(newarg))
                     newargs = list_concat(newargs, ((BoolExpr *) newarg)->args);
                 else
                     newargs = lappend(newargs, newarg);
             }
             return (Node *) make_andclause(newargs);
         }
     
         if (or_clause(node))//OR语句
         {
             List       *newargs = NIL;
             ListCell   *l;
     
             /* Still at qual top-level */
             locContext.isTopQual = context->isTopQual;
     
             foreach(l, ((BoolExpr *) node)->args)
             {
                 Node       *newarg;
     
                 newarg = process_sublinks_mutator(lfirst(l), &locContext);
                 if (or_clause(newarg))
                     newargs = list_concat(newargs, ((BoolExpr *) newarg)->args);
                 else
                     newargs = lappend(newargs, newarg);
             }
             return (Node *) make_orclause(newargs);
         }
     
         /*
          * If we recurse down through anything other than an AND or OR node, we
          * are definitely not at top qual level anymore.
          */
         locContext.isTopQual = false;
     
         return expression_tree_mutator(node,
                                        process_sublinks_mutator,
                                        (void *) &locContext);
     }
    

### 三、使用Param替换上层变量

SQL例子参考"二、生成子链接执行计划"中的例2,这也是使用Param替代Var的一个例子.  
源代码如下:

    
    
     /*
      * Replace correlation vars (uplevel vars) with Params.
      *
      * Uplevel PlaceHolderVars and aggregates are replaced, too.
      *
      * Note: it is critical that this runs immediately after SS_process_sublinks.
      * Since we do not recurse into the arguments of uplevel PHVs and aggregates,
      * they will get copied to the appropriate subplan args list in the parent
      * query with uplevel vars not replaced by Params, but only adjusted in level
      * (see replace_outer_placeholdervar and replace_outer_agg).  That's exactly
      * what we want for the vars of the parent level --- but if a PHV's or
      * aggregate's argument contains any further-up variables, they have to be
      * replaced with Params in their turn. That will happen when the parent level
      * runs SS_replace_correlation_vars.  Therefore it must do so after expanding
      * its sublinks to subplans.  And we don't want any steps in between, else
      * those steps would never get applied to the argument expressions, either in
      * the parent or the child level.
      *
      * Another fairly tricky thing going on here is the handling of SubLinks in
      * the arguments of uplevel PHVs/aggregates.  Those are not touched inside the
      * intermediate query level, either.  Instead, SS_process_sublinks recurses on
      * them after copying the PHV or Aggref expression into the parent plan level
      * (this is actually taken care of in build_subplan).
      */
     Node *
     SS_replace_correlation_vars(PlannerInfo *root, Node *expr)
     {
         /* No setup needed for tree walk, so away we go */
         //调用XX_mutator遍历处理
         return replace_correlation_vars_mutator(expr, root);
     }
     
     static Node *
     replace_correlation_vars_mutator(Node *node, PlannerInfo *root)
     {
         if (node == NULL)
             return NULL;
         if (IsA(node, Var))//Var
         {
             if (((Var *) node)->varlevelsup > 0)
                 return (Node *) replace_outer_var(root, (Var *) node);//使用Param替换
         }
         if (IsA(node, PlaceHolderVar))
         {
             if (((PlaceHolderVar *) node)->phlevelsup > 0)
                 return (Node *) replace_outer_placeholdervar(root,
                                                              (PlaceHolderVar *) node);
         }
         if (IsA(node, Aggref))
         {
             if (((Aggref *) node)->agglevelsup > 0)
                 return (Node *) replace_outer_agg(root, (Aggref *) node);
         }
         if (IsA(node, GroupingFunc))
         {
             if (((GroupingFunc *) node)->agglevelsup > 0)
                 return (Node *) replace_outer_grouping(root, (GroupingFunc *) node);
         }
         return expression_tree_mutator(node,
                                        replace_correlation_vars_mutator,
                                        (void *) root);
     }
    
     /*
      * Generate a Param node to replace the given Var,
      * which is expected to have varlevelsup > 0 (ie, it is not local).
      */
     static Param *
     replace_outer_var(PlannerInfo *root, Var *var)//构造Param替换Var
     {
         Param      *retval;
         int         i;
     
         Assert(var->varlevelsup > 0 && var->varlevelsup < root->query_level);
     
         /* Find the Var in the appropriate plan_params, or add it if not present */
         i = assign_param_for_var(root, var);
     
         retval = makeNode(Param);
         retval->paramkind = PARAM_EXEC;
         retval->paramid = i;
         retval->paramtype = var->vartype;
         retval->paramtypmod = var->vartypmod;
         retval->paramcollid = var->varcollid;
         retval->location = var->location;
     
         return retval;
     }
    
    

### 四、转换表达式为隐式AND格式

源码如下:

    
    
     List *
     make_ands_implicit(Expr *clause)
     {
         /*
          * NB: because the parser sets the qual field to NULL in a query that has
          * no WHERE clause, we must consider a NULL input clause as TRUE, even
          * though one might more reasonably think it FALSE.  Grumble. If this
          * causes trouble, consider changing the parser's behavior.
          */
         if (clause == NULL)//如为NULL,返回空指针
             return NIL;             /* NULL -> NIL list == TRUE */
         else if (and_clause((Node *) clause))//AND语句,直接返回AND中的args参数
             return ((BoolExpr *) clause)->args;
         else if (IsA(clause, Const) &&
                  !((Const *) clause)->constisnull &&
                  DatumGetBool(((Const *) clause)->constvalue))
             return NIL;             /* 常量TRUE ,返回空指针constant TRUE input -> NIL list */
         else
             return list_make1(clause);//返回List
     }
    

### 五、参考资料

[subselect.c](https://doxygen.postgresql.org/subselect_8c_source.html)  
[clauses.c](https://doxygen.postgresql.org/clauses_8c_source.html)

