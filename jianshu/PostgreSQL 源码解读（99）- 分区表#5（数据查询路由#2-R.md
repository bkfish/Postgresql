上一章节已介绍了如何识别分区表并找到所有的分区子表以及函数expand_inherited_tables的主要实现逻辑,该函数把子表信息放在root(PlannerInfo)->append_rel_list链表中。为了更好的理解相关的逻辑,本节重点介绍分区表查询相关的重要数据结构,包括RelOptInfo/PlannerInfo/AppendRelInfo.

### 一、数据结构

**RelOptInfo**  
RelOptInfo是规划器/优化器使用的关系信息结构体  
在规划过程中已存在的基表或者是在关系运算过程中产生的中间关系或者是最终产生的关系,都使用RelOptInfo结构体进行封装表示.  
查询分区表时该结构体中的part_scheme存储分区的schema,nparts存储分区数,boundinfo是分区边界信息,partition_qual是分区约束条件,part_rels是分区表中每个分区的每个分区的RelOptInfo,partexprs是分区键表达式,partitioned_child_rels是关系中未修剪(unpruned)分区的RT索引.unpruned是指查询分区表时,涉及的相关分区,比如
where c1 = 1 OR c1 =
2涉及的分区只有t_hash_partition_1和t_hash_partition_3,则在partitioned_child_rels中只有这两个分区的信息.  
**如何pruned,下节介绍**

    
    
    /*----------     * RelOptInfo
     *      Per-relation information for planning/optimization
     *      规划器/优化器使用的关系信息结构体
     *
     * For planning purposes, a "base rel" is either a plain relation (a table)
     * or the output of a sub-SELECT or function that appears in the range table.
     * In either case it is uniquely identified by an RT index.  A "joinrel"
     * is the joining of two or more base rels.  A joinrel is identified by
     * the set of RT indexes for its component baserels.  We create RelOptInfo
     * nodes for each baserel and joinrel, and store them in the PlannerInfo's
     * simple_rel_array and join_rel_list respectively.
     * 出于计划目的，“base rel”要么是一个普通关系(表)，
     *   要么是出现在范围表中的子查询或函数的输出。
     * 在这两种情况下，它都是由RT索引惟一标识的。
     * "joinrel"是两个或两个以上的base rels连接。
     * 一个joinrel是由它的baserels的RT索引集标识。
     * 我们为每个baserel和joinrel分别创建RelOptInfo节点，
     *   并将它们分别存储在PlannerInfo的simple_rel_array和join_rel_list中。
     *
     * Note that there is only one joinrel for any given set of component
     * baserels, no matter what order we assemble them in; so an unordered
     * set is the right datatype to identify it with.
     * 请注意，对于任何给定的base rels，无论我们以何种顺序组合它们，
     *   都只有一个连接件;因此，一个无序的集合正好是标识它的数据类型。
     *
     * We also have "other rels", which are like base rels in that they refer to
     * single RT indexes; but they are not part of the join tree, and are given
     * a different RelOptKind to identify them.
     * Currently the only kind of otherrels are those made for member relations
     * of an "append relation", that is an inheritance set or UNION ALL subquery.
     * An append relation has a parent RTE that is a base rel, which represents
     * the entire append relation.  The member RTEs are otherrels.  The parent
     * is present in the query join tree but the members are not.  The member
     * RTEs and otherrels are used to plan the scans of the individual tables or
     * subqueries of the append set; then the parent baserel is given Append
     * and/or MergeAppend paths comprising the best paths for the individual
     * member rels.  (See comments for AppendRelInfo for more information.)
     * 同时存在“other rels”，类似于base rels，它们同样都指向单个RT索引;
     *   但是它们不是join树的一部分，并且被赋予不同的RelOptKind来标识它们。
     * 目前唯一的其他类型是那些为“append relation”的成员关系而创建的，
     *   即继承集或UNION ALL子查询。
     * 一个append relation有一个父RTE，它是一个基础rel，表示整个append relation。
     * 其他成员RTEs是otherrel.。
     * 父节点存在于查询的连接树中，但成员无需存储在连接树中。
     * 成员RTEs和otherrels用于计划对append set的单表或子查询的扫描;
     *   然后给出parent base rel的APPEND路径和/或MergeAppend路径，这些路径包含单个成员rels的最佳路径。
     * (更多信息请参见AppendRelInfo的注释。)
     * 
     * At one time we also made otherrels to represent join RTEs, for use in
     * handling join alias Vars.  Currently this is not needed because all join
     * alias Vars are expanded to non-aliased form during preprocess_expression.
     * 曾经，还制作了其他的树来表示连接rte，用于处理连接别名Vars。
     * 目前不需要这样做，因为在preprocess_expression期间，所有连接别名Vars都被扩展为非别名形式。
     *
     * We also have relations representing joins between child relations of
     * different partitioned tables. These relations are not added to
     * join_rel_level lists as they are not joined directly by the dynamic
     * programming algorithm.
     * 还有表示不同分区表的子关系之间的连接的关系。
     * 这些关系不会添加到join_rel_level链表中，因为动态规划算法不会直接连接它们。
     *
     * There is also a RelOptKind for "upper" relations, which are RelOptInfos
     * that describe post-scan/join processing steps, such as aggregation.
     * Many of the fields in these RelOptInfos are meaningless, but their Path
     * fields always hold Paths showing ways to do that processing step.
     * 还有一种RelOptKind表示“upper”关系，
     *   即描述扫描/连接后处理步骤(如聚合)的RelOptInfos。
     * 这些RelOptInfos中的许多字段都是没有意义的，
     *   但是它们的Path字段总是包含显示执行该处理步骤的路径。
     * 
     * Lastly, there is a RelOptKind for "dead" relations, which are base rels
     * that we have proven we don't need to join after all.
     * 最后，还有一种关系是“DEAD”的关系，在规划期间已经证明不需要把此关系加入连接。
     * 
     * Parts of this data structure are specific to various scan and join
     * mechanisms.  It didn't seem worth creating new node types for them.
     * 该数据结构的某些部分特定于各种扫描和连接机制。
     * 但似乎不值得为它们创建新的节点类型。
     * 
     *      relids - Set of base-relation identifiers; it is a base relation
     *              if there is just one, a join relation if more than one
     *      relids - 基础关系标识符的集合;如只有一个则为基础关系,
     *               如有多个则为连接关系
     *      rows - estimated number of tuples in the relation after restriction
     *             clauses have been applied (ie, output rows of a plan for it)
     *      rows - 应用约束条件子句后关系中元组的估算数目(即计划的输出行数)
     *      consider_startup - true if there is any value in keeping plain paths for
     *                         this rel on the basis of having cheap startup cost
     *      consider_startup - 如果在具有低启动成本的基础上为这个rel保留访问路径有价值，则为真
     *      consider_param_startup - the same for parameterized paths
     *      consider_param_startup - 与parameterized访问路径一致
     *      reltarget - Default Path output tlist for this rel; normally contains
     *                  Var and PlaceHolderVar nodes for the values we need to
     *                  output from this relation.
     *                  List is in no particular order, but all rels of an
     *                  appendrel set must use corresponding orders.
     *                  NOTE: in an appendrel child relation, may contain
     *                  arbitrary expressions pulled up from a subquery!
     *      reltarget - 该rel的默认路径输出投影列;通常会包含Var和PlaceHolderVar
     *      pathlist - List of Path nodes, one for each potentially useful
     *                 method of generating the relation
     *                 访问路径节点链表, 存储每一种可能有用的生成关系的方法
     *      ppilist - ParamPathInfo nodes for parameterized Paths, if any
     *                参数化路径的ParamPathInfo节点(如果有的话)
     *      cheapest_startup_path - the pathlist member with lowest startup cost
     *          (regardless of ordering) among the unparameterized paths;
     *          or NULL if there is no unparameterized path
     *          在非参数化路径中启动成本最低(无论顺序如何)的路径链表成员;
     *              如果没有非参数化路径，则为NULL
     *      cheapest_total_path - the pathlist member with lowest total cost
     *          (regardless of ordering) among the unparameterized paths;
     *          or if there is no unparameterized path, the path with lowest
     *          total cost among the paths with minimum parameterization
     *          在非参数化路径中总成本最低(无论顺序如何)的路径列表成员;
     *              如果没有非参数化路径，则在参数化最少的路径中总成本最低的路径
     *      cheapest_unique_path - for caching cheapest path to produce unique
     *          (no duplicates) output from relation; NULL if not yet requested
     *          用于缓存最便宜的路径，以便从关系中产生唯一(无重复)输出;
     *              如果无此要求，则为NULL
     *      cheapest_parameterized_paths - best paths for their parameterizations;
     *          always includes cheapest_total_path, even if that's unparameterized
     *          参数化的最佳路径;总是包含cheapest_total_path，即使它是非参数化的       
     *      direct_lateral_relids - rels this rel has direct LATERAL references to
     *          该rel直接LATERAL依赖的rels
     *      lateral_relids - required outer rels for LATERAL, as a Relids set
     *          (includes both direct and indirect lateral references)
     *          LATERAL所需的外部rels，作为Relids集合(包括直接和间接的侧向参考)     
     *
     * If the relation is a base relation it will have these fields set:
     * 如果关系是一个基本关系，它将设置这些字段:
     *      relid - RTE index (this is redundant with the relids field, but
     *              is provided for convenience of access)
     *          RTE索引(这对于relids字段来说是冗余的，但是为了方便访问而提供)            
     *      rtekind - copy of RTE's rtekind field
     *          RTE's rtekind字段的拷贝
     *      min_attr, max_attr - range of valid AttrNumbers for rel
     *          关系有效AttrNumbers的范围(最大/最小编号)
     *      attr_needed - array of bitmapsets indicating the highest joinrel
     *              in which each attribute is needed; if bit 0 is set then
     *              the attribute is needed as part of final targetlist
     *          位图集数组，表示每个属性所需的最高层joinrel;
     *              如果设置为0，则需要将该属性作为最终targetlist的一部分      
     *      attr_widths - cache space for per-attribute width estimates;
     *                    zero means not computed yet
     *          用于每个属性宽度估计的缓存空间;0表示还没有计算
     *      lateral_vars - lateral cross-references of rel, if any (list of
     *                     Vars and PlaceHolderVars)
     *          rel的lateral交叉参照，如果有的话(Vars和PlaceHolderVars链表)
     *      lateral_referencers - relids of rels that reference this one laterally
     *              (includes both direct and indirect lateral references)
     *          lateral依赖此关系的relids(包括直接&间接lateral依赖)
     *      indexlist - list of IndexOptInfo nodes for relation's indexes
     *                  (always NIL if it's not a table)
     *          关系索引的IndexOptInfo节点链表(如果不是表，总是NIL)  
     *      pages - number of disk pages in relation (zero if not a table)
     *          关系的磁盘页数(如不是表,则为0)
     *      tuples - number of tuples in relation (not considering restrictions)
     *          关系的元组数统计(还没有考虑约束条件)
     *      allvisfrac - fraction of disk pages that are marked all-visible
     *          标记为all-visible的磁盘页数比例
     *      subroot - PlannerInfo for subquery (NULL if it's not a subquery)
     *          用于子查询的PlannerInfo(如果不是子查询，则为NULL)
     *      subplan_params - list of PlannerParamItems to be passed to subquery
     *          要传递给子查询的PlannerParamItems的链表
     *      Note: for a subquery, tuples and subroot are not set immediately
     *      upon creation of the RelOptInfo object; they are filled in when
     *      set_subquery_pathlist processes the object.
     *      对于子查询，元组和subroot不会在创建RelOptInfo对象时立即设置;
     *          它们是在set_subquery_pathlist处理对象时填充的。
     *
     *      For otherrels that are appendrel members, these fields are filled
     *      in just as for a baserel, except we don't bother with lateral_vars.
     *      对于其他appendrel成员，这些字段就像base rels一样被填充，
     *          除了我们不关心的lateral_vars。
     *
     * If the relation is either a foreign table or a join of foreign tables that
     * all belong to the same foreign server and are assigned to the same user to
     * check access permissions as (cf checkAsUser), these fields will be set:
     * 如果关系是一个外表或一个外表的连接，这些表都属于相同的外服务器
     *   并被分配给相同的用户来检查访问权限(cf checkAsUser)，这些字段将被设置:
     *
     *      serverid - OID of foreign server, if foreign table (else InvalidOid)
     *          外部服务器的OID,如不为外部表则为InvalidOid
     *      userid - OID of user to check access as (InvalidOid means current user)
     *          检查访问权限的用户OID(InvalidOid表示当前用户)
     *      useridiscurrent - we've assumed that userid equals current user
     *          我们假设userid为当前用户
     *      fdwroutine - function hooks for FDW, if foreign table (else NULL)
     *          FDW的函数钩子，如不是外部表则为NULL
     *      fdw_private - private state for FDW, if foreign table (else NULL)
     *          FDW的私有状态,不是外部表则为NULL
     *
     * Two fields are used to cache knowledge acquired during the join search
     * about whether this rel is provably unique when being joined to given other
     * relation(s), ie, it can have at most one row matching any given row from
     * that join relation.  Currently we only attempt such proofs, and thus only
     * populate these fields, for base rels; but someday they might be used for
     * join rels too:
     * 下面的两个字段用于缓存在连接搜索过程中获取的信息，
     *     这些信息是关于当这个rel被连接到给定的其他关系时是否被证明是唯一的，
     *     也就是说，它最多只能有一行匹配来自该连接关系的任何给定行。
     * 目前我们只尝试这样的证明，因此只填充这些字段，用于base rels;
     * 但总有一天它们也可以被用来加入join rels:
     *
     *      unique_for_rels - list of Relid sets, each one being a set of other
     *                  rels for which this one has been proven unique
     *          Relid集合的链表，每一个都是一组other rels,这些rels已经被证明是唯一的
     *      non_unique_for_rels - list of Relid sets, each one being a set of
     *                  other rels for which we have tried and failed to prove
     *                  this one unique
     *          Relid集合的链表，每个集合都是other rels，这些rels试图证明唯一的，但失败了
     *
     * The presence of the following fields depends on the restrictions
     * and joins that the relation participates in:
     * 以下字段的存在取决于约束条件和关系所参与的连接:
     *
     *      baserestrictinfo - List of RestrictInfo nodes, containing info about
     *                  each non-join qualification clause in which this relation
     *                  participates (only used for base rels)
     *          RestrictInfo节点链表，其中包含关于此关系参与的每个非连接限定子句的信息(仅用于基础rels)
     *      baserestrictcost - Estimated cost of evaluating the baserestrictinfo
     *                  clauses at a single tuple (only used for base rels)
     *          在单个元组中解析baserestrictinfo子句的估算成本(仅用于基础rels)
     *      baserestrict_min_security - Smallest security_level found among
     *                  clauses in baserestrictinfo
     *          在baserestrictinfo子句中找到的最小security_level
     *      joininfo  - List of RestrictInfo nodes, containing info about each
     *                  join clause in which this relation participates (but
     *                  note this excludes clauses that might be derivable from
     *                  EquivalenceClasses)
     *          RestrictInfo节点链表，其中包含关于此关系参与的每个连接条件子句的信息
     *          (但请注意，这排除了可能从等价类派生的子句)
     *      has_eclass_joins - flag that EquivalenceClass joins are possible
     *          用于标记等价类连接是可能的
     *
     * Note: Keeping a restrictinfo list in the RelOptInfo is useful only for
     * base rels, because for a join rel the set of clauses that are treated as
     * restrict clauses varies depending on which sub-relations we choose to join.
     * (For example, in a 3-base-rel join, a clause relating rels 1 and 2 must be
     * treated as a restrictclause if we join {1} and {2 3} to make {1 2 3}; but
     * if we join {1 2} and {3} then that clause will be a restrictclause in {1 2}
     * and should not be processed again at the level of {1 2 3}.)  Therefore,
     * the restrictinfo list in the join case appears in individual JoinPaths
     * (field joinrestrictinfo), not in the parent relation.  But it's OK for
     * the RelOptInfo to store the joininfo list, because that is the same
     * for a given rel no matter how we form it.
     * 注意:在RelOptInfo中保存一个restrictinfo链表只对基础rels有用，
     *     因为对于一个join rel，被视为限制子句的子句集会根据我们选择加入的子关系而变化。(例如，在一个3-base-rel连接中，如果我们加入{1}和{2 3}以生成{1 2 3}，则与efs 1和2相关的子句必须被视为限制性子句;但是如果我们加入{1 2}和{3}，那么该子句将是{1 2}中的一个限制性子句，不应该在{1 2 3}的级别上再次处理)。因此，在join案例中，节流信息列表出现在单独的连接路径(字段join节流信息)中，而不是在父关系中。但是RelOptInfo可以存储joininfo列表，因为对于给定的rel，无论我们如何形成它都是一样的。
     *
     * We store baserestrictcost in the RelOptInfo (for base relations) because
     * we know we will need it at least once (to price the sequential scan)
     * and may need it multiple times to price index scans.
     * 我们将baserestrictcost存储在RelOptInfo(用于基本关系)中，
     *     因为我们知道至少需要它一次(为顺序扫描计算成本)，并且可能需要它多次来为索引扫描计算成本。
     *
     * If the relation is partitioned, these fields will be set:
     * 如果关系是分区表,会设置这些字段:
     *
     *      part_scheme - Partitioning scheme of the relation
     *          关系的分区schema
     *      nparts - Number of partitions
     *          关系的分区数
     *      boundinfo - Partition bounds
     *          分区边界信息
     *      partition_qual - Partition constraint if not the root
     *          如非root,则该字段存储分区约束条件
     *      part_rels - RelOptInfos for each partition
     *          每个分区的RelOptInfos
     *      partexprs, nullable_partexprs - Partition key expressions
     *          分区键表达式
     *      partitioned_child_rels - RT indexes of unpruned partitions of
     *                               this relation that are partitioned tables
     *                               themselves, in hierarchical order
     *          关系中未修剪(unpruned)分区的RT索引，
     *              这些分区本身就是分区表，按层次顺序排列
     *
     * Note: A base relation always has only one set of partition keys, but a join
     * relation may have as many sets of partition keys as the number of relations
     * being joined. partexprs and nullable_partexprs are arrays containing
     * part_scheme->partnatts elements each. Each of these elements is a list of
     * partition key expressions.  For a base relation each list in partexprs
     * contains only one expression and nullable_partexprs is not populated. For a
     * join relation, partexprs and nullable_partexprs contain partition key
     * expressions from non-nullable and nullable relations resp. Lists at any
     * given position in those arrays together contain as many elements as the
     * number of joining relations.
     * 注意:一个基本关系总是只有一组分区键，
     *   但是连接关系的分区键可能与被连接的关系的数量一样多。
     * partexprs和nullable_partexp是分别包含part_scheme->partnatts元素的数组。
     * 每个元素都是分区键表达式的链表。
     * 对于基本关系，partexprs中的每个链表只包含一个表达式，
     *   并且不填充nullable_partexprs。
     * 对于连接关系，partexprs和nullable_partexprs包含来自非空和可空关系resp的分区键表达式。
     * 这些数组中任意给定位置的链表包含的元素与连接关系的数量一样多。
     *----------     */
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
        //节点标识
        NodeTag     type;
        //RelOpt类型
        RelOptKind  reloptkind;
    
        /* all relations included in this RelOptInfo */
        //所有关系都有的属性
        //Relids(rtindex)集合
        Relids      relids;         /* set of base relids (rangetable indexes) */
    
        /* size estimates generated by planner */
        //规划器生成的大小估算
        //结果元组的估算数量
        double      rows;           /* estimated number of result tuples */
    
        /* per-relation planner control flags */
        //规划器使用的每个关系的控制标记
        //是否考虑启动成本?是,需要保留启动成本低的路径
        bool        consider_startup;   /* keep cheap-startup-cost paths? */
        //是否考虑参数化?的路径 
        bool        consider_param_startup; /* ditto for parameterized paths? */
        //是否考虑并行处理路径
        bool        consider_parallel;  /* consider parallel paths? */
    
        /* default result targetlist for Paths scanning this relation */
        //扫描该Relation时默认的结果投影列
        //Vars/Exprs,成本,行平均大小链表
        struct PathTarget *reltarget;   /* list of Vars/Exprs, cost, width */
    
        /* materialization information */
        //物化信息
        //访问路径链表
        List       *pathlist;       /* Path structures */
        //路径链表中的ParamPathInfos链表
        List       *ppilist;        /* ParamPathInfos used in pathlist */
        //并行部分路径
        List       *partial_pathlist;   /* partial Paths */
        //启动代价最低的路径
        struct Path *cheapest_startup_path;
        //整体代价最低的路径
        struct Path *cheapest_total_path;
        //获取唯一值代价最低的路径
        struct Path *cheapest_unique_path;
        //参数化代价最低的路径
        List       *cheapest_parameterized_paths;
    
        /* parameterization information needed for both base rels and join rels */
        /* (see also lateral_vars and lateral_referencers) */
        //基本通道和连接通道都需要参数化信息
        //(参考lateral_vars和lateral_referencers)
        //使用lateral语法,需依赖的Relids
        Relids      direct_lateral_relids;  /* rels directly laterally referenced */
        //rel的最小化参数信息
        Relids      lateral_relids; /* minimum parameterization of rel */
    
        /* information about a base rel (not set for join rels!) */
        //reloptkind=RELOPT_BASEREL时使用的数据结构
        //relid
        Index       relid;           /* Relation ID */
        //表空间
        Oid         reltablespace;  /*  containing tablespace */
        //类型:基表?子查询?还是函数等等?
        RTEKind     rtekind;        /* RELATION, SUBQUERY, FUNCTION, etc */
        //最小的属性编号
        AttrNumber  min_attr;       /*  smallest attrno of rel (often <0) */
        //最大的属性编号
        AttrNumber  max_attr;       /*  largest attrno of rel */
        //属性数组
        Relids     *attr_needed;    /*  array indexed [min_attr .. max_attr] */
        //属性宽度
        int32      *attr_widths;    /*  array indexed [min_attr .. max_attr] */
        //关系依赖的LATERAL Vars/PHVs
        List       *lateral_vars;   /*  LATERAL Vars and PHVs referenced by rel */
        //依赖该关系的Relids
        Relids      lateral_referencers;    /* rels that reference me laterally */
        //该关系的IndexOptInfo链表
        List       *indexlist;      /*  list of IndexOptInfo */
        //统计信息链表
        List       *statlist;       /*  list of StatisticExtInfo */
        //块数
        BlockNumber pages;          /*  size estimates derived from pg_class */
        //元组数
        double      tuples;      /*  */
        //
        double      allvisfrac;  /* ? */
        //如为子查询,存储子查询的root
        PlannerInfo *subroot;       /*  if subquery */
        //如为子查询,存储子查询的参数
        List       *subplan_params; /*  if subquery */
        //并行执行,需要多少个workers?
        int         rel_parallel_workers;   /*  wanted number of parallel workers */
    
        /* Information about foreign tables and foreign joins */
        //FWD相关信息
        //表或连接的服务器标识符
        Oid         serverid;       /* identifies server for the table or join */
        //用户id标识
        Oid         userid;         /* identifies user to check access as */
        //对于当前用户来说,连接才是有效的
        bool        useridiscurrent;    /* join is only valid for current user */
        /* use "struct FdwRoutine" to avoid including fdwapi.h here */
        //使用结构体FdwRoutine,避免包含头文件fdwapi.h
        struct FdwRoutine *fdwroutine;
        void       *fdw_private;
    
        /* cache space for remembering if we have proven this relation unique */
        //如果已证明该关系是唯一的,那么这些是用于缓存这些信息的字段
        //已知的,可保证唯一的Relids链表
        List       *unique_for_rels;    /* known unique for these other relid
                                         * set(s) */
        //已知的,不唯一的Relids链表
        List       *non_unique_for_rels;    /*  known not unique for these set(s) */
    
        /* used by various scans and joins: */
        //用于各种扫描和连接
        //如为基本关系,存储约束条件链表
        List       *baserestrictinfo;   /*  RestrictInfo structures (if base rel) */
        //解析约束表达式的成本?
        QualCost    baserestrictcost;   /*  cost of evaluating the above */
        //最低安全等级
        Index       baserestrict_min_security;  /*  min security_level found in
                                                 * baserestrictinfo */
        //连接语句的约束条件信息
        List       *joininfo;       /*  RestrictInfo structures for join clauses
                                     * involving this rel */
        //是否存在等价类连接?
        bool        has_eclass_joins;   /*  T means joininfo is incomplete */
    
        /* used by partitionwise joins: */
        //partitionwise连接使用的字段
        //是否考虑使用partitionwise join?
        bool        consider_partitionwise_join;    /* consider partitionwise
                                                     * join paths? (if
                                                     * partitioned rel) */
        //最高层的父关系Relids
        Relids      top_parent_relids;  /* Relids of topmost parents (if "other"
                                         * rel) */
    
        /* used for partitioned relations */
         //分区表使用
        //分区的schema 
        PartitionScheme part_scheme;    /* Partitioning scheme. */
        //分区数 
        int         nparts;         /* number of partitions */
        //分区边界信息 
        struct PartitionBoundInfoData *boundinfo;   /* Partition bounds */
        //分区约束
        List       *partition_qual; /*  partition constraint */
        //分区的RelOptInfo数组 
        struct RelOptInfo **part_rels;  /* Array of RelOptInfos of partitions,
                                         * stored in the same order of bounds */
        //非空分区键表达式链表
        List      **partexprs;      /* Non-nullable partition key expressions. */
        //可为空的分区键表达式 
        List      **nullable_partexprs; /* Nullable partition key expressions. */
        // 分区子表RT Indexes链表 
        List       *partitioned_child_rels; /*List of RT indexes. */
    } RelOptInfo;
    
    

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
    
    

### 二、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

