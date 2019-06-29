在查询分区表的时候PG如何确定查询的是哪个分区？如何确定？相关的机制是什么？接下来几个章节将一一介绍，本节是第一部分。

### 零、实现机制

我们先看下面的例子,两个普通表t_normal_1和t_normal_2,执行UNION ALL操作:

    
    
    drop table if exists t_normal_1;
    drop table if exists t_normal_2;
    create table t_normal_1 (c1 int not null,c2  varchar(40),c3 varchar(40));
    create table t_normal_2 (c1 int not null,c2  varchar(40),c3 varchar(40));
    
    insert into t_normal_1(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    insert into t_normal_2(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    
    testdb=# explain verbose select * from t_normal_1 where c1 = 0
    testdb-# union all
    testdb-# select * from t_normal_2 where c1 <> 0;
                                     QUERY PLAN                                 
    ----------------------------------------------------------------------------     Append  (cost=0.00..34.00 rows=350 width=200)
       ->  Seq Scan on public.t_normal_1  (cost=0.00..14.38 rows=2 width=200)
             Output: t_normal_1.c1, t_normal_1.c2, t_normal_1.c3
             Filter: (t_normal_1.c1 = 0)
       ->  Seq Scan on public.t_normal_2  (cost=0.00..14.38 rows=348 width=200)
             Output: t_normal_2.c1, t_normal_2.c2, t_normal_2.c3
             Filter: (t_normal_2.c1 <> 0)
    (7 rows)
    

两张普通表的UNION
ALL,PG使用APPEND操作符把t_normal_1顺序扫描的结果集和t_normal_2顺序扫描的结果集"APPEND"在一起作为最终的结果集输出.

分区表的查询也是类似的机制,把各个分区的结果集APPEND在一起,然后作为最终的结果集输出,如下例所示:

    
    
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
    
    

查询分区表t_hash_partition,条件为c1 = 1 OR c1 =
2,从执行计划可见是把t_hash_partition_1顺序扫描的结果集和t_hash_partition_3顺序扫描的结果集"APPEND"在一起作为最终的结果集输出.

这里面有几个问题需要解决:  
1.识别分区表并找到所有的分区子表;  
2.根据约束条件识别需要查询的分区,这是出于性能的考虑;  
3.对结果集执行APPEND,作为最终结果输出.  
本节介绍了PG如何识别分区表并找到所有的分区子表,实现的函数是expand_inherited_tables.

### 一、数据结构

**AppendRelInfo**  
Append-relation信息.  
当我们将可继承表(分区表)或UNION-ALL子查询展开为“追加关系”(本质上是子RTE的链表)时，为每个子RTE构建一个AppendRelInfo。  
AppendRelInfos链表指示在展开父节点时必须包含哪些子rte，每个节点具有将引用父节点的Vars转换为引用该子节点的Vars所需的所有信息。

    
    
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
    
    

**PlannerInfo**  
该数据结构用于存储查询语句在规划/优化过程中的相关信息

    
    
    /*----------     * PlannerInfo
     *      Per-query information for planning/optimization
     *      用于规划/优化的每个查询信息
     * 
     * This struct is conventionally called "root" in all the planner routines.
     * It holds links to all of the planner's working state, in addition to the
     * original Query.  Note that at present the planner extensively modifies
     * the passed-in Query data structure; someday that should stop.
     * 在所有计划程序例程中，这个结构通常称为“root”。
     * 除了原始查询之外，它还保存到所有计划器工作状态的链接。
     * 注意，目前计划器会毫无节制的修改传入的查询数据结构,相信总有一天这种情况会停止的。
     *----------     */
    struct AppendRelInfo;
    
    typedef struct PlannerInfo
    {
        NodeTag     type;//Node标识
        //查询树
        Query      *parse;          /* the Query being planned */
        //当前的planner全局信息
        PlannerGlobal *glob;        /* global info for current planner run */
        //查询层次,1标识最高层
        Index       query_level;    /* 1 at the outermost Query */
        // 如为子计划,则这里存储父计划器指针,NULL标识最高层
        struct PlannerInfo *parent_root;    /* NULL at outermost Query */
    
        /*
         * plan_params contains the expressions that this query level needs to
         * make available to a lower query level that is currently being planned.
         * outer_params contains the paramIds of PARAM_EXEC Params that outer
         * query levels will make available to this query level.
         * plan_params包含该查询级别需要提供给当前计划的较低查询级别的表达式。
         * outer_params包含PARAM_EXEC Params的参数，外部查询级别将使该查询级别可用这些参数。
         */
        List       *plan_params;    /* list of PlannerParamItems, see below */
        Bitmapset  *outer_params;
    
        /*
         * simple_rel_array holds pointers to "base rels" and "other rels" (see
         * comments for RelOptInfo for more info).  It is indexed by rangetable
         * index (so entry 0 is always wasted).  Entries can be NULL when an RTE
         * does not correspond to a base relation, such as a join RTE or an
         * unreferenced view RTE; or if the RelOptInfo hasn't been made yet.
         * simple_rel_array保存指向“base rels”和“other rels”的指针
         * (有关RelOptInfo的更多信息，请参见注释)。
         * 它由可范围表索引建立索引(因此条目0总是被浪费)。
         * 当RTE与基本关系(如JOIN RTE或未被引用的视图RTE时)不相对应
         *   或者如果RelOptInfo还没有生成，条目可以为NULL。
         */
        //RelOptInfo数组,存储"base rels",比如基表/子查询等.
        //该数组与RTE的顺序一一对应,而且是从1开始,因此[0]无用 */
        struct RelOptInfo **simple_rel_array;   /* All 1-rel RelOptInfos */
        int         simple_rel_array_size;  /* 数组大小,allocated size of array */
    
        /*
         * simple_rte_array is the same length as simple_rel_array and holds
         * pointers to the associated rangetable entries.  This lets us avoid
         * rt_fetch(), which can be a bit slow once large inheritance sets have
         * been expanded.
         * simple_rte_array的长度与simple_rel_array相同，
         *   并保存指向相应范围表条目的指针。
         * 这使我们可以避免执行rt_fetch()，因为一旦扩展了大型继承集，rt_fetch()可能会有点慢。
         */
        //RTE数组
        RangeTblEntry **simple_rte_array;   /* rangetable as an array */
    
        /*
         * append_rel_array is the same length as the above arrays, and holds
         * pointers to the corresponding AppendRelInfo entry indexed by
         * child_relid, or NULL if none.  The array itself is not allocated if
         * append_rel_list is empty.
         * append_rel_array与上述数组的长度相同，
         *   并保存指向对应的AppendRelInfo条目的指针，该条目由child_relid索引，
         *   如果没有索引则为NULL。
         * 如果append_rel_list为空，则不分配数组本身。
         */
        //处理集合操作如UNION ALL时使用和分区表时使用
        struct AppendRelInfo **append_rel_array;
    
        /*
         * all_baserels is a Relids set of all base relids (but not "other"
         * relids) in the query; that is, the Relids identifier of the final join
         * we need to form.  This is computed in make_one_rel, just before we
         * start making Paths.
         * all_baserels是查询中所有base relids(但不是“other” relids)的一个Relids集合;
         *   也就是说，这是需要形成的最终连接的Relids标识符。
         * 这是在开始创建路径之前在make_one_rel中计算的。
         */
        Relids      all_baserels;//"base rels"
    
        /*
         * nullable_baserels is a Relids set of base relids that are nullable by
         * some outer join in the jointree; these are rels that are potentially
         * nullable below the WHERE clause, SELECT targetlist, etc.  This is
         * computed in deconstruct_jointree.
         * nullable_baserels是由jointree中的某些外连接中值可为空的base Relids集合;
         *   这些是在WHERE子句、SELECT targetlist等下面可能为空的树。
         * 这是在deconstruct_jointree中处理获得的。
         */
        //Nullable-side端的"base rels"
        Relids      nullable_baserels;
    
        /*
         * join_rel_list is a list of all join-relation RelOptInfos we have
         * considered in this planning run.  For small problems we just scan the
         * list to do lookups, but when there are many join relations we build a
         * hash table for faster lookups.  The hash table is present and valid
         * when join_rel_hash is not NULL.  Note that we still maintain the list
         * even when using the hash table for lookups; this simplifies life for
         * GEQO.
         * join_rel_list是在计划执行中考虑的所有连接关系RelOptInfos的链表。
         * 对于小问题，只需要扫描链表执行查找，但是当存在许多连接关系时，
         *    需要构建一个散列表来进行更快的查找。
         * 当join_rel_hash不为空时，哈希表是有效可用于查询的。
         * 注意，即使在使用哈希表进行查找时，仍然维护该链表;这简化了GEQO(遗传算法)的生命周期。
         */
        //参与连接的Relation的RelOptInfo链表
        List       *join_rel_list;  /* list of join-relation RelOptInfos */
        //可加快链表访问的hash表
        struct HTAB *join_rel_hash; /* optional hashtable for join relations */
    
        /*
         * When doing a dynamic-programming-style join search, join_rel_level[k]
         * is a list of all join-relation RelOptInfos of level k, and
         * join_cur_level is the current level.  New join-relation RelOptInfos are
         * automatically added to the join_rel_level[join_cur_level] list.
         * join_rel_level is NULL if not in use.
         * 在执行动态规划算法的连接搜索时，join_rel_level[k]是k级的所有连接关系RelOptInfos的列表，
         * join_cur_level是当前级别。
         * 新的连接关系RelOptInfos会自动添加到join_rel_level[join_cur_level]链表中。
         * 如果不使用join_rel_level，则为NULL。
         */
        //RelOptInfo指针链表数组,k层的join存储在[k]中
        List      **join_rel_level; /* lists of join-relation RelOptInfos */
        //当前的join层次
        int         join_cur_level; /* index of list being extended */
        //查询的初始化计划链表
        List       *init_plans;     /* init SubPlans for query */
        //CTE子计划ID链表
        List       *cte_plan_ids;   /* per-CTE-item list of subplan IDs */
        //MULTIEXPR子查询输出的参数链表的链表
        List       *multiexpr_params;   /* List of Lists of Params for MULTIEXPR
                                         * subquery outputs */
        //活动的等价类链表
        List       *eq_classes;     /* list of active EquivalenceClasses */
        //规范化的PathKey链表
        List       *canon_pathkeys; /* list of "canonical" PathKeys */
        //外连接约束条件链表(左)
        List       *left_join_clauses;  /* list of RestrictInfos for mergejoinable
                                         * outer join clauses w/nonnullable var on
                                         * left */
        //外连接约束条件链表(右)
        List       *right_join_clauses; /* list of RestrictInfos for mergejoinable
                                         * outer join clauses w/nonnullable var on
                                         * right */
        //全连接约束条件链表
        List       *full_join_clauses;  /* list of RestrictInfos for mergejoinable
                                         * full join clauses */
        //特殊连接信息链表
        List       *join_info_list; /* list of SpecialJoinInfos */
        //AppendRelInfo链表
        List       *append_rel_list;    /* list of AppendRelInfos */
        //PlanRowMarks链表
        List       *rowMarks;       /* list of PlanRowMarks */
        //PHI链表
        List       *placeholder_list;   /* list of PlaceHolderInfos */
        // 外键信息链表
        List       *fkey_list;      /* list of ForeignKeyOptInfos */
        //query_planner()要求的PathKeys链表
        List       *query_pathkeys; /* desired pathkeys for query_planner() */
        //分组子句路径键
        List       *group_pathkeys; /* groupClause pathkeys, if any */
        //窗口函数路径键
        List       *window_pathkeys;    /* pathkeys of bottom window, if any */
        //distinctClause路径键
        List       *distinct_pathkeys;  /* distinctClause pathkeys, if any */
        //排序路径键
        List       *sort_pathkeys;  /* sortClause pathkeys, if any */
        //已规范化的分区Schema
        List       *part_schemes;   /* Canonicalised partition schemes used in the
                                     * query. */
        //尝试连接的RelOptInfo链表
        List       *initial_rels;   /* RelOptInfos we are now trying to join */
    
        /* Use fetch_upper_rel() to get any particular upper rel */
        //上层的RelOptInfo链表
        List       *upper_rels[UPPERREL_FINAL + 1]; /*  upper-rel RelOptInfos */
    
        /* Result tlists chosen by grouping_planner for upper-stage processing */
        //grouping_planner为上层处理选择的结果tlists
        struct PathTarget *upper_targets[UPPERREL_FINAL + 1];//
    
        /*
         * grouping_planner passes back its final processed targetlist here, for
         * use in relabeling the topmost tlist of the finished Plan.
         * grouping_planner在这里传回它最终处理过的targetlist，用于重新标记已完成计划的最顶层tlist。
         */
        ////最后需处理的投影列
        List       *processed_tlist;
    
        /* Fields filled during create_plan() for use in setrefs.c */
        //setrefs.c中在create_plan()函数调用期间填充的字段
        //分组函数属性映射
        AttrNumber *grouping_map;   /* for GroupingFunc fixup */
        //MinMaxAggInfos链表
        List       *minmax_aggs;    /* List of MinMaxAggInfos */
        //内存上下文
        MemoryContext planner_cxt;  /* context holding PlannerInfo */
        //关系的page计数
        double      total_table_pages;  /* # of pages in all tables of query */
        //query_planner输入参数:元组处理比例
        double      tuple_fraction; /* tuple_fraction passed to query_planner */
        //query_planner输入参数:limit_tuple
        double      limit_tuples;   /* limit_tuples passed to query_planner */
        //表达式的最小安全等级
        Index       qual_security_level;    /* minimum security_level for quals */
        /* Note: qual_security_level is zero if there are no securityQuals */
        //注意:如果没有securityQuals, 则qual_security_level是NULL(0)
    
        //如目标relation是分区表的child/partition/分区表,则通过此字段标记
        InheritanceKind inhTargetKind;  /* indicates if the target relation is an
                                         * inheritance child or partition or a
                                         * partitioned table */
        //是否存在RTE_JOIN的RTE
        bool        hasJoinRTEs;    /* true if any RTEs are RTE_JOIN kind */
        //是否存在标记为LATERAL的RTE
        bool        hasLateralRTEs; /* true if any RTEs are marked LATERAL */
        //是否存在已在jointree删除的RTE
        bool        hasDeletedRTEs; /* true if any RTE was deleted from jointree */
        //是否存在Having子句
        bool        hasHavingQual;  /* true if havingQual was non-null */
        //如约束条件中存在pseudoconstant = true,则此字段为T
        bool        hasPseudoConstantQuals; /* true if any RestrictInfo has
                                             * pseudoconstant = true */
        //是否存在递归语句
        bool        hasRecursion;   /* true if planning a recursive WITH item */
    
        /* These fields are used only when hasRecursion is true: */
        //这些字段仅在hasRecursion为T时使用:
        //工作表的PARAM_EXEC ID
        int         wt_param_id;    /* PARAM_EXEC ID for the work table */
        //非递归模式的访问路径
        struct Path *non_recursive_path;    /* a path for non-recursive term */
    
        /* These fields are workspace for createplan.c */
        //这些字段用于createplan.c
        //当前节点之上的外部rels
        Relids      curOuterRels;   /* outer rels above current node */
        //未赋值的NestLoopParams参数
        List       *curOuterParams; /* not-yet-assigned NestLoopParams */
    
        /* optional private data for join_search_hook, e.g., GEQO */
        //可选的join_search_hook私有数据，例如GEQO
        void       *join_search_private;
    
        /* Does this query modify any partition key columns? */
        //该查询是否更新分区键列?
        bool        partColsUpdated;
    } PlannerInfo;
    
    

### 二、源码解读

expand_inherited_tables函数将表示继承集合的每个范围表条目展开为“append relation”。

    
    
    /*
     * expand_inherited_tables
     *      Expand each rangetable entry that represents an inheritance set
     *      into an "append relation".  At the conclusion of this process,
     *      the "inh" flag is set in all and only those RTEs that are append
     *      relation parents.
     *      将表示继承集合的每个范围表条目展开为“append relation”。
     *      在这个过程结束时，“inh”标志被设置在所有且只有那些作为append
     *      relation parents的RTEs中。
     */
    void
    expand_inherited_tables(PlannerInfo *root)
    {
        Index       nrtes;
        Index       rti;
        ListCell   *rl;
    
        /*
         * expand_inherited_rtentry may add RTEs to parse->rtable. The function is
         * expected to recursively handle any RTEs that it creates with inh=true.
         * So just scan as far as the original end of the rtable list.
         * expand_inherited_rtentry可以添加RTEs到parse->rtable中。
         * 这个函数被期望递归地处理它用inh = true创建的所有RTEs。
         * 所以只要扫描到rtable链表最开始的末尾即可。
         */
        nrtes = list_length(root->parse->rtable);
        rl = list_head(root->parse->rtable);
        for (rti = 1; rti <= nrtes; rti++)
        {
            RangeTblEntry *rte = (RangeTblEntry *) lfirst(rl);
    
            expand_inherited_rtentry(root, rte, rti);
            rl = lnext(rl);
        }
    }
    
    /*
     * expand_inherited_rtentry
     *      Check whether a rangetable entry represents an inheritance set.
     *      If so, add entries for all the child tables to the query's
     *      rangetable, and build AppendRelInfo nodes for all the child tables
     *      and add them to root->append_rel_list.  If not, clear the entry's
     *      "inh" flag to prevent later code from looking for AppendRelInfos.
     *      检查范围表条目是否表示继承集合。
     *      如是，将所有子表的条目添加到查询的范围表中，
     *        并为所有子表构建AppendRelInfo节点，并将它们添加到root->append_rel_list。
     *      如没有，清除条目的“inh”标志，以防止以后的代码寻找AppendRelInfos。
     *
     * Note that the original RTE is considered to represent the whole
     * inheritance set.  The first of the generated RTEs is an RTE for the same
     * table, but with inh = false, to represent the parent table in its role
     * as a simple member of the inheritance set.
     * 注意，原始的RTEs被认为代表了整个继承集合。
     * 生成的第一个RTE是同一个表的RTE，但inh = false表示父表作为继承集的一个简单成员的角色。
     *
     * A childless table is never considered to be an inheritance set. For
     * regular inheritance, a parent RTE must always have at least two associated
     * AppendRelInfos: one corresponding to the parent table as a simple member of
     * inheritance set and one or more corresponding to the actual children.
     * Since a partitioned table is not scanned, it might have only one associated
     * AppendRelInfo.
     * 无子表的关系永远不会被认为是继承集合。
     * 对于常规继承，父RTE必须始终至少有两个相关的AppendRelInfos:
     *   一个作为继承集的简单成员与父表相对应，
     *   另一个或多个与实际的子表相对应。
     * 因为没有扫描分区表，所以它可能只有一个关联的AppendRelInfo。
     */
    static void
    expand_inherited_rtentry(PlannerInfo *root, RangeTblEntry *rte, Index rti)
    {
        Oid         parentOID;
        PlanRowMark *oldrc;
        Relation    oldrelation;
        LOCKMODE    lockmode;
        List       *inhOIDs;
        ListCell   *l;
    
        /* Does RT entry allow inheritance? */
        //是否分区表?
        if (!rte->inh)
            return;
        /* Ignore any already-expanded UNION ALL nodes */
        //忽略所有已扩展的UNION ALL节点
        if (rte->rtekind != RTE_RELATION)
        {
            Assert(rte->rtekind == RTE_SUBQUERY);
            return;//返回
        }
        /* Fast path for common case of childless table */
        //对于常规的无子表的关系,快速判断
        parentOID = rte->relid;
        if (!has_subclass(parentOID))
        {
            /* Clear flag before returning */
            //无子表,设置标记并返回
            rte->inh = false;
            return;
        }
    
        /*
         * The rewriter should already have obtained an appropriate lock on each
         * relation named in the query.  However, for each child relation we add
         * to the query, we must obtain an appropriate lock, because this will be
         * the first use of those relations in the parse/rewrite/plan pipeline.
         * Child rels should use the same lockmode as their parent.
         * 查询rewriter程序应该已经在查询中命名的每个关系上获得了适当的锁。
         * 但是，对于添加到查询中的每个子关系，必须获得适当的锁，
         *   因为这将是解析/重写/计划过程中这些关系的第一次使用。
         * 子树应该使用与父树相同的锁模式。
         */
        lockmode = rte->rellockmode;
    
        /* Scan for all members of inheritance set, acquire needed locks */
        //扫描继承集的所有成员，获取所需的锁
        inhOIDs = find_all_inheritors(parentOID, lockmode, NULL);
    
        /*
         * Check that there's at least one descendant, else treat as no-child
         * case.  This could happen despite above has_subclass() check, if table
         * once had a child but no longer does.
         * 检查是否至少有一个后代，否则视为无子女情况。
         * 尽管上面有has_subclass()检查，但如果table曾经有一个子元素，
         *   但现在不再有了，则可能发生这种情况。
         */
        if (list_length(inhOIDs) < 2)
        {
            /* Clear flag before returning */
            //清除标记,返回
            rte->inh = false;
            return;
        }
    
        /*
         * If parent relation is selected FOR UPDATE/SHARE, we need to mark its
         * PlanRowMark as isParent = true, and generate a new PlanRowMark for each
         * child.
         * 如果父关系是 selected FOR UPDATE/SHARE，
         *   则需要将其PlanRowMark标记为isParent = true，
         *   并为每个子关系生成一个新的PlanRowMark。
         */
        oldrc = get_plan_rowmark(root->rowMarks, rti);
        if (oldrc)
            oldrc->isParent = true;
    
        /*
         * Must open the parent relation to examine its tupdesc.  We need not lock
         * it; we assume the rewriter already did.
         * 必须打开父关系以检查其tupdesc。
         * 不需要锁定,我们假设查询重写已经这么做了。
         */
        oldrelation = heap_open(parentOID, NoLock);
    
        /* Scan the inheritance set and expand it */
        //扫描继承集合并扩展之
        if (RelationGetPartitionDesc(oldrelation) != NULL)//
        {
            Assert(rte->relkind == RELKIND_PARTITIONED_TABLE);
    
            /*
             * If this table has partitions, recursively expand them in the order
             * in which they appear in the PartitionDesc.  While at it, also
             * extract the partition key columns of all the partitioned tables.
             * 如果这个表有分区，则按分区在PartitionDesc中出现的顺序递归展开它们。
             * 同时，还提取所有分区表的分区键列。
             */
            expand_partitioned_rtentry(root, rte, rti, oldrelation, oldrc,
                                       lockmode, &root->append_rel_list);
        }
        else
        {
            //分区描述符获取不成功(没有分区信息)
            List       *appinfos = NIL;
            RangeTblEntry *childrte;
            Index       childRTindex;
    
            /*
             * This table has no partitions.  Expand any plain inheritance
             * children in the order the OIDs were returned by
             * find_all_inheritors.
             * 这个表没有分区。
             * 按find_all_inheritors返回的OIDs的顺序展开所有普通继承子元素。
             */
            foreach(l, inhOIDs)//遍历OIDs
            {
                Oid         childOID = lfirst_oid(l);
                Relation    newrelation;
    
                /* Open rel if needed; we already have required locks */
                //如有需要，打开rel(已获得锁)
                if (childOID != parentOID)
                    newrelation = heap_open(childOID, NoLock);
                else
                    newrelation = oldrelation;
    
                /*
                 * It is possible that the parent table has children that are temp
                 * tables of other backends.  We cannot safely access such tables
                 * (because of buffering issues), and the best thing to do seems
                 * to be to silently ignore them.
                 * 父表的子表可能是其他后台的临时表。
                 * 我们不能安全地访问这些表(因为存在缓冲问题)，最好的办法似乎是悄悄地忽略它们。
                 */
                if (childOID != parentOID && RELATION_IS_OTHER_TEMP(newrelation))
                {
                    heap_close(newrelation, lockmode);//忽略它们
                    continue;
                }
    
                expand_single_inheritance_child(root, rte, rti, oldrelation, oldrc,
                                                newrelation,
                                                &appinfos, &childrte,
                                                &childRTindex);//展开
    
                /* Close child relations, but keep locks */
                //关闭子表,但仍持有锁
                if (childOID != parentOID)
                    heap_close(newrelation, NoLock);
            }
    
            /*
             * If all the children were temp tables, pretend it's a
             * non-inheritance situation; we don't need Append node in that case.
             * The duplicate RTE we added for the parent table is harmless, so we
             * don't bother to get rid of it; ditto for the useless PlanRowMark
             * node.
             * 如果所有的子表都是临时表，则假设这是非继承情况;
             *   在这种情况下，不需要APPEND NODE。
             * 我们为父表添加重复的RTE是无关紧要的，
             *   因此我们不必费心删除它;无用的PlanRowMark节点也是如此。
             */
            if (list_length(appinfos) < 2)
                rte->inh = false;//设置标记
            else
                root->append_rel_list = list_concat(root->append_rel_list,
                                                    appinfos);//添加到链表中
    
        }
    
        heap_close(oldrelation, NoLock);//关闭relation
    }
    
    /*
     * expand_partitioned_rtentry
     *      Recursively expand an RTE for a partitioned table.
     *      递归扩展分区表RTE
     */
    static void
    expand_partitioned_rtentry(PlannerInfo *root, RangeTblEntry *parentrte,
                               Index parentRTindex, Relation parentrel,
                               PlanRowMark *top_parentrc, LOCKMODE lockmode,
                               List **appinfos)
    {
        int         i;
        RangeTblEntry *childrte;
        Index       childRTindex;
        PartitionDesc partdesc = RelationGetPartitionDesc(parentrel);
    
        check_stack_depth();
    
        /* A partitioned table should always have a partition descriptor. */
        //分配表通常应具备分区描述符
        Assert(partdesc);
    
        Assert(parentrte->inh);
    
        /*
         * Note down whether any partition key cols are being updated. Though it's
         * the root partitioned table's updatedCols we are interested in, we
         * instead use parentrte to get the updatedCols. This is convenient
         * because parentrte already has the root partrel's updatedCols translated
         * to match the attribute ordering of parentrel.
         * 请注意是否正在更新分区键cols。
         * 虽然感兴趣的是根分区表的updatedCols，但是使用parentrte来获取updatedCols。
         * 这很方便，因为parentrte已经将root partrel的updatedCols转换为匹配parentrel的属性顺序。
         */
        if (!root->partColsUpdated)
            root->partColsUpdated =
                has_partition_attrs(parentrel, parentrte->updatedCols, NULL);
    
        /* First expand the partitioned table itself. */
        //
        expand_single_inheritance_child(root, parentrte, parentRTindex, parentrel,
                                        top_parentrc, parentrel,
                                        appinfos, &childrte, &childRTindex);
    
        /*
         * If the partitioned table has no partitions, treat this as the
         * non-inheritance case.
         * 如果分区表没有分区，则将其视为非继承情况。
         */
        if (partdesc->nparts == 0)
        {
            parentrte->inh = false;
            return;
        }
    
        for (i = 0; i < partdesc->nparts; i++)
        {
            Oid         childOID = partdesc->oids[i];
            Relation    childrel;
    
            /* Open rel; we already have required locks */
            //打开rel
            childrel = heap_open(childOID, NoLock);
    
            /*
             * Temporary partitions belonging to other sessions should have been
             * disallowed at definition, but for paranoia's sake, let's double
             * check.
             * 属于其他会话的临时分区在定义时应该是不允许的，但是出于偏执狂的考虑，再检查一下。
             */
            if (RELATION_IS_OTHER_TEMP(childrel))
                elog(ERROR, "temporary relation from another session found as partition");
            //扩展之
            expand_single_inheritance_child(root, parentrte, parentRTindex,
                                            parentrel, top_parentrc, childrel,
                                            appinfos, &childrte, &childRTindex);
    
            /* If this child is itself partitioned, recurse */
            //子关系是分区表,递归扩展
            if (childrel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE)
                expand_partitioned_rtentry(root, childrte, childRTindex,
                                           childrel, top_parentrc, lockmode,
                                           appinfos);
    
            /* Close child relation, but keep locks */
            //关闭子关系,但仍持有锁
            heap_close(childrel, NoLock);
        }
    }
    
    
     /* expand_single_inheritance_child
     *      Build a RangeTblEntry and an AppendRelInfo, if appropriate, plus
     *      maybe a PlanRowMark.
     *      构建一个RangeTblEntry和一个AppendRelInfo，如果合适的话，再加上一个PlanRowMark。
     *
     * We now expand the partition hierarchy level by level, creating a
     * corresponding hierarchy of AppendRelInfos and RelOptInfos, where each
     * partitioned descendant acts as a parent of its immediate partitions.
     * (This is a difference from what older versions of PostgreSQL did and what
     * is still done in the case of table inheritance for unpartitioned tables,
     * where the hierarchy is flattened during RTE expansion.)
     * 现在我们逐层扩展分区层次结构，创建一个对应的AppendRelInfos和RelOptInfos层次结构，
     *   其中每个分区的后代充当其直接分区的父级。
     * (在未分区表的表继承中，
     *    层次结构在RTE扩展期间被扁平化，这与老版本的PostgreSQL有所不同。)
     *
     * PlanRowMarks still carry the top-parent's RTI, and the top-parent's
     * allMarkTypes field still accumulates values from all descendents.
     * PlanRowMarks仍然具有顶级父类的RTI信息，
     *   而顶级父类的allMarkTypes字段仍然从所有子类累积。
     * 
     * "parentrte" and "parentRTindex" are immediate parent's RTE and
     * RTI. "top_parentrc" is top parent's PlanRowMark.
     * “parentrte”和“parentRTindex”是直接父级的RTE和RTI。
     * “top_parentrc”是top父类的PlanRowMark。
     *
     * The child RangeTblEntry and its RTI are returned in "childrte_p" and
     * "childRTindex_p" resp.
     * 子RTE及其RTI在“childrte_p”和“childRTindex_p”resp中返回。
     */
    static void
    expand_single_inheritance_child(PlannerInfo *root, RangeTblEntry *parentrte,
                                    Index parentRTindex, Relation parentrel,
                                    PlanRowMark *top_parentrc, Relation childrel,
                                    List **appinfos, RangeTblEntry **childrte_p,
                                    Index *childRTindex_p)
    {
        Query      *parse = root->parse;
        Oid         parentOID = RelationGetRelid(parentrel);//父关系
        Oid         childOID = RelationGetRelid(childrel);//子关系
        RangeTblEntry *childrte;
        Index       childRTindex;
        AppendRelInfo *appinfo;
    
        /*
         * Build an RTE for the child, and attach to query's rangetable list. We
         * copy most fields of the parent's RTE, but replace relation OID and
         * relkind, and set inh = false.  Also, set requiredPerms to zero since
         * all required permissions checks are done on the original RTE. Likewise,
         * set the child's securityQuals to empty, because we only want to apply
         * the parent's RLS conditions regardless of what RLS properties
         * individual children may have.  (This is an intentional choice to make
         * inherited RLS work like regular permissions checks.) The parent
         * securityQuals will be propagated to children along with other base
         * restriction clauses, so we don't need to do it here.
         * 为子元素构建一个RTE，并附加到query的范围表链表中。
         * 我们复制父RTE的大部分字段，但是替换关系OID和relkind，并设置inh = false。
         * 另外，将requiredPerms设置为0，因为所有需要的权限检查都是在原始RTE上完成的。
         * 同样，将子元素securityQuals设置为空，因为只想应用父元素的RLS条件，
         *   而不管每个子元素可能具有什么RLS属性。
         *   (这是一种有意的选择，目的是让继承的RLS像常规权限检查一样工作。)
         * 父安全条件quals将与其他基本限制条款一起传播到子级，因此不需要在这里这样做。
         */
        childrte = copyObject(parentrte);
        *childrte_p = childrte;
        childrte->relid = childOID;
        childrte->relkind = childrel->rd_rel->relkind;
        /* A partitioned child will need to be expanded further. */
        //分区表的子关系会在"将来"扩展
        if (childOID != parentOID &&
            childrte->relkind == RELKIND_PARTITIONED_TABLE)
            childrte->inh = true;
        else
            childrte->inh = false;
        childrte->requiredPerms = 0;
        childrte->securityQuals = NIL;
        parse->rtable = lappend(parse->rtable, childrte);
        childRTindex = list_length(parse->rtable);
        *childRTindex_p = childRTindex;
    
        /*
         * We need an AppendRelInfo if paths will be built for the child RTE. If
         * childrte->inh is true, then we'll always need to generate append paths
         * for it.  If childrte->inh is false, we must scan it if it's not a
         * partitioned table; but if it is a partitioned table, then it never has
         * any data of its own and need not be scanned.
         * 如果要为子RTE构建路径，则需要一个AppendRelInfo。
         * 如果children ->inh为真，那么我们总是需要为它生成APPEND访问路径。
         * 如果children ->inh为假，则必须扫描它，如果它不是分区表;
         *   但是如果它是一个分区表，那么它永远不会有任何自己的数据，也不需要扫描。
         */
        if (childrte->relkind != RELKIND_PARTITIONED_TABLE || childrte->inh)
        {
            appinfo = makeNode(AppendRelInfo);
            appinfo->parent_relid = parentRTindex;
            appinfo->child_relid = childRTindex;
            appinfo->parent_reltype = parentrel->rd_rel->reltype;
            appinfo->child_reltype = childrel->rd_rel->reltype;
            make_inh_translation_list(parentrel, childrel, childRTindex,
                                      &appinfo->translated_vars);
            appinfo->parent_reloid = parentOID;
            *appinfos = lappend(*appinfos, appinfo);
    
            /*
             * Translate the column permissions bitmaps to the child's attnums (we
             * have to build the translated_vars list before we can do this). But
             * if this is the parent table, leave copyObject's result alone.
             * 将列权限位图转换为子节点的attnums(在此之前必须构建translated_vars列表)。
             * 但是，如果这是父表，则不要理会copyObject的结果。
             *
             * Note: we need to do this even though the executor won't run any
             * permissions checks on the child RTE.  The insertedCols/updatedCols
             * bitmaps may be examined for trigger-firing purposes.
             * 注意:即使执行程序不会在子RTE上运行任何权限检查，我们也需要这样做。
             * 可以检查插入的tedcols /updatedCols位图是否具有触发目的。
             */
            if (childOID != parentOID)
            {
                childrte->selectedCols = translate_col_privs(parentrte->selectedCols,
                                                             appinfo->translated_vars);
                childrte->insertedCols = translate_col_privs(parentrte->insertedCols,
                                                             appinfo->translated_vars);
                childrte->updatedCols = translate_col_privs(parentrte->updatedCols,
                                                            appinfo->translated_vars);
            }
        }
    
        /*
         * Build a PlanRowMark if parent is marked FOR UPDATE/SHARE.
         * 如父关系标记为FOR UPDATE/SHARE,则创建PlanRowMark
         */
        if (top_parentrc)
        {
            PlanRowMark *childrc = makeNode(PlanRowMark);
    
            childrc->rti = childRTindex;
            childrc->prti = top_parentrc->rti;
            childrc->rowmarkId = top_parentrc->rowmarkId;
            /* Reselect rowmark type, because relkind might not match parent */
            //重新选择rowmark类型，因为relkind可能与父类不匹配
            childrc->markType = select_rowmark_type(childrte,
                                                    top_parentrc->strength);
            childrc->allMarkTypes = (1 << childrc->markType);
            childrc->strength = top_parentrc->strength;
            childrc->waitPolicy = top_parentrc->waitPolicy;
    
            /*
             * We mark RowMarks for partitioned child tables as parent RowMarks so
             * that the executor ignores them (except their existence means that
             * the child tables be locked using appropriate mode).
             * 我们将分区的子表的RowMarks标记为父RowMarks，
             *   以便执行程序忽略它们(除非它们的存在意味着子表使用适当的模式被锁定)。
             */
            childrc->isParent = (childrte->relkind == RELKIND_PARTITIONED_TABLE);
    
            /* Include child's rowmark type in top parent's allMarkTypes */
            //在父类的allMarkTypes中包含子类的rowmark类型
            top_parentrc->allMarkTypes |= childrc->allMarkTypes;
    
            root->rowMarks = lappend(root->rowMarks, childrc);
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

    
    
    (gdb) b expand_inherited_tables
    Breakpoint 1 at 0x7e53ba: file prepunion.c, line 1483.
    (gdb) c
    Continuing.
    
    Breakpoint 1, expand_inherited_tables (root=0x28fcdc8) at prepunion.c:1483
    1483        nrtes = list_length(root->parse->rtable);
    

获取RTE的个数和链表元素

    
    
    (gdb) n
    1484        rl = list_head(root->parse->rtable);
    (gdb) 
    1485        for (rti = 1; rti <= nrtes; rti++)
    (gdb) p nrtes
    $1 = 1
    (gdb) p *rl
    $2 = {data = {ptr_value = 0x28d83d0, int_value = 42828752, oid_value = 42828752}, next = 0x0}
    (gdb) 
    

循环处理RTE

    
    
    (gdb) n
    1487            RangeTblEntry *rte = (RangeTblEntry *) lfirst(rl);
    (gdb) 
    1489            expand_inherited_rtentry(root, rte, rti);
    (gdb) p *rte
    $3 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16986, relkind = 112 'p', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x28d84e8, lateral = false, 
      inh = true, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x28d8c40, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    

进入expand_inherited_rtentry

    
    
    (gdb) step
    expand_inherited_rtentry (root=0x28fcdc8, rte=0x28d83d0, rti=1) at prepunion.c:1517
    1517        Query      *parse = root->parse;
    

expand_inherited_rtentry->分区表标记为T

    
    
    1526        if (!rte->inh)
    (gdb) p rte->inh
    $4 = true
    

expand_inherited_rtentry->执行相关判断

    
    
    (gdb) n
    1529        if (rte->rtekind != RTE_RELATION)
    (gdb) p rte->rtekind
    $5 = RTE_RELATION
    (gdb) n
    1535        parentOID = rte->relid;
    (gdb) 
    1536        if (!has_subclass(parentOID))
    (gdb) p parentOID
    $6 = 16986
    (gdb) n
    1556        oldrc = get_plan_rowmark(root->rowMarks, rti);
    (gdb) 
    1557        if (rti == parse->resultRelation)
    (gdb) p *oldrc
    Cannot access memory at address 0x0
    

expand_inherited_rtentry->扫描继承集的所有成员，获取所需的锁,并构建OIDs链表

    
    
    (gdb) n
    1559        else if (oldrc && RowMarkRequiresRowShareLock(oldrc->markType))
    (gdb) 
    1562            lockmode = AccessShareLock;
    (gdb) 
    1565        inhOIDs = find_all_inheritors(parentOID, lockmode, NULL);
    (gdb) 
    1572        if (list_length(inhOIDs) < 2)
    (gdb) p inhOIDs
    $7 = (List *) 0x28fd208
    (gdb) p *inhOIDs
    $8 = {type = T_OidList, length = 7, head = 0x28fd1e0, tail = 0x28fd778}
    (gdb) 
    

expand_inherited_rtentry->打开relation

    
    
    (gdb) n
    1584        if (oldrc)
    (gdb) 
    1591        oldrelation = heap_open(parentOID, NoLock);
    

expand_inherited_rtentry->成功获取分区描述符,调用expand_partitioned_rtentry

    
    
    (gdb) 
    1594        if (RelationGetPartitionDesc(oldrelation) != NULL)
    (gdb) 
    1596            Assert(rte->relkind == RELKIND_PARTITIONED_TABLE);
    (gdb) 
    1603            expand_partitioned_rtentry(root, rte, rti, oldrelation, oldrc,
    (gdb) 
    

expand_inherited_rtentry->进入expand_partitioned_rtentry

    
    
    (gdb) step
    expand_partitioned_rtentry (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, parentrel=0x7f4e66827980, 
        top_parentrc=0x0, lockmode=1, appinfos=0x28fce98) at prepunion.c:1684
    1684        PartitionDesc partdesc = RelationGetPartitionDesc(parentrel);
    

expand_partitioned_rtentry->获取分区描述符

    
    
    1684        PartitionDesc partdesc = RelationGetPartitionDesc(parentrel);
    (gdb) n
    1686        check_stack_depth();
    (gdb) p *partdesc
    $9 = {nparts = 6, oids = 0x298e4f8, boundinfo = 0x298e530}
    

expand_partitioned_rtentry->执行相关校验

    
    
    (gdb) n
    1689        Assert(partdesc);
    (gdb) 
    1691        Assert(parentrte->inh);
    (gdb) 
    1700        if (!root->partColsUpdated)
    (gdb) 
    1702                has_partition_attrs(parentrel, parentrte->updatedCols, NULL);
    (gdb) 
    1701            root->partColsUpdated =
    (gdb) 
    1705        expand_single_inheritance_child(root, parentrte, parentRTindex, parentrel,
    

expand_partitioned_rtentry->首先展开分区表本身,进入expand_single_inheritance_child

    
    
    (gdb) step
    expand_single_inheritance_child (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, parentrel=0x7f4e66827980, 
        top_parentrc=0x0, childrel=0x7f4e66827980, appinfos=0x28fce98, childrte_p=0x7ffd1928d2f8, childRTindex_p=0x7ffd1928d2f4)
        at prepunion.c:1778
    1778        Query      *parse = root->parse;
    

expand_single_inheritance_child->执行相关初始化(childrte)

    
    
    (gdb) n
    1779        Oid         parentOID = RelationGetRelid(parentrel);
    (gdb) 
    1780        Oid         childOID = RelationGetRelid(childrel);
    (gdb) 
    1797        childrte = copyObject(parentrte);
    (gdb) p parentOID
    $10 = 16986
    (gdb) p childOID
    $11 = 16986
    (gdb) n
    1798        *childrte_p = childrte;
    (gdb) 
    1799        childrte->relid = childOID;
    (gdb) 
    1800        childrte->relkind = childrel->rd_rel->relkind;
    (gdb) 
    1802        if (childOID != parentOID &&
    (gdb) 
    1806            childrte->inh = false;
    (gdb) 
    1807        childrte->requiredPerms = 0;
    (gdb) 
    1808        childrte->securityQuals = NIL;
    (gdb) 
    1809        parse->rtable = lappend(parse->rtable, childrte);
    (gdb) 
    1810        childRTindex = list_length(parse->rtable);
    (gdb) 
    1811        *childRTindex_p = childRTindex;
    (gdb) p *childrte -->relid = 16986,仍为分区表
    $12 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16986, relkind = 112 'p', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x28fd268, lateral = false, 
      inh = false, inFromCl = true, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x28fd898, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) p *childRTindex_p
    $13 = 0
    

expand_single_inheritance_child->完成分区表本身的扩展,回到expand_partitioned_rtentry

    
    
    (gdb) p *childRTindex_p
    $13 = 0
    (gdb) n
    1820        if (childrte->relkind != RELKIND_PARTITIONED_TABLE || childrte->inh)
    (gdb) 
    1855        if (top_parentrc)
    (gdb) 
    1881    }
    (gdb) 
    expand_partitioned_rtentry (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, parentrel=0x7f4e66827980, 
        top_parentrc=0x0, lockmode=1, appinfos=0x28fce98) at prepunion.c:1713
    1713        if (partdesc->nparts == 0)
    

expand_partitioned_rtentry->开始遍历分区描述符中的分区

    
    
    1713        if (partdesc->nparts == 0)
    (gdb) n
    1719        for (i = 0; i < partdesc->nparts; i++)
    (gdb) 
    1721            Oid         childOID = partdesc->oids[i];
    (gdb) 
    1725            childrel = heap_open(childOID, NoLock);
    (gdb) 
    1732            if (RELATION_IS_OTHER_TEMP(childrel))
    (gdb) 
    1735            expand_single_inheritance_child(root, parentrte, parentRTindex,
    (gdb) p childOID
    $14 = 16989 
    ----------------------------------------    testdb=# select relname from pg_class where oid=16989;
          relname       
    --------------------     t_hash_partition_1
    (1 row)
    ----------------------------------------    

expand_single_inheritance_child->再次进入expand_single_inheritance_child

    
    
    (gdb) step
    expand_single_inheritance_child (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, parentrel=0x7f4e66827980, 
        top_parentrc=0x0, childrel=0x7f4e668306a0, appinfos=0x28fce98, childrte_p=0x7ffd1928d2f8, childRTindex_p=0x7ffd1928d2f4)
        at prepunion.c:1778
    1778        Query      *parse = root->parse;
    

expand_single_inheritance_child->开始构建AppendRelInfo

    
    
    ...
    1820        if (childrte->relkind != RELKIND_PARTITIONED_TABLE || childrte->inh)
    (gdb) 
    1822            appinfo = makeNode(AppendRelInfo);
    (gdb) p *childrte
    $17 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16989, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x28fd9d0, lateral = false, 
      inh = false, inFromCl = true, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x28fdbc8, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) p *childrte->relkind
    Cannot access memory at address 0x72
    (gdb) p childrte->relkind
    $18 = 114 'r'
    (gdb) p childrte->inh
    $19 = false
    

expand_single_inheritance_child->构建完毕,查看AppendRelInfo结构体

    
    
    (gdb) n
    1823            appinfo->parent_relid = parentRTindex;
    (gdb) 
    1824            appinfo->child_relid = childRTindex;
    (gdb) 
    1825            appinfo->parent_reltype = parentrel->rd_rel->reltype;
    (gdb) 
    1826            appinfo->child_reltype = childrel->rd_rel->reltype;
    (gdb) 
    1827            make_inh_translation_list(parentrel, childrel, childRTindex,
    (gdb) 
    1829            appinfo->parent_reloid = parentOID;
    (gdb) 
    1830            *appinfos = lappend(*appinfos, appinfo);
    (gdb) 
    1841            if (childOID != parentOID)
    (gdb) 
    1843                childrte->selectedCols = translate_col_privs(parentrte->selectedCols,
    (gdb) 
    1845                childrte->insertedCols = translate_col_privs(parentrte->insertedCols,
    (gdb) 
    1847                childrte->updatedCols = translate_col_privs(parentrte->updatedCols,
    (gdb) 
    1855        if (top_parentrc)
    (gdb) p *appinfo
    $20 = {type = T_AppendRelInfo, parent_relid = 1, child_relid = 3, parent_reltype = 16988, child_reltype = 16991, 
      translated_vars = 0x28fdc90, parent_reloid = 16986}
    

expand_single_inheritance_child->完成调用,返回

    
    
    (gdb) 
    1855        if (top_parentrc)
    (gdb) p *appinfo
    $20 = {type = T_AppendRelInfo, parent_relid = 1, child_relid = 3, parent_reltype = 16988, child_reltype = 16991, 
      translated_vars = 0x28fdc90, parent_reloid = 16986}
    (gdb) n
    1881    }
    (gdb) 
    expand_partitioned_rtentry (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, parentrel=0x7f4e66827980, 
        top_parentrc=0x0, lockmode=1, appinfos=0x28fce98) at prepunion.c:1740
    1740            if (childrel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE)
    

expand_inherited_rtentry->完成expand_partitioned_rtentry过程调用,回到expand_inherited_rtentry

    
    
    (gdb) finish
    Run till exit from #0  expand_partitioned_rtentry (root=0x28fcdc8, parentrte=0x28d83d0, parentRTindex=1, 
        parentrel=0x7f4e66827980, top_parentrc=0x0, lockmode=1, appinfos=0x28fce98) at prepunion.c:1740
    0x00000000007e55e3 in expand_inherited_rtentry (root=0x28fcdc8, rte=0x28d83d0, rti=1) at prepunion.c:1603
    1603            expand_partitioned_rtentry(root, rte, rti, oldrelation, oldrc,
    (gdb) 
    

expand_inherited_rtentry->完成expand_inherited_rtentry的调用,回到expand_inherited_tables

    
    
    (gdb) n
    1665        heap_close(oldrelation, NoLock);
    (gdb) 
    1666    }
    (gdb) 
    expand_inherited_tables (root=0x28fcdc8) at prepunion.c:1490
    1490            rl = lnext(rl);
    (gdb) 
    

expand_inherited_tables->完成expand_inherited_tables调用,回到subquery_planner

    
    
    (gdb) n
    1485        for (rti = 1; rti <= nrtes; rti++)
    (gdb) 
    1492    }
    (gdb) 
    subquery_planner (glob=0x28fcd30, parse=0x28d82b8, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:719
    719     root->hasHavingQual = (parse->havingQual != NULL);
    (gdb) 
    

DONE!

### 四、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

