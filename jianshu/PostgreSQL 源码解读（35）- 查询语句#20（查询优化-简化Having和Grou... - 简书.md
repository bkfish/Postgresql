本节简单介绍了PG查询优化中对Having和Group By子句的简化处理。

### 一、基本概念

**简化Having语句**  
把Having中的约束条件,如满足可以提升到Where条件中的,则移动到Where子句中,否则仍保留在Having语句中.这样做的目的是因为Having过滤在Group
by之后执行,如能把Having中的过滤提升到Where中,则可以提前执行"选择"运算,减少Group by的开销.  
以下语句,条件dwbh='1002'提升到Where中执行:

    
    
    testdb=# explain verbose select a.dwbh,a.xb,count(*) 
    testdb-# from t_grxx a 
    testdb-# group by a.dwbh,a.xb
    testdb-# having count(*) >= 1 and dwbh = '1002';
                                     QUERY PLAN                                  
    -----------------------------------------------------------------------------     GroupAggregate  (cost=15.01..15.06 rows=1 width=84)
       Output: dwbh, xb, count(*)
       Group Key: a.dwbh, a.xb
       Filter: (count(*) >= 1) -- count(*) >= 1 仍保留在Having中
       ->  Sort  (cost=15.01..15.02 rows=2 width=76)
             Output: dwbh, xb
             Sort Key: a.xb
             ->  Seq Scan on public.t_grxx a  (cost=0.00..15.00 rows=2 width=76)
                   Output: dwbh, xb
                   Filter: ((a.dwbh)::text = '1002'::text) -- 提升到Where中,扫描时过滤Tuple
    (10 rows)
    

如存在Group by & Grouping sets则不作处理:

    
    
    testdb=# explain verbose
    testdb-# select a.dwbh,a.xb,count(*) 
    testdb-# from t_grxx a 
    testdb-# group by 
    testdb-# grouping sets ((a.dwbh),(a.xb),())
    testdb-# having count(*) >= 1 and dwbh = '1002'
    testdb-# order by a.dwbh,a.xb;
                                      QUERY PLAN                                   
    -------------------------------------------------------------------------------     Sort  (cost=28.04..28.05 rows=3 width=84)
       Output: dwbh, xb, (count(*))
       Sort Key: a.dwbh, a.xb
       ->  MixedAggregate  (cost=0.00..28.02 rows=3 width=84)
             Output: dwbh, xb, count(*)
             Hash Key: a.dwbh
             Hash Key: a.xb
             Group Key: ()
             Filter: ((count(*) >= 1) AND ((a.dwbh)::text = '1002'::text)) -- 扫描数据表后再过滤
             ->  Seq Scan on public.t_grxx a  (cost=0.00..14.00 rows=400 width=76)
                   Output: dwbh, grbh, xm, xb, nl
    (11 rows)
    

**简化Group by语句**  
如Group by中的字段列表已包含某个表主键的所有列,则该表在Group by语句中的其他列可以删除,这样的做法有利于提升在Group
by过程中排序或Hash的性能,减少不必要的开销.

    
    
    testdb=# explain verbose select a.dwbh,a.dwmc,count(*) 
    testdb-# from t_dwxx a 
    testdb-# group by a.dwbh,a.dwmc
    testdb-# having count(*) >= 1;
                                    QUERY PLAN                                
    --------------------------------------------------------------------------     HashAggregate  (cost=13.20..15.20 rows=53 width=264)
       Output: dwbh, dwmc, count(*)
       Group Key: a.dwbh, a.dwmc -- 分组键为dwbh & dwmc
       Filter: (count(*) >= 1)
       ->  Seq Scan on public.t_dwxx a  (cost=0.00..11.60 rows=160 width=256)
             Output: dwmc, dwbh, dwdz
    (6 rows)
    
    testdb=# alter table t_dwxx add primary key(dwbh); -- 添加主键
    ALTER TABLE
    testdb=# explain verbose select a.dwbh,a.dwmc,count(*) 
    from t_dwxx a 
    group by a.dwbh,a.dwmc
    having count(*) >= 1;
                                  QUERY PLAN                               
    -----------------------------------------------------------------------     HashAggregate  (cost=1.05..1.09 rows=1 width=264)
       Output: dwbh, dwmc, count(*)
       Group Key: a.dwbh -- 分组键只保留dwbh
       Filter: (count(*) >= 1)
       ->  Seq Scan on public.t_dwxx a  (cost=0.00..1.03 rows=3 width=256)
             Output: dwmc, dwbh, dwdz
    (6 rows)
    

### 二、源码解读

相关处理的源码位于文件subquery_planner.c中,主函数为subquery_planner,代码片段如下:

    
    
         /*
          * In some cases we may want to transfer a HAVING clause into WHERE. We
          * cannot do so if the HAVING clause contains aggregates (obviously) or
          * volatile functions (since a HAVING clause is supposed to be executed
          * only once per group).  We also can't do this if there are any nonempty
          * grouping sets; moving such a clause into WHERE would potentially change
          * the results, if any referenced column isn't present in all the grouping
          * sets.  (If there are only empty grouping sets, then the HAVING clause
          * must be degenerate as discussed below.)
          *
          * Also, it may be that the clause is so expensive to execute that we're
          * better off doing it only once per group, despite the loss of
          * selectivity.  This is hard to estimate short of doing the entire
          * planning process twice, so we use a heuristic: clauses containing
          * subplans are left in HAVING.  Otherwise, we move or copy the HAVING
          * clause into WHERE, in hopes of eliminating tuples before aggregation
          * instead of after.
          *
          * If the query has explicit grouping then we can simply move such a
          * clause into WHERE; any group that fails the clause will not be in the
          * output because none of its tuples will reach the grouping or
          * aggregation stage.  Otherwise we must have a degenerate (variable-free)
          * HAVING clause, which we put in WHERE so that query_planner() can use it
          * in a gating Result node, but also keep in HAVING to ensure that we
          * don't emit a bogus aggregated row. (This could be done better, but it
          * seems not worth optimizing.)
          *
          * Note that both havingQual and parse->jointree->quals are in
          * implicitly-ANDed-list form at this point, even though they are declared
          * as Node *.
          */
         newHaving = NIL;
         foreach(l, (List *) parse->havingQual)//存在Having条件语句
         {
             Node       *havingclause = (Node *) lfirst(l);//获取谓词
     
             if ((parse->groupClause && parse->groupingSets) ||
                 contain_agg_clause(havingclause) ||
                 contain_volatile_functions(havingclause) ||
                 contain_subplans(havingclause))
             {
                 /* keep it in HAVING */
                 //如果有Group&&Group Sets语句
                 //保持不变
                 newHaving = lappend(newHaving, havingclause);
             }
             else if (parse->groupClause && !parse->groupingSets)
             {
                 /* move it to WHERE */
                 //只有group语句,可以加入到jointree的条件中
                 parse->jointree->quals = (Node *)
                     lappend((List *) parse->jointree->quals, havingclause);
             }
             else//既没有group也没有grouping set,拷贝一份到jointree的条件中
             {
                 /* put a copy in WHERE, keep it in HAVING */
                 parse->jointree->quals = (Node *)
                     lappend((List *) parse->jointree->quals,
                             copyObject(havingclause));
                 newHaving = lappend(newHaving, havingclause);
             }
         }
         parse->havingQual = (Node *) newHaving;//调整having子句
     
         /* Remove any redundant GROUP BY columns */
         remove_useless_groupby_columns(root);//去掉group by中无用的数据列
    
    

**remove_useless_groupby_columns**

    
    
     /*
      * remove_useless_groupby_columns
      *      Remove any columns in the GROUP BY clause that are redundant due to
      *      being functionally dependent on other GROUP BY columns.
      *
      * Since some other DBMSes do not allow references to ungrouped columns, it's
      * not unusual to find all columns listed in GROUP BY even though listing the
      * primary-key columns would be sufficient.  Deleting such excess columns
      * avoids redundant sorting work, so it's worth doing.  When we do this, we
      * must mark the plan as dependent on the pkey constraint (compare the
      * parser's check_ungrouped_columns() and check_functional_grouping()).
      *
      * In principle, we could treat any NOT-NULL columns appearing in a UNIQUE
      * index as the determining columns.  But as with check_functional_grouping(),
      * there's currently no way to represent dependency on a NOT NULL constraint,
      * so we consider only the pkey for now.
      */
     static void
     remove_useless_groupby_columns(PlannerInfo *root)
     {
         Query      *parse = root->parse;//查询树
         Bitmapset **groupbyattnos;//位图集合
         Bitmapset **surplusvars;//位图集合
         ListCell   *lc;
         int         relid;
     
         /* No chance to do anything if there are less than two GROUP BY items */
         if (list_length(parse->groupClause) < 2)//如果只有1个ITEMS,无需处理
             return;
     
         /* Don't fiddle with the GROUP BY clause if the query has grouping sets */
         if (parse->groupingSets)//存在Grouping sets,不作处理
             return;
     
         /*
          * Scan the GROUP BY clause to find GROUP BY items that are simple Vars.
          * Fill groupbyattnos[k] with a bitmapset of the column attnos of RTE k
          * that are GROUP BY items.
          */
         //用于分组的属性 
         groupbyattnos = (Bitmapset **) palloc0(sizeof(Bitmapset *) *
                                                (list_length(parse->rtable) + 1));
         foreach(lc, parse->groupClause)
         {
             SortGroupClause *sgc = lfirst_node(SortGroupClause, lc);
             TargetEntry *tle = get_sortgroupclause_tle(sgc, parse->targetList);
             Var        *var = (Var *) tle->expr;
     
             /*
              * Ignore non-Vars and Vars from other query levels.
              *
              * XXX in principle, stable expressions containing Vars could also be
              * removed, if all the Vars are functionally dependent on other GROUP
              * BY items.  But it's not clear that such cases occur often enough to
              * be worth troubling over.
              */
             if (!IsA(var, Var) ||
                 var->varlevelsup > 0)
                 continue;
     
             /* OK, remember we have this Var */
             relid = var->varno;
             Assert(relid <= list_length(parse->rtable));
             groupbyattnos[relid] = bms_add_member(groupbyattnos[relid],
                                                   var->varattno - FirstLowInvalidHeapAttributeNumber);
         }
     
         /*
          * Consider each relation and see if it is possible to remove some of its
          * Vars from GROUP BY.  For simplicity and speed, we do the actual removal
          * in a separate pass.  Here, we just fill surplusvars[k] with a bitmapset
          * of the column attnos of RTE k that are removable GROUP BY items.
          */
         surplusvars = NULL;         /* don't allocate array unless required */
         relid = 0;
         //如某个Relation的分组键中已含主键列,去掉其他列
         foreach(lc, parse->rtable)
         {
             RangeTblEntry *rte = lfirst_node(RangeTblEntry, lc);
             Bitmapset  *relattnos;
             Bitmapset  *pkattnos;
             Oid         constraintOid;
     
             relid++;
     
             /* Only plain relations could have primary-key constraints */
             if (rte->rtekind != RTE_RELATION)
                 continue;
     
             /* Nothing to do unless this rel has multiple Vars in GROUP BY */
             relattnos = groupbyattnos[relid];
             if (bms_membership(relattnos) != BMS_MULTIPLE)
                 continue;
     
             /*
              * Can't remove any columns for this rel if there is no suitable
              * (i.e., nondeferrable) primary key constraint.
              */
             pkattnos = get_primary_key_attnos(rte->relid, false, &constraintOid);
             if (pkattnos == NULL)
                 continue;
     
             /*
              * If the primary key is a proper subset of relattnos then we have
              * some items in the GROUP BY that can be removed.
              */
             if (bms_subset_compare(pkattnos, relattnos) == BMS_SUBSET1)
             {
                 /*
                  * To easily remember whether we've found anything to do, we don't
                  * allocate the surplusvars[] array until we find something.
                  */
                 if (surplusvars == NULL)
                     surplusvars = (Bitmapset **) palloc0(sizeof(Bitmapset *) *
                                                          (list_length(parse->rtable) + 1));
     
                 /* Remember the attnos of the removable columns */
                 surplusvars[relid] = bms_difference(relattnos, pkattnos);
     
                 /* Also, mark the resulting plan as dependent on this constraint */
                 parse->constraintDeps = lappend_oid(parse->constraintDeps,
                                                     constraintOid);
             }
         }
     
         /*
          * If we found any surplus Vars, build a new GROUP BY clause without them.
          * (Note: this may leave some TLEs with unreferenced ressortgroupref
          * markings, but that's harmless.)
          */
         if (surplusvars != NULL)
         {
             List       *new_groupby = NIL;
     
             foreach(lc, parse->groupClause)
             {
                 SortGroupClause *sgc = lfirst_node(SortGroupClause, lc);
                 TargetEntry *tle = get_sortgroupclause_tle(sgc, parse->targetList);
                 Var        *var = (Var *) tle->expr;
     
                 /*
                  * New list must include non-Vars, outer Vars, and anything not
                  * marked as surplus.
                  */
                 if (!IsA(var, Var) ||
                     var->varlevelsup > 0 ||
                     !bms_is_member(var->varattno - FirstLowInvalidHeapAttributeNumber,
                                    surplusvars[var->varno]))
                     new_groupby = lappend(new_groupby, sgc);
             }
     
             parse->groupClause = new_groupby;
         }
     }
    

### 三、参考资料

[planner.c](https://doxygen.postgresql.org/planner_8c_source.html)

