先前的章节已介绍了函数query_planner中子函数remove_useless_joins、reduce_unique_semijoins和add_placeholders_to_base_rels的主要实现逻辑,本节继续介绍create_lateral_join_info、match_foreign_keys_to_quals和extract_restriction_or_clauses的实现逻辑。

query_planner代码片段:

    
    
         //...
     
         /*
          * Construct the lateral reference sets now that we have finalized
          * PlaceHolderVar eval levels.
          */
         create_lateral_join_info(root);//创建Lateral连接信息
     
         /*
          * Match foreign keys to equivalence classes and join quals.  This must be
          * done after finalizing equivalence classes, and it's useful to wait till
          * after join removal so that we can skip processing foreign keys
          * involving removed relations.
          */
         match_foreign_keys_to_quals(root);//匹配外键信息
     
         /*
          * Look for join OR clauses that we can extract single-relation
          * restriction OR clauses from.
          */
         extract_restriction_or_clauses(root);//在OR语句中抽取约束条件
     
         /*
          * We should now have size estimates for every actual table involved in
          * the query, and we also know which if any have been deleted from the
          * query by join removal; so we can compute total_table_pages.
          *
          * Note that appendrels are not double-counted here, even though we don't
          * bother to distinguish RelOptInfos for appendrel parents, because the
          * parents will still have size zero.
          *
          * XXX if a table is self-joined, we will count it once per appearance,
          * which perhaps is the wrong thing ... but that's not completely clear,
          * and detecting self-joins here is difficult, so ignore it for now.
          */
         total_pages = 0;
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//计算总pages
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
     
             if (brel == NULL)
                 continue;
     
             Assert(brel->relid == rti); /* sanity check on array */
     
             if (IS_SIMPLE_REL(brel))
                 total_pages += (double) brel->pages;
         }
         root->total_table_pages = total_pages;//赋值
    
         //...
    

### 一、数据结构

**RelOptInfo**  
RelOptInfo中,与LATERAL相关的数据结构

    
    
     typedef struct RelOptInfo
     {
         NodeTag     type;//节点标识
     
         RelOptKind  reloptkind;//RelOpt类型
     
         //...
     
         /* parameterization information needed for both base rels and join rels */
         /* (see also lateral_vars and lateral_referencers) */
         Relids      direct_lateral_relids;  /*使用lateral语法,需依赖的Relids rels directly laterally referenced */
         Relids      lateral_relids; /* minimum parameterization of rel */
     
         //...
         List       *lateral_vars;   /* 关系依赖的Vars/PHVs LATERAL Vars and PHVs referenced by rel */
         Relids      lateral_referencers;    /*依赖该关系的Relids rels that reference me laterally */
         //...
     } RelOptInfo;
    
    

### 二、源码解读

**create_lateral_join_info**  
PG在提供LATERAL语法之前,假定所有的子查询都可以独立存在,不能互相引用属性或者引用上层的属性,为了可以引用其他或上层的属性,需要在子查询前面显式指定LATERAL关键字.  
比如以下的SQL语句,不显式指定LATERAL关键字无法正常运行:

    
    
    testdb=# select a.*,b.grbh,b.je 
    testdb-# from t_dwxx a,(select t1.dwbh,t1.grbh,t2.je from t_grxx t1 inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    testdb-# where a.dwbh = '1001'
    testdb-# order by b.dwbh;
    ERROR:  invalid reference to FROM-clause entry for table "a"
    LINE 2: ... from t_grxx t1 inner join t_jfxx t2 on t1.dwbh = a.dwbh and...
                                                                 ^
    HINT:  There is an entry for table "a", but it cannot be referenced from this part of the query.
    

在子查询前显式指定LATERAL后,可以正常运行:

    
    
    testdb=# select a.*,b.grbh,b.je 
    testdb-# from t_dwxx a,lateral (select t1.dwbh,t1.grbh,t2.je from t_grxx t1 inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    testdb-# where a.dwbh = '1001'
    testdb-# order by b.dwbh;
       dwmc    | dwbh |        dwdz        | grbh |  je   
    -----------+------+--------------------+------+-------     X有限公司 | 1001 | 广东省广州市荔湾区 | 901  | 401.3
     X有限公司 | 1001 | 广东省广州市荔湾区 | 901  | 401.3
     X有限公司 | 1001 | 广东省广州市荔湾区 | 901  | 401.3
    

如函数注释所描述的,create_lateral_join_info函数的作用是填充RelOptInfo中的相关四个变量,"Fill in the per-base-relation direct_lateral_relids, lateral_relids和and lateral_referencers
sets"

源代码如下:

    
    
     /*
      * create_lateral_join_info
      *    Fill in the per-base-relation direct_lateral_relids, lateral_relids
      *    and lateral_referencers sets.
      *
      * This has to run after deconstruct_jointree, because we need to know the
      * final ph_eval_at values for PlaceHolderVars.
      */
     void
     create_lateral_join_info(PlannerInfo *root)
     {
         bool        found_laterals = false;
         Index       rti;
         ListCell   *lc;
     
         /* We need do nothing if the query contains no LATERAL RTEs */
         if (!root->hasLateralRTEs)//是否存在LateralRTE
             return;
     
         /*
          * Examine all baserels (the rel array has been set up by now).
          */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//遍历
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
             Relids      lateral_relids;
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (brel == NULL)
                 continue;
     
             Assert(brel->relid == rti); /* sanity check on array */
     
             /* ignore RTEs that are "other rels" */
             if (brel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             lateral_relids = NULL;
     
             /* consider each laterally-referenced Var or PHV */
             foreach(lc, brel->lateral_vars)
             {
                 Node       *node = (Node *) lfirst(lc);
     
                 if (IsA(node, Var))
                 {
                     Var        *var = (Var *) node;
     
                     found_laterals = true;
                     lateral_relids = bms_add_member(lateral_relids,
                                                     var->varno);
                 }
                 else if (IsA(node, PlaceHolderVar))
                 {
                     PlaceHolderVar *phv = (PlaceHolderVar *) node;
                     PlaceHolderInfo *phinfo = find_placeholder_info(root, phv,
                                                                     false);
     
                     found_laterals = true;
                     lateral_relids = bms_add_members(lateral_relids,
                                                      phinfo->ph_eval_at);
                 }
                 else
                     Assert(false);
             }
     
             /* We now have all the simple lateral refs from this rel */
             brel->direct_lateral_relids = lateral_relids;
             brel->lateral_relids = bms_copy(lateral_relids);
         }
     
         /*
          * Now check for lateral references within PlaceHolderVars, and mark their
          * eval_at rels as having lateral references to the source rels.
          *
          * For a PHV that is due to be evaluated at a baserel, mark its source(s)
          * as direct lateral dependencies of the baserel (adding onto the ones
          * recorded above).  If it's due to be evaluated at a join, mark its
          * source(s) as indirect lateral dependencies of each baserel in the join,
          * ie put them into lateral_relids but not direct_lateral_relids.  This is
          * appropriate because we can't put any such baserel on the outside of a
          * join to one of the PHV's lateral dependencies, but on the other hand we
          * also can't yet join it directly to the dependency.
          */
         foreach(lc, root->placeholder_list)
         {
             PlaceHolderInfo *phinfo = (PlaceHolderInfo *) lfirst(lc);
             Relids      eval_at = phinfo->ph_eval_at;
             int         varno;
     
             if (phinfo->ph_lateral == NULL)
                 continue;           /* PHV is uninteresting if no lateral refs */
     
             found_laterals = true;
     
             if (bms_get_singleton_member(eval_at, &varno))
             {
                 /* Evaluation site is a baserel */
                 RelOptInfo *brel = find_base_rel(root, varno);
     
                 brel->direct_lateral_relids =
                     bms_add_members(brel->direct_lateral_relids,
                                     phinfo->ph_lateral);
                 brel->lateral_relids =
                     bms_add_members(brel->lateral_relids,
                                     phinfo->ph_lateral);
             }
             else
             {
                 /* Evaluation site is a join */
                 varno = -1;
                 while ((varno = bms_next_member(eval_at, varno)) >= 0)
                 {
                     RelOptInfo *brel = find_base_rel(root, varno);
     
                     brel->lateral_relids = bms_add_members(brel->lateral_relids,
                                                            phinfo->ph_lateral);
                 }
             }
         }
     
         /*
          * If we found no actual lateral references, we're done; but reset the
          * hasLateralRTEs flag to avoid useless work later.
          */
         if (!found_laterals)
         {
             root->hasLateralRTEs = false;
             return;
         }
     
         /*
          * Calculate the transitive closure of the lateral_relids sets, so that
          * they describe both direct and indirect lateral references.  If relation
          * X references Y laterally, and Y references Z laterally, then we will
          * have to scan X on the inside of a nestloop with Z, so for all intents
          * and purposes X is laterally dependent on Z too.
          *
          * This code is essentially Warshall's algorithm for transitive closure.
          * The outer loop considers each baserel, and propagates its lateral
          * dependencies to those baserels that have a lateral dependency on it.
          */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
             Relids      outer_lateral_relids;
             Index       rti2;
     
             if (brel == NULL || brel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             /* need not consider baserel further if it has no lateral refs */
             outer_lateral_relids = brel->lateral_relids;
             if (outer_lateral_relids == NULL)
                 continue;
     
             /* else scan all baserels */
             for (rti2 = 1; rti2 < root->simple_rel_array_size; rti2++)
             {
                 RelOptInfo *brel2 = root->simple_rel_array[rti2];
     
                 if (brel2 == NULL || brel2->reloptkind != RELOPT_BASEREL)
                     continue;
     
                 /* if brel2 has lateral ref to brel, propagate brel's refs */
                 if (bms_is_member(rti, brel2->lateral_relids))
                     brel2->lateral_relids = bms_add_members(brel2->lateral_relids,
                                                             outer_lateral_relids);
             }
         }
     
         /*
          * Now that we've identified all lateral references, mark each baserel
          * with the set of relids of rels that reference it laterally (possibly
          * indirectly) --- that is, the inverse mapping of lateral_relids.
          */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
             Relids      lateral_relids;
             int         rti2;
     
             if (brel == NULL || brel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             /* Nothing to do at rels with no lateral refs */
             lateral_relids = brel->lateral_relids;
             if (lateral_relids == NULL)
                 continue;
     
             /*
              * We should not have broken the invariant that lateral_relids is
              * exactly NULL if empty.
              */
             Assert(!bms_is_empty(lateral_relids));
     
             /* Also, no rel should have a lateral dependency on itself */
             Assert(!bms_is_member(rti, lateral_relids));
     
             /* Mark this rel's referencees */
             rti2 = -1;
             while ((rti2 = bms_next_member(lateral_relids, rti2)) >= 0)
             {
                 RelOptInfo *brel2 = root->simple_rel_array[rti2];
     
                 Assert(brel2 != NULL && brel2->reloptkind == RELOPT_BASEREL);
                 brel2->lateral_referencers =
                     bms_add_member(brel2->lateral_referencers, rti);
             }
         }
     
         /*
          * Lastly, propagate lateral_relids and lateral_referencers from appendrel
          * parent rels to their child rels.  We intentionally give each child rel
          * the same minimum parameterization, even though it's quite possible that
          * some don't reference all the lateral rels.  This is because any append
          * path for the parent will have to have the same parameterization for
          * every child anyway, and there's no value in forcing extra
          * reparameterize_path() calls.  Similarly, a lateral reference to the
          * parent prevents use of otherwise-movable join rels for each child.
          */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
             RangeTblEntry *brte = root->simple_rte_array[rti];
     
             /*
              * Skip empty slots. Also skip non-simple relations i.e. dead
              * relations.
              */
             if (brel == NULL || !IS_SIMPLE_REL(brel))
                 continue;
     
             /*
              * In the case of table inheritance, the parent RTE is directly linked
              * to every child table via an AppendRelInfo.  In the case of table
              * partitioning, the inheritance hierarchy is expanded one level at a
              * time rather than flattened.  Therefore, an other member rel that is
              * a partitioned table may have children of its own, and must
              * therefore be marked with the appropriate lateral info so that those
              * children eventually get marked also.
              */
             Assert(brte);
             if (brel->reloptkind == RELOPT_OTHER_MEMBER_REL &&
                 (brte->rtekind != RTE_RELATION ||
                  brte->relkind != RELKIND_PARTITIONED_TABLE))
                 continue;
     
             if (brte->inh)
             {
                 foreach(lc, root->append_rel_list)
                 {
                     AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(lc);
                     RelOptInfo *childrel;
     
                     if (appinfo->parent_relid != rti)
                         continue;
                     childrel = root->simple_rel_array[appinfo->child_relid];
                     Assert(childrel->reloptkind == RELOPT_OTHER_MEMBER_REL);
                     Assert(childrel->direct_lateral_relids == NULL);
                     childrel->direct_lateral_relids = brel->direct_lateral_relids;
                     Assert(childrel->lateral_relids == NULL);
                     childrel->lateral_relids = brel->lateral_relids;
                     Assert(childrel->lateral_referencers == NULL);
                     childrel->lateral_referencers = brel->lateral_referencers;
                 }
             }
         }
     }
    

跟踪分析:

    
    
    (gdb) b planmain.c:173
    Breakpoint 1 at 0x76961a: file planmain.c, line 173.
    (gdb) c
    Continuing.
    
    Breakpoint 1, query_planner (root=0x1702b80, tlist=0x174a870, qp_callback=0x76e97d <standard_qp_callback>, 
        qp_extra=0x7ffd35e059c0) at planmain.c:177
    ...
    (gdb) 
    212   create_lateral_join_info(root);
    

查看root变量:

    
    
    (gdb) p *root
    $11 = {..., hasLateralRTEs = false, ...}
    

经过处理后,LATERAL已经消失(hasLateralRTEs = false),不需要进行处理.

**match_foreign_keys_to_quals**  
这是外键相关的处理,等价类与外键约束进行匹配并加入到条件语句(quals)中

    
    
     /*
      * match_foreign_keys_to_quals
      *      Match foreign-key constraints to equivalence classes and join quals
      *
      * The idea here is to see which query join conditions match equality
      * constraints of a foreign-key relationship.  For such join conditions,
      * we can use the FK semantics to make selectivity estimates that are more
      * reliable than estimating from statistics, especially for multiple-column
      * FKs, where the normal assumption of independent conditions tends to fail.
      *
      * In this function we annotate the ForeignKeyOptInfos in root->fkey_list
      * with info about which eclasses and join qual clauses they match, and
      * discard any ForeignKeyOptInfos that are irrelevant for the query.
      */
     void
     match_foreign_keys_to_quals(PlannerInfo *root)
     {
         List       *newlist = NIL;
         ListCell   *lc;
     
         foreach(lc, root->fkey_list)
         {
             ForeignKeyOptInfo *fkinfo = (ForeignKeyOptInfo *) lfirst(lc);
             RelOptInfo *con_rel;
             RelOptInfo *ref_rel;
             int         colno;
     
             /*
              * Either relid might identify a rel that is in the query's rtable but
              * isn't referenced by the jointree so won't have a RelOptInfo.  Hence
              * don't use find_base_rel() here.  We can ignore such FKs.
              */
             if (fkinfo->con_relid >= root->simple_rel_array_size ||
                 fkinfo->ref_relid >= root->simple_rel_array_size)
                 continue;           /* just paranoia */
             con_rel = root->simple_rel_array[fkinfo->con_relid];
             if (con_rel == NULL)
                 continue;
             ref_rel = root->simple_rel_array[fkinfo->ref_relid];
             if (ref_rel == NULL)
                 continue;
     
             /*
              * Ignore FK unless both rels are baserels.  This gets rid of FKs that
              * link to inheritance child rels (otherrels) and those that link to
              * rels removed by join removal (dead rels).
              */
             if (con_rel->reloptkind != RELOPT_BASEREL ||
                 ref_rel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             /*
              * Scan the columns and try to match them to eclasses and quals.
              *
              * Note: for simple inner joins, any match should be in an eclass.
              * "Loose" quals that syntactically match an FK equality must have
              * been rejected for EC status because they are outer-join quals or
              * similar.  We can still consider them to match the FK if they are
              * not outerjoin_delayed.
              */
             for (colno = 0; colno < fkinfo->nkeys; colno++)
             {
                 AttrNumber  con_attno,
                             ref_attno;
                 Oid         fpeqop;
                 ListCell   *lc2;
     
                 fkinfo->eclass[colno] = match_eclasses_to_foreign_key_col(root,
                                                                           fkinfo,
                                                                           colno);
                 /* Don't bother looking for loose quals if we got an EC match */
                 if (fkinfo->eclass[colno] != NULL)
                 {
                     fkinfo->nmatched_ec++;
                     continue;
                 }
     
                 /*
                  * Scan joininfo list for relevant clauses.  Either rel's joininfo
                  * list would do equally well; we use con_rel's.
                  */
                 con_attno = fkinfo->conkey[colno];
                 ref_attno = fkinfo->confkey[colno];
                 fpeqop = InvalidOid;    /* we'll look this up only if needed */
     
                 foreach(lc2, con_rel->joininfo)
                 {
                     RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc2);
                     OpExpr     *clause = (OpExpr *) rinfo->clause;
                     Var        *leftvar;
                     Var        *rightvar;
     
                     /* Ignore outerjoin-delayed clauses */
                     if (rinfo->outerjoin_delayed)
                         continue;
     
                     /* Only binary OpExprs are useful for consideration */
                     if (!IsA(clause, OpExpr) ||
                         list_length(clause->args) != 2)
                         continue;
                     leftvar = (Var *) get_leftop((Expr *) clause);
                     rightvar = (Var *) get_rightop((Expr *) clause);
     
                     /* Operands must be Vars, possibly with RelabelType */
                     while (leftvar && IsA(leftvar, RelabelType))
                         leftvar = (Var *) ((RelabelType *) leftvar)->arg;
                     if (!(leftvar && IsA(leftvar, Var)))
                         continue;
                     while (rightvar && IsA(rightvar, RelabelType))
                         rightvar = (Var *) ((RelabelType *) rightvar)->arg;
                     if (!(rightvar && IsA(rightvar, Var)))
                         continue;
     
                     /* Now try to match the vars to the current foreign key cols */
                     if (fkinfo->ref_relid == leftvar->varno &&
                         ref_attno == leftvar->varattno &&
                         fkinfo->con_relid == rightvar->varno &&
                         con_attno == rightvar->varattno)
                     {
                         /* Vars match, but is it the right operator? */
                         if (clause->opno == fkinfo->conpfeqop[colno])
                         {
                             fkinfo->rinfos[colno] = lappend(fkinfo->rinfos[colno],
                                                             rinfo);
                             fkinfo->nmatched_ri++;
                         }
                     }
                     else if (fkinfo->ref_relid == rightvar->varno &&
                              ref_attno == rightvar->varattno &&
                              fkinfo->con_relid == leftvar->varno &&
                              con_attno == leftvar->varattno)
                     {
                         /*
                          * Reverse match, must check commutator operator.  Look it
                          * up if we didn't already.  (In the worst case we might
                          * do multiple lookups here, but that would require an FK
                          * equality operator without commutator, which is
                          * unlikely.)
                          */
                         if (!OidIsValid(fpeqop))
                             fpeqop = get_commutator(fkinfo->conpfeqop[colno]);
                         if (clause->opno == fpeqop)
                         {
                             fkinfo->rinfos[colno] = lappend(fkinfo->rinfos[colno],
                                                             rinfo);
                             fkinfo->nmatched_ri++;
                         }
                     }
                 }
                 /* If we found any matching loose quals, count col as matched */
                 if (fkinfo->rinfos[colno])
                     fkinfo->nmatched_rcols++;
             }
     
             /*
              * Currently, we drop multicolumn FKs that aren't fully matched to the
              * query.  Later we might figure out how to derive some sort of
              * estimate from them, in which case this test should be weakened to
              * "if ((fkinfo->nmatched_ec + fkinfo->nmatched_rcols) > 0)".
              */
             if ((fkinfo->nmatched_ec + fkinfo->nmatched_rcols) == fkinfo->nkeys)
                 newlist = lappend(newlist, fkinfo);
         }
         /* Replace fkey_list, thereby discarding any useless entries */
         root->fkey_list = newlist;
     }
    

**extract_restriction_or_clauses**  
检查join连接条件的OR-of-AND语句,如存在有用的OR约束条件,则提取出来.  
如代码注释所描述,((a.x = 42 AND b.y = 43) OR (a.x = 44 AND b.z = 45)),可以提取条件(a.x = 42
OR a.x = 44) AND (b.y = 43 OR b.z =
45),提取这些条件的目的是为了在连接前能把这些条件下推到关系中,减少参与连接运算的元组数量.  
比如:

    
    
    testdb=# explain verbose select t1.*
    from t_dwxx t1 inner join t_grxx t2 
        on (t1.dwbh = '1001' and t2.grbh = '901') OR (t1.dwbh = '1002' and t2.grbh = '902');
                                                                                QUERY PLAN                                       
                                          
    -----------------------------------------------------------------------------------------------------------------------------    --------------------------------------     Nested Loop  (cost=0.00..17.23 rows=5 width=474)
       Output: t1.dwmc, t1.dwbh, t1.dwdz
       Join Filter: ((((t1.dwbh)::text = '1001'::text) AND ((t2.grbh)::text = '901'::text)) OR (((t1.dwbh)::text = '1002'::text) 
    AND ((t2.grbh)::text = '902'::text)))
       ->  Seq Scan on public.t_grxx t2  (cost=0.00..16.00 rows=4 width=38)
             Output: t2.dwbh, t2.grbh, t2.xm, t2.xb, t2.nl
             Filter: (((t2.grbh)::text = '901'::text) OR ((t2.grbh)::text = '902'::text))
       ->  Materialize  (cost=0.00..1.05 rows=2 width=474)
             Output: t1.dwmc, t1.dwbh, t1.dwdz
             ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.04 rows=2 width=474)
                   Output: t1.dwmc, t1.dwbh, t1.dwdz
                   Filter: (((t1.dwbh)::text = '1001'::text) OR ((t1.dwbh)::text = '1002'::text))
    (11 rows)
    
    

可以看到,t1.dwbh = '1001' OR t1.dwbh = '1002'和t2.grbh = '901' OR t2.grbh =
'902'在连接前下推到数据表扫描作为过滤条件.

    
    
     /*
      * extract_restriction_or_clauses
      *    Examine join OR-of-AND clauses to see if any useful restriction OR
      *    clauses can be extracted.  If so, add them to the query.
      *
      * Although a join clause must reference multiple relations overall,
      * an OR of ANDs clause might contain sub-clauses that reference just one
      * relation and can be used to build a restriction clause for that rel.
      * For example consider
      *      WHERE ((a.x = 42 AND b.y = 43) OR (a.x = 44 AND b.z = 45));
      * We can transform this into
      *      WHERE ((a.x = 42 AND b.y = 43) OR (a.x = 44 AND b.z = 45))
      *          AND (a.x = 42 OR a.x = 44)
      *          AND (b.y = 43 OR b.z = 45);
      * which allows the latter clauses to be applied during the scans of a and b,
      * perhaps as index qualifications, and in any case reducing the number of
      * rows arriving at the join.  In essence this is a partial transformation to
      * CNF (AND of ORs format).  It is not complete, however, because we do not
      * unravel the original OR --- doing so would usually bloat the qualification
      * expression to little gain.
      *
      * The added quals are partially redundant with the original OR, and therefore
      * would cause the size of the joinrel to be underestimated when it is finally
      * formed.  (This would be true of a full transformation to CNF as well; the
      * fault is not really in the transformation, but in clauselist_selectivity's
      * inability to recognize redundant conditions.)  We can compensate for this
      * redundancy by changing the cached selectivity of the original OR clause,
      * canceling out the (valid) reduction in the estimated sizes of the base
      * relations so that the estimated joinrel size remains the same.  This is
      * a MAJOR HACK: it depends on the fact that clause selectivities are cached
      * and on the fact that the same RestrictInfo node will appear in every
      * joininfo list that might be used when the joinrel is formed.
      * And it doesn't work in cases where the size estimation is nonlinear
      * (i.e., outer and IN joins).  But it beats not doing anything.
      *
      * We examine each base relation to see if join clauses associated with it
      * contain extractable restriction conditions.  If so, add those conditions
      * to the rel's baserestrictinfo and update the cached selectivities of the
      * join clauses.  Note that the same join clause will be examined afresh
      * from the point of view of each baserel that participates in it, so its
      * cached selectivity may get updated multiple times.
      */
     void
     extract_restriction_or_clauses(PlannerInfo *root)
     {
         Index       rti;
     
         /* Examine each baserel for potential join OR clauses */
         for (rti = 1; rti < root->simple_rel_array_size; rti++)
         {
             RelOptInfo *rel = root->simple_rel_array[rti];
             ListCell   *lc;
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (rel == NULL)
                 continue;
     
             Assert(rel->relid == rti);  /* sanity check on array */
     
             /* ignore RTEs that are "other rels" */
             if (rel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             /*
              * Find potentially interesting OR joinclauses.  We can use any
              * joinclause that is considered safe to move to this rel by the
              * parameterized-path machinery, even though what we are going to do
              * with it is not exactly a parameterized path.
              *
              * However, it seems best to ignore clauses that have been marked
              * redundant (by setting norm_selec > 1).  That likely can't happen
              * for OR clauses, but let's be safe.
              */
             foreach(lc, rel->joininfo)
             {
                 RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc);
     
                 if (restriction_is_or_clause(rinfo) &&
                     join_clause_is_movable_to(rinfo, rel) &&
                     rinfo->norm_selec <= 1)
                 {
                     /* Try to extract a qual for this rel only */
                     Expr       *orclause = extract_or_clause(rinfo, rel);
     
                     /*
                      * If successful, decide whether we want to use the clause,
                      * and insert it into the rel's restrictinfo list if so.
                      */
                     if (orclause)
                         consider_new_or_clause(root, rel, orclause, rinfo);
                 }
             }
         }
     }
    

### 三、参考资料

[planmain.c](https://doxygen.postgresql.org/planmain_8c_source.html)

