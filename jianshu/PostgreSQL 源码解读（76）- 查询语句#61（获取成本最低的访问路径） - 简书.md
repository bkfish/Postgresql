本节介绍了standard_planner函数中的fetch_upper_rel和get_cheapest_fractional_path函数，其中fetch_upper_rel函数构建用以表示查询优化器生成的最终关系（用RelOptInfo数据结构表示），get_cheapest_fractional_path函数根据输入的RelOptInfo数据结构找到成本最低的访问路径。

### 一、源码解读

**fetch_upper_rel**  
从优化器信息中的upper_rels数据中获取用以表示查询优化器生成的最终关系.

    
    
    //--------------------------------------------------------------------------- fetch_upper_rel
    
    /*
     * fetch_upper_rel
     *      Build a RelOptInfo describing some post-scan/join query processing,
     *      or return a pre-existing one if somebody already built it.
     *      构建RelOptInfo数据结构,用以表示post-scan/join查询过程.如已存在,则直接返回.
     *
     * An "upper" relation is identified by an UpperRelationKind and a Relids set.
     * The meaning of the Relids set is not specified here, and very likely will
     * vary for different relation kinds.
     * 一个“上级”关系是由一个UpperRelationKind和一个Relids集合来标识。
     * 在这里没有指定Relids集合的含义，并且很可能会因不同的关系类型而不同。
     *
     * Most of the fields in an upper-level RelOptInfo are not used and are not
     * set here (though makeNode should ensure they're zeroes).  We basically only
     * care about fields that are of interest to add_path() and set_cheapest().
     * 上层RelOptInfo中的大多数字段都没有使用，也没有在这里设置(尽管makeNode应该确保它们是NULL)。
     * 基本上只关心add_path()和set_cheap()函数所感兴趣的字段。
     */
    RelOptInfo *
    fetch_upper_rel(PlannerInfo *root, UpperRelationKind kind, Relids relids)
    {
        RelOptInfo *upperrel;
        ListCell   *lc;
    
        /*
         * For the moment, our indexing data structure is just a List for each
         * relation kind.  If we ever get so many of one kind that this stops
         * working well, we can improve it.  No code outside this function should
         * assume anything about how to find a particular upperrel.
         * 目前，我们已索引的数据结构只是每个关系类型的链表。
         * 这个函数之外的任何代码都不应该假定如何找到一个特定的上层关系。
         */
    
        /* If we already made this upperrel for the query, return it */
        //如果已经构造了该查询的上层关系,直接返回
        foreach(lc, root->upper_rels[kind])
        {
            upperrel = (RelOptInfo *) lfirst(lc);
    
            if (bms_equal(upperrel->relids, relids))
                return upperrel;
        }
    
        upperrel = makeNode(RelOptInfo);
        upperrel->reloptkind = RELOPT_UPPER_REL;
        upperrel->relids = bms_copy(relids);
    
        /* cheap startup cost is interesting iff not all tuples to be retrieved */
        //低廉的启动成本在较少的元组返回的情况是比较让人关心的.
        upperrel->consider_startup = (root->tuple_fraction > 0);//如非全部返回元组,则需要考虑启动成本
        upperrel->consider_param_startup = false;
        upperrel->consider_parallel = false;    /* 以后可能会有变化;might get changed later */
        upperrel->reltarget = create_empty_pathtarget();
        upperrel->pathlist = NIL;
        upperrel->cheapest_startup_path = NULL;
        upperrel->cheapest_total_path = NULL;
        upperrel->cheapest_unique_path = NULL;
        upperrel->cheapest_parameterized_paths = NIL;
    
        root->upper_rels[kind] = lappend(root->upper_rels[kind], upperrel);
    
        return upperrel;
    }
    

**get_cheapest_fractional_path**  
get_cheapest_fractional_path通过对RelOptInfo中的访问路径两两比较,获取成本最低的访问路径.

    
    
    //--------------------------------------------------------------------------- get_cheapest_fractional_path
    
    /*
     * get_cheapest_fractional_path
     *    Find the cheapest path for retrieving a specified fraction of all
     *    the tuples expected to be returned by the given relation.
     *    给定关系,找到成本最低的访问路径,该路径返回返回预期设定元组数。
     * 
     * We interpret tuple_fraction the same way as grouping_planner.
     * 我们用grouping_planner函数来获取tuple_fraction。
     *
     * We assume set_cheapest() has been run on the given rel.
     * 假定set_cheapest()函数已在给定的关系上执行.
     */
    Path *
    get_cheapest_fractional_path(RelOptInfo *rel, double tuple_fraction)
    {
        Path       *best_path = rel->cheapest_total_path;
        ListCell   *l;
    
        /* If all tuples will be retrieved, just return the cheapest-total path */
        //如果需要检索所有元组,则返回总成本最低的访问路径
        if (tuple_fraction <= 0.0)
            return best_path;
    
        /* Convert absolute # of tuples to a fraction; no need to clamp to 0..1 */
        //根据比例给出需返回行数
        if (tuple_fraction >= 1.0 && best_path->rows > 0)
            tuple_fraction /= best_path->rows;
    
        foreach(l, rel->pathlist)
        {
            Path       *path = (Path *) lfirst(l);
            //best_path的成本比path要低或相同,保留
            if (path == rel->cheapest_total_path ||
                compare_fractional_path_costs(best_path, path, tuple_fraction) <= 0)
                continue;
            //否则,选择新的访问路径
            best_path = path;
        }
    
        return best_path;
    }
    
    
    //------------------------------------------------------ compare_path_costs
    
     /*
      * compare_path_fractional_costs
      *    Return -1, 0, or +1 according as path1 is cheaper, the same cost,
      *    or more expensive than path2 for fetching the specified fraction
      *    of the total tuples.
      *    返回值:
      *    -1:相对于path2,path1成本更低;
      *     0:与path2成本相同
      *     1:比path2成本更高
      *
      * If fraction is <= 0 or > 1, we interpret it as 1, ie, we select the
      * path with the cheaper total_cost.
      * 如果fraction≤0或者大于1,则选择总成本最低的访问路径
      */
     int
     compare_fractional_path_costs(Path *path1, Path *path2,
                                   double fraction)
     {
         Cost        cost1,
                     cost2;
     
         if (fraction <= 0.0 || fraction >= 1.0)
             return compare_path_costs(path1, path2, TOTAL_COST);
         cost1 = path1->startup_cost +
             fraction * (path1->total_cost - path1->startup_cost);
         cost2 = path2->startup_cost +
             fraction * (path2->total_cost - path2->startup_cost);
         if (cost1 < cost2)
             return -1;
         if (cost1 > cost2)
             return +1;
         return 0;
     }
    
    //--------------------------------------- compare_path_costs
     /*
      * compare_path_costs
      *    Return -1, 0, or +1 according as path1 is cheaper, the same cost,
      *    or more expensive than path2 for the specified criterion.
      * 给定标准,返回比较结果.
      * 返回值:
      *    -1:相对于path2,path1成本更低;
      *     0:与path2成本相同
      *     1:比path2成本更高
      */
     int
     compare_path_costs(Path *path1, Path *path2, CostSelector criterion)
     {
         if (criterion == STARTUP_COST)//启动成本
         {
             if (path1->startup_cost < path2->startup_cost)
                 return -1;
             if (path1->startup_cost > path2->startup_cost)
                 return +1;
     
             /*
              * If paths have the same startup cost (not at all unlikely), order
              * them by total cost.
              */
             if (path1->total_cost < path2->total_cost)
                 return -1;
             if (path1->total_cost > path2->total_cost)
                 return +1;
         }
         else//总成本
         {
             if (path1->total_cost < path2->total_cost)
                 return -1;
             if (path1->total_cost > path2->total_cost)
                 return +1;
     
             /*
              * If paths have the same total cost, order them by startup cost.
              */
             if (path1->startup_cost < path2->startup_cost)
                 return -1;
             if (path1->startup_cost > path2->startup_cost)
                 return +1;
         }
         return 0;
     }
    
    

### 二、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

