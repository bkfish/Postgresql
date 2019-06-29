先前的章节已介绍了函数query_planner中子函数query_planner中qp_callback（回调函数）和fix_placeholder_input_needed_levels的主要实现逻辑,本节继续介绍remove_useless_joins、reduce_unique_semijoins和add_placeholders_to_base_rels的实现逻辑。

query_planner代码片段:

    
    
         //...
     
         /*
          * Remove any useless outer joins.  Ideally this would be done during
          * jointree preprocessing, but the necessary information isn't available
          * until we've built baserel data structures and classified qual clauses.
          */
         joinlist = remove_useless_joins(root, joinlist);//清除无用的外连接
     
         /*
          * Also, reduce any semijoins with unique inner rels to plain inner joins.
          * Likewise, this can't be done until now for lack of needed info.
          */
         reduce_unique_semijoins(root);//消除半连接
     
         /*
          * Now distribute "placeholders" to base rels as needed.  This has to be
          * done after join removal because removal could change whether a
          * placeholder is evaluable at a base rel.
          */
         add_placeholders_to_base_rels(root);//在"base rels"中添加PH
     
          //...
    

### 一、数据结构

**PlaceHolderVar**  
上一小节已介绍过PHInfo

    
    
     /*
      * Placeholder node for an expression to be evaluated below the top level
      * of a plan tree.  This is used during planning to represent the contained
      * expression.  At the end of the planning process it is replaced by either
      * the contained expression or a Var referring to a lower-level evaluation of
      * the contained expression.  Typically the evaluation occurs below an outer
      * join, and Var references above the outer join might thereby yield NULL
      * instead of the expression value.
      *
      * Although the planner treats this as an expression node type, it is not
      * recognized by the parser or executor, so we declare it here rather than
      * in primnodes.h.
      */
     
     typedef struct PlaceHolderVar
     {
         Expr        xpr;
         Expr       *phexpr;         /* the represented expression */
         Relids      phrels;         /* base relids syntactically within expr src */
         Index       phid;           /* ID for PHV (unique within planner run) */
         Index       phlevelsup;     /* > 0 if PHV belongs to outer query */
     } PlaceHolderVar;
    

**SpecialJoinInfo**

    
    
     /*
      * "Special join" info.
      *
      * One-sided outer joins constrain the order of joining partially but not
      * completely.  We flatten such joins into the planner's top-level list of
      * relations to join, but record information about each outer join in a
      * SpecialJoinInfo struct.  These structs are kept in the PlannerInfo node's
      * join_info_list.
      *
      * Similarly, semijoins and antijoins created by flattening IN (subselect)
      * and EXISTS(subselect) clauses create partial constraints on join order.
      * These are likewise recorded in SpecialJoinInfo structs.
      *
      * We make SpecialJoinInfos for FULL JOINs even though there is no flexibility
      * of planning for them, because this simplifies make_join_rel()'s API.
      *
      * min_lefthand and min_righthand are the sets of base relids that must be
      * available on each side when performing the special join.  lhs_strict is
      * true if the special join's condition cannot succeed when the LHS variables
      * are all NULL (this means that an outer join can commute with upper-level
      * outer joins even if it appears in their RHS).  We don't bother to set
      * lhs_strict for FULL JOINs, however.
      *
      * It is not valid for either min_lefthand or min_righthand to be empty sets;
      * if they were, this would break the logic that enforces join order.
      *
      * syn_lefthand and syn_righthand are the sets of base relids that are
      * syntactically below this special join.  (These are needed to help compute
      * min_lefthand and min_righthand for higher joins.)
      *
      * delay_upper_joins is set true if we detect a pushed-down clause that has
      * to be evaluated after this join is formed (because it references the RHS).
      * Any outer joins that have such a clause and this join in their RHS cannot
      * commute with this join, because that would leave noplace to check the
      * pushed-down clause.  (We don't track this for FULL JOINs, either.)
      *
      * For a semijoin, we also extract the join operators and their RHS arguments
      * and set semi_operators, semi_rhs_exprs, semi_can_btree, and semi_can_hash.
      * This is done in support of possibly unique-ifying the RHS, so we don't
      * bother unless at least one of semi_can_btree and semi_can_hash can be set
      * true.  (You might expect that this information would be computed during
      * join planning; but it's helpful to have it available during planning of
      * parameterized table scans, so we store it in the SpecialJoinInfo structs.)
      *
      * jointype is never JOIN_RIGHT; a RIGHT JOIN is handled by switching
      * the inputs to make it a LEFT JOIN.  So the allowed values of jointype
      * in a join_info_list member are only LEFT, FULL, SEMI, or ANTI.
      *
      * For purposes of join selectivity estimation, we create transient
      * SpecialJoinInfo structures for regular inner joins; so it is possible
      * to have jointype == JOIN_INNER in such a structure, even though this is
      * not allowed within join_info_list.  We also create transient
      * SpecialJoinInfos with jointype == JOIN_INNER for outer joins, since for
      * cost estimation purposes it is sometimes useful to know the join size under
      * plain innerjoin semantics.  Note that lhs_strict, delay_upper_joins, and
      * of course the semi_xxx fields are not set meaningfully within such structs.
      */
     
     typedef struct SpecialJoinInfo
     {
         NodeTag     type;
         Relids      min_lefthand;   /* base relids in minimum LHS for join */
         Relids      min_righthand;  /* base relids in minimum RHS for join */
         Relids      syn_lefthand;   /* base relids syntactically within LHS */
         Relids      syn_righthand;  /* base relids syntactically within RHS */
         JoinType    jointype;       /* always INNER, LEFT, FULL, SEMI, or ANTI */
         bool        lhs_strict;     /* joinclause is strict for some LHS rel */
         bool        delay_upper_joins;  /* can't commute with upper RHS */
         /* Remaining fields are set only for JOIN_SEMI jointype: */
         bool        semi_can_btree; /* true if semi_operators are all btree */
         bool        semi_can_hash;  /* true if semi_operators are all hash */
         List       *semi_operators; /* OIDs of equality join operators */
         List       *semi_rhs_exprs; /* righthand-side expressions of these ops */
     } SpecialJoinInfo;
    

### 二、源码解读

**remove_useless_joins**  
清除无用的连接,比如以下的SQL语句:

    
    
    select t1.dwbh 
    from t_grxx t1 left join t_dwxx t2 on t1.dwbh = t2.dwbh;
    

左连接,而且t_dwxx.dwbh唯一,这样的连接是不需要的连接,直接查询t_grxx即可.  
从执行计划来看,PG只对t_grxx进行扫描:

    
    
    testdb=# explain verbose select t1.dwbh from t_grxx t1 left join t_dwxx t2 on t1.dwbh = t2.dwbh;
                                 QUERY PLAN                             
    --------------------------------------------------------------------     Seq Scan on public.t_grxx t1  (cost=0.00..14.00 rows=400 width=38)
       Output: t1.dwbh
    (2 rows)
    

源代码如下:

    
    
     /*
      * remove_useless_joins
      *      Check for relations that don't actually need to be joined at all,
      *      and remove them from the query.
      *
      * We are passed the current joinlist and return the updated list.  Other
      * data structures that have to be updated are accessible via "root".
      */
     List *
     remove_useless_joins(PlannerInfo *root, List *joinlist)
     {
         ListCell   *lc;
     
         /*
          * We are only interested in relations that are left-joined to, so we can
          * scan the join_info_list to find them easily.
          */
     restart:
         foreach(lc, root->join_info_list)//遍历连接信息链表
         {
             SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(lc);
             int         innerrelid;
             int         nremoved;
     
             /* Skip if not removable */
             if (!join_is_removable(root, sjinfo))//判断是否可以清除连接
                 continue;
     
             /*
              * Currently, join_is_removable can only succeed when the sjinfo's
              * righthand is a single baserel.  Remove that rel from the query and
              * joinlist.
              */
             innerrelid = bms_singleton_member(sjinfo->min_righthand);
     
             remove_rel_from_query(root, innerrelid,
                                   bms_union(sjinfo->min_lefthand,
                                             sjinfo->min_righthand));//从查询中删除相应的Rel
     
             /* We verify that exactly one reference gets removed from joinlist */
             nremoved = 0;
             joinlist = remove_rel_from_joinlist(joinlist, innerrelid, &nremoved);
             if (nremoved != 1)
                 elog(ERROR, "failed to find relation %d in joinlist", innerrelid);
     
             /*
              * We can delete this SpecialJoinInfo from the list too, since it's no
              * longer of interest.
              */
             //更新连接链表信息 
             root->join_info_list = list_delete_ptr(root->join_info_list, sjinfo);
     
             /*
              * Restart the scan.  This is necessary to ensure we find all
              * removable joins independently of ordering of the join_info_list
              * (note that removal of attr_needed bits may make a join appear
              * removable that did not before).  Also, since we just deleted the
              * current list cell, we'd have to have some kluge to continue the
              * list scan anyway.
              */
             goto restart;
         }
     
         return joinlist;
     }
    

**reduce_unique_semijoins**  
把可以简化的半连接转化为内连接.  
比如以下的SQL语句:

    
    
    select t1.*
    from t_grxx t1 
    where dwbh IN (select t2.dwbh from t_dwxx t2);
    

由于子查询"select t2.dwbh from t_dwxx
t2"的dwbh是PK,子查询提升后,t_grxx的dwbh只对应t_dwxx唯一的一条记录,因此可以把半连接转换为内连接,执行计划如下:

    
    
    testdb=# explain verbose select t1.*
    from t_grxx t1 
    where dwbh IN (select t2.dwbh from t_dwxx t2);
                                     QUERY PLAN                                  
    -----------------------------------------------------------------------------     Hash Join  (cost=1.07..20.10 rows=6 width=176)
       Output: t1.dwbh, t1.grbh, t1.xm, t1.xb, t1.nl
       Inner Unique: true
       Hash Cond: ((t1.dwbh)::text = (t2.dwbh)::text)
       ->  Seq Scan on public.t_grxx t1  (cost=0.00..14.00 rows=400 width=176)
             Output: t1.dwbh, t1.grbh, t1.xm, t1.xb, t1.nl
       ->  Hash  (cost=1.03..1.03 rows=3 width=38)
             Output: t2.dwbh
             ->  Seq Scan on public.t_dwxx t2  (cost=0.00..1.03 rows=3 width=38)
                   Output: t2.dwbh
    (10 rows)
    

跟踪分析:

    
    
    (gdb) n
    199   reduce_unique_semijoins(root);
    (gdb) step
    reduce_unique_semijoins (root=0x1702968) at analyzejoins.c:520
    520   for (lc = list_head(root->join_info_list); lc != NULL; lc = next)
    (gdb) n
    522     SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(lc);
    (gdb) 
    

查看SpecialJoinInfo内存结构:

    
    
    528     next = lnext(lc);
    (gdb) p *sjinfo
    $1 = {type = T_SpecialJoinInfo, min_lefthand = 0x1749818, min_righthand = 0x1749830, syn_lefthand = 0x1749570, 
      syn_righthand = 0x17495d0, jointype = JOIN_SEMI, lhs_strict = true, delay_upper_joins = false, semi_can_btree = true, 
      semi_can_hash = true, semi_operators = 0x17496c8, semi_rhs_exprs = 0x17497b8}
    

内表(innerrel,即t_dwxx)如支持唯一性,则可以考虑把半连接转换为内连接

    
    
    550     if (!rel_supports_distinctness(root, innerrel))
    ...
    575     root->join_info_list = list_delete_ptr(root->join_info_list, sjinfo);
    ...
    

源代码如下:

    
    
     /*
      * reduce_unique_semijoins
      *      Check for semijoins that can be simplified to plain inner joins
      *      because the inner relation is provably unique for the join clauses.
      *
      * Ideally this would happen during reduce_outer_joins, but we don't have
      * enough information at that point.
      *
      * To perform the strength reduction when applicable, we need only delete
      * the semijoin's SpecialJoinInfo from root->join_info_list.  (We don't
      * bother fixing the join type attributed to it in the query jointree,
      * since that won't be consulted again.)
      */
     void
     reduce_unique_semijoins(PlannerInfo *root)
     {
         ListCell   *lc;
         ListCell   *next;
     
         /*
          * Scan the join_info_list to find semijoins.  We can't use foreach
          * because we may delete the current cell.
          */
         for (lc = list_head(root->join_info_list); lc != NULL; lc = next)
         {
             SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(lc);//特殊连接信息,先前通过deconstruct函数生成
             int         innerrelid;
             RelOptInfo *innerrel;
             Relids      joinrelids;
             List       *restrictlist;
     
             next = lnext(lc);
     
             /*
              * Must be a non-delaying semijoin to a single baserel, else we aren't
              * going to be able to do anything with it.  (It's probably not
              * possible for delay_upper_joins to be set on a semijoin, but we
              * might as well check.)
              */
             if (sjinfo->jointype != JOIN_SEMI ||
                 sjinfo->delay_upper_joins)
                 continue;
     
             if (!bms_get_singleton_member(sjinfo->min_righthand, &innerrelid))
                 continue;
     
             innerrel = find_base_rel(root, innerrelid);
     
             /*
              * Before we trouble to run generate_join_implied_equalities, make a
              * quick check to eliminate cases in which we will surely be unable to
              * prove uniqueness of the innerrel.
              */
             if (!rel_supports_distinctness(root, innerrel))
                 continue;
     
             /* Compute the relid set for the join we are considering */
             joinrelids = bms_union(sjinfo->min_lefthand, sjinfo->min_righthand);
     
             /*
              * Since we're only considering a single-rel RHS, any join clauses it
              * has must be clauses linking it to the semijoin's min_lefthand.  We
              * can also consider EC-derived join clauses.
              */
             restrictlist =
                 list_concat(generate_join_implied_equalities(root,
                                                              joinrelids,
                                                              sjinfo->min_lefthand,
                                                              innerrel),
                             innerrel->joininfo);
     
             /* Test whether the innerrel is unique for those clauses. */
             if (!innerrel_is_unique(root,
                                     joinrelids, sjinfo->min_lefthand, innerrel,
                                     JOIN_SEMI, restrictlist, true))
                 continue;
     
             /* OK, remove the SpecialJoinInfo from the list. */
             root->join_info_list = list_delete_ptr(root->join_info_list, sjinfo);//删除特殊连接信息
         }
     }
    

**add_placeholders_to_base_rels**  
把PHV分发到base rels中,代码较为简单

    
    
     /*
      * add_placeholders_to_base_rels
      *      Add any required PlaceHolderVars to base rels' targetlists.
      *
      * If any placeholder can be computed at a base rel and is needed above it,
      * add it to that rel's targetlist.  This might look like it could be merged
      * with fix_placeholder_input_needed_levels, but it must be separate because
      * join removal happens in between, and can change the ph_eval_at sets.  There
      * is essentially the same logic in add_placeholders_to_joinrel, but we can't
      * do that part until joinrels are formed.
      */
     void
     add_placeholders_to_base_rels(PlannerInfo *root)
     {
         ListCell   *lc;
     
         foreach(lc, root->placeholder_list)//遍历PH链表
         {
             PlaceHolderInfo *phinfo = (PlaceHolderInfo *) lfirst(lc);
             Relids      eval_at = phinfo->ph_eval_at;
             int         varno;
     
             if (bms_get_singleton_member(eval_at, &varno) &&
                 bms_nonempty_difference(phinfo->ph_needed, eval_at))//添加到需要的RelOptInfo中
             {
                 RelOptInfo *rel = find_base_rel(root, varno);
     
                 rel->reltarget->exprs = lappend(rel->reltarget->exprs,
                                                 copyObject(phinfo->ph_var));
                 /* reltarget's cost and width fields will be updated later */
             }
         }
     }
    
    

### 三、参考资料

[planmain.c](https://doxygen.postgresql.org/planmain_8c.html)  
[relation.h](https://doxygen.postgresql.org/relation_8h_source.html)

