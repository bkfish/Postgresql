本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths函数的主实现逻辑，该函数为连接新生成的joinrel构造访问路径。

### 一、数据结构

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
    
    

**RelOptInfo**

    
    
     typedef enum RelOptKind
     {
         RELOPT_BASEREL,//基本关系(如基表/子查询等)
         RELOPT_JOINREL,//连接产生的关系,要注意的是通过连接等方式产生的结果亦可以视为关系
         RELOPT_OTHER_MEMBER_REL,
         RELOPT_OTHER_JOINREL,
         RELOPT_UPPER_REL,//上层的关系
         RELOPT_OTHER_UPPER_REL,
         RELOPT_DEADREL
     } RelOptKind;
     
     /*
      * Is the given relation a simple relation i.e a base or "other" member
      * relation?
      */
     #define IS_SIMPLE_REL(rel) \
         ((rel)->reloptkind == RELOPT_BASEREL || \
          (rel)->reloptkind == RELOPT_OTHER_MEMBER_REL)
     
     /* Is the given relation a join relation? */
     #define IS_JOIN_REL(rel)    \
         ((rel)->reloptkind == RELOPT_JOINREL || \
          (rel)->reloptkind == RELOPT_OTHER_JOINREL)
     
     /* Is the given relation an upper relation? */
     #define IS_UPPER_REL(rel)   \
         ((rel)->reloptkind == RELOPT_UPPER_REL || \
          (rel)->reloptkind == RELOPT_OTHER_UPPER_REL)
     
     /* Is the given relation an "other" relation? */
     #define IS_OTHER_REL(rel) \
         ((rel)->reloptkind == RELOPT_OTHER_MEMBER_REL || \
          (rel)->reloptkind == RELOPT_OTHER_JOINREL || \
          (rel)->reloptkind == RELOPT_OTHER_UPPER_REL)
     
     typedef struct RelOptInfo
     {
         NodeTag     type;//节点标识
     
         RelOptKind  reloptkind;//RelOpt类型
     
         /* all relations included in this RelOptInfo */
         Relids      relids;         /*Relids(rtindex)集合 set of base relids (rangetable indexes) */
     
         /* size estimates generated by planner */
         double      rows;           /*结果元组的估算数量 estimated number of result tuples */
     
         /* per-relation planner control flags */
         bool        consider_startup;   /*是否考虑启动成本?是,需要保留启动成本低的路径 keep cheap-startup-cost paths? */
         bool        consider_param_startup; /*是否考虑参数化?的路径 ditto, for parameterized paths? */
         bool        consider_parallel;  /*是否考虑并行处理路径 consider parallel paths? */
     
         /* default result targetlist for Paths scanning this relation */
         struct PathTarget *reltarget;   /*扫描该Relation时默认的结果 list of Vars/Exprs, cost, width */
     
         /* materialization information */
         List       *pathlist;       /*访问路径链表 Path structures */
         List       *ppilist;        /*路径链表中使用参数化路径进行 ParamPathInfos used in pathlist */
         List       *partial_pathlist;   /* partial Paths */
         struct Path *cheapest_startup_path;//代价最低的启动路径
         struct Path *cheapest_total_path;//代价最低的整体路径
         struct Path *cheapest_unique_path;//代价最低的获取唯一值的路径
         List       *cheapest_parameterized_paths;//代价最低的参数化?路径链表
     
         /* parameterization information needed for both base rels and join rels */
         /* (see also lateral_vars and lateral_referencers) */
         Relids      direct_lateral_relids;  /*使用lateral语法,需依赖的Relids rels directly laterally referenced */
         Relids      lateral_relids; /* minimum parameterization of rel */
     
         /* information about a base rel (not set for join rels!) */
         //reloptkind=RELOPT_BASEREL时使用的数据结构
         Index       relid;          /* Relation ID */
         Oid         reltablespace;  /* 表空间 containing tablespace */
         RTEKind     rtekind;        /* 基表?子查询?还是函数等等?RELATION, SUBQUERY, FUNCTION, etc */
         AttrNumber  min_attr;       /* 最小的属性编号 smallest attrno of rel (often <0) */
         AttrNumber  max_attr;       /* 最大的属性编号 largest attrno of rel */
         Relids     *attr_needed;    /* 数组 array indexed [min_attr .. max_attr] */
         int32      *attr_widths;    /* 属性宽度 array indexed [min_attr .. max_attr] */
         List       *lateral_vars;   /* 关系依赖的Vars/PHVs LATERAL Vars and PHVs referenced by rel */
         Relids      lateral_referencers;    /*依赖该关系的Relids rels that reference me laterally */
         List       *indexlist;      /* 该关系的IndexOptInfo链表 list of IndexOptInfo */
         List       *statlist;       /* 统计信息链表 list of StatisticExtInfo */
         BlockNumber pages;          /* 块数 size estimates derived from pg_class */
         double      tuples;         /* 元组数 */
         double      allvisfrac;     /* ? */
         PlannerInfo *subroot;       /* 如为子查询,存储子查询的root if subquery */
         List       *subplan_params; /* 如为子查询,存储子查询的参数 if subquery */
         int         rel_parallel_workers;   /* 并行执行,需要多少个workers? wanted number of parallel workers */
     
         /* Information about foreign tables and foreign joins */
         //FWD相关信息
         Oid         serverid;       /* identifies server for the table or join */
         Oid         userid;         /* identifies user to check access as */
         bool        useridiscurrent;    /* join is only valid for current user */
         /* use "struct FdwRoutine" to avoid including fdwapi.h here */
         struct FdwRoutine *fdwroutine;
         void       *fdw_private;
     
         /* cache space for remembering if we have proven this relation unique */
         //已知的,可保证唯一的Relids链表
         List       *unique_for_rels;    /* known unique for these other relid
                                          * set(s) */
         List       *non_unique_for_rels;    /* 已知的,不唯一的Relids链表 known not unique for these set(s) */
     
         /* used by various scans and joins: */
         List       *baserestrictinfo;   /* 如为基本关系,存储约束条件 RestrictInfo structures (if base rel) */
         QualCost    baserestrictcost;   /* 解析约束表达式的成本? cost of evaluating the above */
         Index       baserestrict_min_security;  /* 最低安全等级 min security_level found in
                                                  * baserestrictinfo */
         List       *joininfo;       /* 连接语句的约束条件信息 RestrictInfo structures for join clauses
                                      * involving this rel */
         bool        has_eclass_joins;   /* 是否存在等价类连接? T means joininfo is incomplete */
     
         /* used by partitionwise joins: */
         bool        consider_partitionwise_join;    /* 分区? consider partitionwise
                                                      * join paths? (if
                                                      * partitioned rel) */
         Relids      top_parent_relids;  /* Relids of topmost parents (if "other"
                                          * rel) */
     
         /* used for partitioned relations */
         //分区表使用
         PartitionScheme part_scheme;    /* 分区的schema Partitioning scheme. */
         int         nparts;         /* 分区数 number of partitions */
         struct PartitionBoundInfoData *boundinfo;   /* 分区边界信息 Partition bounds */
         List       *partition_qual; /* 分区约束 partition constraint */
         struct RelOptInfo **part_rels;  /* 分区的RelOptInfo数组 Array of RelOptInfos of partitions,
                                          * stored in the same order of bounds */
         List      **partexprs;      /* 非空分区键表达式 Non-nullable partition key expressions. */
         List      **nullable_partexprs; /* 可为空的分区键表达式 Nullable partition key expressions. */
         List       *partitioned_child_rels; /* RT Indexes链表 List of RT indexes. */
     } RelOptInfo;
    
    

### 二、源码解读

join_search_one_level->...(如make_rels_by_clause_joins)->make_join_rel->populate_joinrel_with_paths函数为新生成的连接joinrel(给定参与连接的relations)构造访问路径.  
输入参数中的sjinfo(SpecialJoinInfo结构体)提供了有关连接的详细信息，限制条件链表restrictlist(List)包含连接条件子句和适用于给定连接关系对的其他条件子句。

    
    
    //-------------------------------------------------------------------- populate_joinrel_with_paths
    /*
     * populate_joinrel_with_paths
     *    Add paths to the given joinrel for given pair of joining relations. The
     *    SpecialJoinInfo provides details about the join and the restrictlist
     *    contains the join clauses and the other clauses applicable for given pair
     *    of the joining relations.
     *    为新生成的连接joinrel(给定参与连接的relations)构造访问路径.
     *    SpecialJoinInfo提供了有关连接的详细信息，
     *    限制条件链表包含连接条件子句和适用于给定连接关系对的其他条件子句。
     */
    static void
    populate_joinrel_with_paths(PlannerInfo *root, RelOptInfo *rel1,
                                RelOptInfo *rel2, RelOptInfo *joinrel,
                                SpecialJoinInfo *sjinfo, List *restrictlist)
    {
        /*
         * Consider paths using each rel as both outer and inner.  Depending on
         * the join type, a provably empty outer or inner rel might mean the join
         * is provably empty too; in which case throw away any previously computed
         * paths and mark the join as dummy.  (We do it this way since it's
         * conceivable that dummy-ness of a multi-element join might only be
         * noticeable for certain construction paths.)
         * 考虑使用每个rel分别作为外表和内表的路径。
         * 根据连接类型的不同，一个可证明为空的外表或内表可能意味着连接结果也是为空;
         * 在这种情况下，丢弃任何以前计算过的路径，并将连接标记为虚(dummy)连接。
         * 
         * Also, a provably constant-false join restriction typically means that
         * we can skip evaluating one or both sides of the join.  We do this by
         * marking the appropriate rel as dummy.  For outer joins, a
         * constant-false restriction that is pushed down still means the whole
         * join is dummy, while a non-pushed-down one means that no inner rows
         * will join so we can treat the inner rel as dummy.
         * 此外，可以证明的常量-false连接限制通常意味着我们可以跳过对连接的一个或两个方面的表达式解析。
       * 我们通过将适当的rel标记为虚(dummy)来完成这一操作。
       * 对于外连接，被下推的常量-false限制仍然意味着整个连接是虚连接，
       * 而非下推的限制意味着没有内部行会连接，因此我们可以将内部rel视为虚拟的。
       * 
         * We need only consider the jointypes that appear in join_info_list, plus
         * JOIN_INNER.
         * 只需要考虑出现在join_info_list和JOIN_INNER中的jointype。
         */
        switch (sjinfo->jointype)
        {
            case JOIN_INNER:
                if (is_dummy_rel(rel1) || is_dummy_rel(rel2) ||
                    restriction_is_constant_false(restrictlist, joinrel, false))
                {
                    mark_dummy_rel(joinrel);//设置为虚连接
                    break;
                }
                add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                     JOIN_INNER, sjinfo,
                                     restrictlist);//添加路径,rel1为外表,rel2为内表
                add_paths_to_joinrel(root, joinrel, rel2, rel1,
                                     JOIN_INNER, sjinfo,
                                     restrictlist);//添加路径,rel2为外表,rel1为内表
                break;
            case JOIN_LEFT://同上
                if (is_dummy_rel(rel1) ||
                    restriction_is_constant_false(restrictlist, joinrel, true))
                {
                    mark_dummy_rel(joinrel);
                    break;
                }
                if (restriction_is_constant_false(restrictlist, joinrel, false) &&
                    bms_is_subset(rel2->relids, sjinfo->syn_righthand))
                    mark_dummy_rel(rel2);
                add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                     JOIN_LEFT, sjinfo,
                                     restrictlist);
                add_paths_to_joinrel(root, joinrel, rel2, rel1,
                                     JOIN_RIGHT, sjinfo,
                                     restrictlist);
                break;
            case JOIN_FULL://同上
                if ((is_dummy_rel(rel1) && is_dummy_rel(rel2)) ||
                    restriction_is_constant_false(restrictlist, joinrel, true))
                {
                    mark_dummy_rel(joinrel);
                    break;
                }
                add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                     JOIN_FULL, sjinfo,
                                     restrictlist);
                add_paths_to_joinrel(root, joinrel, rel2, rel1,
                                     JOIN_FULL, sjinfo,
                                     restrictlist);
    
                /*
                 * If there are join quals that aren't mergeable or hashable, we
                 * may not be able to build any valid plan.  Complain here so that
                 * we can give a somewhat-useful error message.  (Since we have no
                 * flexibility of planning for a full join, there's no chance of
                 * succeeding later with another pair of input rels.)
                 * 如果有无法合并或不能合并的join quals，我们可能无法建立任何有效的计划。
             * 在这里报错，这样就可以给出一些有用的错误消息。
           * (由于无法灵活地为全连接添加计划，所以以后再加入另一对rels就没有成功的机会了。)
                 */
                if (joinrel->pathlist == NIL)
                    ereport(ERROR,
                            (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                             errmsg("FULL JOIN is only supported with merge-joinable or hash-joinable join conditions")));
                break;
            case JOIN_SEMI://半连接
    
                /*
                 * We might have a normal semijoin, or a case where we don't have
                 * enough rels to do the semijoin but can unique-ify the RHS and
                 * then do an innerjoin (see comments in join_is_legal).  In the
                 * latter case we can't apply JOIN_SEMI joining.
                 * 可能有一个普通的半连接，或者我们没有足够的rels来做半连接，
           * 但是可以通过唯一化RHS，然后做一个innerjoin(请参阅join_is_legal中的注释)。
           * 在后一种情况下，我们不能应用JOIN_SEMI join。
                 */
                if (bms_is_subset(sjinfo->min_lefthand, rel1->relids) &&
                    bms_is_subset(sjinfo->min_righthand, rel2->relids))
                {
                    if (is_dummy_rel(rel1) || is_dummy_rel(rel2) ||
                        restriction_is_constant_false(restrictlist, joinrel, false))
                    {
                        mark_dummy_rel(joinrel);
                        break;
                    }
                    add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                         JOIN_SEMI, sjinfo,
                                         restrictlist);
                }
    
                /*
                 * If we know how to unique-ify the RHS and one input rel is
                 * exactly the RHS (not a superset) we can consider unique-ifying
                 * it and then doing a regular join.  (The create_unique_path
                 * check here is probably redundant with what join_is_legal did,
                 * but if so the check is cheap because it's cached.  So test
                 * anyway to be sure.)
           * 如果我们知道如何唯一化RHS，一个输入rel恰好是RHS(不是超集)，
           * 我们可以考虑唯一化它，然后进行常规连接。
           * (这里的create_unique_path检查与join_is_legal的检查可能是冗余的，
             * 但是如果是的话，这样的检查的成本就显得很低，因为它可以缓存。所以不管怎样，还是要检查一下。
                 */
                if (bms_equal(sjinfo->syn_righthand, rel2->relids) &&
                    create_unique_path(root, rel2, rel2->cheapest_total_path,
                                       sjinfo) != NULL)
                {
                    if (is_dummy_rel(rel1) || is_dummy_rel(rel2) ||
                        restriction_is_constant_false(restrictlist, joinrel, false))
                    {
                        mark_dummy_rel(joinrel);
                        break;
                    }
                    add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                         JOIN_UNIQUE_INNER, sjinfo,
                                         restrictlist);
                    add_paths_to_joinrel(root, joinrel, rel2, rel1,
                                         JOIN_UNIQUE_OUTER, sjinfo,
                                         restrictlist);
                }
                break;
            case JOIN_ANTI://反连接
                if (is_dummy_rel(rel1) ||
                    restriction_is_constant_false(restrictlist, joinrel, true))
                {
                    mark_dummy_rel(joinrel);
                    break;
                }
                if (restriction_is_constant_false(restrictlist, joinrel, false) &&
                    bms_is_subset(rel2->relids, sjinfo->syn_righthand))
                    mark_dummy_rel(rel2);
                add_paths_to_joinrel(root, joinrel, rel1, rel2,
                                     JOIN_ANTI, sjinfo,
                                     restrictlist);
                break;
            default://非法的连接类型
                /* other values not expected here */
                elog(ERROR, "unrecognized join type: %d", (int) sjinfo->jointype);
                break;
        }
    
        /* 尝试partitionwise技术. Apply partitionwise join technique, if possible. */
        try_partitionwise_join(root, rel1, rel2, joinrel, sjinfo, restrictlist);
    }
    
    
    //------------------------------------------------------------------- add_paths_to_joinrel
    /*
     * add_paths_to_joinrel
     *    Given a join relation and two component rels from which it can be made,
     *    consider all possible paths that use the two component rels as outer
     *    and inner rel respectively.  Add these paths to the join rel's pathlist
     *    if they survive comparison with other paths (and remove any existing
     *    paths that are dominated by these paths).
     *    给出组成连接的两个组合rels,尝试所有可能的路径进行连接,比如分别设置为outer和inner表等.
     *    如果连接的路径在与其他路径的比较中可以留存下来，
     *    则将这些路径添加到连接rel的路径列表中(并删除现有的由这些路径控制的其他路径)。
     * 
     * Modifies the pathlist field of the joinrel node to contain the best
     * paths found so far.
     * 更新joinrel->pathlist链表已容纳最优的访问路径.
     *
     * jointype is not necessarily the same as sjinfo->jointype; it might be
     * "flipped around" if we are considering joining the rels in the opposite
     * direction from what's indicated in sjinfo.
     * jointype不需要与sjinfo->jointype一致,如果我们考虑加入与sjinfo所示相反方向的rels，它可能是“翻转”的。
     * 
     * Also, this routine and others in this module accept the special JoinTypes
     * JOIN_UNIQUE_OUTER and JOIN_UNIQUE_INNER to indicate that we should
     * unique-ify the outer or inner relation and then apply a regular inner
     * join.  These values are not allowed to propagate outside this module,
     * however.  Path cost estimation code may need to recognize that it's
     * dealing with such a case --- the combination of nominal jointype INNER
     * with sjinfo->jointype == JOIN_SEMI indicates that.
     * 此外，这个处理过程和这个模块中的其他过程接受特殊的JoinTypes(JOIN_UNIQUE_OUTER和JOIN_UNIQUE_INNER)，
     * 以表明应该对外部或内部关系进行唯一化，然后应用一个常规的内部连接。
     * 但是，这些值不允许传播到这个模块之外。
     * 访问路径成本估算过程可能需要认识到，
     * 它正在处理这样的情况———名义上的INNER jointype与sjinfo->jointype == JOIN_SEMI的组合。
     */
    void
    add_paths_to_joinrel(PlannerInfo *root,
                         RelOptInfo *joinrel,
                         RelOptInfo *outerrel,
                         RelOptInfo *innerrel,
                         JoinType jointype,
                         SpecialJoinInfo *sjinfo,
                         List *restrictlist)
    {
        JoinPathExtraData extra;
        bool        mergejoin_allowed = true;
        ListCell   *lc;
        Relids      joinrelids;
    
        /*
         * PlannerInfo doesn't contain the SpecialJoinInfos created for joins
         * between child relations, even if there is a SpecialJoinInfo node for
         * the join between the topmost parents. So, while calculating Relids set
         * representing the restriction, consider relids of topmost parent of
         * partitions.
         * PlannerInfo不包含为子关系之间的连接创建的SpecialJoinInfo，
       * 即使最顶层的父关系之间有一个SpecialJoinInfo节点。
       * 因此，在计算表示限制条件的Relids集合时，需考虑分区的最顶层父类的Relids。
         */
        if (joinrel->reloptkind == RELOPT_OTHER_JOINREL)
            joinrelids = joinrel->top_parent_relids;
        else
            joinrelids = joinrel->relids;
    
        extra.restrictlist = restrictlist;
        extra.mergeclause_list = NIL;
        extra.sjinfo = sjinfo;
        extra.param_source_rels = NULL;
    
        /*
         * See if the inner relation is provably unique for this outer rel.
         * 判断内表是否已被验证为唯一.
         * 
         * We have some special cases: for JOIN_SEMI and JOIN_ANTI, it doesn't
         * matter since the executor can make the equivalent optimization anyway;
         * we need not expend planner cycles on proofs.  For JOIN_UNIQUE_INNER, we
         * must be considering a semijoin whose inner side is not provably unique
         * (else reduce_unique_semijoins would've simplified it), so there's no
         * point in calling innerrel_is_unique.  However, if the LHS covers all of
         * the semijoin's min_lefthand, then it's appropriate to set inner_unique
         * because the path produced by create_unique_path will be unique relative
         * to the LHS.  (If we have an LHS that's only part of the min_lefthand,
         * that is *not* true.)  For JOIN_UNIQUE_OUTER, pass JOIN_INNER to avoid
         * letting that value escape this module.
         * 存在一些特殊的情况:
         * 1.对于JOIN_SEMI和JOIN_ANTI，这无关紧要，因为执行器无论如何都可以进行等价的优化;
         *   这些不需要在证明上花费时间证明。
       *  2.对于JOIN_UNIQUE_INNER，必须考虑一个内部不是唯一的半连接(否则reduce_unique_semijoin会简化它)，
       *    所以调用innerrel_is_unique没有任何意义。
       *    但是，如果LHS覆盖了半连接的所有min_left，那么就应该设置inner_unique，
       *    因为create_unique_path生成的路径相对于LHS是唯一的。
       *    (如果LHS只是min_left的一部分，那就不是真的)
       *    对于JOIN_UNIQUE_OUTER，传递JOIN_INNER以避免让该值转义这个模块。
         */
        switch (jointype)
        {
            case JOIN_SEMI:
            case JOIN_ANTI:
                extra.inner_unique = false; /* well, unproven */
                break;
            case JOIN_UNIQUE_INNER:
                extra.inner_unique = bms_is_subset(sjinfo->min_lefthand,
                                                   outerrel->relids);
                break;
            case JOIN_UNIQUE_OUTER:
                extra.inner_unique = innerrel_is_unique(root,
                                                        joinrel->relids,
                                                        outerrel->relids,
                                                        innerrel,
                                                        JOIN_INNER,
                                                        restrictlist,
                                                        false);
                break;
            default:
                extra.inner_unique = innerrel_is_unique(root,
                                                        joinrel->relids,
                                                        outerrel->relids,
                                                        innerrel,
                                                        jointype,
                                                        restrictlist,
                                                        false);
                break;
        }
    
        /*
         * Find potential mergejoin clauses.  We can skip this if we are not
         * interested in doing a mergejoin.  However, mergejoin may be our only
         * way of implementing a full outer join, so override enable_mergejoin if
         * it's a full join.
         * 寻找潜在的mergejoin条件。如果不允许Merge Join,则跳过。
       * 然而，mergejoin可能是实现完整外部连接的唯一方法，
         * 因此，如果它是完全连接，则不理会enable_mergejoin参数。
         */
        if (enable_mergejoin || jointype == JOIN_FULL)
            extra.mergeclause_list = select_mergejoin_clauses(root,
                                                              joinrel,
                                                              outerrel,
                                                              innerrel,
                                                              restrictlist,
                                                              jointype,
                                                              &mergejoin_allowed);
    
        /*
         * If it's SEMI, ANTI, or inner_unique join, compute correction factors
         * for cost estimation.  These will be the same for all paths.
         * 如果是半连接、反连接或inner_unique连接，则计算成本估算的相关因子,该值对所有路径都是一样的。
         */
        if (jointype == JOIN_SEMI || jointype == JOIN_ANTI || extra.inner_unique)
            compute_semi_anti_join_factors(root, joinrel, outerrel, innerrel,
                                           jointype, sjinfo, restrictlist,
                                           &extra.semifactors);
    
        /*
         * Decide whether it's sensible to generate parameterized paths for this
         * joinrel, and if so, which relations such paths should require.  There
         * is usually no need to create a parameterized result path unless there
         * is a join order restriction that prevents joining one of our input rels
         * directly to the parameter source rel instead of joining to the other
         * input rel.  (But see allow_star_schema_join().)  This restriction
         * reduces the number of parameterized paths we have to deal with at
         * higher join levels, without compromising the quality of the resulting
         * plan.  We express the restriction as a Relids set that must overlap the
         * parameterization of any proposed join path.
         * 确定为这个连接生成参数化路径是否合理，如果是，这些路径应该需要哪些关系。
       * 通常不需要创建一个参数化的结果路径，除非存在一个连接顺序限制，
       * 阻止将一个关系直接连接到参数源rel，而不是连接到另一个input rel(但请参阅allow_star_schema_join())。
       * 这种限制减少了在更高的连接级别上必须处理的参数化路径的数量，而不会影响最终计划的质量。
       * 把这个限制表示为一个Relids集合，它必须与任何建议的连接路径的参数化重叠。
         */
        foreach(lc, root->join_info_list)
        {
            SpecialJoinInfo *sjinfo2 = (SpecialJoinInfo *) lfirst(lc);
    
            /*
             * SJ is relevant to this join if we have some part of its RHS
             * (possibly not all of it), and haven't yet joined to its LHS.  (This
             * test is pretty simplistic, but should be sufficient considering the
             * join has already been proven legal.)  If the SJ is relevant, it
             * presents constraints for joining to anything not in its RHS.
         * 与这个连接相关的SJ，如果有它的部分RHS(可能不是全部)，并且还没有加入它的LHS。
           * (这个验证非常简单，但是考虑到连接已经被证明是合法的，这个验证就足够了。)
         * 如果SJ是相关的，那么它就为连接到其RHS之外的任何内容提供了约束。
             */
            if (bms_overlap(joinrelids, sjinfo2->min_righthand) &&
                !bms_overlap(joinrelids, sjinfo2->min_lefthand))
                extra.param_source_rels = bms_join(extra.param_source_rels,
                                                   bms_difference(root->all_baserels,
                                                                  sjinfo2->min_righthand));
    
            /* 全连接在语法上约束左右两边.full joins constrain both sides symmetrically */
            if (sjinfo2->jointype == JOIN_FULL &&
                bms_overlap(joinrelids, sjinfo2->min_lefthand) &&
                !bms_overlap(joinrelids, sjinfo2->min_righthand))
                extra.param_source_rels = bms_join(extra.param_source_rels,
                                                   bms_difference(root->all_baserels,
                                                                  sjinfo2->min_lefthand));
        }
    
        /*
         * However, when a LATERAL subquery is involved, there will simply not be
         * any paths for the joinrel that aren't parameterized by whatever the
         * subquery is parameterized by, unless its parameterization is resolved
         * within the joinrel.  So we might as well allow additional dependencies
         * on whatever residual lateral dependencies the joinrel will have.
         * 然而，当涉及到一个LATERAL子查询时，除非在joinrel中解析其参数化，
       * 否则joinrel的任何路径都不会被子查询参数化。
       * 因此，也可以允许对joinrel将拥有的任何剩余的LATERAL依赖进行额外依赖。
         */
        extra.param_source_rels = bms_add_members(extra.param_source_rels,
                                                  joinrel->lateral_relids);
    
        /*
         * 1. Consider mergejoin paths where both relations must be explicitly
         * sorted.  Skip this if we can't mergejoin.
         * 1. 尝试merge join访问路径，其中两个关系必须执行显式的排序。
         *    如果禁用merge join，则跳过。
         */
        if (mergejoin_allowed)
            sort_inner_and_outer(root, joinrel, outerrel, innerrel,
                                 jointype, &extra);
    
        /*
         * 2. Consider paths where the outer relation need not be explicitly
         * sorted. This includes both nestloops and mergejoins where the outer
         * path is already ordered.  Again, skip this if we can't mergejoin.
         * (That's okay because we know that nestloop can't handle right/full
         * joins at all, so it wouldn't work in the prohibited cases either.)
         * 2. 考虑外部关系不需要显式排序的路径。
       *    这包括nestloop和mergejoin，它们的外部路径已经排序。
       *    再一次的，如果禁用merge join，则跳过。
         *    (nestloop无法处理正确/完全连接，所以在禁止的情况下它也无法工作)
         */
        if (mergejoin_allowed)
            match_unsorted_outer(root, joinrel, outerrel, innerrel,
                                 jointype, &extra);
    
    #ifdef NOT_USED
    
        /*
         * 3. Consider paths where the inner relation need not be explicitly
         * sorted.  This includes mergejoins only (nestloops were already built in
         * match_unsorted_outer).
         * 3. 尝试内部关系不需要显式排序的路径。这只包括mergejoin(在match_unsorted_outer中已经构建了nestloop)。
         * (已废弃)
       * 
         * Diked out as redundant 2/13/2000 -- tgl.  There isn't any really
         * significant difference between the inner and outer side of a mergejoin,
         * so match_unsorted_inner creates no paths that aren't equivalent to
         * those made by match_unsorted_outer when add_paths_to_joinrel() is
         * invoked with the two rels given in the other order.
         */
        if (mergejoin_allowed)
            match_unsorted_inner(root, joinrel, outerrel, innerrel,
                                 jointype, &extra);
    #endif
    
        /*
         * 4. Consider paths where both outer and inner relations must be hashed
         * before being joined.  As above, disregard enable_hashjoin for full
         * joins, because there may be no other alternative.
       * 4. 考虑在连接之前必须对外部/内部关系进行散列处理的路径。
       *    如上所述，对于完全连接，忽略enable_hashjoin，因为可能没有其他选择。
         */
        if (enable_hashjoin || jointype == JOIN_FULL)
            hash_inner_and_outer(root, joinrel, outerrel, innerrel,
                                 jointype, &extra);
    
        /*
         * 5. If inner and outer relations are foreign tables (or joins) belonging
         * to the same server and assigned to the same user to check access
         * permissions as, give the FDW a chance to push down joins.
         * 如果内部和外部关系是属于同一服务器的外部表(或连接)，
         * 并分配给同一用户以检查访问权限，则FDW有机会下推连接。
         */
        if (joinrel->fdwroutine &&
            joinrel->fdwroutine->GetForeignJoinPaths)
            joinrel->fdwroutine->GetForeignJoinPaths(root, joinrel,
                                                     outerrel, innerrel,
                                                     jointype, &extra);
    
        /*
         * 6. Finally, give extensions a chance to manipulate the path list.
         * 6. 最后,调用扩展钩子函数.
         */
        if (set_join_pathlist_hook)
            set_join_pathlist_hook(root, joinrel, outerrel, innerrel,
                                   jointype, &extra);
    }
    
    

### 三、跟踪分析

SQL语句如下:

    
    
    testdb=# explain verbose select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
                            from t_grxx gr inner join t_jfxx jf 
                                           on gr.dwbh = dw.dwbh 
                                              and gr.grbh = jf.grbh) grjf 
    order by dw.dwbh;
                                                  QUERY PLAN                                               
    -------------------------------------------------------------------------------------------------------     Merge Join  (cost=18841.64..21009.94 rows=99850 width=47)
       Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
       Merge Cond: ((dw.dwbh)::text = (gr.dwbh)::text)
       ->  Index Scan using t_dwxx_pkey on public.t_dwxx dw  (cost=0.29..399.62 rows=10000 width=20)
             Output: dw.dwmc, dw.dwbh, dw.dwdz
       ->  Materialize  (cost=18836.82..19336.82 rows=100000 width=31)
             Output: gr.grbh, gr.xm, gr.dwbh, jf.ny, jf.je
             ->  Sort  (cost=18836.82..19086.82 rows=100000 width=31)
                   Output: gr.grbh, gr.xm, gr.dwbh, jf.ny, jf.je
                   Sort Key: gr.dwbh
                   ->  Hash Join  (cost=3465.00..8138.00 rows=100000 width=31)
                         Output: gr.grbh, gr.xm, gr.dwbh, jf.ny, jf.je
                         Hash Cond: ((jf.grbh)::text = (gr.grbh)::text)
                         ->  Seq Scan on public.t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                               Output: jf.ny, jf.je, jf.grbh
                         ->  Hash  (cost=1726.00..1726.00 rows=100000 width=16)
                               Output: gr.grbh, gr.xm, gr.dwbh
                               ->  Seq Scan on public.t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
                                     Output: gr.grbh, gr.xm, gr.dwbh
    (19 rows)
    

参与连接的有3张基表,分别是t_dwxx/t_grxx/t_jfxx,从执行计划可见,由于存在order by
dwbh排序子句,优化器"聪明"的选择Merge Join.

启动gdb,设置断点,只考察level=3的情况(最终结果)

    
    
    (gdb) b join_search_one_level
    Breakpoint 1 at 0x755667: file joinrels.c, line 67.
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x1cae678, level=2) at joinrels.c:67
    67    List    **joinrels = root->join_rel_level;
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x1cae678, level=3) at joinrels.c:67
    67    List    **joinrels = root->join_rel_level;
    (gdb) 
    

跟踪populate_joinrel_with_paths

    
    
    (gdb) b populate_joinrel_with_paths
    Breakpoint 2 at 0x75646d: file joinrels.c, line 780.
    

进入populate_joinrel_with_paths函数

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 2, populate_joinrel_with_paths (root=0x1cae678, rel1=0x1d10978, rel2=0x1d09610, joinrel=0x1d131b8, 
        sjinfo=0x7ffef59baf20, restrictlist=0x1d135e8) at joinrels.c:780
    780   switch (sjinfo->jointype)
    

查看输入参数  
1.root:simple_rte_array数组,其中simple_rel_array_size =
6,存在6个Item,1->16734/t_dwxx,3->16742/t_grxx,4->16747/t_jfxx  
2.rel1:1号和3号连接生成的Relation,即t_dwxx和t_grxx连接  
3.rel2:4号RTE,即t_jfxx  
4.joinrel:rel1和rel2通过build_join_rel函数生成的连接Relation  
5.sjinfo:连接信息,连接类型为内连接JOIN_INNER  
6.restrictlist:约束条件链表,t_grxx.grbh=t_jfxx.grbh

    
    
    (gdb) p *root
    $3 = {type = T_PlannerInfo, parse = 0x1cd7830, glob = 0x1cb8d38, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x1d07af8, simple_rel_array_size = 6, simple_rte_array = 0x1d07b48, 
      all_baserels = 0x1d0ada8, nullable_baserels = 0x0, join_rel_list = 0x1d10e48, join_rel_hash = 0x0, 
      join_rel_level = 0x1d10930, join_cur_level = 3, init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, 
      eq_classes = 0x1d0a6d8, canon_pathkeys = 0x1d0ad28, left_join_clauses = 0x0, right_join_clauses = 0x0, 
      full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, rowMarks = 0x0, placeholder_list = 0x0, 
      fkey_list = 0x0, query_pathkeys = 0x1d0ad78, group_pathkeys = 0x0, window_pathkeys = 0x0, distinct_pathkeys = 0x0, 
      sort_pathkeys = 0x1d0ad78, part_schemes = 0x0, initial_rels = 0x1d108c0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 
        0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, processed_tlist = 0x1cbb608, grouping_map = 0x0, 
      minmax_aggs = 0x0, planner_cxt = 0x1bfa040, total_table_pages = 1427, tuple_fraction = 0, limit_tuples = -1, 
      qual_security_level = 0, inhTargetKind = INHKIND_NONE, hasJoinRTEs = true, hasLateralRTEs = false, 
      hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, hasRecursion = false, wt_param_id = -1, 
      non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, join_search_private = 0x0, partColsUpdated = false}
    (gdb) p *root->simple_rte_array[1]
    $4 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16734, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, ...
    ...
    (gdb) p *rel1->relids
    $10 = {nwords = 1, words = 0x1d10b8c}
    (gdb) p *rel1->relids->words
    $11 = 10
    (gdb) p *rel2->relids->words
    $13 = 16
    (gdb) p *joinrel->relids->words
    $15 = 26
    (gdb) p *sjinfo
    $16 = {type = T_SpecialJoinInfo, min_lefthand = 0x1d10b88, min_righthand = 0x1d09518, syn_lefthand = 0x1d10b88, 
      syn_righthand = 0x1d09518, jointype = JOIN_INNER, lhs_strict = false, delay_upper_joins = false, semi_can_btree = false, 
      semi_can_hash = false, semi_operators = 0x0, semi_rhs_exprs = 0x0}
    ...
    (gdb) p *(Var *)((RelabelType *)$args->head->data.ptr_value)->arg
    $34 = {xpr = {type = T_Var}, varno = 3, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 2, location = 273}  -->t_grxx.grbh
    (gdb) p *(Var *)((RelabelType *)$args->head->next->data.ptr_value)->arg
    $35 = {xpr = {type = T_Var}, varno = 4, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 4, varoattno = 1, location = 283}  -->t_jfxx.grbh
    

进入JOIN_INNER分支,调用函数add_paths_to_joinrel

    
    
    (gdb) 
    789       add_paths_to_joinrel(root, joinrel, rel1, rel2,
    

进入add_paths_to_joinrel函数

    
    
    (gdb) step
    add_paths_to_joinrel (root=0x1cae678, joinrel=0x1d131b8, outerrel=0x1d10978, innerrel=0x1d09610, jointype=JOIN_INNER, 
        sjinfo=0x7ffef59baf20, restrictlist=0x1d135e8) at joinpath.c:126
    126   bool    mergejoin_allowed = true;
    

判断内表是否已被验证为唯一

    
    
    162   switch (jointype)
    (gdb) 
    182       extra.inner_unique = innerrel_is_unique(root,
    (gdb) 
    189       break;
    (gdb) p extra.inner_unique
    $36 = false
    

寻找潜在的mergejoin条件。如果不允许Merge Join,则跳过  
merge join的条件是t_grxx.grbh=t_jfxx.grbh

    
    
    (gdb) n
    198   if (enable_mergejoin || jointype == JOIN_FULL)
    (gdb) 
    199     extra.mergeclause_list = select_mergejoin_clauses(root,
    (gdb) 
    211   if (jointype == JOIN_SEMI || jointype == JOIN_ANTI || extra.inner_unique)
    (gdb) p *(Var *)((RelabelType *)$args->head->data.ptr_value)->arg
    $47 = {xpr = {type = T_Var}, varno = 3, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 2, location = 273} -->t_grxx.grbh
    (gdb) p *(Var *)((RelabelType *)$args->head->next->data.ptr_value)->arg
    $48 = {xpr = {type = T_Var}, varno = 4, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 4, varoattno = 1, location = 283} -->t_jfxx.grbh
    

确定为这个连接生成参数化路径是否合理，如果是，这些路径应该需要哪些关系(结果为:NULL)

    
    
    (gdb) 
    261   extra.param_source_rels = bms_add_members(extra.param_source_rels,
    (gdb) 
    268   if (mergejoin_allowed)
    (gdb) p *extra.param_source_rels
    Cannot access memory at address 0x0
    

尝试merge join访问路径，其中两个关系必须执行显式的排序.  
注:joinrel->pathlist在执行前为NULL,执行后生成了访问路径.

    
    
    (gdb) p *joinrel->pathlist
    Cannot access memory at address 0x0
    (gdb) n
    269     sort_inner_and_outer(root, joinrel, outerrel, innerrel,
    (gdb) 
    279   if (mergejoin_allowed)
    (gdb) p *joinrel->pathlist
    $50 = {type = T_List, length = 1, head = 0x1d13850, tail = 0x1d13850}
    

其他实现逻辑类似,sort_inner_and_outer等函数的实现逻辑,后续再行详细解读.  
最终结果是生成了2条访问路径,存储在pathlist链表中.

    
    
    324   if (set_join_pathlist_hook)
    (gdb) 
    327 }
    (gdb) p *joinrel->pathlist
    $51 = {type = T_List, length = 2, head = 0x1d13850, tail = 0x1d13930}
    (gdb) p *(Node *)joinrel->pathlist->head->data.ptr_value
    $52 = {type = T_HashPath}
    (gdb) p *(HashPath *)joinrel->pathlist->head->data.ptr_value
    $53 = {jpath = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x1d131b8, pathtarget = 0x1d133c8, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 99850, 
          startup_cost = 3762, total_cost = 10075.348750000001, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
        outerjoinpath = 0x1d11f48, innerjoinpath = 0x1d0f548, joinrestrictinfo = 0x1d135e8}, path_hashclauses = 0x1d13aa0, 
      num_batches = 2, inner_rows_total = 100000}
    (gdb) p *(Node *)joinrel->pathlist->head->next->data.ptr_value
    $54 = {type = T_NestPath}
    (gdb) p *(NestPath *)joinrel->pathlist->head->next->data.ptr_value
    $55 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x1d131b8, pathtarget = 0x1d133c8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 99850, startup_cost = 39.801122856046675, 
        total_cost = 41318.966172885761, pathkeys = 0x1d0b818}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x1d119d8, innerjoinpath = 0x1d0f9d8, joinrestrictinfo = 0x0}
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

