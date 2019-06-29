本文简单介绍了PG查询逻辑优化中的子查询链接(subLink),以ANY子链接为例介绍了子查询链接上拉主函数处理逻辑以及使用gdb跟踪分析。

### 一、子链接上拉简介

PG尝试对ANY_SUBLINK和EXISTS_SUBLINK两种类型的子链接尝试提升,ANY_SUBLINK表示IN/ANY子句,EXISTS_SUBLINK表示EXISTS子句.子链接上拉(pull
up)的目的是为了提升性能,把查询一条记录对比一条记录的逻辑实现变换为半连接或反半连接实现.  
上拉样例:

    
    
    -- 原始SQL
    select * 
    from t_dwxx a
    where dwbh > any (select b.dwbh from t_grxx b);
    -- 上拉后的SQL(示意,实际不能执行)
    select a.* 
    from t_dwxx a semi join select b.dwbh from t_grxx b on a.dwbh > pullup.dwbh;
    

执行了子链接上拉后的执行计划:

    
    
    testdb=# explain select * 
    testdb-# from t_dwxx a
    testdb-# where dwbh > any (select b.dwbh from t_grxx b);
                                   QUERY PLAN                               
    ------------------------------------------------------------------------     Nested Loop Semi Join  (cost=0.00..815.76 rows=53 width=474)
       Join Filter: ((a.dwbh)::text > (b.dwbh)::text)
       ->  Seq Scan on t_dwxx a  (cost=0.00..11.60 rows=160 width=474)
       ->  Materialize  (cost=0.00..17.35 rows=490 width=38)
             ->  Seq Scan on t_grxx b  (cost=0.00..14.90 rows=490 width=38)
    (5 rows)
    

没有上拉的执行计划:

    
    
    testdb=# explain select * 
    testdb-# from t_dwxx a
    testdb-# where dwbh > any (select b.dwbh from t_grxx b where a.dwbh = b.dwbh);
                                QUERY PLAN                            
    ------------------------------------------------------------------     Seq Scan on t_dwxx a  (cost=0.00..1302.40 rows=80 width=474)
       Filter: (SubPlan 1)
       SubPlan 1
         ->  Seq Scan on t_grxx b  (cost=0.00..16.12 rows=2 width=38)
               Filter: ((a.dwbh)::text = (dwbh)::text)
    (5 rows)
    

### 二、源码解读

**pull_up_sublinks**

    
    
     /*
      * pull_up_sublinks
      *      Attempt to pull up ANY and EXISTS SubLinks to be treated as
      *      semijoins or anti-semijoins.
      * 尝试上拉(pull up) ANY/IN 和 EXISTS 子链接,变换为半连接或反半连接
      * 
      * A clause "foo op ANY (sub-SELECT)" can be processed by pulling the
      * sub-SELECT up to become a rangetable entry and treating the implied
      * comparisons as quals of a semijoin.  However, this optimization *only*
      * works at the top level of WHERE or a JOIN/ON clause, because we cannot
      * distinguish whether the ANY ought to return FALSE or NULL in cases
      * involving NULL inputs.  Also, in an outer join's ON clause we can only
      * do this if the sublink is degenerate (ie, references only the nullable
      * side of the join).  In that case it is legal to push the semijoin
      * down into the nullable side of the join.  If the sublink references any
      * nonnullable-side variables then it would have to be evaluated as part
      * of the outer join, which makes things way too complicated.
      *
      * Under similar conditions, EXISTS and NOT EXISTS clauses can be handled
      * by pulling up the sub-SELECT and creating a semijoin or anti-semijoin.
      *
      * express op ANY (sub-SELECT) 语句可以通过上拉子查询为RTE并且把对比操作变换
      * 为半连接操作,这种优化只能在最上层的WHERE语句中实现
      * EXISTS/NOT EXISTS语句类似
      *
      *
      * This routine searches for such clauses and does the necessary parsetree
      * transformations if any are found.
      *
      * This routine has to run before preprocess_expression(), so the quals
      * clauses are not yet reduced to implicit-AND format, and are not guaranteed
      * to be AND/OR-flat either.  That means we need to recursively search through
      * explicit AND clauses.  We stop as soon as we hit a non-AND item.
      */
     void
     pull_up_sublinks(PlannerInfo *root)
     {
         Node       *jtnode;
         Relids      relids;
     
         /* Begin recursion through the jointree */
         jtnode = pull_up_sublinks_jointree_recurse(root,
                                                    (Node *) root->parse->jointree,
                                                    &relids);//执行上拉操作
     
         /*
          * root->parse->jointree must always be a FromExpr, so insert a dummy one
          * if we got a bare RangeTblRef or JoinExpr out of the recursion.
          */
         if (IsA(jtnode, FromExpr))//jointree要求FromExpr类型
             root->parse->jointree = (FromExpr *) jtnode;
         else
             root->parse->jointree = makeFromExpr(list_make1(jtnode), NULL);
     }
     
    

**pull_up_sublinks_jointree_recurse**  
处理流程如下图所示:  

![](https://upload-images.jianshu.io/upload_images/8194836-dc9ba0d51e289a1f.png)

pull_up_sublinks_jointree_recurse处理流程

查询树结构如下:

  

![](https://upload-images.jianshu.io/upload_images/8194836-2a84259749e75e8a.png)

查询树

    
    
     /*
      * Recurse through jointree nodes for pull_up_sublinks()
      *
      * In addition to returning the possibly-modified jointree node, we return
      * a relids set of the contained rels into *relids.
      */
     static Node *
     pull_up_sublinks_jointree_recurse(PlannerInfo *root, Node *jtnode,
                                       Relids *relids)
     {
         if (jtnode == NULL)
         {
             *relids = NULL;
         }
         else if (IsA(jtnode, RangeTblRef))//如为RangeTblRef类型
         {
             int         varno = ((RangeTblRef *) jtnode)->rtindex;
     
             *relids = bms_make_singleton(varno);
             /* jtnode is returned unmodified */
         }
         else if (IsA(jtnode, FromExpr))//如为FromExpr类型
         {
             FromExpr   *f = (FromExpr *) jtnode;
             List       *newfromlist = NIL;
             Relids      frelids = NULL;
             FromExpr   *newf;
             Node       *jtlink;
             ListCell   *l;
     
             /* First, recurse to process children and collect their relids */
             foreach(l, f->fromlist)//
             {
                 Node       *newchild;
                 Relids      childrelids;
                 //对fromlist中的元素执行上拉操作
                 //如能够上拉,则把子查询从WHERE子句中提升到FROM子句,newchild作为连接的一部分
                 newchild = pull_up_sublinks_jointree_recurse(root,
                                                              lfirst(l),
                                                              &childrelids);
                 newfromlist = lappend(newfromlist, newchild);
                 frelids = bms_join(frelids, childrelids);
             }
             /* Build the replacement FromExpr; no quals yet */
             newf = makeFromExpr(newfromlist, NULL);//创建新的FromExpr
             /* Set up a link representing the rebuilt jointree */
             jtlink = (Node *) newf;
             /* Now process qual --- all children are available for use */
             //处理子链接中的表达式
             //newf(指针,相当于jtlink)
             newf->quals = pull_up_sublinks_qual_recurse(root, f->quals,
                                                         &jtlink, frelids,
                                                         NULL, NULL);//
     
             /*
              * Note that the result will be either newf, or a stack of JoinExprs
              * with newf at the base.  We rely on subsequent optimization steps to
              * flatten this and rearrange the joins as needed.
              *
              * Although we could include the pulled-up subqueries in the returned
              * relids, there's no need since upper quals couldn't refer to their
              * outputs anyway.
              */
             *relids = frelids;//设置相关的relids
             jtnode = jtlink;//返回值
         }
         else if (IsA(jtnode, JoinExpr))
         {
             JoinExpr   *j;
             Relids      leftrelids;
             Relids      rightrelids;
             Node       *jtlink;
     
             /*
              * Make a modifiable copy of join node, but don't bother copying its
              * subnodes (yet).
              */
             j = (JoinExpr *) palloc(sizeof(JoinExpr));
             memcpy(j, jtnode, sizeof(JoinExpr));
             jtlink = (Node *) j;
     
             /* Recurse to process children and collect their relids */
             //递归处理左边&右边子树
             j->larg = pull_up_sublinks_jointree_recurse(root, j->larg,
                                                         &leftrelids);
             j->rarg = pull_up_sublinks_jointree_recurse(root, j->rarg,
                                                         &rightrelids);
     
             /*
              * Now process qual, showing appropriate child relids as available,
              * and attach any pulled-up jointree items at the right place. In the
              * inner-join case we put new JoinExprs above the existing one (much
              * as for a FromExpr-style join).  In outer-join cases the new
              * JoinExprs must go into the nullable side of the outer join. The
              * point of the available_rels machinations is to ensure that we only
              * pull up quals for which that's okay.
              *
              * We don't expect to see any pre-existing JOIN_SEMI or JOIN_ANTI
              * nodes here.
              */
             switch (j->jointype)
             {
                 case JOIN_INNER:
                     j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
                                                              &jtlink,
                                                              bms_union(leftrelids,
                                                                        rightrelids),
                                                              NULL, NULL);
                     break;
                 case JOIN_LEFT:
                     j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
                                                              &j->rarg,
                                                              rightrelids,
                                                              NULL, NULL);
                     break;
                 case JOIN_FULL:
                     /* can't do anything with full-join quals */
                     break;
                 case JOIN_RIGHT:
                     j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
                                                              &j->larg,
                                                              leftrelids,
                                                              NULL, NULL);
                     break;
                 default:
                     elog(ERROR, "unrecognized join type: %d",
                          (int) j->jointype);
                     break;
             }
     
             /*
              * Although we could include the pulled-up subqueries in the returned
              * relids, there's no need since upper quals couldn't refer to their
              * outputs anyway.  But we *do* need to include the join's own rtindex
              * because we haven't yet collapsed join alias variables, so upper
              * levels would mistakenly think they couldn't use references to this
              * join.
              */
             *relids = bms_join(leftrelids, rightrelids);
             if (j->rtindex)
                 *relids = bms_add_member(*relids, j->rtindex);
             jtnode = jtlink;
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
         return jtnode;
     }
    

**pull_up_sublinks_qual_recurse**

    
    
     /*
      * Recurse through top-level qual nodes for pull_up_sublinks()
      *
      * jtlink1 points to the link in the jointree where any new JoinExprs should
      * be inserted if they reference available_rels1 (i.e., available_rels1
      * denotes the relations present underneath jtlink1).  Optionally, jtlink2 can
      * point to a second link where new JoinExprs should be inserted if they
      * reference available_rels2 (pass NULL for both those arguments if not used).
      * Note that SubLinks referencing both sets of variables cannot be optimized.
      * If we find multiple pull-up-able SubLinks, they'll get stacked onto jtlink1
      * and/or jtlink2 in the order we encounter them.  We rely on subsequent
      * optimization to rearrange the stack if appropriate.
      *
      * Returns the replacement qual node, or NULL if the qual should be removed.
      */
     static Node *
     pull_up_sublinks_qual_recurse(PlannerInfo *root, Node *node,
                                   Node **jtlink1, Relids available_rels1,
                                   Node **jtlink2, Relids available_rels2)
     {
         if (node == NULL)
             return NULL;
         if (IsA(node, SubLink))//子链接
         {
             SubLink    *sublink = (SubLink *) node;
             JoinExpr   *j;
             Relids      child_rels;
     
             /* Is it a convertible ANY or EXISTS clause? */
             if (sublink->subLinkType == ANY_SUBLINK)//ANY子链接
             {
                 if ((j = convert_ANY_sublink_to_join(root, sublink,
                                                      available_rels1)) != NULL)
                 {
                     /* Yes; insert the new join node into the join tree */
                     j->larg = *jtlink1;
                     *jtlink1 = (Node *) j;
                     /* Recursively process pulled-up jointree nodes */
                     j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                 j->rarg,
                                                                 &child_rels);
     
                     /*
                      * Now recursively process the pulled-up quals.  Any inserted
                      * joins can get stacked onto either j->larg or j->rarg,
                      * depending on which rels they reference.
                      */
                     j->quals = pull_up_sublinks_qual_recurse(root,
                                                              j->quals,
                                                              &j->larg,
                                                              available_rels1,
                                                              &j->rarg,
                                                              child_rels);
                     /* Return NULL representing constant TRUE */
                     return NULL;
                 }
                 if (available_rels2 != NULL &&
                     (j = convert_ANY_sublink_to_join(root, sublink,
                                                      available_rels2)) != NULL)
                 {
                     /* Yes; insert the new join node into the join tree */
                     j->larg = *jtlink2;
                     *jtlink2 = (Node *) j;
                     /* Recursively process pulled-up jointree nodes */
                     j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                 j->rarg,
                                                                 &child_rels);
     
                     /*
                      * Now recursively process the pulled-up quals.  Any inserted
                      * joins can get stacked onto either j->larg or j->rarg,
                      * depending on which rels they reference.
                      */
                     j->quals = pull_up_sublinks_qual_recurse(root,
                                                              j->quals,
                                                              &j->larg,
                                                              available_rels2,
                                                              &j->rarg,
                                                              child_rels);
                     /* Return NULL representing constant TRUE */
                     return NULL;
                 }
             }
             else if (sublink->subLinkType == EXISTS_SUBLINK)//EXISTS子链接
             {
                 if ((j = convert_EXISTS_sublink_to_join(root, sublink, false,
                                                         available_rels1)) != NULL)
                 {
                     /* Yes; insert the new join node into the join tree */
                     j->larg = *jtlink1;
                     *jtlink1 = (Node *) j;
                     /* Recursively process pulled-up jointree nodes */
                     j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                 j->rarg,
                                                                 &child_rels);
     
                     /*
                      * Now recursively process the pulled-up quals.  Any inserted
                      * joins can get stacked onto either j->larg or j->rarg,
                      * depending on which rels they reference.
                      */
                     j->quals = pull_up_sublinks_qual_recurse(root,
                                                              j->quals,
                                                              &j->larg,
                                                              available_rels1,
                                                              &j->rarg,
                                                              child_rels);
                     /* Return NULL representing constant TRUE */
                     return NULL;
                 }
                 if (available_rels2 != NULL &&
                     (j = convert_EXISTS_sublink_to_join(root, sublink, false,
                                                         available_rels2)) != NULL)
                 {
                     /* Yes; insert the new join node into the join tree */
                     j->larg = *jtlink2;
                     *jtlink2 = (Node *) j;
                     /* Recursively process pulled-up jointree nodes */
                     j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                 j->rarg,
                                                                 &child_rels);
     
                     /*
                      * Now recursively process the pulled-up quals.  Any inserted
                      * joins can get stacked onto either j->larg or j->rarg,
                      * depending on which rels they reference.
                      */
                     j->quals = pull_up_sublinks_qual_recurse(root,
                                                              j->quals,
                                                              &j->larg,
                                                              available_rels2,
                                                              &j->rarg,
                                                              child_rels);
                     /* Return NULL representing constant TRUE */
                     return NULL;
                 }
             }
             /* Else return it unmodified */
             return node;
         }
         if (not_clause(node))//NOT语句
         {
             /* If the immediate argument of NOT is EXISTS, try to convert */
             SubLink    *sublink = (SubLink *) get_notclausearg((Expr *) node);
             JoinExpr   *j;
             Relids      child_rels;
     
             if (sublink && IsA(sublink, SubLink))
             {
                 if (sublink->subLinkType == EXISTS_SUBLINK)
                 {
                     if ((j = convert_EXISTS_sublink_to_join(root, sublink, true,
                                                             available_rels1)) != NULL)
                     {
                         /* Yes; insert the new join node into the join tree */
                         j->larg = *jtlink1;
                         *jtlink1 = (Node *) j;
                         /* Recursively process pulled-up jointree nodes */
                         j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                     j->rarg,
                                                                     &child_rels);
     
                         /*
                          * Now recursively process the pulled-up quals.  Because
                          * we are underneath a NOT, we can't pull up sublinks that
                          * reference the left-hand stuff, but it's still okay to
                          * pull up sublinks referencing j->rarg.
                          */
                         j->quals = pull_up_sublinks_qual_recurse(root,
                                                                  j->quals,
                                                                  &j->rarg,
                                                                  child_rels,
                                                                  NULL, NULL);
                         /* Return NULL representing constant TRUE */
                         return NULL;
                     }
                     if (available_rels2 != NULL &&
                         (j = convert_EXISTS_sublink_to_join(root, sublink, true,
                                                             available_rels2)) != NULL)
                     {
                         /* Yes; insert the new join node into the join tree */
                         j->larg = *jtlink2;
                         *jtlink2 = (Node *) j;
                         /* Recursively process pulled-up jointree nodes */
                         j->rarg = pull_up_sublinks_jointree_recurse(root,
                                                                     j->rarg,
                                                                     &child_rels);
     
                         /*
                          * Now recursively process the pulled-up quals.  Because
                          * we are underneath a NOT, we can't pull up sublinks that
                          * reference the left-hand stuff, but it's still okay to
                          * pull up sublinks referencing j->rarg.
                          */
                         j->quals = pull_up_sublinks_qual_recurse(root,
                                                                  j->quals,
                                                                  &j->rarg,
                                                                  child_rels,
                                                                  NULL, NULL);
                         /* Return NULL representing constant TRUE */
                         return NULL;
                     }
                 }
             }
             /* Else return it unmodified */
             return node;
         }
         if (and_clause(node))//AND语句
         {
             /* Recurse into AND clause */
             List       *newclauses = NIL;
             ListCell   *l;
     
             foreach(l, ((BoolExpr *) node)->args)
             {
                 Node       *oldclause = (Node *) lfirst(l);
                 Node       *newclause;
     
                 newclause = pull_up_sublinks_qual_recurse(root,
                                                           oldclause,
                                                           jtlink1,
                                                           available_rels1,
                                                           jtlink2,
                                                           available_rels2);
                 if (newclause)
                     newclauses = lappend(newclauses, newclause);
             }
             /* We might have got back fewer clauses than we started with */
             if (newclauses == NIL)
                 return NULL;
             else if (list_length(newclauses) == 1)
                 return (Node *) linitial(newclauses);
             else
                 return (Node *) make_andclause(newclauses);
         }
         /* Stop if not an AND */
         return node;
     }
    
    

**convert_ANY_sublink_to_join**

    
    
     /*
      * convert_ANY_sublink_to_join: try to convert an ANY SubLink to a join
      *
      * The caller has found an ANY SubLink at the top level of one of the query's
      * qual clauses, but has not checked the properties of the SubLink further.
      * Decide whether it is appropriate to process this SubLink in join style.
      * If so, form a JoinExpr and return it.  Return NULL if the SubLink cannot
      * be converted to a join.
      *
      * The only non-obvious input parameter is available_rels: this is the set
      * of query rels that can safely be referenced in the sublink expression.
      * (We must restrict this to avoid changing the semantics when a sublink
      * is present in an outer join's ON qual.)  The conversion must fail if
      * the converted qual would reference any but these parent-query relids.
      *
      * On success, the returned JoinExpr has larg = NULL and rarg = the jointree
      * item representing the pulled-up subquery.  The caller must set larg to
      * represent the relation(s) on the lefthand side of the new join, and insert
      * the JoinExpr into the upper query's jointree at an appropriate place
      * (typically, where the lefthand relation(s) had been).  Note that the
      * passed-in SubLink must also be removed from its original position in the
      * query quals, since the quals of the returned JoinExpr replace it.
      * (Notionally, we replace the SubLink with a constant TRUE, then elide the
      * redundant constant from the qual.)
      *
      * On success, the caller is also responsible for recursively applying
      * pull_up_sublinks processing to the rarg and quals of the returned JoinExpr.
      * (On failure, there is no need to do anything, since pull_up_sublinks will
      * be applied when we recursively plan the sub-select.)
      *
      * Side effects of a successful conversion include adding the SubLink's
      * subselect to the query's rangetable, so that it can be referenced in
      * the JoinExpr's rarg.
      */
     JoinExpr *
     convert_ANY_sublink_to_join(PlannerInfo *root, SubLink *sublink,
                                 Relids available_rels)
     {
         JoinExpr   *result;
         Query      *parse = root->parse;
         Query      *subselect = (Query *) sublink->subselect;
         Relids      upper_varnos;
         int         rtindex;
         RangeTblEntry *rte;
         RangeTblRef *rtr;
         List       *subquery_vars;
         Node       *quals;
         ParseState *pstate;
     
         Assert(sublink->subLinkType == ANY_SUBLINK);
     
         /*
          * The sub-select must not refer to any Vars of the parent query. (Vars of
          * higher levels should be okay, though.)
          */
         //ANY类型的子链接,子查询不能依赖父查询的任何变量,否则返回NULL(不能上拉)
         if (contain_vars_of_level((Node *) subselect, 1))
             return NULL;
     
         /*
          * The test expression must contain some Vars of the parent query, else
          * it's not gonna be a join.  (Note that it won't have Vars referring to
          * the subquery, rather Params.)
          */
         //子查询的testexpr变量中必须含有父查询的某些Vars,否则不能上拉(join),返回NULL
         upper_varnos = pull_varnos(sublink->testexpr);
         if (bms_is_empty(upper_varnos))
             return NULL;
     
         /*
          * However, it can't refer to anything outside available_rels.
          */
         //但是不能够依赖超出可用的范围之内,否则一样不能上拉
         if (!bms_is_subset(upper_varnos, available_rels))
             return NULL;
     
         /*
          * The combining operators and left-hand expressions mustn't be volatile.
          */
         //组合操作符和左侧的表达式不能是易变(volatile)的,比如随机数等
         if (contain_volatile_functions(sublink->testexpr))
             return NULL;
       
         //校验通过,上拉
         /* Create a dummy ParseState for addRangeTableEntryForSubquery */
         pstate = make_parsestate(NULL);
     
         /*
          * Okay, pull up the sub-select into upper range table.
          *
          * We rely here on the assumption that the outer query has no references
          * to the inner (necessarily true, other than the Vars that we build
          * below). Therefore this is a lot easier than what pull_up_subqueries has
          * to go through.
          */
         //子链接上拉后,子查询变成了上层的RTE,在这里构造
         rte = addRangeTableEntryForSubquery(pstate,
                                             subselect,
                                             makeAlias("ANY_subquery", NIL),
                                             false,
                                             false);
         //添加到上层的rtable中
         parse->rtable = lappend(parse->rtable, rte);
         //rtable中的索引
         rtindex = list_length(parse->rtable);
     
         /*
          * Form a RangeTblRef for the pulled-up sub-select.
          */
         //产生了RTE,自然要生成RTR(RangeTblRef)
         rtr = makeNode(RangeTblRef);
         rtr->rtindex = rtindex;
     
         /*
          * Build a list of Vars representing the subselect outputs.
          */
         //创建子查询的输出列(投影)
         subquery_vars = generate_subquery_vars(root,
                                                subselect->targetList,
                                                rtindex);
     
         /*
          * Build the new join's qual expression, replacing Params with these Vars.
          */
         //构造上层的条件表达式
         quals = convert_testexpr(root, sublink->testexpr, subquery_vars);
     
         /*
          * And finally, build the JoinExpr node.
          */
         //构造返回结果
         result = makeNode(JoinExpr);
         result->jointype = JOIN_SEMI;//变换为半连接
         result->isNatural = false;
         result->larg = NULL;        /* caller must fill this in */
         result->rarg = (Node *) rtr;
         result->usingClause = NIL;
         result->quals = quals;
         result->alias = NULL;
         result->rtindex = 0;        /* we don't need an RTE for it */
     
         return result;
     }
    
    

### 三、数据结构

**Param**

    
    
     
     /*
      * Param
      *
      *      paramkind specifies the kind of parameter. The possible values
      *      for this field are:
      *
      *      PARAM_EXTERN:  The parameter value is supplied from outside the plan.
      *              Such parameters are numbered from 1 to n.
      *
      *      PARAM_EXEC:  The parameter is an internal executor parameter, used
      *              for passing values into and out of sub-queries or from
      *              nestloop joins to their inner scans.
      *              For historical reasons, such parameters are numbered from 0.
      *              These numbers are independent of PARAM_EXTERN numbers.
      *
      *      PARAM_SUBLINK:  The parameter represents an output column of a SubLink
      *              node's sub-select.  The column number is contained in the
      *              `paramid' field.  (This type of Param is converted to
      *              PARAM_EXEC during planning.)
      *
      *      PARAM_MULTIEXPR:  Like PARAM_SUBLINK, the parameter represents an
      *              output column of a SubLink node's sub-select, but here, the
      *              SubLink is always a MULTIEXPR SubLink.  The high-order 16 bits
      *              of the `paramid' field contain the SubLink's subLinkId, and
      *              the low-order 16 bits contain the column number.  (This type
      *              of Param is also converted to PARAM_EXEC during planning.)
      */
     typedef enum ParamKind
     {
         PARAM_EXTERN,
         PARAM_EXEC,
         PARAM_SUBLINK,
         PARAM_MULTIEXPR
     } ParamKind;
     
     typedef struct Param
     {
         Expr        xpr;
         ParamKind   paramkind;      /* kind of parameter. See above */
         int         paramid;        /* numeric ID for parameter */
         Oid         paramtype;      /* pg_type OID of parameter's datatype */
         int32       paramtypmod;    /* typmod value, if known */
         Oid         paramcollid;    /* OID of collation, or InvalidOid if none */
         int         location;       /* token location, or -1 if unknown */
     } Param;
    

### 四、跟踪分析

测试脚本:

    
    
    select * 
    from t_dwxx a
    where dwbh > any (select b.dwbh from t_grxx b);
    
    

gdb跟踪:

    
    
    (gdb) b pull_up_sublinks
    Breakpoint 1 at 0x77cbc6: file prepjointree.c, line 157.
    (gdb) c
    Continuing.
    
    Breakpoint 1, pull_up_sublinks (root=0x249f318) at prepjointree.c:157
    157                                                (Node *) root->parse->jointree,
    (gdb) 
    #输入参数root->parse是查询树
    #查询树结构见第2小结中的查询树图
    ...
    (gdb) p *root->parse
    $2 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = true, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, hasForUpdate = false, 
      hasRowSecurity = false, cteList = 0x0, rtable = 0x23b0798, jointree = 0x23d3290, targetList = 0x23b0bc8, 
      override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, groupClause = 0x0, groupingSets = 0x0, 
      havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, 
      rowMarks = 0x0, setOperations = 0x0, constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 70}
    (gdb) 
    #第1次进入pull_up_sublinks_jointree_recurse
    (gdb) n
    156     jtnode = pull_up_sublinks_jointree_recurse(root,
    (gdb) step
    pull_up_sublinks_jointree_recurse (root=0x249f318, jtnode=0x23d3290, relids=0x7ffc4fd1ad90) at prepjointree.c:180
    180     if (jtnode == NULL)
    (gdb) n
    184     else if (IsA(jtnode, RangeTblRef))
    (gdb) 
    206             newchild = pull_up_sublinks_jointree_recurse(root,
    (gdb) p *jtnode
    $3 = {type = T_FromExpr}
    (gdb) 
    #第2次调用pull_up_sublinks_jointree_recurse,输入参数的jtnode为RangeTblRef
    #第2次调用后返回信息
    (gdb) n
    209             newfromlist = lappend(newfromlist, newchild);
    (gdb) p *(RangeTblRef *)newchild
    $8 = {type = T_RangeTblRef, rtindex = 1}
    ...
    #进入pull_up_sublinks_qual_recurse
    (gdb) step
    pull_up_sublinks_qual_recurse (root=0x249f318, node=0x23b00e8, jtlink1=0x7ffc4fd1ad28, available_rels1=0x249fa98, 
        jtlink2=0x0, available_rels2=0x0) at prepjointree.c:335
    335     if (node == NULL)
    #1.root=PlannerInfo
    #2.node=f->quals,即SubLink(结构参见查询树图)
    #3.jtlink1=FromExpr(指针数组)
    (gdb) p **(FromExpr **)jtlink1
    $29 = {type = T_FromExpr, fromlist = 0x246e0b8, quals = 0x0}
    ...
    #进入convert_ANY_sublink_to_join
    (gdb) 
    346       if ((j = convert_ANY_sublink_to_join(root, sublink,
    (gdb) 
    #输入参数
    #1.root见上
    #2.sublink,子链接
    #3.available_rels,可用的rels
    ...
    #sublink中的子查询
    1322    Query    *subselect = (Query *) sublink->subselect;
    (gdb) 
    1337    if (contain_vars_of_level((Node *) subselect, 1))
    (gdb) p *subselect
    $2 = {type = T_Query, commandType = CMD_SELECT, 
      querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, 
      hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, 
      hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, 
      rtable = 0x1cb1030, jointree = 0x1cb1240, 
      targetList = 0x1cb1210, override = OVERRIDING_NOT_SET, 
      onConflict = 0x0, returningList = 0x0, groupClause = 0x0, 
      groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, 
      distinctClause = 0x0, sortClause = 0x0, limitOffset = 0x0, 
      limitCount = 0x0, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, 
      stmt_location = 0, stmt_len = 0}
    ...
    (gdb) p *upper_varnos.words
    $4 = 2
    (gdb) p available_rels
    $5 = (Relids) 0x1c8ab68
    (gdb) p *available_rels
    $6 = {nwords = 1, words = 0x1c8ab6c}
    (gdb) p *available_rels.words
    $7 = 2
    ...
    #子链接被上拉为上一层jointree的rarg
    #larg由上层填充
    1411    return result;
    (gdb) p *result
    $10 = {type = T_JoinExpr, jointype = JOIN_SEMI, 
      isNatural = false, larg = 0x0, rarg = 0x1c96e88, 
      usingClause = 0x0, quals = 0x1c96ef0, alias = 0x0, 
      rtindex = 0}
    (gdb) n
    1412  }
    (gdb) n
    #回到pull_up_sublinks_qual_recurse
    pull_up_sublinks_qual_recurse (root=0x1c8a3e8, node=0x1bc1038, 
        jtlink1=0x7ffc99e060e8, available_rels1=0x1c8ab68, 
        jtlink2=0x0, available_rels2=0x0) at prepjointree.c:350
    350         j->larg = *jtlink1;
    (gdb) n
    351         *jtlink1 = (Node *) j;
    #larg为RTF(rtindex=1)
    (gdb) p *jtlink1
    $13 = (Node *) 0x1c96ca8
    (gdb) p **jtlink1
    $14 = {type = T_FromExpr}
    (gdb) p **(FromExpr **)jtlink1
    $15 = {type = T_FromExpr, fromlist = 0x1c96c78, quals = 0x0}
    (gdb) p **(FromExpr **)jtlink1->fromlist
    There is no member named fromlist.
    (gdb) p *(*(FromExpr **)jtlink1)->fromlist
    $16 = {type = T_List, length = 1, head = 0x1c96c58, 
      tail = 0x1c96c58}
    (gdb) p *(Node *)(*(FromExpr **)jtlink1)->fromlist->head->data.ptr_value
    $17 = {type = T_RangeTblRef}
    (gdb) p *(RangeTblRef *)(*(FromExpr **)jtlink1)->fromlist->head->data.ptr_value
    $18 = {type = T_RangeTblRef, rtindex = 1}
    #递归上拉右树
    (gdb) step
    pull_up_sublinks_jointree_recurse (root=0x1c8a3e8, 
        jtnode=0x1c96e88, relids=0x7ffc99e06048)
        at prepjointree.c:180
    180   if (jtnode == NULL)
    (gdb) p *jtnode
    $19 = {type = T_RangeTblRef}
    #RTR,退出
    (gdb) finish
    Run till exit from #0  pull_up_sublinks_jointree_recurse (
        root=0x1c8a3e8, jtnode=0x1c96e88, relids=0x7ffc99e06048)
        at prepjointree.c:180
    0x000000000077d00a in pull_up_sublinks_qual_recurse (
        root=0x1c8a3e8, node=0x1bc1038, jtlink1=0x7ffc99e060e8, 
        available_rels1=0x1c8ab68, jtlink2=0x0, available_rels2=0x0)
        at prepjointree.c:353
    353         j->rarg = pull_up_sublinks_jointree_recurse(root,
    Value returned is $20 = (Node *) 0x1c96e88
    (gdb) n
    362         j->quals = pull_up_sublinks_qual_recurse(root,
    (gdb) step
    #递归上拉条件表达式,节点类型为OpExpr,无需处理
    Breakpoint 1, pull_up_sublinks_qual_recurse (root=0x1c8a3e8, 
        node=0x1c96ef0, jtlink1=0x1c97100, 
        available_rels1=0x1c8ab68, jtlink2=0x1c97108, 
        available_rels2=0x1c97140) at prepjointree.c:335
    335   if (node == NULL)
    (gdb) n
    337   if (IsA(node, SubLink))
    (gdb) p *node
    $21 = {type = T_OpExpr}
    ...
    369         return NULL;
    (gdb) 
    552 }
    (gdb) 
    #回到pull_up_sublinks_jointree_recurse
    pull_up_sublinks_jointree_recurse (root=0x1c8a3e8, 
        jtnode=0x1cb1540, relids=0x7ffc99e06150)
        at prepjointree.c:230
    230     *relids = frelids;
    (gdb) p *newf
    #newf的条件表达式为NULL(TRUE)
    $22 = {type = T_FromExpr, fromlist = 0x1c96c78, quals = 0x0}
    (gdb) p *jtlink
    $23 = {type = T_JoinExpr}
    (gdb) n
    231     jtnode = jtlink;
    (gdb) n
    312   return jtnode;
    (gdb) 
    313 }
    (gdb) 
    pull_up_sublinks (root=0x1c8a3e8) at prepjointree.c:164
    164   if (IsA(jtnode, FromExpr))
    (gdb) 
    167     root->parse->jointree = makeFromExpr(list_make1(jtnode), NULL);
    (gdb) 
    168 }
    (gdb) 
    subquery_planner (glob=0x1bc1d50, parse=0x1bc1328, 
        parent_root=0x0, hasRecursion=false, tuple_fraction=0)
        at planner.c:656
    warning: Source file is more recent than executable.
    656   inline_set_returning_functions(root);
    (gdb) finish
    Run till exit from #0  subquery_planner (glob=0x1bc1d50, 
        parse=0x1bc1328, parent_root=0x0, hasRecursion=false, 
        tuple_fraction=0) at planner.c:656
    0x0000000000769a49 in standard_planner (parse=0x1bc1328, 
        cursorOptions=256, boundParams=0x0) at planner.c:405
    405   root = subquery_planner(glob, parse, NULL,
    Value returned is $24 = (PlannerInfo *) 0x1c8a3e8
    (gdb) 
    #DONE!
    

### 五、小结

1、上拉过程：上拉FromExpr中的子链接->上拉条件表达式quals中的子链接。处理过程使用递归；  
2、上拉条件：ANY子句，子查询中含有父查询的Vars（数据列）、依赖更高层次的RTE或者存在易变函数，不能上拉。

参考资料：  
[prepjointree.c](https://doxygen.postgresql.org/prepjointree_8c_source.html#l02353)  
[list.c](https://doxygen.postgresql.org/prepjointree_8c_source.html#l01408)

