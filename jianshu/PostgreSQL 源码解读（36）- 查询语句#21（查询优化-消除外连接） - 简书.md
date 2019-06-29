本节简单介绍了PG查询优化中对消除外连接的处理过程。  
使用的测试脚本:

    
    
    drop table if exists t_null1;
    create table t_null1(c1 int);
    insert into t_null1 values(1);
    insert into t_null1 values(2);
    insert into t_null1 values(null);
    
    drop table if exists t_null2;
    create table t_null2(c1 int);
    insert into t_null2 values(1);
    insert into t_null2 values(null);
    
    

### 一、基本概念

消除外连接的代码注释说明如下:

    
    
     /*
      * reduce_outer_joins
      *      Attempt to reduce outer joins to plain inner joins.
      *
      * The idea here is that given a query like
      *      SELECT ... FROM a LEFT JOIN b ON (...) WHERE b.y = 42;
      * we can reduce the LEFT JOIN to a plain JOIN if the "=" operator in WHERE
      * is strict.  The strict operator will always return NULL, causing the outer
      * WHERE to fail, on any row where the LEFT JOIN filled in NULLs for b's
      * columns.  Therefore, there's no need for the join to produce null-extended
      * rows in the first place --- which makes it a plain join not an outer join.
      * (This scenario may not be very likely in a query written out by hand, but
      * it's reasonably likely when pushing quals down into complex views.)
      *
      * More generally, an outer join can be reduced in strength if there is a
      * strict qual above it in the qual tree that constrains a Var from the
      * nullable side of the join to be non-null.  (For FULL joins this applies
      * to each side separately.)
      *
      * Another transformation we apply here is to recognize cases like
      *      SELECT ... FROM a LEFT JOIN b ON (a.x = b.y) WHERE b.y IS NULL;
      * If the join clause is strict for b.y, then only null-extended rows could
      * pass the upper WHERE, and we can conclude that what the query is really
      * specifying is an anti-semijoin.  We change the join type from JOIN_LEFT
      * to JOIN_ANTI.  The IS NULL clause then becomes redundant, and must be
      * removed to prevent bogus selectivity calculations, but we leave it to
      * distribute_qual_to_rels to get rid of such clauses.
      *
      * Also, we get rid of JOIN_RIGHT cases by flipping them around to become
      * JOIN_LEFT.  This saves some code here and in some later planner routines,
      * but the main reason to do it is to not need to invent a JOIN_REVERSE_ANTI
      * join type.
      *
      * To ease recognition of strict qual clauses, we require this routine to be
      * run after expression preprocessing (i.e., qual canonicalization and JOIN
      * alias-var expansion).
      */
    

有两种类型的外连接可以被消除,第一种是形如以下形式的语句:  
_SELECT ... FROM a LEFT JOIN b ON (...) WHERE b.y = 42;_  
这种语句如满足条件可变换为内连接(INNER_JOIN).  
之所以可以变换为内连接,那是因为这样的语句与内连接处理的结果是一样的,原因是在Nullable-Side端(需要填充NULL值的一端),存在过滤条件保证这一端不可能是NULL值,比如IS NOT NULL/y = 42这类强(strict)过滤条件.

    
    
    testdb=# explain verbose select * from t_null1 a left join t_null2 b on a.c1 = b.c1;
                                       QUERY PLAN                                   
    --------------------------------------------------------------------------------     Merge Left Join  (cost=359.57..860.00 rows=32512 width=8) -- 外连接
       Output: a.c1, b.c1
       Merge Cond: (a.c1 = b.c1)
       ->  Sort  (cost=179.78..186.16 rows=2550 width=4)
             Output: a.c1
             Sort Key: a.c1
             ->  Seq Scan on public.t_null1 a  (cost=0.00..35.50 rows=2550 width=4)
                   Output: a.c1
       ->  Sort  (cost=179.78..186.16 rows=2550 width=4)
             Output: b.c1
             Sort Key: b.c1
             ->  Seq Scan on public.t_null2 b  (cost=0.00..35.50 rows=2550 width=4)
                   Output: b.c1
    (13 rows)
    
    testdb=# explain verbose select * from t_null1 a left join t_null2 b on a.c1 = b.c1 where b.c1 = 1;
                                      QUERY PLAN                                  
    ------------------------------------------------------------------------------     Nested Loop  (cost=0.00..85.89 rows=169 width=8) -- 外连接(Left关键字)已被消除
       Output: a.c1, b.c1
       ->  Seq Scan on public.t_null1 a  (cost=0.00..41.88 rows=13 width=4)
             Output: a.c1
             Filter: (a.c1 = 1)
       ->  Materialize  (cost=0.00..41.94 rows=13 width=4)
             Output: b.c1
             ->  Seq Scan on public.t_null2 b  (cost=0.00..41.88 rows=13 width=4)
                   Output: b.c1
                   Filter: (b.c1 = 1)
    (10 rows)
    

第二种形如:  
_SELECT ... FROM a LEFT JOIN b ON (a.x = b.y) WHERE b.y IS NULL;_  
这种语句如满足条件可以变换为反半连接(ANTI-SEMIJOIN).  
过滤条件已明确要求Nullable-Side端y IS NULL,如果连接条件是a.x =
b.y这类严格(strict)的条件,那么这样的外连接与反半连接的结果是一样的.

    
    
    testdb=# explain verbose select * from t_null1 a left join t_null2 b on a.c1 = b.c1 where b.c1 is null;
                                       QUERY PLAN                                   
    --------------------------------------------------------------------------------     Hash Anti Join  (cost=67.38..152.44 rows=1275 width=8) -- 变换为反连接
       Output: a.c1, b.c1
       Hash Cond: (a.c1 = b.c1)
       ->  Seq Scan on public.t_null1 a  (cost=0.00..35.50 rows=2550 width=4)
             Output: a.c1
       ->  Hash  (cost=35.50..35.50 rows=2550 width=4)
             Output: b.c1
             ->  Seq Scan on public.t_null2 b  (cost=0.00..35.50 rows=2550 width=4)
                   Output: b.c1
    (9 rows)
    

值得一提的是,在PG中,形如 _SELECT ... FROM a LEFT JOIN b ON (...) WHERE b.y = 42;_
这样的SQL语句, _WHERE b.y = 42_
这类条件可以视为连接的上层过滤条件,在查询树中,Jointree->fromlist(元素类型为JoinExpr)与Jointree->quals处于同一层次,由于JoinExpr中的quals为同层条件,因此其上层即为Jointree->quals.有兴趣的可以查看日志输出查看Query查询树结构.

### 二、源码解读

消除外连接的代码在主函数subquery_planner中,通过调用reduce_outer_joins函数实现,代码片段如下:

    
    
         /*
          * If we have any outer joins, try to reduce them to plain inner joins.
          * This step is most easily done after we've done expression
          * preprocessing.
          */
         if (hasOuterJoins)
             reduce_outer_joins(root);
    
    

**reduce_outer_joins**

    
    
    

相关的数据结构和依赖的子函数:  
**reduce_outer_joins_state**

    
    
     typedef struct reduce_outer_joins_state
     {
         Relids      relids;         /* base relids within this subtree */
         bool        contains_outer; /* does subtree contain outer join(s)? */
         List       *sub_states;     /* List of states for subtree components */
     } reduce_outer_joins_state;
    

**BitmapXX**

    
    
     typedef struct Bitmapset
     {
         int         nwords;         /* number of words in array */
         bitmapword  words[FLEXIBLE_ARRAY_MEMBER];   /* really [nwords] */
     } Bitmapset;
     
     #define WORDNUM(x)  ((x) / BITS_PER_BITMAPWORD)
     #define BITNUM(x)   ((x) % BITS_PER_BITMAPWORD)
     /* The unit size can be adjusted by changing these three declarations: */
     #define BITS_PER_BITMAPWORD 32
     typedef uint32 bitmapword; /* must be an unsigned type */
    
    
     /*
      * bms_make_singleton - build a bitmapset containing a single member
      */
     Bitmapset *
     bms_make_singleton(int x)
     {
         Bitmapset  *result;
         int         wordnum,
                     bitnum;
     
         if (x < 0)
             elog(ERROR, "negative bitmapset member not allowed");
         wordnum = WORDNUM(x);
         bitnum = BITNUM(x);
         result = (Bitmapset *) palloc0(BITMAPSET_SIZE(wordnum + 1));
         result->nwords = wordnum + 1;
         result->words[wordnum] = ((bitmapword) 1 << bitnum);
         return result;
     }
    
    
     /*
      * bms_add_member - add a specified member to set
      *
      * Input set is modified or recycled!
      */
     Bitmapset *
     bms_add_member(Bitmapset *a, int x)
     {
         int         wordnum,
                     bitnum;
     
         if (x < 0)
             elog(ERROR, "negative bitmapset member not allowed");
         if (a == NULL)
             return bms_make_singleton(x);
         wordnum = WORDNUM(x);
         bitnum = BITNUM(x);
     
         /* enlarge the set if necessary */
         if (wordnum >= a->nwords)
         {
             int         oldnwords = a->nwords;
             int         i;
     
             a = (Bitmapset *) repalloc(a, BITMAPSET_SIZE(wordnum + 1));
             a->nwords = wordnum + 1;
             /* zero out the enlarged portion */
             for (i = oldnwords; i < a->nwords; i++)
                 a->words[i] = 0;
         }
     
         a->words[wordnum] |= ((bitmapword) 1 << bitnum);
         return a;
     }
    
    

**find_nonnullable_rels**

    
    
     /*
      * find_nonnullable_rels
      *      Determine which base rels are forced nonnullable by given clause.
      *
      * Returns the set of all Relids that are referenced in the clause in such
      * a way that the clause cannot possibly return TRUE if any of these Relids
      * is an all-NULL row.  (It is OK to err on the side of conservatism; hence
      * the analysis here is simplistic.)
      *
      * The semantics here are subtly different from contain_nonstrict_functions:
      * that function is concerned with NULL results from arbitrary expressions,
      * but here we assume that the input is a Boolean expression, and wish to
      * see if NULL inputs will provably cause a FALSE-or-NULL result.  We expect
      * the expression to have been AND/OR flattened and converted to implicit-AND
      * format.
      *
      * Note: this function is largely duplicative of find_nonnullable_vars().
      * The reason not to simplify this function into a thin wrapper around
      * find_nonnullable_vars() is that the tested conditions really are different:
      * a clause like "t1.v1 IS NOT NULL OR t1.v2 IS NOT NULL" does not prove
      * that either v1 or v2 can't be NULL, but it does prove that the t1 row
      * as a whole can't be all-NULL.
      *
      * top_level is true while scanning top-level AND/OR structure; here, showing
      * the result is either FALSE or NULL is good enough.  top_level is false when
      * we have descended below a NOT or a strict function: now we must be able to
      * prove that the subexpression goes to NULL.
      *
      * We don't use expression_tree_walker here because we don't want to descend
      * through very many kinds of nodes; only the ones we can be sure are strict.
      */
     Relids
     find_nonnullable_rels(Node *clause)
     {
         return find_nonnullable_rels_walker(clause, true);
     }
     
     static Relids
     find_nonnullable_rels_walker(Node *node, bool top_level)
     {
         Relids      result = NULL;
         ListCell   *l;
     
         if (node == NULL)
             return NULL;
         if (IsA(node, Var))
         {
             Var        *var = (Var *) node;
     
             if (var->varlevelsup == 0)
                 result = bms_make_singleton(var->varno);
         }
         else if (IsA(node, List))
         {
             /*
              * At top level, we are examining an implicit-AND list: if any of the
              * arms produces FALSE-or-NULL then the result is FALSE-or-NULL. If
              * not at top level, we are examining the arguments of a strict
              * function: if any of them produce NULL then the result of the
              * function must be NULL.  So in both cases, the set of nonnullable
              * rels is the union of those found in the arms, and we pass down the
              * top_level flag unmodified.
              */
             foreach(l, (List *) node)
             {
                 result = bms_join(result,
                                   find_nonnullable_rels_walker(lfirst(l),
                                                                top_level));
             }
         }
         else if (IsA(node, FuncExpr))
         {
             FuncExpr   *expr = (FuncExpr *) node;
     
             if (func_strict(expr->funcid))
                 result = find_nonnullable_rels_walker((Node *) expr->args, false);
         }
         else if (IsA(node, OpExpr))
         {
             OpExpr     *expr = (OpExpr *) node;
     
             set_opfuncid(expr);
             if (func_strict(expr->opfuncid))
                 result = find_nonnullable_rels_walker((Node *) expr->args, false);
         }
         else if (IsA(node, ScalarArrayOpExpr))
         {
             ScalarArrayOpExpr *expr = (ScalarArrayOpExpr *) node;
     
             if (is_strict_saop(expr, true))
                 result = find_nonnullable_rels_walker((Node *) expr->args, false);
         }
         else if (IsA(node, BoolExpr))
         {
             BoolExpr   *expr = (BoolExpr *) node;
     
             switch (expr->boolop)
             {
                 case AND_EXPR:
                     /* At top level we can just recurse (to the List case) */
                     if (top_level)
                     {
                         result = find_nonnullable_rels_walker((Node *) expr->args,
                                                               top_level);
                         break;
                     }
     
                     /*
                      * Below top level, even if one arm produces NULL, the result
                      * could be FALSE (hence not NULL).  However, if *all* the
                      * arms produce NULL then the result is NULL, so we can take
                      * the intersection of the sets of nonnullable rels, just as
                      * for OR.  Fall through to share code.
                      */
                     /* FALL THRU */
                 case OR_EXPR:
     
                     /*
                      * OR is strict if all of its arms are, so we can take the
                      * intersection of the sets of nonnullable rels for each arm.
                      * This works for both values of top_level.
                      */
                     foreach(l, expr->args)
                     {
                         Relids      subresult;
     
                         subresult = find_nonnullable_rels_walker(lfirst(l),
                                                                  top_level);
                         if (result == NULL) /* first subresult? */
                             result = subresult;
                         else
                             result = bms_int_members(result, subresult);
     
                         /*
                          * If the intersection is empty, we can stop looking. This
                          * also justifies the test for first-subresult above.
                          */
                         if (bms_is_empty(result))
                             break;
                     }
                     break;
                 case NOT_EXPR:
                     /* NOT will return null if its arg is null */
                     result = find_nonnullable_rels_walker((Node *) expr->args,
                                                           false);
                     break;
                 default:
                     elog(ERROR, "unrecognized boolop: %d", (int) expr->boolop);
                     break;
             }
         }
         else if (IsA(node, RelabelType))
         {
             RelabelType *expr = (RelabelType *) node;
     
             result = find_nonnullable_rels_walker((Node *) expr->arg, top_level);
         }
         else if (IsA(node, CoerceViaIO))
         {
             /* not clear this is useful, but it can't hurt */
             CoerceViaIO *expr = (CoerceViaIO *) node;
     
             result = find_nonnullable_rels_walker((Node *) expr->arg, top_level);
         }
         else if (IsA(node, ArrayCoerceExpr))
         {
             /* ArrayCoerceExpr is strict at the array level; ignore elemexpr */
             ArrayCoerceExpr *expr = (ArrayCoerceExpr *) node;
     
             result = find_nonnullable_rels_walker((Node *) expr->arg, top_level);
         }
         else if (IsA(node, ConvertRowtypeExpr))
         {
             /* not clear this is useful, but it can't hurt */
             ConvertRowtypeExpr *expr = (ConvertRowtypeExpr *) node;
     
             result = find_nonnullable_rels_walker((Node *) expr->arg, top_level);
         }
         else if (IsA(node, CollateExpr))
         {
             CollateExpr *expr = (CollateExpr *) node;
     
             result = find_nonnullable_rels_walker((Node *) expr->arg, top_level);
         }
         else if (IsA(node, NullTest))
         {
             /* IS NOT NULL can be considered strict, but only at top level */
             NullTest   *expr = (NullTest *) node;
     
             if (top_level && expr->nulltesttype == IS_NOT_NULL && !expr->argisrow)
                 result = find_nonnullable_rels_walker((Node *) expr->arg, false);
         }
         else if (IsA(node, BooleanTest))
         {
             /* Boolean tests that reject NULL are strict at top level */
             BooleanTest *expr = (BooleanTest *) node;
     
             if (top_level &&
                 (expr->booltesttype == IS_TRUE ||
                  expr->booltesttype == IS_FALSE ||
                  expr->booltesttype == IS_NOT_UNKNOWN))
                 result = find_nonnullable_rels_walker((Node *) expr->arg, false);
         }
         else if (IsA(node, PlaceHolderVar))
         {
             PlaceHolderVar *phv = (PlaceHolderVar *) node;
     
             result = find_nonnullable_rels_walker((Node *) phv->phexpr, top_level);
         }
         return result;
     }
    

### 三、跟踪分析

**外连接- >内连接**  
测试脚本:

    
    
    select * 
    from t_null1 a left join t_null2 b on a.c1 = b.c1 
    where b.c1 = 1;
    

gdb跟踪:

    
    
    (gdb) b reduce_outer_joins
    Breakpoint 1 at 0x77faf5: file prepjointree.c, line 2484.
    (gdb) c
    Continuing.
    
    Breakpoint 1, reduce_outer_joins (root=0x2eafa98) at prepjointree.c:2484
    2484    state = reduce_outer_joins_pass1((Node *) root->parse->jointree);
    
    (gdb) b reduce_outer_joins
    Breakpoint 1 at 0x77faf5: file prepjointree.c, line 2484.
    ...
    #进入reduce_outer_joins_pass1
    (gdb) step
    reduce_outer_joins_pass1 (jtnode=0x2ea4af8) at prepjointree.c:2504
    ...
    (gdb) p *f
    $2 = {type = T_FromExpr, fromlist = 0x2ea4668, quals = 0x2edcae8}
    #进入FromExpr分支
    (gdb) n
    2527        sub_state = reduce_outer_joins_pass1(lfirst(l));
    ##递归调用
    (gdb) step
    reduce_outer_joins_pass1 (jtnode=0x2dd40f0) at prepjointree.c:2504
    2504    result = (reduce_outer_joins_state *)
    ##进入JoinExpr分支
    2534    else if (IsA(jtnode, JoinExpr))
    (gdb) 
    2536      JoinExpr   *j = (JoinExpr *) jtnode;
    (gdb) p *j
    $3 = {type = T_JoinExpr, jointype = JOIN_LEFT, isNatural = false, larg = 0x2dd49e0, rarg = 0x2dd4c68, usingClause = 0x0, 
      quals = 0x2edc828, alias = 0x0, rtindex = 3}
    ###递归调用
    2543      sub_state = reduce_outer_joins_pass1(j->larg);
    (gdb) step
    reduce_outer_joins_pass1 (jtnode=0x2dd49e0) at prepjointree.c:2504
    2504    result = (reduce_outer_joins_state *)
    ###进入RangeTblRef分支
    2512    if (IsA(jtnode, RangeTblRef))
    (gdb) 
    2514      int     varno = ((RangeTblRef *) jtnode)->rtindex;
    ...
    ###JoinExpr处理完毕
    2549      sub_state = reduce_outer_joins_pass1(j->rarg);
    (gdb) 
    2551                       sub_state->relids);
    (gdb) 
    2550      result->relids = bms_add_members(result->relids,
    (gdb) 
    2552      result->contains_outer |= sub_state->contains_outer;
    (gdb) 
    2553      result->sub_states = lappend(result->sub_states, sub_state);
    (gdb) 
    2558    return result;
    (gdb) p *result
    $12 = {relids = 0x2ea61e8, contains_outer = true, sub_states = 0x2edcbc8}
    (gdb) p *result->relids
    $13 = {nwords = 1, words = 0x2ea61ec}
    (gdb) p *result->sub_states
    $14 = {type = T_List, length = 2, head = 0x2edcba8, tail = 0x2edcc40}
    (gdb) n
    2559  }
    (gdb) 
    ##回到FromExpr
    ...
    2523      foreach(l, f->fromlist)
    (gdb) p *result
    $18 = {relids = 0x2edcc60, contains_outer = true, sub_states = 0x2edcc98}
    #回到reduce_outer_joins
    reduce_outer_joins (root=0x2eafa98) at prepjointree.c:2487
    2487    if (state == NULL || !state->contains_outer)
    (gdb) p *state
    $21 = {relids = 0x2edcc60, contains_outer = true, sub_states = 0x2edcc98}
    (gdb) p *state->relids[0]->words
    $30 = 6 -->Relids,1 & 2,即1<<1 | 1 << 2
    #进入reduce_outer_joins_pass2
    (gdb) step
    reduce_outer_joins_pass2 (jtnode=0x2ea4af8, state=0x2edcb18, root=0x2eafa98, nonnullable_rels=0x0, nonnullable_vars=0x0, 
        forced_null_vars=0x0) at prepjointree.c:2583
    2583    if (jtnode == NULL)
    ...
    #进入FromExpr分支
    2587    else if (IsA(jtnode, FromExpr))
    (gdb) 
    2589      FromExpr   *f = (FromExpr *) jtnode;
    ...
    #寻找FromExpr中存在过滤条件为NOT NULL的Relids
    (gdb) n
    2597      pass_nonnullable_rels = find_nonnullable_rels(f->quals);
    (gdb) 
    2598      pass_nonnullable_rels = bms_add_members(pass_nonnullable_rels,
    (gdb) p pass_nonnullable_rels->words[0]
    $34 = 4 -- rtindex = 2的Relid
    #寻找NOT NULL's Vars
    2601      pass_nonnullable_vars = find_nonnullable_vars(f->quals);
    (gdb) 
    2602      pass_nonnullable_vars = list_concat(pass_nonnullable_vars,
    (gdb) 
    2604      pass_forced_null_vars = find_forced_null_vars(f->quals);
    (gdb) p *pass_nonnullable_vars
    $35 = {type = T_List, length = 1, head = 0x2edcce0, tail = 0x2edcce0}
    (gdb) p *(Node *)pass_nonnullable_vars->head->data.ptr_value
    $36 = {type = T_Var}
    (gdb) p *(Var *)pass_nonnullable_vars->head->data.ptr_value
    $37 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 65} -- rtindex=2的RTE,属性编号为1的字段
    ##递归调用reduce_outer_joins_pass2
    (gdb) 
    2614          reduce_outer_joins_pass2(lfirst(l), sub_state, root,
    (gdb) step
    reduce_outer_joins_pass2 (jtnode=0x2dd40f0, state=0x2edcb48, root=0x2eafa98, nonnullable_rels=0x2edccc8, 
        nonnullable_vars=0x2edcd00, forced_null_vars=0x0) at prepjointree.c:2583
    2583    if (jtnode == NULL)
    ##进入JoinExpr分支
    2622    else if (IsA(jtnode, JoinExpr))
    (gdb) 
    2624      JoinExpr   *j = (JoinExpr *) jtnode;
    ...
    (gdb) p *right_state->relids[0]->words
    $44 = 4 -->2号RTE
    (gdb) p *left_state->relids[0]->words
    $45 = 2 -->1号RTE
    (gdb) p rtindex
    $46 = 3 -->Jointree整体作为3号RTE存在
    ...
    2633      switch (jointype)
    (gdb) 
    2638          if (bms_overlap(nonnullable_rels, right_state->relids))
    (gdb) p *nonnullable_rels->words
    $49 = 4 -->2号RTE
    (gdb) p right_state->relids->words[0]
    $50 = 4 -->2号RTE
    ##转换为内连接
    (gdb) n
    2639            jointype = JOIN_INNER;
    ...
    ##修改RTE的连接类型
    (gdb) 
    2724      if (rtindex && jointype != j->jointype)
    (gdb)
    2726        RangeTblEntry *rte = rt_fetch(rtindex, root->parse->rtable);
    (gdb) 
    2730        rte->jointype = jointype;
    (gdb) p *rte
    $54 = {type = T_RangeTblEntry, rtekind = RTE_JOIN, relid = 0, relkind = 0 '\000', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_LEFT, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x2ea4498, lateral = false, 
      inh = false, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) n
    2732      j->jointype = jointype;
    ...
    #回到上层的reduce_outer_joins_pass2
    (gdb) 
    reduce_outer_joins_pass2 (jtnode=0x2ea4af8, state=0x2edcb18, root=0x2eafa98, nonnullable_rels=0x0, nonnullable_vars=0x0, 
        forced_null_vars=0x0) at prepjointree.c:2609
    2609      forboth(l, f->fromlist, s, state->sub_states)
    #回到主函数reduce_outer_joins
    2844  }
    (gdb) 
    reduce_outer_joins (root=0x2eafa98) at prepjointree.c:2492
    2492  }
    (gdb) 
    #DONE!
    

**外连接- >反连接**  
测试脚本:

    
    
    select * 
    from t_null1 a left join t_null2 b on a.c1 = b.c1 
    where b.c1 is null;
    

gdb跟踪,与转换为内连接不同的地方在于reduce_outer_joins_pass2函数中find_forced_null_vars在这里是可以找到相应的Vars的:

    
    
    (gdb) b prepjointree.c:2702
    Breakpoint 3 at 0x78023c: file prepjointree.c, line 2702.
    (gdb) c
    Continuing.
    
    Breakpoint 3, reduce_outer_joins_pass2 (jtnode=0x2dd40f0, state=0x2edc9b8, root=0x2eafa98, nonnullable_rels=0x0, 
        nonnullable_vars=0x0, forced_null_vars=0x2edcb70) at prepjointree.c:2703
    2703      if (jointype == JOIN_LEFT)
    #尝试转换为内连接,但不成功,仍为左连接
    2707        local_nonnullable_vars = find_nonnullable_vars(j->quals);
    (gdb) 
    2708        computed_local_nonnullable_vars = true;
    (gdb) p *local_nonnullable_vars
    $56 = {type = T_List, length = 2, head = 0x2edcba0, tail = 0x2edcbf0}
    (gdb) p *(Var *)local_nonnullable_vars->head->data.ptr_value
    $57 = {xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 1, varoattno = 1, location = 47}
    (gdb) p *(Var *)local_nonnullable_vars->head->next->data.ptr_value
    $58 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 54}
    (gdb) p *(Var *)forced_null_vars->head->data.ptr_value
    $61 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 65}
    (gdb) p *(Var *)overlap->head->data.ptr_value
    $63 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 2, varoattno = 1, location = 54}
    ...
    #转换为反连接
    2717        if (overlap != NIL &&
    (gdb) 
    2720          jointype = JOIN_ANTI;
    (gdb) c
    Continuing.
    

### 四、参考资料

[prepjointree.c](https://doxygen.postgresql.org/prepjointree_8c_source.html)

