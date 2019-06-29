本节介绍了PG如何为append
relation(分区表)构建访问路径，主要的实现逻辑在函数set_rel_pathlist中实现，该函数首先判断RTE是否分区表，如是则调用set_append_rel_pathlist函数对仍“存活”的子关系构建访问路径，然后把这些访问路径Merge到APPEND访问路径中。

### 一、数据结构

**AppendRelInfo**  
当我们将可继承表(分区表)或UNION-ALL子查询展开为“追加关系”(本质上是子RTE的链表)时，为每个子RTE构建一个AppendRelInfo。

    
    
    /*
     * Append-relation info.
     * Append-relation信息.
     * 
     * When we expand an inheritable table or a UNION-ALL subselect into an
     * "append relation" (essentially, a list of child RTEs), we build an
     * AppendRelInfo for each child RTE.  The list of AppendRelInfos indicates
     * which child RTEs must be included when expanding the parent, and each node
     * carries information needed to translate Vars referencing the parent into
     * Vars referencing that child.
     * 当我们将可继承表(分区表)或UNION-ALL子查询展开为“追加关系”(本质上是子RTE的链表)时，
     *   为每个子RTE构建一个AppendRelInfo。
     * AppendRelInfos链表指示在展开父节点时必须包含哪些子rte，
     *   每个节点具有将引用父节点的Vars转换为引用该子节点的Vars所需的所有信息。
     * 
     * These structs are kept in the PlannerInfo node's append_rel_list.
     * Note that we just throw all the structs into one list, and scan the
     * whole list when desiring to expand any one parent.  We could have used
     * a more complex data structure (eg, one list per parent), but this would
     * be harder to update during operations such as pulling up subqueries,
     * and not really any easier to scan.  Considering that typical queries
     * will not have many different append parents, it doesn't seem worthwhile
     * to complicate things.
     * 这些结构体保存在PlannerInfo节点的append_rel_list中。
     * 注意，只是将所有的结构体放入一个链表中，并在希望展开任何父类时扫描整个链表。
     * 本可以使用更复杂的数据结构(例如，每个父节点一个列表)，
     *   但是在提取子查询之类的操作中更新它会更困难，
     *   而且实际上也不会更容易扫描。
     * 考虑到典型的查询不会有很多不同的附加项，因此似乎不值得将事情复杂化。
     * 
     * Note: after completion of the planner prep phase, any given RTE is an
     * append parent having entries in append_rel_list if and only if its
     * "inh" flag is set.  We clear "inh" for plain tables that turn out not
     * to have inheritance children, and (in an abuse of the original meaning
     * of the flag) we set "inh" for subquery RTEs that turn out to be
     * flattenable UNION ALL queries.  This lets us avoid useless searches
     * of append_rel_list.
     * 注意:计划准备阶段完成后,
     *   当且仅当它的“inh”标志已设置时,给定的RTE是一个append parent在append_rel_list中的一个条目。
     * 我们为没有child的平面表清除“inh”标记,
     *   同时(有滥用标记的嫌疑)为UNION ALL查询中的子查询RTEs设置“inh”标记。
     * 这样可以避免对append_rel_list进行无用的搜索。
     * 
     * Note: the data structure assumes that append-rel members are single
     * baserels.  This is OK for inheritance, but it prevents us from pulling
     * up a UNION ALL member subquery if it contains a join.  While that could
     * be fixed with a more complex data structure, at present there's not much
     * point because no improvement in the plan could result.
     * 注意:数据结构假定附加的rel成员是独立的baserels。
     * 这对于继承来说是可以的，但是如果UNION ALL member子查询包含一个join，
     *   那么它将阻止我们提取UNION ALL member子查询。
     * 虽然可以用更复杂的数据结构解决这个问题，但目前没有太大意义，因为该计划可能不会有任何改进。
     */
    
    typedef struct AppendRelInfo
    {
        NodeTag     type;
    
        /*
         * These fields uniquely identify this append relationship.  There can be
         * (in fact, always should be) multiple AppendRelInfos for the same
         * parent_relid, but never more than one per child_relid, since a given
         * RTE cannot be a child of more than one append parent.
         * 这些字段惟一地标识这个append relationship。
         * 对于同一个parent_relid可以有(实际上应该总是)多个AppendRelInfos，
         *   但是每个child_relid不能有多个AppendRelInfos，
         *   因为给定的RTE不能是多个append parent的子节点。
         */
        Index       parent_relid;   /* parent rel的RT索引;RT index of append parent rel */
        Index       child_relid;    /* child rel的RT索引;RT index of append child rel */
    
        /*
         * For an inheritance appendrel, the parent and child are both regular
         * relations, and we store their rowtype OIDs here for use in translating
         * whole-row Vars.  For a UNION-ALL appendrel, the parent and child are
         * both subqueries with no named rowtype, and we store InvalidOid here.
         * 对于继承appendrel，父类和子类都是普通关系，
         *   我们将它们的rowtype OIDs存储在这里，用于转换whole-row Vars。
         * 对于UNION-ALL appendrel，父查询和子查询都是没有指定行类型的子查询，
         * 我们在这里存储InvalidOid。
         */
        Oid         parent_reltype; /* OID of parent's composite type */
        Oid         child_reltype;  /* OID of child's composite type */
    
        /*
         * The N'th element of this list is a Var or expression representing the
         * child column corresponding to the N'th column of the parent. This is
         * used to translate Vars referencing the parent rel into references to
         * the child.  A list element is NULL if it corresponds to a dropped
         * column of the parent (this is only possible for inheritance cases, not
         * UNION ALL).  The list elements are always simple Vars for inheritance
         * cases, but can be arbitrary expressions in UNION ALL cases.
         * 这个列表的第N个元素是一个Var或表达式，表示与父元素的第N列对应的子列。
         * 这用于将引用parent rel的Vars转换为对子rel的引用。
         * 如果链表元素与父元素的已删除列相对应，则该元素为NULL
         *   (这只适用于继承情况，而不是UNION ALL)。
         * 对于继承情况，链表元素总是简单的变量，但是可以是UNION ALL情况下的任意表达式。
         *
         * Notice we only store entries for user columns (attno > 0).  Whole-row
         * Vars are special-cased, and system columns (attno < 0) need no special
         * translation since their attnos are the same for all tables.
         * 注意，我们只存储用户列的条目(attno > 0)。
         * Whole-row Vars是大小写敏感的，系统列(attno < 0)不需要特别的转换，
         *   因为它们的attno对所有表都是相同的。
         *
         * Caution: the Vars have varlevelsup = 0.  Be careful to adjust as needed
         * when copying into a subquery.
         * 注意:Vars的varlevelsup = 0。
         * 在将数据复制到子查询时，要注意根据需要进行调整。
         */
        //child's Vars中的表达式
        List       *translated_vars;    /* Expressions in the child's Vars */
    
        /*
         * We store the parent table's OID here for inheritance, or InvalidOid for
         * UNION ALL.  This is only needed to help in generating error messages if
         * an attempt is made to reference a dropped parent column.
         * 我们将父表的OID存储在这里用于继承，
         *   如为UNION ALL,则这里存储的是InvalidOid。
         * 只有在试图引用已删除的父列时，才需要这样做来帮助生成错误消息。
         */
        Oid         parent_reloid;  /* OID of parent relation */
    } AppendRelInfo;
    
    

**RelOptInfo**  
规划器/优化器使用的关系信息结构体  
参见[PostgreSQL 源码解读（99）-分区表#5（数据查询路由#2-RelOptInfo数据结构）](https://www.jianshu.com/p/ab982c65a65b)

### 二、源码解读

set_rel_pathlist函数为基础关系构建访问路径.

    
    
    //是否DUMMY访问路径
    #define IS_DUMMY_PATH(p) \
        (IsA((p), AppendPath) && ((AppendPath *) (p))->subpaths == NIL)
    
    /* A relation that's been proven empty will have one path that is dummy */
    //已被证明为空的关系会含有一个虚拟dummy的访问路径
    #define IS_DUMMY_REL(r) \
        ((r)->cheapest_total_path != NULL && \
         IS_DUMMY_PATH((r)->cheapest_total_path))
    
    
    /*
     * set_rel_pathlist
     *    Build access paths for a base relation
     *    为基础关系构建访问路径
     */
    static void
    set_rel_pathlist(PlannerInfo *root, RelOptInfo *rel,
                     Index rti, RangeTblEntry *rte)
    {
        if (IS_DUMMY_REL(rel))
        {
            /* We already proved the relation empty, so nothing more to do */
            //不需要做任何处理
        }
        else if (rte->inh)
        {
            /* It's an "append relation", process accordingly */
            //append relation,调用set_append_rel_pathlist处理
            set_append_rel_pathlist(root, rel, rti, rte);
        }
        else
        {//其他类型的关系
            switch (rel->rtekind)
            {
                case RTE_RELATION://基础关系
                    if (rte->relkind == RELKIND_FOREIGN_TABLE)
                    {
                        /* Foreign table */
                        //外部表
                        set_foreign_pathlist(root, rel, rte);
                    }
                    else if (rte->tablesample != NULL)
                    {
                        /* Sampled relation */
                        //数据表采样
                        set_tablesample_rel_pathlist(root, rel, rte);
                    }
                    else
                    {
                        /* Plain relation */
                        //常规关系
                        set_plain_rel_pathlist(root, rel, rte);
                    }
                    break;
                case RTE_SUBQUERY:
                    /* Subquery --- fully handled during set_rel_size */
                    //子查询
                    break;
                case RTE_FUNCTION:
                    /* RangeFunction */
                    set_function_pathlist(root, rel, rte);
                    break;
                case RTE_TABLEFUNC:
                    /* Table Function */
                    set_tablefunc_pathlist(root, rel, rte);
                    break;
                case RTE_VALUES:
                    /* Values list */
                    set_values_pathlist(root, rel, rte);
                    break;
                case RTE_CTE:
                    /* CTE reference --- fully handled during set_rel_size */
                    break;
                case RTE_NAMEDTUPLESTORE:
                    /* tuplestore reference --- fully handled during set_rel_size */
                    break;
                default:
                    elog(ERROR, "unexpected rtekind: %d", (int) rel->rtekind);
                    break;
            }
        }
    
        /*
         * If this is a baserel, we should normally consider gathering any partial
         * paths we may have created for it.
         * 如为基础关系,通常来说需要考虑聚集gathering所有的部分路径(先前已构建)
         *
         * However, if this is an inheritance child, skip it.  Otherwise, we could
         * end up with a very large number of gather nodes, each trying to grab
         * its own pool of workers.  Instead, we'll consider gathering partial
         * paths for the parent appendrel.
         * 但是,如果这是一个继承关系中的子表,忽略它.
         * 否则的话,我们最好可能会有很大量的Gather节点,每一个都尝试grab它自己worker的输出.
         * 相反我们的处理是为父append relation生成gathering部分访问路径.
         *
         * Also, if this is the topmost scan/join rel (that is, the only baserel),
         * we postpone this until the final scan/join targelist is available (see
         * grouping_planner).
         * 另外,如果这是最高层的scan/join关系(基础关系),
         *   在完成了最后的scan/join投影列处理后才进行相应的处理.
         */
        if (rel->reloptkind == RELOPT_BASEREL &&
            bms_membership(root->all_baserels) != BMS_SINGLETON)
            generate_gather_paths(root, rel, false);
    
        /*
         * Allow a plugin to editorialize on the set of Paths for this base
         * relation.  It could add new paths (such as CustomPaths) by calling
         * add_path(), or delete or modify paths added by the core code.
         * 调用钩子函数.
         * 插件可以通过调用add_path添加新的访问路径(比如CustomPaths等),
         *   或者删除/修改核心代码生成的访问路径.
         */
        if (set_rel_pathlist_hook)
            (*set_rel_pathlist_hook) (root, rel, rti, rte);
    
        /* Now find the cheapest of the paths for this rel */
        //为rel找到成本最低的访问路径
        set_cheapest(rel);
    
    #ifdef OPTIMIZER_DEBUG
        debug_print_rel(root, rel);
    #endif
    }
    
    
     /*
     * set_append_rel_pathlist
     *    Build access paths for an "append relation"
     *    为append relation构建访问路径
     */
    static void
    set_append_rel_pathlist(PlannerInfo *root, RelOptInfo *rel,
                            Index rti, RangeTblEntry *rte)
    {
        int         parentRTindex = rti;
        List       *live_childrels = NIL;
        ListCell   *l;
    
        /*
         * Generate access paths for each member relation, and remember the
         * non-dummy children.
         * 为每个成员关系生成访问路径，并"记住"非虚拟子节点。
         */
        foreach(l, root->append_rel_list)//遍历链表
        {
            AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(l);//获取AppendRelInfo
            int         childRTindex;
            RangeTblEntry *childRTE;
            RelOptInfo *childrel;
    
            /* append_rel_list contains all append rels; ignore others */
            //append_rel_list含有所有的append relations,忽略其他的rels
            if (appinfo->parent_relid != parentRTindex)
                continue;
    
            /* Re-locate the child RTE and RelOptInfo */
            //重新定位子RTE和RelOptInfo
            childRTindex = appinfo->child_relid;
            childRTE = root->simple_rte_array[childRTindex];
            childrel = root->simple_rel_array[childRTindex];
    
            /*
             * If set_append_rel_size() decided the parent appendrel was
             * parallel-unsafe at some point after visiting this child rel, we
             * need to propagate the unsafety marking down to the child, so that
             * we don't generate useless partial paths for it.
             * 如果set_append_rel_size()函数在访问了子关系后,
             *   在某些点上断定父append relation是非并行安全的,
             *   我们需要分发不安全的标记到子关系中,以免产生无用的部分访问路径.
             */
            if (!rel->consider_parallel)
                childrel->consider_parallel = false;
    
            /*
             * Compute the child's access paths.
             * "计算"子关系的访问路径
             */
            set_rel_pathlist(root, childrel, childRTindex, childRTE);
    
            /*
             * If child is dummy, ignore it.
             * 如为虚拟关系,忽略之
             */
            if (IS_DUMMY_REL(childrel))
                continue;
    
            /* Bubble up childrel's partitioned children. */
            //
            if (rel->part_scheme)
                rel->partitioned_child_rels =
                    list_concat(rel->partitioned_child_rels,
                                list_copy(childrel->partitioned_child_rels));
    
            /*
             * Child is live, so add it to the live_childrels list for use below.
             * 添加到live_childrels链表中
             */
            live_childrels = lappend(live_childrels, childrel);
        }
    
        /* Add paths to the append relation. */
        //添加访问路径到append relation中
        add_paths_to_append_rel(root, rel, live_childrels);
    }
    
    
    /*
     * add_paths_to_append_rel
     *      Generate paths for the given append relation given the set of non-dummy
     *      child rels.
     *      基于非虚拟子关系集合为给定的append relation生成访问路径
     *
     * The function collects all parameterizations and orderings supported by the
     * non-dummy children. For every such parameterization or ordering, it creates
     * an append path collecting one path from each non-dummy child with given
     * parameterization or ordering. Similarly it collects partial paths from
     * non-dummy children to create partial append paths.
     * 该函数收集所有的非虚拟关系支持的参数化和排序访问路径.
     * 对于每一个参数化或排序的访问路径,它创建一个附加路径，
     *   从每个具有给定参数化或排序的非虚拟子节点收集相关信息。
     * 类似的,从非虚拟子关系中收集部分访问路径用以创建部分append路径.
     */
    void
    add_paths_to_append_rel(PlannerInfo *root, RelOptInfo *rel,
                            List *live_childrels)
    {
        List       *subpaths = NIL;
        bool        subpaths_valid = true;
        List       *partial_subpaths = NIL;
        List       *pa_partial_subpaths = NIL;
        List       *pa_nonpartial_subpaths = NIL;
        bool        partial_subpaths_valid = true;
        bool        pa_subpaths_valid;
        List       *all_child_pathkeys = NIL;
        List       *all_child_outers = NIL;
        ListCell   *l;
        List       *partitioned_rels = NIL;
        double      partial_rows = -1;
    
        /* If appropriate, consider parallel append */
        //如可以,考虑并行append
        pa_subpaths_valid = enable_parallel_append && rel->consider_parallel;
    
        /*
         * AppendPath generated for partitioned tables must record the RT indexes
         * of partitioned tables that are direct or indirect children of this
         * Append rel.
         * 为分区表生成的AppendPath必须记录属于该分区表(Append Rel)的直接或间接子关系的RT索引
         *
         * AppendPath may be for a sub-query RTE (UNION ALL), in which case, 'rel'
         * itself does not represent a partitioned relation, but the child sub-         * queries may contain references to partitioned relations.  The loop
         * below will look for such children and collect them in a list to be
         * passed to the path creation function.  (This assumes that we don't need
         * to look through multiple levels of subquery RTEs; if we ever do, we
         * could consider stuffing the list we generate here into sub-query RTE's
         * RelOptInfo, just like we do for partitioned rels, which would be used
         * when populating our parent rel with paths.  For the present, that
         * appears to be unnecessary.)
         * AppendPath可能是子查询RTE(UNION ALL),在这种情况下,rel自身并不代表分区表,
         *   但child sub-queries可能含有分区表的依赖.
         * 以下的循环会寻找这样的子关系,存储在链表中,并作为参数传给访问路径创建函数.
         * (这是假设我们不需要遍历多个层次的子查询RTEs,如果我们曾经这样做了,
         *  这好比分区表的做法,可以考虑把生成的链表放到子查询的RTE's RelOptInfo结构体中,
         *  用于使用访问路径填充父关系.不过目前来说,这样做是不需要的.)
         */
        if (rel->part_scheme != NULL)
        {
            if (IS_SIMPLE_REL(rel))
                partitioned_rels = list_make1(rel->partitioned_child_rels);
            else if (IS_JOIN_REL(rel))
            {
                int         relid = -1;
                List       *partrels = NIL;
    
                /*
                 * For a partitioned joinrel, concatenate the component rels'
                 * partitioned_child_rels lists.
                 * 对于分区连接rel,连接rels的partitioned_child_rels链表
                 *
                 */
                while ((relid = bms_next_member(rel->relids, relid)) >= 0)
                {
                    RelOptInfo *component;
    
                    Assert(relid >= 1 && relid < root->simple_rel_array_size);
                    component = root->simple_rel_array[relid];
                    Assert(component->part_scheme != NULL);
                    Assert(list_length(component->partitioned_child_rels) >= 1);
                    partrels =
                        list_concat(partrels,
                                    list_copy(component->partitioned_child_rels));
                }
    
                partitioned_rels = list_make1(partrels);
            }
    
            Assert(list_length(partitioned_rels) >= 1);
        }
    
        /*
         * For every non-dummy child, remember the cheapest path.  Also, identify
         * all pathkeys (orderings) and parameterizations (required_outer sets)
         * available for the non-dummy member relations.
         * 对于每一个非虚拟子关系,记录成本最低的访问路径.
         * 同时,为每一个非虚拟成员关系标识所有的路径键(排序)和可用的参数化信息(required_outer集合)
         */
        foreach(l, live_childrels)//遍历
        {
            RelOptInfo *childrel = lfirst(l);
            ListCell   *lcp;
            Path       *cheapest_partial_path = NULL;
    
            /*
             * For UNION ALLs with non-empty partitioned_child_rels, accumulate
             * the Lists of child relations.
             * 对于包含非空的partitioned_child_rels的UNION ALLs操作,
             *   累积子关系链表
             */
            if (rel->rtekind == RTE_SUBQUERY && childrel->partitioned_child_rels != NIL)
                partitioned_rels = lappend(partitioned_rels,
                                           childrel->partitioned_child_rels);
    
            /*
             * If child has an unparameterized cheapest-total path, add that to
             * the unparameterized Append path we are constructing for the parent.
             * If not, there's no workable unparameterized path.
             * 如果子关系存在非参数化的总成本最低的访问路径,
             *   添加此路径到我们为父关系构建的非参数化的Append访问路径中.
             *
             * With partitionwise aggregates, the child rel's pathlist may be
             * empty, so don't assume that a path exists here.
             * 使用partitionwise聚合,子关系的访问路径链表可能是空的,不能假设其中存在访问路径
             */
            if (childrel->pathlist != NIL &&
                childrel->cheapest_total_path->param_info == NULL)
                accumulate_append_subpath(childrel->cheapest_total_path,
                                          &subpaths, NULL);
            else
                subpaths_valid = false;
    
            /* Same idea, but for a partial plan. */
            //同样的思路,处理并行处理中的部分计划
            if (childrel->partial_pathlist != NIL)
            {
                cheapest_partial_path = linitial(childrel->partial_pathlist);
                accumulate_append_subpath(cheapest_partial_path,
                                          &partial_subpaths, NULL);
            }
            else
                partial_subpaths_valid = false;
    
            /*
             * Same idea, but for a parallel append mixing partial and non-partial
             * paths.
             * 同样的,处理并行append混合并行/非并行访问路径
             */
            if (pa_subpaths_valid)
            {
                Path       *nppath = NULL;
    
                nppath =
                    get_cheapest_parallel_safe_total_inner(childrel->pathlist);
    
                if (cheapest_partial_path == NULL && nppath == NULL)
                {
                    /* Neither a partial nor a parallel-safe path?  Forget it. */
                    //不是部分路径,也不是并行安全的路径,跳过
                    pa_subpaths_valid = false;
                }
                else if (nppath == NULL ||
                         (cheapest_partial_path != NULL &&
                          cheapest_partial_path->total_cost < nppath->total_cost))
                {
                    /* Partial path is cheaper or the only option. */
                    //部分路径成本更低或者是唯一的选项
                    Assert(cheapest_partial_path != NULL);
                    accumulate_append_subpath(cheapest_partial_path,
                                              &pa_partial_subpaths,
                                              &pa_nonpartial_subpaths);
    
                }
                else
                {
                    /*
                     * Either we've got only a non-partial path, or we think that
                     * a single backend can execute the best non-partial path
                     * faster than all the parallel backends working together can
                     * execute the best partial path.
                     * 这时候,要么得到了一个非并行访问路径,或者我们认为一个单独的后台进程
                     *   执行最好的非并行访问路径会比索引的并行进程一起执行最好的部分路径还要好.
                     *
                     * It might make sense to be more aggressive here.  Even if
                     * the best non-partial path is more expensive than the best
                     * partial path, it could still be better to choose the
                     * non-partial path if there are several such paths that can
                     * be given to different workers.  For now, we don't try to
                     * figure that out.
                     * 在这里,采取更积极的态度是有道理的.
                     * 甚至最好的非部分路径比最好的并行部分路径成本更高,仍然需要选择非并行路径,
                     *   如果多个这样的路径可能会分派到不同的worker上.
                     * 现在不需要指出这一点.
                     */
                    accumulate_append_subpath(nppath,
                                              &pa_nonpartial_subpaths,
                                              NULL);
                }
            }
    
            /*
             * Collect lists of all the available path orderings and
             * parameterizations for all the children.  We use these as a
             * heuristic to indicate which sort orderings and parameterizations we
             * should build Append and MergeAppend paths for.
             * 收集子关系所有可用的排序和参数化路径链表.
             * 我们采用启发式的规则判断Append和MergeAppend访问路径使用哪个排序和参数化信息
             */
            foreach(lcp, childrel->pathlist)
            {
                Path       *childpath = (Path *) lfirst(lcp);
                List       *childkeys = childpath->pathkeys;
                Relids      childouter = PATH_REQ_OUTER(childpath);
    
                /* Unsorted paths don't contribute to pathkey list */
                //未排序的访问路径,不需要分发到路径键链表中
                if (childkeys != NIL)
                {
                    ListCell   *lpk;
                    bool        found = false;
    
                    /* Have we already seen this ordering? */
                    foreach(lpk, all_child_pathkeys)
                    {
                        List       *existing_pathkeys = (List *) lfirst(lpk);
    
                        if (compare_pathkeys(existing_pathkeys,
                                             childkeys) == PATHKEYS_EQUAL)
                        {
                            found = true;
                            break;
                        }
                    }
                    if (!found)
                    {
                        /* No, so add it to all_child_pathkeys */
                        all_child_pathkeys = lappend(all_child_pathkeys,
                                                     childkeys);
                    }
                }
    
                /* Unparameterized paths don't contribute to param-set list */
                //非参数访问路径无需分发到参数化集合链表中
                if (childouter)
                {
                    ListCell   *lco;
                    bool        found = false;
    
                    /* Have we already seen this param set? */
                    foreach(lco, all_child_outers)
                    {
                        Relids      existing_outers = (Relids) lfirst(lco);
    
                        if (bms_equal(existing_outers, childouter))
                        {
                            found = true;
                            break;
                        }
                    }
                    if (!found)
                    {
                        /* No, so add it to all_child_outers */
                        all_child_outers = lappend(all_child_outers,
                                                   childouter);
                    }
                }
            }
        }
    
        /*
         * If we found unparameterized paths for all children, build an unordered,
         * unparameterized Append path for the rel.  (Note: this is correct even
         * if we have zero or one live subpath due to constraint exclusion.)
         * 如存在子关系的非参数化访问路径,构建未排序/未参数化的Append访问路径.
         * (注意:如果存在约束排除,我们只剩下有0或1个存活的subpath,这样的处理也说OK的)
         */
        if (subpaths_valid)
            add_path(rel, (Path *) create_append_path(root, rel, subpaths, NIL,
                                                      NULL, 0, false,
                                                      partitioned_rels, -1));
    
        /*
         * Consider an append of unordered, unparameterized partial paths.  Make
         * it parallel-aware if possible.
         * 尝试未排序/未参数化的部分Append访问路径.
         * 如可能,构建parallel-aware访问路径.
         */
        if (partial_subpaths_valid)
        {
            AppendPath *appendpath;
            ListCell   *lc;
            int         parallel_workers = 0;
    
            /* Find the highest number of workers requested for any subpath. */
            //为子访问路径寻找最多数量的wokers
            foreach(lc, partial_subpaths)
            {
                Path       *path = lfirst(lc);
    
                parallel_workers = Max(parallel_workers, path->parallel_workers);
            }
            Assert(parallel_workers > 0);
    
            /*
             * If the use of parallel append is permitted, always request at least
             * log2(# of children) workers.  We assume it can be useful to have
             * extra workers in this case because they will be spread out across
             * the children.  The precise formula is just a guess, but we don't
             * want to end up with a radically different answer for a table with N
             * partitions vs. an unpartitioned table with the same data, so the
             * use of some kind of log-scaling here seems to make some sense.
             * 如允许使用并行append,那么至少需要log2(子关系个数)个workers.
             * 经过扩展后,假定可以使用额外的workers.
             * 并不存在精确的计算公式,目前只是猜测而已,但是对于有相同数据的N个分区的分区表和非分区表来说,
             *   答案是不一样的,因此在这里使用对数的计算方法是OK的.
             */
            if (enable_parallel_append)
            {
                parallel_workers = Max(parallel_workers,
                                       fls(list_length(live_childrels)));//上限值
                parallel_workers = Min(parallel_workers,
                                       max_parallel_workers_per_gather);//下限值
            }
            Assert(parallel_workers > 0);
    
            /* Generate a partial append path. */
            //生成并行部分append访问路径
            appendpath = create_append_path(root, rel, NIL, partial_subpaths,
                                            NULL, parallel_workers,
                                            enable_parallel_append,
                                            partitioned_rels, -1);
    
            /*
             * Make sure any subsequent partial paths use the same row count
             * estimate.
             * 确保所有的子部分路径使用相同的行数估算
             */
            partial_rows = appendpath->path.rows;
    
            /* Add the path. */
            //添加路径
            add_partial_path(rel, (Path *) appendpath);
        }
    
        /*
         * Consider a parallel-aware append using a mix of partial and non-partial
         * paths.  (This only makes sense if there's at least one child which has
         * a non-partial path that is substantially cheaper than any partial path;
         * otherwise, we should use the append path added in the previous step.)
         * 使用混合的部分和非部分并行的append.
         */
        if (pa_subpaths_valid && pa_nonpartial_subpaths != NIL)
        {
            AppendPath *appendpath;
            ListCell   *lc;
            int         parallel_workers = 0;
    
            /*
             * Find the highest number of workers requested for any partial
             * subpath.
             */
            foreach(lc, pa_partial_subpaths)
            {
                Path       *path = lfirst(lc);
    
                parallel_workers = Max(parallel_workers, path->parallel_workers);
            }
    
            /*
             * Same formula here as above.  It's even more important in this
             * instance because the non-partial paths won't contribute anything to
             * the planned number of parallel workers.
             */
            parallel_workers = Max(parallel_workers,
                                   fls(list_length(live_childrels)));
            parallel_workers = Min(parallel_workers,
                                   max_parallel_workers_per_gather);
            Assert(parallel_workers > 0);
    
            appendpath = create_append_path(root, rel, pa_nonpartial_subpaths,
                                            pa_partial_subpaths,
                                            NULL, parallel_workers, true,
                                            partitioned_rels, partial_rows);
            add_partial_path(rel, (Path *) appendpath);
        }
    
        /*
         * Also build unparameterized MergeAppend paths based on the collected
         * list of child pathkeys.
         * 基于收集的子路径键,构建非参数化的MergeAppend访问路径
         */
        if (subpaths_valid)
            generate_mergeappend_paths(root, rel, live_childrels,
                                       all_child_pathkeys,
                                       partitioned_rels);
    
        /*
         * Build Append paths for each parameterization seen among the child rels.
         * (This may look pretty expensive, but in most cases of practical
         * interest, the child rels will expose mostly the same parameterizations,
         * so that not that many cases actually get considered here.)
         * 为每一个参数化的子关系构建Append访问路径.
         * (看起来成本客观,但在大多数情况下,子关系使用同样的参数化信息,因此实际上并不是经常发现)
         *
         * The Append node itself cannot enforce quals, so all qual checking must
         * be done in the child paths.  This means that to have a parameterized
         * Append path, we must have the exact same parameterization for each
         * child path; otherwise some children might be failing to check the
         * moved-down quals.  To make them match up, we can try to increase the
         * parameterization of lesser-parameterized paths.
         */
        foreach(l, all_child_outers)
        {
            Relids      required_outer = (Relids) lfirst(l);
            ListCell   *lcr;
    
            /* Select the child paths for an Append with this parameterization */
            subpaths = NIL;
            subpaths_valid = true;
            foreach(lcr, live_childrels)
            {
                RelOptInfo *childrel = (RelOptInfo *) lfirst(lcr);
                Path       *subpath;
    
                if (childrel->pathlist == NIL)
                {
                    /* failed to make a suitable path for this child */
                    subpaths_valid = false;
                    break;
                }
    
                subpath = get_cheapest_parameterized_child_path(root,
                                                                childrel,
                                                                required_outer);
                if (subpath == NULL)
                {
                    /* failed to make a suitable path for this child */
                    subpaths_valid = false;
                    break;
                }
                accumulate_append_subpath(subpath, &subpaths, NULL);
            }
    
            if (subpaths_valid)
                add_path(rel, (Path *)
                         create_append_path(root, rel, subpaths, NIL,
                                            required_outer, 0, false,
                                            partitioned_rels, -1));
        }
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain verbose select * from t_hash_partition where c1 = 1 OR c1 = 2;
                                         QUERY PLAN                                      
    -------------------------------------------------------------------------------------     Append  (cost=0.00..30.53 rows=6 width=200)
       ->  Seq Scan on public.t_hash_partition_1  (cost=0.00..15.25 rows=3 width=200)
             Output: t_hash_partition_1.c1, t_hash_partition_1.c2, t_hash_partition_1.c3
             Filter: ((t_hash_partition_1.c1 = 1) OR (t_hash_partition_1.c1 = 2))
       ->  Seq Scan on public.t_hash_partition_3  (cost=0.00..15.25 rows=3 width=200)
             Output: t_hash_partition_3.c1, t_hash_partition_3.c2, t_hash_partition_3.c3
             Filter: ((t_hash_partition_3.c1 = 1) OR (t_hash_partition_3.c1 = 2))
    (7 rows)
    

启动gdb,设置断点

    
    
    (gdb) b set_rel_pathlist
    Breakpoint 1 at 0x796823: file allpaths.c, line 425.
    (gdb) c
    Continuing.
    
    Breakpoint 1, set_rel_pathlist (root=0x1f1e400, rel=0x1efaba0, rti=1, rte=0x1efa3d0) at allpaths.c:425
    425     if (IS_DUMMY_REL(rel))
    (gdb) 
    

通过rte->inh判断是否分区表或者UNION ALL

    
    
    (gdb) p rte->inh
    $1 = true
    (gdb) 
    

进入set_append_rel_pathlist函数

    
    
    (gdb) n
    429     else if (rte->inh)
    (gdb) 
    432         set_append_rel_pathlist(root, rel, rti, rte);
    (gdb) step
    set_append_rel_pathlist (root=0x1f1e400, rel=0x1efaba0, rti=1, rte=0x1efa3d0) at allpaths.c:1296
    1296        int         parentRTindex = rti;
    

遍历子关系

    
    
    (gdb) n
    1297        List       *live_childrels = NIL;
    (gdb) 
    1304        foreach(l, root->append_rel_list)
    (gdb) 
    (gdb) p *root->append_rel_list
    $2 = {type = T_List, length = 6, head = 0x1fc1f98, tail = 0x1fc2ae8}
    

获取AppendRelInfo,判断父关系是否正在处理的父关系

    
    
    (gdb) n
    1306            AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(l);
    (gdb) 
    1312            if (appinfo->parent_relid != parentRTindex)
    (gdb) p *appinfo
    $3 = {type = T_AppendRelInfo, parent_relid = 1, child_relid = 3, parent_reltype = 16988, child_reltype = 16991, 
      translated_vars = 0x1fc1e60, parent_reloid = 16986}
    (gdb) 
    

获取子关系的相关信息,递归调用set_rel_pathlist

    
    
    (gdb) n
    1316            childRTindex = appinfo->child_relid;
    (gdb) 
    1317            childRTE = root->simple_rte_array[childRTindex];
    (gdb) 
    1318            childrel = root->simple_rel_array[childRTindex];
    (gdb) 
    1326            if (!rel->consider_parallel)
    (gdb) 
    1332            set_rel_pathlist(root, childrel, childRTindex, childRTE);
    (gdb) 
    (gdb) 
    
    Breakpoint 1, set_rel_pathlist (root=0x1f1e400, rel=0x1f1c8a0, rti=3, rte=0x1efa658) at allpaths.c:425
    425     if (IS_DUMMY_REL(rel))
    (gdb) finish
    Run till exit from #0  set_rel_pathlist (root=0x1f1e400, rel=0x1f1c8a0, rti=3, rte=0x1efa658) at allpaths.c:425
    set_append_rel_pathlist (root=0x1f1e400, rel=0x1efaba0, rti=1, rte=0x1efa3d0) at allpaths.c:1337
    

如为虚拟关系,则忽略之

    
    
    1337            if (IS_DUMMY_REL(childrel))
    
    

该子关系不是虚拟关系,继续处理,加入到rel->partitioned_child_rels和live_childrels链表中

    
    
    (gdb) n
    1341            if (rel->part_scheme)
    (gdb) 
    1344                                list_copy(childrel->partitioned_child_rels));
    (gdb) 
    1343                    list_concat(rel->partitioned_child_rels,
    (gdb) 
    1342                rel->partitioned_child_rels =
    (gdb) 
    1349            live_childrels = lappend(live_childrels, childrel);
    (gdb) p *rel->partitioned_child_rels
    $4 = {type = T_IntList, length = 1, head = 0x1fc4d78, tail = 0x1fc4d78}
    (gdb) p rel->partitioned_child_rels->head->data.int_value
    $6 = 1
    (gdb) 
    (gdb) n
    1304        foreach(l, root->append_rel_list)
    (gdb) p live_childrels
    $7 = (List *) 0x1fd0a60
    (gdb) p *live_childrels
    $8 = {type = T_List, length = 1, head = 0x1fd0a38, tail = 0x1fd0a38}
    (gdb) p *(Node *)live_childrels->head->data.ptr_value
    $9 = {type = T_RelOptInfo}
    (gdb) p *(RelOptInfo *)live_childrels->head->data.ptr_value
    $10 = {type = T_RelOptInfo, reloptkind = RELOPT_OTHER_MEMBER_REL, relids = 0x1fc3590, rows = 3, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x1fc35b0, pathlist = 0x1fd0940, ppilist = 0x0, 
      partial_pathlist = 0x1fd09a0, cheapest_startup_path = 0x1fc44f8, cheapest_total_path = 0x1fc44f8, 
      cheapest_unique_path = 0x0, cheapest_parameterized_paths = 0x1fd0a00, direct_lateral_relids = 0x0, lateral_relids = 0x0, 
      relid = 3, reltablespace = 0, rtekind = RTE_RELATION, min_attr = -7, max_attr = 3, attr_needed = 0x1fc2e38, 
      attr_widths = 0x1fc3628, lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 10, 
      tuples = 350, allvisfrac = 0, subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, 
      useridiscurrent = false, fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, 
      baserestrictinfo = 0x1fc68d8, baserestrictcost = {startup = 0, per_tuple = 0.0050000000000000001}, 
      baserestrict_min_security = 0, joininfo = 0x0, has_eclass_joins = false, consider_partitionwise_join = false, 
      top_parent_relids = 0x1fc3608, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, 
      partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    

对于虚拟子关系(上一节介绍的被裁剪的分区),直接跳过

    
    
    (gdb) n
    1306            AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(l);
    (gdb) 
    1312            if (appinfo->parent_relid != parentRTindex)
    (gdb) 
    1316            childRTindex = appinfo->child_relid;
    (gdb) 
    1317            childRTE = root->simple_rte_array[childRTindex];
    (gdb) 
    1318            childrel = root->simple_rel_array[childRTindex];
    (gdb) 
    1326            if (!rel->consider_parallel)
    (gdb) 
    1332            set_rel_pathlist(root, childrel, childRTindex, childRTE);
    (gdb) 
    1337            if (IS_DUMMY_REL(childrel))
    (gdb) 
    1338                continue;
    

设置断点,进入add_paths_to_append_rel函数

    
    
    (gdb) b add_paths_to_append_rel
    Breakpoint 2 at 0x797d88: file allpaths.c, line 1372.
    (gdb) c
    Continuing.
    
    Breakpoint 2, add_paths_to_append_rel (root=0x1f1cdb8, rel=0x1fc1800, live_childrels=0x1fcfb10) at allpaths.c:1372
    1372        List       *subpaths = NIL;
    (gdb) 
    

输入参数,其中rel是父关系,live_childrels是经裁剪后仍存活的分区(子关系)

    
    
    (gdb) n
    1373        bool        subpaths_valid = true;
    (gdb) p *rel
    $11 = {type = T_RelOptInfo, reloptkind = RELOPT_BASEREL, relids = 0x1fc1a18, rows = 6, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x1fc1a38, pathlist = 0x0, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 1, reltablespace = 0, 
      rtekind = RTE_RELATION, min_attr = -7, max_attr = 3, attr_needed = 0x1fc1a90, attr_widths = 0x1fc1b28, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 6, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x1fc3e20, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 0, joininfo = 0x0, has_eclass_joins = false, 
      consider_partitionwise_join = false, top_parent_relids = 0x0, part_scheme = 0x1fc1b80, nparts = 6, boundinfo = 0x1fc1d30, 
      partition_qual = 0x0, part_rels = 0x1fc2000, partexprs = 0x1fc1f08, nullable_partexprs = 0x1fc1fe0, 
      partitioned_child_rels = 0x1fc3ea0}
    

初始化变量

    
    
    (gdb) n
    1374        List       *partial_subpaths = NIL;
    (gdb) 
    1375        List       *pa_partial_subpaths = NIL;
    (gdb) 
    1376        List       *pa_nonpartial_subpaths = NIL;
    (gdb) 
    1377        bool        partial_subpaths_valid = true;
    (gdb) 
    1379        List       *all_child_pathkeys = NIL;
    (gdb) 
    1380        List       *all_child_outers = NIL;
    (gdb) 
    1382        List       *partitioned_rels = NIL;
    (gdb) 
    1383        double      partial_rows = -1;
    (gdb) 
    1386        pa_subpaths_valid = enable_parallel_append && rel->consider_parallel;
    (gdb) 
    1404        if (rel->part_scheme != NULL)
    (gdb) p pa_subpaths_valid
    $17 = true
    

构建partitioned_rels链表

    
    
    (gdb) n
    1406            if (IS_SIMPLE_REL(rel))
    (gdb) 
    1407                partitioned_rels = list_make1(rel->partitioned_child_rels);
    (gdb) 
    1433            Assert(list_length(partitioned_rels) >= 1);
    (gdb) 
    1441        foreach(l, live_childrels)
    (gdb) p *partitioned_rels
    $18 = {type = T_List, length = 1, head = 0x1fcff40, tail = 0x1fcff40}
    

开始遍历live_childrels,对于每一个非虚拟子关系,记录成本最低的访问路径.  
如果子关系存在非参数化的总成本最低的访问路径,添加此路径到我们为父关系构建的非参数化的Append访问路径中.

    
    
    (gdb) n
    1443            RelOptInfo *childrel = lfirst(l);
    (gdb) 
    1445            Path       *cheapest_partial_path = NULL;
    (gdb) 
    1451            if (rel->rtekind == RTE_SUBQUERY && childrel->partitioned_child_rels != NIL)
    (gdb) 
    1463            if (childrel->pathlist != NIL &&
    (gdb) 
    1464                childrel->cheapest_total_path->param_info == NULL)
    (gdb) 
    1463            if (childrel->pathlist != NIL &&
    (gdb) 
    1465                accumulate_append_subpath(childrel->cheapest_total_path,
    

同样的思路,处理并行处理中的部分计划

    
    
    (gdb) n
    1471            if (childrel->partial_pathlist != NIL)
    (gdb) 
    1473                cheapest_partial_path = linitial(childrel->partial_pathlist);
    (gdb) 
    1474                accumulate_append_subpath(cheapest_partial_path,
    

同样的,处理并行append混合并行/非并行访问路径

    
    
    (gdb) n
    1486                Path       *nppath = NULL;
    (gdb) 
    1489                    get_cheapest_parallel_safe_total_inner(childrel->pathlist);
    (gdb) 
    1488                nppath =
    (gdb) 
    1491                if (cheapest_partial_path == NULL && nppath == NULL)
    (gdb) 
    1496                else if (nppath == NULL ||
    (gdb) 
    1498                          cheapest_partial_path->total_cost < nppath->total_cost))
    (gdb) 
    1497                         (cheapest_partial_path != NULL &&
    (gdb) 
    1501                    Assert(cheapest_partial_path != NULL);
    (gdb) 
    1502                    accumulate_append_subpath(cheapest_partial_path,
    

收集子关系所有可用的排序和参数化路径链表.

    
    
    (gdb) 
    1534            foreach(lcp, childrel->pathlist)
    (gdb) 
    (gdb) n
    1536                Path       *childpath = (Path *) lfirst(lcp);
    (gdb) 
    1537                List       *childkeys = childpath->pathkeys;
    (gdb) 
    1538                Relids      childouter = PATH_REQ_OUTER(childpath);
    (gdb) 
    1541                if (childkeys != NIL)
    (gdb) 
    1567                if (childouter)
    (gdb) 
    1534            foreach(lcp, childrel->pathlist)
    

继续下一个子关系,完成处理

    
    
    ...
    (gdb) 
    1441        foreach(l, live_childrels)
    (gdb) 
    1598        if (subpaths_valid)
    

如存在子关系的非参数化访问路径,构建未排序/未参数化的Append访问路径.

    
    
    (gdb) n
    1599            add_path(rel, (Path *) create_append_path(root, rel, subpaths, NIL,
        (gdb) p *rel->pathlist
    Cannot access memory at address 0x0
    (gdb) n
    1607        if (partial_subpaths_valid)
    (gdb) p *rel->pathlist
    $22 = {type = T_List, length = 1, head = 0x1fd0230, tail = 0x1fd0230}
    

尝试未排序/未参数化的部分Append访问路径.如可能,构建parallel-aware访问路径.

    
    
    ...
    (gdb) 
    1641            appendpath = create_append_path(root, rel, NIL, partial_subpaths,
    (gdb) 
    1650            partial_rows = appendpath->path.rows;
    (gdb) 
    1653            add_partial_path(rel, (Path *) appendpath);
    (gdb) 
    

使用混合的部分和非部分并行的append.

    
    
    1662        if (pa_subpaths_valid && pa_nonpartial_subpaths != NIL)
    (gdb) 
    

基于收集的子路径键,构建非参数化的MergeAppend访问路径

    
    
    1701        if (subpaths_valid)
    (gdb) 
    1702            generate_mergeappend_paths(root, rel, live_childrels,
    

完成调用

    
    
    (gdb) 
    1719        foreach(l, all_child_outers)
    (gdb) 
    1757    }
    (gdb) 
    set_append_rel_pathlist (root=0x1f1cdb8, rel=0x1fc1800, rti=1, rte=0x1efa3d0) at allpaths.c:1354
    1354    }
    (gdb) p *rel->pathlist
    $23 = {type = T_List, length = 1, head = 0x1fd0230, tail = 0x1fd0230}
    (gdb) 
    (gdb) p *(Node *)rel->pathlist->head->data.ptr_value
    $24 = {type = T_AppendPath}
    (gdb) p *(AppendPath *)rel->pathlist->head->data.ptr_value
    $25 = {path = {type = T_AppendPath, pathtype = T_Append, parent = 0x1fc1800, pathtarget = 0x1fc1a38, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 6, startup_cost = 0, 
        total_cost = 30.530000000000001, pathkeys = 0x0}, partitioned_rels = 0x1fd01f8, subpaths = 0x1fcffc8, 
      first_partial_path = 2}
    

结束调用

    
    
    (gdb) n
    set_rel_pathlist (root=0x1f1cdb8, rel=0x1fc1800, rti=1, rte=0x1efa3d0) at allpaths.c:495
    495     if (rel->reloptkind == RELOPT_BASEREL &&
    (gdb) 
    496         bms_membership(root->all_baserels) != BMS_SINGLETON)
    (gdb) 
    495     if (rel->reloptkind == RELOPT_BASEREL &&
    (gdb) 
    504     if (set_rel_pathlist_hook)
    (gdb) 
    508     set_cheapest(rel);
    (gdb) 
    513 }
    (gdb) 
    

DONE!

### 四、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

