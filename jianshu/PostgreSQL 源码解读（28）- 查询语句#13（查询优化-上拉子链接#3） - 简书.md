本节简单介绍了PG查询逻辑优化中的子查询链接(subLink),以EXISTS子链接为例介绍了子查询链接上拉主函数处理逻辑以及使用gdb跟踪分析。

### 一、源码解读

上一节介绍了ANY子链接,本节介绍了EXISTS子链接.  
为便于方便解析,根据日志分析,得出查询树如下图所示:

  

![](https://upload-images.jianshu.io/upload_images/8194836-4671ca16e13b16d0.png)

查询树

convert_EXISTS_sublink_to_join函数源码:

    
    
     /*
      * convert_EXISTS_sublink_to_join: try to convert an EXISTS SubLink to a join
      *
      * The API of this function is identical to convert_ANY_sublink_to_join's,
      * except that we also support the case where the caller has found NOT EXISTS,
      * so we need an additional input parameter "under_not".
      * 逻辑与ANY一致,为了支持NOT EXISTS,多加了一个参数under_not
      */
      
     JoinExpr *
     convert_EXISTS_sublink_to_join(PlannerInfo *root, SubLink *sublink,
                                    bool under_not, Relids available_rels)
     {
         JoinExpr   *result;//返回结果
         Query      *parse = root->parse;//查询树
         Query      *subselect = (Query *) sublink->subselect;//子查询
         Node       *whereClause;//where语句
         int         rtoffset;//
         int         varno;
         Relids      clause_varnos;
         Relids      upper_varnos;
     
         Assert(sublink->subLinkType == EXISTS_SUBLINK);
     
         /*
          * Can't flatten if it contains WITH.  (We could arrange to pull up the
          * WITH into the parent query's cteList, but that risks changing the
          * semantics, since a WITH ought to be executed once per associated query
          * call.)  Note that convert_ANY_sublink_to_join doesn't have to reject
          * this case, since it just produces a subquery RTE that doesn't have to
          * get flattened into the parent query.
          */
         if (subselect->cteList)//存在With子句,返回
             return NULL;
     
         /*
          * Copy the subquery so we can modify it safely (see comments in
          * make_subplan).
          */
         subselect = copyObject(subselect);
     
         /*
          * See if the subquery can be simplified based on the knowledge that it's
          * being used in EXISTS().  If we aren't able to get rid of its
          * targetlist, we have to fail, because the pullup operation leaves us
          * with noplace to evaluate the targetlist.
          */
         //能否handle targetList?不行,则退出
         //如果含有集合操作/聚合操作/Having子句等,子链接不能提升
         if (!simplify_EXISTS_query(root, subselect))
             return NULL;
     
         /*
          * The subquery must have a nonempty jointree, else we won't have a join.
          */
         if (subselect->jointree->fromlist == NIL)//子查询没有查询主体,退出
             return NULL;
     
         /*
          * Separate out the WHERE clause.  (We could theoretically also remove
          * top-level plain JOIN/ON clauses, but it's probably not worth the
          * trouble.)
          */
         whereClause = subselect->jointree->quals;//子查询条件语句单独保存
         subselect->jointree->quals = NULL;//子查询的条件语句设置为NULL
     
         /*
          * The rest of the sub-select must not refer to any Vars of the parent
          * query.  (Vars of higher levels should be okay, though.)
          */
         if (contain_vars_of_level((Node *) subselect, 1))//去掉条件语句后,如仍依赖父查询的Vars,退出
             return NULL;
     
         /*
          * On the other hand, the WHERE clause must contain some Vars of the
          * parent query, else it's not gonna be a join.
          */
         if (!contain_vars_of_level(whereClause, 1))//条件语句必须含有父查询的Vars,否则不构成连接,退出
             return NULL;
     
         /*
          * We don't risk optimizing if the WHERE clause is volatile, either.
          */
         if (contain_volatile_functions(whereClause))//条件语句存在易变函数(如随机函数等)
             return NULL;
     
         /*
          * Prepare to pull up the sub-select into top range table.
          *
          * We rely here on the assumption that the outer query has no references
          * to the inner (necessarily true). Therefore this is a lot easier than
          * what pull_up_subqueries has to go through.
          *
          * In fact, it's even easier than what convert_ANY_sublink_to_join has to
          * do.  The machinations of simplify_EXISTS_query ensured that there is
          * nothing interesting in the subquery except an rtable and jointree, and
          * even the jointree FromExpr no longer has quals.  So we can just append
          * the rtable to our own and use the FromExpr in our jointree. But first,
          * adjust all level-zero varnos in the subquery to account for the rtable
          * merger.
          */
         rtoffset = list_length(parse->rtable);//获取rtable的长度(新RTE插入的偏移)
         //调整子查询中varno为0(指向rtable)的Vars,varno调整为父查询rtable的index
         //(详见依赖函数解析)
         OffsetVarNodes((Node *) subselect, rtoffset, 0);
         OffsetVarNodes(whereClause, rtoffset, 0);
     
         /*
          * Upper-level vars in subquery will now be one level closer to their
          * parent than before; in particular, anything that had been level 1
          * becomes level zero.
          */
         //子查询中与父查询相关的Vars,varlevelsup需要从Leve 1变为Level 0
         //在子查询中,这些Vars的varlevelsup为1,表示依赖于父查询(上一层)的Vars,提升后不存在此依赖,需改为0
         IncrementVarSublevelsUp((Node *) subselect, -1, 1);
         IncrementVarSublevelsUp(whereClause, -1, 1);
     
         /*
          * Now that the WHERE clause is adjusted to match the parent query
          * environment, we can easily identify all the level-zero rels it uses.
          * The ones <= rtoffset belong to the upper query; the ones > rtoffset do
          * not.
          */
         clause_varnos = pull_varnos(whereClause);
         upper_varnos = NULL;
         while ((varno = bms_first_member(clause_varnos)) >= 0)
         {
             if (varno <= rtoffset)
                 upper_varnos = bms_add_member(upper_varnos, varno);
         }
         bms_free(clause_varnos);
         Assert(!bms_is_empty(upper_varnos));
     
         /*
          * Now that we've got the set of upper-level varnos, we can make the last
          * check: only available_rels can be referenced.
          */
         if (!bms_is_subset(upper_varnos, available_rels))
             return NULL;
     
         /* Now we can attach the modified subquery rtable to the parent */
         //把子查询的rtable拼接到父查询的rtable中
         parse->rtable = list_concat(parse->rtable, subselect->rtable);
     
         //构造JoinExpr
         /*
          * And finally, build the JoinExpr node.
          */
         result = makeNode(JoinExpr);
         result->jointype = under_not ? JOIN_ANTI : JOIN_SEMI;
         result->isNatural = false;
         result->larg = NULL;        /* caller must fill this in */
         /* flatten out the FromExpr node if it's useless */
         if (list_length(subselect->jointree->fromlist) == 1)
             result->rarg = (Node *) linitial(subselect->jointree->fromlist);
         else
             result->rarg = (Node *) subselect->jointree;
         result->usingClause = NIL;
         result->quals = whereClause;
         result->alias = NULL;
         result->rtindex = 0;        /* we don't need an RTE for it */
     
         return result;
     }
    

### 二、基础信息

**相关数据结构**  
_1、Var_

    
    
     /*
      * Var - expression node representing a variable (ie, a table column)
      *
      * Note: during parsing/planning, varnoold/varoattno are always just copies
      * of varno/varattno.  At the tail end of planning, Var nodes appearing in
      * upper-level plan nodes are reassigned to point to the outputs of their
      * subplans; for example, in a join node varno becomes INNER_VAR or OUTER_VAR
      * and varattno becomes the index of the proper element of that subplan's
      * target list.  Similarly, INDEX_VAR is used to identify Vars that reference
      * an index column rather than a heap column.  (In ForeignScan and CustomScan
      * plan nodes, INDEX_VAR is abused to signify references to columns of a
      * custom scan tuple type.)  In all these cases, varnoold/varoattno hold the
      * original values.  The code doesn't really need varnoold/varoattno, but they
      * are very useful for debugging and interpreting completed plans, so we keep
      * them around.
      */
     #define    INNER_VAR        65000   /* reference to inner subplan */
     #define    OUTER_VAR        65001   /* reference to outer subplan */
     #define    INDEX_VAR        65002   /* reference to index column */
     
     #define IS_SPECIAL_VARNO(varno)     ((varno) >= INNER_VAR)
     
     /* Symbols for the indexes of the special RTE entries in rules */
     #define    PRS2_OLD_VARNO           1
     #define    PRS2_NEW_VARNO           2
     
     typedef struct Var
     {
         Expr        xpr;
         Index       varno;          /* index of this var's relation in the range
                                      * table, or INNER_VAR/OUTER_VAR/INDEX_VAR */
         AttrNumber  varattno;       /* attribute number of this var, or zero for
                                      * all attrs ("whole-row Var") */
         Oid         vartype;        /* pg_type OID for the type of this var */
         int32       vartypmod;      /* pg_attribute typmod value */
         Oid         varcollid;      /* OID of collation, or InvalidOid if none */
         Index       varlevelsup;    /* for subquery variables referencing outer
                                      * relations; 0 in a normal var, >0 means N
                                      * levels up */
         Index       varnoold;       /* original value of varno, for debugging */
         AttrNumber  varoattno;      /* original value of varattno */
         int         location;       /* token location, or -1 if unknown */
     } Var;
     
    

_XX_one_pos_

    
    
     /*
      * Lookup tables to avoid need for bit-by-bit groveling
      *
      * rightmost_one_pos[x] gives the bit number (0-7) of the rightmost one bit
      * in a nonzero byte value x.  The entry for x=0 is never used.
      *
      * leftmost_one_pos[x] gives the bit number (0-7) of the leftmost one bit in a
      * nonzero byte value x.  The entry for x=0 is never used.
      *
      * number_of_ones[x] gives the number of one-bits (0-8) in a byte value x.
      *
      * We could make these tables larger and reduce the number of iterations
      * in the functions that use them, but bytewise shifts and masks are
      * especially fast on many machines, so working a byte at a time seems best.
      */
     
     static const uint8 rightmost_one_pos[256] = {
         0, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         7, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
         4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0
     };
     
     static const uint8 leftmost_one_pos[256] = {
         0, 0, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3,
         4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
         5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
         5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
         6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6,
         6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6,
         6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6,
         6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7,
         7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7
     };
     
     static const uint8 number_of_ones[256] = {
         0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2, 3, 2, 3, 3, 4,
         1, 2, 2, 3, 2, 3, 3, 4, 2, 3, 3, 4, 3, 4, 4, 5,
         1, 2, 2, 3, 2, 3, 3, 4, 2, 3, 3, 4, 3, 4, 4, 5,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         1, 2, 2, 3, 2, 3, 3, 4, 2, 3, 3, 4, 3, 4, 4, 5,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         3, 4, 4, 5, 4, 5, 5, 6, 4, 5, 5, 6, 5, 6, 6, 7,
         1, 2, 2, 3, 2, 3, 3, 4, 2, 3, 3, 4, 3, 4, 4, 5,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         3, 4, 4, 5, 4, 5, 5, 6, 4, 5, 5, 6, 5, 6, 6, 7,
         2, 3, 3, 4, 3, 4, 4, 5, 3, 4, 4, 5, 4, 5, 5, 6,
         3, 4, 4, 5, 4, 5, 5, 6, 4, 5, 5, 6, 5, 6, 6, 7,
         3, 4, 4, 5, 4, 5, 5, 6, 4, 5, 5, 6, 5, 6, 6, 7,
         4, 5, 5, 6, 5, 6, 6, 7, 5, 6, 6, 7, 6, 7, 7, 8
     };
     
    

**依赖的函数**  
_simplify_EXISTS_query_

    
    
     /*
      * simplify_EXISTS_query: remove any useless stuff in an EXISTS's subquery
      *
      * The only thing that matters about an EXISTS query is whether it returns
      * zero or more than zero rows.  Therefore, we can remove certain SQL features
      * that won't affect that.  The only part that is really likely to matter in
      * typical usage is simplifying the targetlist: it's a common habit to write
      * "SELECT * FROM" even though there is no need to evaluate any columns.
      *
      * Note: by suppressing the targetlist we could cause an observable behavioral
      * change, namely that any errors that might occur in evaluating the tlist
      * won't occur, nor will other side-effects of volatile functions.  This seems
      * unlikely to bother anyone in practice.
      *
      * Returns true if was able to discard the targetlist, else false.
      */
     static bool
     simplify_EXISTS_query(PlannerInfo *root, Query *query)
     {
         /*
          * We don't try to simplify at all if the query uses set operations,
          * aggregates, grouping sets, SRFs, modifying CTEs, HAVING, OFFSET, or FOR
          * UPDATE/SHARE; none of these seem likely in normal usage and their
          * possible effects are complex.  (Note: we could ignore an "OFFSET 0"
          * clause, but that traditionally is used as an optimization fence, so we
          * don't.)
          */
         if (query->commandType != CMD_SELECT ||
             query->setOperations ||
             query->hasAggs ||
             query->groupingSets ||
             query->hasWindowFuncs ||
             query->hasTargetSRFs ||
             query->hasModifyingCTE ||
             query->havingQual ||
             query->limitOffset ||
             query->rowMarks)
             return false;
     
         /*
          * LIMIT with a constant positive (or NULL) value doesn't affect the
          * semantics of EXISTS, so let's ignore such clauses.  This is worth doing
          * because people accustomed to certain other DBMSes may be in the habit
          * of writing EXISTS(SELECT ... LIMIT 1) as an optimization.  If there's a
          * LIMIT with anything else as argument, though, we can't simplify.
          */
         if (query->limitCount)
         {
             /*
              * The LIMIT clause has not yet been through eval_const_expressions,
              * so we have to apply that here.  It might seem like this is a waste
              * of cycles, since the only case plausibly worth worrying about is
              * "LIMIT 1" ... but what we'll actually see is "LIMIT int8(1::int4)",
              * so we have to fold constants or we're not going to recognize it.
              */
             Node       *node = eval_const_expressions(root, query->limitCount);
             Const      *limit;
     
             /* Might as well update the query if we simplified the clause. */
             query->limitCount = node;
     
             if (!IsA(node, Const))
                 return false;
     
             limit = (Const *) node;
             Assert(limit->consttype == INT8OID);
             if (!limit->constisnull && DatumGetInt64(limit->constvalue) <= 0)
                 return false;
     
             /* Whether or not the targetlist is safe, we can drop the LIMIT. */
             query->limitCount = NULL;
         }
     
         /*
          * Otherwise, we can throw away the targetlist, as well as any GROUP,
          * WINDOW, DISTINCT, and ORDER BY clauses; none of those clauses will
          * change a nonzero-rows result to zero rows or vice versa.  (Furthermore,
          * since our parsetree representation of these clauses depends on the
          * targetlist, we'd better throw them away if we drop the targetlist.)
          */
         query->targetList = NIL;
         query->groupClause = NIL;
         query->windowClause = NIL;
         query->distinctClause = NIL;
         query->sortClause = NIL;
         query->hasDistinctOn = false;
     
         return true;
     }
     
    

_OffsetVarNodes_

    
    
     void
     OffsetVarNodes(Node *node, int offset, int sublevels_up)
     {
         OffsetVarNodes_context context;//上下文
     
         context.offset = offset;//保存传入的offset
         context.sublevels_up = sublevels_up;
     
         /*
          * Must be prepared to start with a Query or a bare expression tree; if
          * it's a Query, go straight to query_tree_walker to make sure that
          * sublevels_up doesn't get incremented prematurely.
          */
         if (node && IsA(node, Query))
         {
             Query      *qry = (Query *) node;
     
             /*
              * If we are starting at a Query, and sublevels_up is zero, then we
              * must also fix rangetable indexes in the Query itself --- namely
              * resultRelation, exclRelIndex and rowMarks entries.  sublevels_up
              * cannot be zero when recursing into a subquery, so there's no need
              * to have the same logic inside OffsetVarNodes_walker.
              */
             if (sublevels_up == 0)
             {
                 ListCell   *l;
     
                 if (qry->resultRelation)
                     qry->resultRelation += offset;
     
                 if (qry->onConflict && qry->onConflict->exclRelIndex)
                     qry->onConflict->exclRelIndex += offset;
     
                 foreach(l, qry->rowMarks)
                 {
                     RowMarkClause *rc = (RowMarkClause *) lfirst(l);
     
                     rc->rti += offset;
                 }
             }
             query_tree_walker(qry, OffsetVarNodes_walker,
                               (void *) &context, 0);
         }
         else
             OffsetVarNodes_walker(node, &context);
     }
     
    

_IncrementVarSublevelsUp_

    
    
     void
     IncrementVarSublevelsUp(Node *node, int delta_sublevels_up,
                             int min_sublevels_up)
     {
         IncrementVarSublevelsUp_context context;
     
         context.delta_sublevels_up = delta_sublevels_up;
         context.min_sublevels_up = min_sublevels_up;
     
         /*
          * Must be prepared to start with a Query or a bare expression tree; if
          * it's a Query, we don't want to increment sublevels_up.
          */
         query_or_expression_tree_walker(node,
                                         IncrementVarSublevelsUp_walker,
                                         (void *) &context,
                                         QTW_EXAMINE_RTES);
     }
    

_IncrementVarSublevelsUp_

    
    
     /*
      * IncrementVarSublevelsUp - adjust Var nodes when pushing them down in tree
      *
      * Find all Var nodes in the given tree having varlevelsup >= min_sublevels_up,
      * and add delta_sublevels_up to their varlevelsup value.  This is needed when
      * an expression that's correct for some nesting level is inserted into a
      * subquery.  Ordinarily the initial call has min_sublevels_up == 0 so that
      * all Vars are affected.  The point of min_sublevels_up is that we can
      * increment it when we recurse into a sublink, so that local variables in
      * that sublink are not affected, only outer references to vars that belong
      * to the expression's original query level or parents thereof.
      *
      * Likewise for other nodes containing levelsup fields, such as Aggref.
      *
      * NOTE: although this has the form of a walker, we cheat and modify the
      * Var nodes in-place.  The given expression tree should have been copied
      * earlier to ensure that no unwanted side-effects occur!
      */
     
     typedef struct
     {
         int         delta_sublevels_up;
         int         min_sublevels_up;
     } IncrementVarSublevelsUp_context;
     
     static bool
     IncrementVarSublevelsUp_walker(Node *node,
                                    IncrementVarSublevelsUp_context *context)
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))
         {
             Var        *var = (Var *) node;
     
             if (var->varlevelsup >= context->min_sublevels_up)
                 var->varlevelsup += context->delta_sublevels_up;
             return false;           /* done here */
         }
         if (IsA(node, CurrentOfExpr))
         {
             /* this should not happen */
             if (context->min_sublevels_up == 0)
                 elog(ERROR, "cannot push down CurrentOfExpr");
             return false;
         }
         if (IsA(node, Aggref))
         {
             Aggref     *agg = (Aggref *) node;
     
             if (agg->agglevelsup >= context->min_sublevels_up)
                 agg->agglevelsup += context->delta_sublevels_up;
             /* fall through to recurse into argument */
         }
         if (IsA(node, GroupingFunc))
         {
             GroupingFunc *grp = (GroupingFunc *) node;
     
             if (grp->agglevelsup >= context->min_sublevels_up)
                 grp->agglevelsup += context->delta_sublevels_up;
             /* fall through to recurse into argument */
         }
         if (IsA(node, PlaceHolderVar))
         {
             PlaceHolderVar *phv = (PlaceHolderVar *) node;
     
             if (phv->phlevelsup >= context->min_sublevels_up)
                 phv->phlevelsup += context->delta_sublevels_up;
             /* fall through to recurse into argument */
         }
         if (IsA(node, RangeTblEntry))
         {
             RangeTblEntry *rte = (RangeTblEntry *) node;
     
             if (rte->rtekind == RTE_CTE)
             {
                 if (rte->ctelevelsup >= context->min_sublevels_up)
                     rte->ctelevelsup += context->delta_sublevels_up;
             }
             return false;           /* allow range_table_walker to continue */
         }
         if (IsA(node, Query))
         {
             /* Recurse into subselects */
             bool        result;
     
             context->min_sublevels_up++;
             result = query_tree_walker((Query *) node,
                                        IncrementVarSublevelsUp_walker,
                                        (void *) context,
                                        QTW_EXAMINE_RTES);
             context->min_sublevels_up--;
             return result;
         }
         return expression_tree_walker(node, IncrementVarSublevelsUp_walker,
                                       (void *) context);
     }
    

_pull_varnos_

    
    
    /*
      * pull_varnos
      *      Create a set of all the distinct varnos present in a parsetree.
      *      Only varnos that reference level-zero rtable entries are considered.
      *
      * NOTE: this is used on not-yet-planned expressions.  It may therefore find
      * bare SubLinks, and if so it needs to recurse into them to look for uplevel
      * references to the desired rtable level!  But when we find a completed
      * SubPlan, we only need to look at the parameters passed to the subplan.
      */
     Relids
     pull_varnos(Node *node)
     {
         pull_varnos_context context;
     
         context.varnos = NULL;
         context.sublevels_up = 0;
     
         /*
          * Must be prepared to start with a Query or a bare expression tree; if
          * it's a Query, we don't want to increment sublevels_up.
          */
         query_or_expression_tree_walker(node,
                                         pull_varnos_walker,
                                         (void *) &context,
                                         0);
     
         return context.varnos;
     }
    

_contain_vars_of_level_

    
    
    /*
      * contain_vars_of_level
      *    Recursively scan a clause to discover whether it contains any Var nodes
      *    of the specified query level.
      *
      *    Returns true if any such Var found.
      *
      * Will recurse into sublinks.  Also, may be invoked directly on a Query.
      */
     bool
     contain_vars_of_level(Node *node, int levelsup)
     {
         int         sublevels_up = levelsup;
     
         return query_or_expression_tree_walker(node,
                                                contain_vars_of_level_walker,
                                                (void *) &sublevels_up,
                                                0);
     }
    

_query_tree_walker_

    
    
     
     /*
      * query_tree_walker --- initiate a walk of a Query's expressions
      *
      * This routine exists just to reduce the number of places that need to know
      * where all the expression subtrees of a Query are.  Note it can be used
      * for starting a walk at top level of a Query regardless of whether the
      * walker intends to descend into subqueries.  It is also useful for
      * descending into subqueries within a walker.
      *
      * Some callers want to suppress visitation of certain items in the sub-Query,
      * typically because they need to process them specially, or don't actually
      * want to recurse into subqueries.  This is supported by the flags argument,
      * which is the bitwise OR of flag values to suppress visitation of
      * indicated items.  (More flag bits may be added as needed.)
      */
     bool
     query_tree_walker(Query *query,
                       bool (*walker) (),
                       void *context,
                       int flags)
     {
         Assert(query != NULL && IsA(query, Query));
     
         if (walker((Node *) query->targetList, context))
             return true;
         if (walker((Node *) query->withCheckOptions, context))
             return true;
         if (walker((Node *) query->onConflict, context))
             return true;
         if (walker((Node *) query->returningList, context))
             return true;
         if (walker((Node *) query->jointree, context))
             return true;
         if (walker(query->setOperations, context))
             return true;
         if (walker(query->havingQual, context))
             return true;
         if (walker(query->limitOffset, context))
             return true;
         if (walker(query->limitCount, context))
             return true;
         if (!(flags & QTW_IGNORE_CTE_SUBQUERIES))
         {
             if (walker((Node *) query->cteList, context))
                 return true;
         }
         if (!(flags & QTW_IGNORE_RANGE_TABLE))
         {
             if (range_table_walker(query->rtable, walker, context, flags))
                 return true;
         }
         return false;
     }
    

_OffsetVarNodes_walker_

    
    
     /*
      * OffsetVarNodes - adjust Vars when appending one query's RT to another
      *
      * Find all Var nodes in the given tree with varlevelsup == sublevels_up,
      * and increment their varno fields (rangetable indexes) by 'offset'.
      * The varnoold fields are adjusted similarly.  Also, adjust other nodes
      * that contain rangetable indexes, such as RangeTblRef and JoinExpr.
      *
      * NOTE: although this has the form of a walker, we cheat and modify the
      * nodes in-place.  The given expression tree should have been copied
      * earlier to ensure that no unwanted side-effects occur!
      */
     
     typedef struct
     {
         int         offset;
         int         sublevels_up;
     } OffsetVarNodes_context;
     
     static bool
     OffsetVarNodes_walker(Node *node, OffsetVarNodes_context *context)
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))
         {
             Var        *var = (Var *) node;
     
             if (var->varlevelsup == context->sublevels_up)
             {
                 var->varno += context->offset;
                 var->varnoold += context->offset;
             }
             return false;
         }
         if (IsA(node, CurrentOfExpr))
         {
             CurrentOfExpr *cexpr = (CurrentOfExpr *) node;
     
             if (context->sublevels_up == 0)
                 cexpr->cvarno += context->offset;
             return false;
         }
         if (IsA(node, RangeTblRef))
         {
             RangeTblRef *rtr = (RangeTblRef *) node;
     
             if (context->sublevels_up == 0)
                 rtr->rtindex += context->offset;
             /* the subquery itself is visited separately */
             return false;
         }
         if (IsA(node, JoinExpr))
         {
             JoinExpr   *j = (JoinExpr *) node;
     
             if (j->rtindex && context->sublevels_up == 0)
                 j->rtindex += context->offset;
             /* fall through to examine children */
         }
         if (IsA(node, PlaceHolderVar))
         {
             PlaceHolderVar *phv = (PlaceHolderVar *) node;
     
             if (phv->phlevelsup == context->sublevels_up)
             {
                 phv->phrels = offset_relid_set(phv->phrels,
                                                context->offset);
             }
             /* fall through to examine children */
         }
         if (IsA(node, AppendRelInfo))
         {
             AppendRelInfo *appinfo = (AppendRelInfo *) node;
     
             if (context->sublevels_up == 0)
             {
                 appinfo->parent_relid += context->offset;
                 appinfo->child_relid += context->offset;
             }
             /* fall through to examine children */
         }
         /* Shouldn't need to handle other planner auxiliary nodes here */
         Assert(!IsA(node, PlanRowMark));
         Assert(!IsA(node, SpecialJoinInfo));
         Assert(!IsA(node, PlaceHolderInfo));
         Assert(!IsA(node, MinMaxAggInfo));
     
         if (IsA(node, Query))
         {
             /* Recurse into subselects */
             bool        result;
     
             context->sublevels_up++;
             result = query_tree_walker((Query *) node, OffsetVarNodes_walker,
                                        (void *) context, 0);
             context->sublevels_up--;
             return result;
         }
         return expression_tree_walker(node, OffsetVarNodes_walker,
                                       (void *) context);
     }
    

_query_or_expression_tree_walker_

    
    
     /*
      * query_or_expression_tree_walker --- hybrid form
      *
      * This routine will invoke query_tree_walker if called on a Query node,
      * else will invoke the walker directly.  This is a useful way of starting
      * the recursion when the walker's normal change of state is not appropriate
      * for the outermost Query node.
      */
     bool
     query_or_expression_tree_walker(Node *node,
                                     bool (*walker) (),
                                     void *context,
                                     int flags)
     {
         if (node && IsA(node, Query))
             return query_tree_walker((Query *) node,
                                      walker,
                                      context,
                                      flags);
         else
             return walker(node, context);
     }
    

_pull_varnos_walker_

    
    
     static bool
     pull_varnos_walker(Node *node, pull_varnos_context *context)
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))
         {
             Var        *var = (Var *) node;
     
             if (var->varlevelsup == context->sublevels_up)
                 context->varnos = bms_add_member(context->varnos, var->varno);
             return false;
         }
         if (IsA(node, CurrentOfExpr))
         {
             CurrentOfExpr *cexpr = (CurrentOfExpr *) node;
     
             if (context->sublevels_up == 0)
                 context->varnos = bms_add_member(context->varnos, cexpr->cvarno);
             return false;
         }
         if (IsA(node, PlaceHolderVar))
         {
             /*
              * A PlaceHolderVar acts as a variable of its syntactic scope, or
              * lower than that if it references only a subset of the rels in its
              * syntactic scope.  It might also contain lateral references, but we
              * should ignore such references when computing the set of varnos in
              * an expression tree.  Also, if the PHV contains no variables within
              * its syntactic scope, it will be forced to be evaluated exactly at
              * the syntactic scope, so take that as the relid set.
              */
             PlaceHolderVar *phv = (PlaceHolderVar *) node;
             pull_varnos_context subcontext;
     
             subcontext.varnos = NULL;
             subcontext.sublevels_up = context->sublevels_up;
             (void) pull_varnos_walker((Node *) phv->phexpr, &subcontext);
             if (phv->phlevelsup == context->sublevels_up)
             {
                 subcontext.varnos = bms_int_members(subcontext.varnos,
                                                     phv->phrels);
                 if (bms_is_empty(subcontext.varnos))
                     context->varnos = bms_add_members(context->varnos,
                                                       phv->phrels);
             }
             context->varnos = bms_join(context->varnos, subcontext.varnos);
             return false;
         }
         if (IsA(node, Query))
         {
             /* Recurse into RTE subquery or not-yet-planned sublink subquery */
             bool        result;
     
             context->sublevels_up++;
             result = query_tree_walker((Query *) node, pull_varnos_walker,
                                        (void *) context, 0);
             context->sublevels_up--;
             return result;
         }
         return expression_tree_walker(node, pull_varnos_walker,
                                       (void *) context);
     }
    

_contain_vars_of_level_walker_

    
    
     static bool
     contain_vars_of_level_walker(Node *node, int *sublevels_up)
     {
         if (node == NULL)
             return false;
         if (IsA(node, Var))
         {
             if (((Var *) node)->varlevelsup == *sublevels_up)
                 return true;        /* abort tree traversal and return true */
             return false;
         }
         if (IsA(node, CurrentOfExpr))
         {
             if (*sublevels_up == 0)
                 return true;
             return false;
         }
         if (IsA(node, PlaceHolderVar))
         {
             if (((PlaceHolderVar *) node)->phlevelsup == *sublevels_up)
                 return true;        /* abort the tree traversal and return true */
             /* else fall through to check the contained expr */
         }
         if (IsA(node, Query))
         {
             /* Recurse into subselects */
             bool        result;
     
             (*sublevels_up)++;
             result = query_tree_walker((Query *) node,
                                        contain_vars_of_level_walker,
                                        (void *) sublevels_up,
                                        0);
             (*sublevels_up)--;
             return result;
         }
         return expression_tree_walker(node,
                                       contain_vars_of_level_walker,
                                       (void *) sublevels_up);
     }
    

_expression_tree_walker_

    
    
     /*
      * Standard expression-tree walking support
      *
      * We used to have near-duplicate code in many different routines that
      * understood how to recurse through an expression node tree.  That was
      * a pain to maintain, and we frequently had bugs due to some particular
      * routine neglecting to support a particular node type.  In most cases,
      * these routines only actually care about certain node types, and don't
      * care about other types except insofar as they have to recurse through
      * non-primitive node types.  Therefore, we now provide generic tree-walking
      * logic to consolidate the redundant "boilerplate" code.  There are
      * two versions: expression_tree_walker() and expression_tree_mutator().
      */
     
     /*
      * expression_tree_walker() is designed to support routines that traverse
      * a tree in a read-only fashion (although it will also work for routines
      * that modify nodes in-place but never add/delete/replace nodes).
      * A walker routine should look like this:
      *
      * bool my_walker (Node *node, my_struct *context)
      * {
      *      if (node == NULL)
      *          return false;
      *      // check for nodes that special work is required for, eg:
      *      if (IsA(node, Var))
      *      {
      *          ... do special actions for Var nodes
      *      }
      *      else if (IsA(node, ...))
      *      {
      *          ... do special actions for other node types
      *      }
      *      // for any node type not specially processed, do:
      *      return expression_tree_walker(node, my_walker, (void *) context);
      * }
      *
      * The "context" argument points to a struct that holds whatever context
      * information the walker routine needs --- it can be used to return data
      * gathered by the walker, too.  This argument is not touched by
      * expression_tree_walker, but it is passed down to recursive sub-invocations
      * of my_walker.  The tree walk is started from a setup routine that
      * fills in the appropriate context struct, calls my_walker with the top-level
      * node of the tree, and then examines the results.
      *
      * The walker routine should return "false" to continue the tree walk, or
      * "true" to abort the walk and immediately return "true" to the top-level
      * caller.  This can be used to short-circuit the traversal if the walker
      * has found what it came for.  "false" is returned to the top-level caller
      * iff no invocation of the walker returned "true".
      *
      * The node types handled by expression_tree_walker include all those
      * normally found in target lists and qualifier clauses during the planning
      * stage.  In particular, it handles List nodes since a cnf-ified qual clause
      * will have List structure at the top level, and it handles TargetEntry nodes
      * so that a scan of a target list can be handled without additional code.
      * Also, RangeTblRef, FromExpr, JoinExpr, and SetOperationStmt nodes are
      * handled, so that query jointrees and setOperation trees can be processed
      * without additional code.
      *
      * expression_tree_walker will handle SubLink nodes by recursing normally
      * into the "testexpr" subtree (which is an expression belonging to the outer
      * plan).  It will also call the walker on the sub-Query node; however, when
      * expression_tree_walker itself is called on a Query node, it does nothing
      * and returns "false".  The net effect is that unless the walker does
      * something special at a Query node, sub-selects will not be visited during
      * an expression tree walk. This is exactly the behavior wanted in many cases
      * --- and for those walkers that do want to recurse into sub-selects, special
      * behavior is typically needed anyway at the entry to a sub-select (such as
      * incrementing a depth counter). A walker that wants to examine sub-selects
      * should include code along the lines of:
      *
      *      if (IsA(node, Query))
      *      {
      *          adjust context for subquery;
      *          result = query_tree_walker((Query *) node, my_walker, context,
      *                                     0); // adjust flags as needed
      *          restore context if needed;
      *          return result;
      *      }
      *
      * query_tree_walker is a convenience routine (see below) that calls the
      * walker on all the expression subtrees of the given Query node.
      *
      * expression_tree_walker will handle SubPlan nodes by recursing normally
      * into the "testexpr" and the "args" list (which are expressions belonging to
      * the outer plan).  It will not touch the completed subplan, however.  Since
      * there is no link to the original Query, it is not possible to recurse into
      * subselects of an already-planned expression tree.  This is OK for current
      * uses, but may need to be revisited in future.
      */
     
     bool
     expression_tree_walker(Node *node,
                            bool (*walker) (),
                            void *context)
     {
         ListCell   *temp;
     
         /*
          * The walker has already visited the current node, and so we need only
          * recurse into any sub-nodes it has.
          *
          * We assume that the walker is not interested in List nodes per se, so
          * when we expect a List we just recurse directly to self without
          * bothering to call the walker.
          */
         if (node == NULL)
             return false;
     
         /* Guard against stack overflow due to overly complex expressions */
         check_stack_depth();
     
         switch (nodeTag(node))
         {
             case T_Var:
             case T_Const:
             case T_Param:
             case T_CaseTestExpr:
             case T_SQLValueFunction:
             case T_CoerceToDomainValue:
             case T_SetToDefault:
             case T_CurrentOfExpr:
             case T_NextValueExpr:
             case T_RangeTblRef:
             case T_SortGroupClause:
                 /* primitive node types with no expression subnodes */
                 break;
             case T_WithCheckOption:
                 return walker(((WithCheckOption *) node)->qual, context);
             case T_Aggref:
                 {
                     Aggref     *expr = (Aggref *) node;
     
                     /* recurse directly on List */
                     if (expression_tree_walker((Node *) expr->aggdirectargs,
                                                walker, context))
                         return true;
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                     if (expression_tree_walker((Node *) expr->aggorder,
                                                walker, context))
                         return true;
                     if (expression_tree_walker((Node *) expr->aggdistinct,
                                                walker, context))
                         return true;
                     if (walker((Node *) expr->aggfilter, context))
                         return true;
                 }
                 break;
             case T_GroupingFunc:
                 {
                     GroupingFunc *grouping = (GroupingFunc *) node;
     
                     if (expression_tree_walker((Node *) grouping->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_WindowFunc:
                 {
                     WindowFunc *expr = (WindowFunc *) node;
     
                     /* recurse directly on List */
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                     if (walker((Node *) expr->aggfilter, context))
                         return true;
                 }
                 break;
             case T_ArrayRef:
                 {
                     ArrayRef   *aref = (ArrayRef *) node;
     
                     /* recurse directly for upper/lower array index lists */
                     if (expression_tree_walker((Node *) aref->refupperindexpr,
                                                walker, context))
                         return true;
                     if (expression_tree_walker((Node *) aref->reflowerindexpr,
                                                walker, context))
                         return true;
                     /* walker must see the refexpr and refassgnexpr, however */
                     if (walker(aref->refexpr, context))
                         return true;
                     if (walker(aref->refassgnexpr, context))
                         return true;
                 }
                 break;
             case T_FuncExpr:
                 {
                     FuncExpr   *expr = (FuncExpr *) node;
     
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_NamedArgExpr:
                 return walker(((NamedArgExpr *) node)->arg, context);
             case T_OpExpr:
             case T_DistinctExpr:    /* struct-equivalent to OpExpr */
             case T_NullIfExpr:      /* struct-equivalent to OpExpr */
                 {
                     OpExpr     *expr = (OpExpr *) node;
     
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_ScalarArrayOpExpr:
                 {
                     ScalarArrayOpExpr *expr = (ScalarArrayOpExpr *) node;
     
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_BoolExpr:
                 {
                     BoolExpr   *expr = (BoolExpr *) node;
     
                     if (expression_tree_walker((Node *) expr->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_SubLink:
                 {
                     SubLink    *sublink = (SubLink *) node;
     
                     if (walker(sublink->testexpr, context))
                         return true;
     
                     /*
                      * Also invoke the walker on the sublink's Query node, so it
                      * can recurse into the sub-query if it wants to.
                      */
                     return walker(sublink->subselect, context);
                 }
                 break;
             case T_SubPlan:
                 {
                     SubPlan    *subplan = (SubPlan *) node;
     
                     /* recurse into the testexpr, but not into the Plan */
                     if (walker(subplan->testexpr, context))
                         return true;
                     /* also examine args list */
                     if (expression_tree_walker((Node *) subplan->args,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_AlternativeSubPlan:
                 return walker(((AlternativeSubPlan *) node)->subplans, context);
             case T_FieldSelect:
                 return walker(((FieldSelect *) node)->arg, context);
             case T_FieldStore:
                 {
                     FieldStore *fstore = (FieldStore *) node;
     
                     if (walker(fstore->arg, context))
                         return true;
                     if (walker(fstore->newvals, context))
                         return true;
                 }
                 break;
             case T_RelabelType:
                 return walker(((RelabelType *) node)->arg, context);
             case T_CoerceViaIO:
                 return walker(((CoerceViaIO *) node)->arg, context);
             case T_ArrayCoerceExpr:
                 {
                     ArrayCoerceExpr *acoerce = (ArrayCoerceExpr *) node;
     
                     if (walker(acoerce->arg, context))
                         return true;
                     if (walker(acoerce->elemexpr, context))
                         return true;
                 }
                 break;
             case T_ConvertRowtypeExpr:
                 return walker(((ConvertRowtypeExpr *) node)->arg, context);
             case T_CollateExpr:
                 return walker(((CollateExpr *) node)->arg, context);
             case T_CaseExpr:
                 {
                     CaseExpr   *caseexpr = (CaseExpr *) node;
     
                     if (walker(caseexpr->arg, context))
                         return true;
                     /* we assume walker doesn't care about CaseWhens, either */
                     foreach(temp, caseexpr->args)
                     {
                         CaseWhen   *when = lfirst_node(CaseWhen, temp);
     
                         if (walker(when->expr, context))
                             return true;
                         if (walker(when->result, context))
                             return true;
                     }
                     if (walker(caseexpr->defresult, context))
                         return true;
                 }
                 break;
             case T_ArrayExpr:
                 return walker(((ArrayExpr *) node)->elements, context);
             case T_RowExpr:
                 /* Assume colnames isn't interesting */
                 return walker(((RowExpr *) node)->args, context);
             case T_RowCompareExpr:
                 {
                     RowCompareExpr *rcexpr = (RowCompareExpr *) node;
     
                     if (walker(rcexpr->largs, context))
                         return true;
                     if (walker(rcexpr->rargs, context))
                         return true;
                 }
                 break;
             case T_CoalesceExpr:
                 return walker(((CoalesceExpr *) node)->args, context);
             case T_MinMaxExpr:
                 return walker(((MinMaxExpr *) node)->args, context);
             case T_XmlExpr:
                 {
                     XmlExpr    *xexpr = (XmlExpr *) node;
     
                     if (walker(xexpr->named_args, context))
                         return true;
                     /* we assume walker doesn't care about arg_names */
                     if (walker(xexpr->args, context))
                         return true;
                 }
                 break;
             case T_NullTest:
                 return walker(((NullTest *) node)->arg, context);
             case T_BooleanTest:
                 return walker(((BooleanTest *) node)->arg, context);
             case T_CoerceToDomain:
                 return walker(((CoerceToDomain *) node)->arg, context);
             case T_TargetEntry:
                 return walker(((TargetEntry *) node)->expr, context);
             case T_Query:
                 /* Do nothing with a sub-Query, per discussion above */
                 break;
             case T_WindowClause:
                 {
                     WindowClause *wc = (WindowClause *) node;
     
                     if (walker(wc->partitionClause, context))
                         return true;
                     if (walker(wc->orderClause, context))
                         return true;
                     if (walker(wc->startOffset, context))
                         return true;
                     if (walker(wc->endOffset, context))
                         return true;
                 }
                 break;
             case T_CommonTableExpr:
                 {
                     CommonTableExpr *cte = (CommonTableExpr *) node;
     
                     /*
                      * Invoke the walker on the CTE's Query node, so it can
                      * recurse into the sub-query if it wants to.
                      */
                     return walker(cte->ctequery, context);
                 }
                 break;
             case T_List:
                 foreach(temp, (List *) node)
                 {
                     if (walker((Node *) lfirst(temp), context))
                         return true;
                 }
                 break;
             case T_FromExpr:
                 {
                     FromExpr   *from = (FromExpr *) node;
     
                     if (walker(from->fromlist, context))
                         return true;
                     if (walker(from->quals, context))
                         return true;
                 }
                 break;
             case T_OnConflictExpr:
                 {
                     OnConflictExpr *onconflict = (OnConflictExpr *) node;
     
                     if (walker((Node *) onconflict->arbiterElems, context))
                         return true;
                     if (walker(onconflict->arbiterWhere, context))
                         return true;
                     if (walker(onconflict->onConflictSet, context))
                         return true;
                     if (walker(onconflict->onConflictWhere, context))
                         return true;
                     if (walker(onconflict->exclRelTlist, context))
                         return true;
                 }
                 break;
             case T_PartitionPruneStepOp:
                 {
                     PartitionPruneStepOp *opstep = (PartitionPruneStepOp *) node;
     
                     if (walker((Node *) opstep->exprs, context))
                         return true;
                 }
                 break;
             case T_PartitionPruneStepCombine:
                 /* no expression subnodes */
                 break;
             case T_JoinExpr:
                 {
                     JoinExpr   *join = (JoinExpr *) node;
     
                     if (walker(join->larg, context))
                         return true;
                     if (walker(join->rarg, context))
                         return true;
                     if (walker(join->quals, context))
                         return true;
     
                     /*
                      * alias clause, using list are deemed uninteresting.
                      */
                 }
                 break;
             case T_SetOperationStmt:
                 {
                     SetOperationStmt *setop = (SetOperationStmt *) node;
     
                     if (walker(setop->larg, context))
                         return true;
                     if (walker(setop->rarg, context))
                         return true;
     
                     /* groupClauses are deemed uninteresting */
                 }
                 break;
             case T_PlaceHolderVar:
                 return walker(((PlaceHolderVar *) node)->phexpr, context);
             case T_InferenceElem:
                 return walker(((InferenceElem *) node)->expr, context);
             case T_AppendRelInfo:
                 {
                     AppendRelInfo *appinfo = (AppendRelInfo *) node;
     
                     if (expression_tree_walker((Node *) appinfo->translated_vars,
                                                walker, context))
                         return true;
                 }
                 break;
             case T_PlaceHolderInfo:
                 return walker(((PlaceHolderInfo *) node)->ph_var, context);
             case T_RangeTblFunction:
                 return walker(((RangeTblFunction *) node)->funcexpr, context);
             case T_TableSampleClause:
                 {
                     TableSampleClause *tsc = (TableSampleClause *) node;
     
                     if (expression_tree_walker((Node *) tsc->args,
                                                walker, context))
                         return true;
                     if (walker((Node *) tsc->repeatable, context))
                         return true;
                 }
                 break;
             case T_TableFunc:
                 {
                     TableFunc  *tf = (TableFunc *) node;
     
                     if (walker(tf->ns_uris, context))
                         return true;
                     if (walker(tf->docexpr, context))
                         return true;
                     if (walker(tf->rowexpr, context))
                         return true;
                     if (walker(tf->colexprs, context))
                         return true;
                     if (walker(tf->coldefexprs, context))
                         return true;
                 }
                 break;
             default:
                 elog(ERROR, "unrecognized node type: %d",
                      (int) nodeTag(node));
                 break;
         }
         return false;
     }
    
    

### 三、跟踪分析

测试脚本:

    
    
    select *                
    from t_dwxx a
    where exists (select b.dwbh from t_grxx b where a.dwbh = b.dwbh);
    

gdb分析:

    
    
    (gdb) b convert_EXISTS_sublink_to_join
    Breakpoint 1 at 0x77a4fe: file subselect.c, line 1426.
    (gdb) c
    Continuing.
    
    Breakpoint 1, convert_EXISTS_sublink_to_join (root=0x22de828, sublink=0x22292a0, under_not=false, available_rels=0x22defa8)
        at subselect.c:1426
    1426        Query      *parse = root->parse;
    (gdb) 
    #1.root见上一节
    #2.sublink,子链接,见本节查询树结构
    #3.under_not,false,表示EXISTS
    #4.available_rels,可用的rels
    (gdb) p *available_rels
    $1 = {nwords = 1, words = 0x22defac}
    (gdb) p available_rels->words[0]
    $3 = 2
    (gdb) 
    ...
    1511        rtoffset = list_length(parse->rtable);
    (gdb) 
    1512        OffsetVarNodes((Node *) subselect, rtoffset, 0);
    (gdb) p rtoffset
    $6 = 1
    (gdb) step
    OffsetVarNodes (node=0x22dee48, offset=1, sublevels_up=0) at rewriteManip.c:428
    428     context.offset = offset;
    ...
    #调整subselect->jointree->fromlist->head(类型为RTR)的rtindex,原为1,调整为2
    (gdb) p *node
    $20 = {type = T_RangeTblRef}
    (gdb) p *(RangeTblRef *)node
    $21 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) n
    368             rtr->rtindex += context->offset;
    (gdb) 
    370         return false;
    (gdb) p *(RangeTblRef *)node
    $22 = {type = T_RangeTblRef, rtindex = 2}
    (gdb) 
    1513        OffsetVarNodes(whereClause, rtoffset, 0);
    (gdb) 
    1520        IncrementVarSublevelsUp((Node *) subselect, -1, 1);
    (gdb) p whereClause
    $23 = (Node *) 0x22def58
    (gdb) p *whereClause
    $24 = {type = T_OpExpr}
    (gdb) set $arg1=(RelabelType *)((OpExpr *)whereClause)->args->head->data.ptr_value
    (gdb) p *$arg1->arg
    $32 = {type = T_Var}
    #第1个参数是t_dwxx.dwbh
    (gdb) p *(Var *)$arg1->arg
    $33 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 1, 
      varnoold = 1, varoattno = 2, location = 72}
    (gdb) set $arg2=(RelabelType *)((OpExpr *)whereClause)->args->tail->data.ptr_value
    #第2个参数是t_grxx.dwbh
    #varno/varnoold从原来的值1调整为2
    (gdb) p *(Var *)$arg2->arg
    $34 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 81}
    (gdb) 
    (gdb) n
    1521        IncrementVarSublevelsUp(whereClause, -1, 1);
    (gdb) 
    1529        clause_varnos = pull_varnos(whereClause);
    #调整varlevelsup为0
    (gdb) p *(Var *)$arg1->arg
    $36 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 2, location = 72}
    (gdb) p *(Var *)$arg2->arg
    $37 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 81}
    ...
    #构造返回值
    (gdb)
    1552        result = makeNode(JoinExpr);
    1566        return result;
    (gdb) 
    1567    }
    (gdb) 
    pull_up_sublinks_qual_recurse (root=0x22de828, node=0x22292a0, jtlink1=0x7ffdcabbc918, available_rels1=0x22defa8, 
        jtlink2=0x0, available_rels2=0x0) at prepjointree.c:404
    404                 j->larg = *jtlink1;
    #上拉成为半连接SEMI JOIN
    (gdb) p *(JoinExpr *)jtnode
    $65 = {type = T_JoinExpr, jointype = JOIN_SEMI, isNatural = false, larg = 0x22e7118, rarg = 0x22e7438, usingClause = 0x0, 
      quals = 0x22def58, alias = 0x0, rtindex = 0}
    #DONE!
    

### 四、小结

1、上拉过程：与ANY子链接类似，主要逻辑是调整varno&varlevelsup等信息；  
2、重要函数：XX_walker，遍历Node节点的函数，用于统计或更新Node信息。

