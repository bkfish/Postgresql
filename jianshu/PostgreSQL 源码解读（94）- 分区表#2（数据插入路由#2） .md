本节介绍了ExecPrepareTupleRouting->ExecFindPartition函数，该函数为heap tuple找到合适的分区。

### 一、数据结构

**ModifyTable**  
ModifyTable Node  
通过插入、更新或删除，将子计划生成的行应用到结果表。

    
    
    /* ----------------     *   ModifyTable node -     *      Apply rows produced by subplan(s) to result table(s),
     *      by inserting, updating, or deleting.
     *      通过插入、更新或删除，将子计划生成的行应用到结果表。
     *
     * If the originally named target table is a partitioned table, both
     * nominalRelation and rootRelation contain the RT index of the partition
     * root, which is not otherwise mentioned in the plan.  Otherwise rootRelation
     * is zero.  However, nominalRelation will always be set, as it's the rel that
     * EXPLAIN should claim is the INSERT/UPDATE/DELETE target.
     * 如果最初命名的目标表是分区表，则nominalRelation和rootRelation都包含分区根的RT索引，计划中没有另外提到这个索引。
     * 否则，根关系为零。但是，总是会设置名义关系，nominalRelation因为EXPLAIN应该声明的rel是INSERT/UPDATE/DELETE目标关系。
     * 
     * Note that rowMarks and epqParam are presumed to be valid for all the
     * subplan(s); they can't contain any info that varies across subplans.
     * 注意，rowMarks和epqParam被假定对所有子计划有效;
     * 它们不能包含任何在子计划中变化的信息。
     * ----------------     */
    typedef struct ModifyTable
    {
        Plan        plan;
        CmdType     operation;      /* 操作类型;INSERT, UPDATE, or DELETE */
        bool        canSetTag;      /* 是否需要设置tag?do we set the command tag/es_processed? */
        Index       nominalRelation;    /* 用于EXPLAIN的父RT索引;Parent RT index for use of EXPLAIN */
        Index       rootRelation;   /* 根Root RT索引(如目标为分区表);Root RT index, if target is partitioned */
        bool        partColsUpdated;    /* 更新了层次结构中的分区关键字;some part key in hierarchy updated */
        List       *resultRelations;    /* RT索引的整型链表;integer list of RT indexes */
        int         resultRelIndex; /* 计划链表中第一个resultRel的索引;index of first resultRel in plan's list */
        int         rootResultRelIndex; /* 分区表根索引;index of the partitioned table root */
        List       *plans;          /* 生成源数据的计划链表;plan(s) producing source data */
        List       *withCheckOptionLists;   /* 每一个目标表均具备的WCO链表;per-target-table WCO lists */
        List       *returningLists; /* 每一个目标表均具备的RETURNING链表;per-target-table RETURNING tlists */
        List       *fdwPrivLists;   /* 每一个目标表的FDW私有数据链表;per-target-table FDW private data lists */
        Bitmapset  *fdwDirectModifyPlans;   /* FDW DM计划索引位图;indices of FDW DM plans */
        List       *rowMarks;       /* rowMarks链表;PlanRowMarks (non-locking only) */
        int         epqParam;       /* EvalPlanQual再解析使用的参数ID;ID of Param for EvalPlanQual re-eval */
        OnConflictAction onConflictAction;  /* ON CONFLICT action */
        List       *arbiterIndexes; /* 冲突仲裁器索引表;List of ON CONFLICT arbiter index OIDs  */
        List       *onConflictSet;  /* SET for INSERT ON CONFLICT DO UPDATE */
        Node       *onConflictWhere;    /* WHERE for ON CONFLICT UPDATE */
        Index       exclRelRTI;     /* RTI of the EXCLUDED pseudo relation */
        List       *exclRelTlist;   /* 已排除伪关系的投影列链表;tlist of the EXCLUDED pseudo relation */
    } ModifyTable;
    

**ResultRelInfo**  
ResultRelInfo结构体  
每当更新一个现有的关系时，我们必须更新关系上的索引，也许还需要触发触发器。ResultRelInfo保存关于结果关系所需的所有信息，包括索引。

    
    
    /*
     * ResultRelInfo
     * ResultRelInfo结构体
     *
     * Whenever we update an existing relation, we have to update indexes on the
     * relation, and perhaps also fire triggers.  ResultRelInfo holds all the
     * information needed about a result relation, including indexes.
     * 每当更新一个现有的关系时，我们必须更新关系上的索引，也许还需要触发触发器。
     * ResultRelInfo保存关于结果关系所需的所有信息，包括索引。
     * 
     * Normally, a ResultRelInfo refers to a table that is in the query's
     * range table; then ri_RangeTableIndex is the RT index and ri_RelationDesc
     * is just a copy of the relevant es_relations[] entry.  But sometimes,
     * in ResultRelInfos used only for triggers, ri_RangeTableIndex is zero
     * and ri_RelationDesc is a separately-opened relcache pointer that needs
     * to be separately closed.  See ExecGetTriggerResultRel.
     * 通常，ResultRelInfo是指查询范围表中的表;
     * ri_RangeTableIndex是RT索引，而ri_RelationDesc只是相关es_relations[]条目的副本。
     * 但有时，在只用于触发器的ResultRelInfos中，ri_RangeTableIndex为零(NULL)，
     *   而ri_RelationDesc是一个需要单独关闭单独打开的relcache指针。
     *   具体可参考ExecGetTriggerResultRel结构体。
     */
    typedef struct ResultRelInfo
    {
        NodeTag     type;
    
        /* result relation's range table index, or 0 if not in range table */
        //RTE索引
        Index       ri_RangeTableIndex;
    
        /* relation descriptor for result relation */
        //结果/目标relation的描述符
        Relation    ri_RelationDesc;
    
        /* # of indices existing on result relation */
        //目标关系中索引数目
        int         ri_NumIndices;
    
        /* array of relation descriptors for indices */
        //索引的关系描述符数组(索引视为一个relation)
        RelationPtr ri_IndexRelationDescs;
    
        /* array of key/attr info for indices */
        //索引的键/属性数组
        IndexInfo **ri_IndexRelationInfo;
    
        /* triggers to be fired, if any */
        //触发的索引
        TriggerDesc *ri_TrigDesc;
    
        /* cached lookup info for trigger functions */
        //触发器函数(缓存)
        FmgrInfo   *ri_TrigFunctions;
    
        /* array of trigger WHEN expr states */
        //WHEN表达式状态的触发器数组
        ExprState **ri_TrigWhenExprs;
    
        /* optional runtime measurements for triggers */
        //可选的触发器运行期度量器
        Instrumentation *ri_TrigInstrument;
    
        /* FDW callback functions, if foreign table */
        //FDW回调函数
        struct FdwRoutine *ri_FdwRoutine;
    
        /* available to save private state of FDW */
        //可用于存储FDW的私有状态
        void       *ri_FdwState;
    
        /* true when modifying foreign table directly */
        //直接更新FDW时为T
        bool        ri_usesFdwDirectModify;
    
        /* list of WithCheckOption's to be checked */
        //WithCheckOption链表
        List       *ri_WithCheckOptions;
    
        /* list of WithCheckOption expr states */
        //WithCheckOption表达式链表
        List       *ri_WithCheckOptionExprs;
    
        /* array of constraint-checking expr states */
        //约束检查表达式状态数组
        ExprState **ri_ConstraintExprs;
    
        /* for removing junk attributes from tuples */
        //用于从元组中删除junk属性
        JunkFilter *ri_junkFilter;
    
        /* list of RETURNING expressions */
        //RETURNING表达式链表
        List       *ri_returningList;
    
        /* for computing a RETURNING list */
        //用于计算RETURNING链表
        ProjectionInfo *ri_projectReturning;
    
        /* list of arbiter indexes to use to check conflicts */
        //用于检查冲突的仲裁器索引的列表
        List       *ri_onConflictArbiterIndexes;
    
        /* ON CONFLICT evaluation state */
        //ON CONFLICT解析状态
        OnConflictSetState *ri_onConflict;
    
        /* partition check expression */
        //分区检查表达式链表
        List       *ri_PartitionCheck;
    
        /* partition check expression state */
        //分区检查表达式状态
        ExprState  *ri_PartitionCheckExpr;
    
        /* relation descriptor for root partitioned table */
        //分区root根表描述符
        Relation    ri_PartitionRoot;
    
        /* Additional information specific to partition tuple routing */
        //额外的分区元组路由信息
        struct PartitionRoutingInfo *ri_PartitionInfo;
    } ResultRelInfo;
    
    

**PartitionRoutingInfo**  
PartitionRoutingInfo结构体  
分区路由信息,用于将元组路由到表分区的结果关系信息。

    
    
    /*
     * PartitionRoutingInfo
     * PartitionRoutingInfo - 分区路由信息
     * 
     * Additional result relation information specific to routing tuples to a
     * table partition.
     * 用于将元组路由到表分区的结果关系信息。
     */
    typedef struct PartitionRoutingInfo
    {
        /*
         * Map for converting tuples in root partitioned table format into
         * partition format, or NULL if no conversion is required.
         * 映射，用于将根分区表格式的元组转换为分区格式，如果不需要转换，则转换为NULL。
         */
        TupleConversionMap *pi_RootToPartitionMap;
    
        /*
         * Map for converting tuples in partition format into the root partitioned
         * table format, or NULL if no conversion is required.
         * 映射，用于将分区格式的元组转换为根分区表格式，如果不需要转换，则转换为NULL。
         */
        TupleConversionMap *pi_PartitionToRootMap;
    
        /*
         * Slot to store tuples in partition format, or NULL when no translation
         * is required between root and partition.
         * 以分区格式存储元组的slot.在根分区和分区之间不需要转换时为NULL。
         */
        TupleTableSlot *pi_PartitionTupleSlot;
    } PartitionRoutingInfo;
    
    

**TupleConversionMap**  
TupleConversionMap结构体,用于存储元组转换映射信息.

    
    
    typedef struct TupleConversionMap
    {
        TupleDesc   indesc;         /* 源行类型的描述符;tupdesc for source rowtype */
        TupleDesc   outdesc;        /* 结果行类型的描述符;tupdesc for result rowtype */
        AttrNumber *attrMap;        /* 输入字段的索引信息,0表示NULL;indexes of input fields, or 0 for null */
        Datum      *invalues;       /* 析构源数据的工作空间;workspace for deconstructing source */
        bool       *inisnull;       //是否为NULL标记数组
        Datum      *outvalues;      /* 构造结果的工作空间;workspace for constructing result */
        bool       *outisnull;      //null标记
    } TupleConversionMap;
    
    

### 二、源码解读

ExecFindPartition函数在以父节点为根的分区树中为包含在*slot中的元组找到目标分区(叶子分区)

    
    
    /*
     * ExecFindPartition -- Find a leaf partition in the partition tree rooted
     * at parent, for the heap tuple contained in *slot
     * ExecFindPartition —— 在以父节点为根的分区树中为包含在*slot中的堆元组找到目标分区(叶子分区)
     * 
     * estate must be non-NULL; we'll need it to compute any expressions in the
     * partition key(s)
     * estate不能为NULL;需要使用它计算分区键上的表达式
     *
     * If no leaf partition is found, this routine errors out with the appropriate
     * error message, else it returns the leaf partition sequence number
     * as an index into the array of (ResultRelInfos of) all leaf partitions in
     * the partition tree.
     * 如果没有找到目标分区，则此例程将输出适当的错误消息，
     *   否则它将分区树中所有叶子分区的数组(ResultRelInfos)的目标分区序列号作为索引返回。
     */
    int
    ExecFindPartition(ResultRelInfo *resultRelInfo, PartitionDispatch *pd,
                      TupleTableSlot *slot, EState *estate)
    {
        int         result;//结果索引号
        Datum       values[PARTITION_MAX_KEYS];//值类型Datum
        bool        isnull[PARTITION_MAX_KEYS];//是否null?
        Relation    rel;//关系
        PartitionDispatch dispatch;//
        ExprContext *ecxt = GetPerTupleExprContext(estate);//表达式上下文
        TupleTableSlot *ecxt_scantuple_old = ecxt->ecxt_scantuple;//原tuple slot
        TupleTableSlot *myslot = NULL;//临时变量
        MemoryContext   oldcxt;//原内存上下文
        HeapTuple       tuple;//tuple
    
        /* use per-tuple context here to avoid leaking memory */
        //使用每个元组上下文来避免内存泄漏
        oldcxt = MemoryContextSwitchTo(GetPerTupleMemoryContext(estate));
    
        /*
         * First check the root table's partition constraint, if any.  No point in
         * routing the tuple if it doesn't belong in the root table itself.
         * 首先检查根表的分区约束(如果有的话)。如果元组不属于根表本身，则没有必要路由它。
         */
        if (resultRelInfo->ri_PartitionCheck)
            ExecPartitionCheck(resultRelInfo, slot, estate, true);
    
        /* start with the root partitioned table */
        //从root分区表开始
        tuple = ExecFetchSlotTuple(slot);//获取tuple
        dispatch = pd[0];//root
        while (true)
        {
            PartitionDesc partdesc;//分区描述符
            TupleConversionMap *map = dispatch->tupmap;//转换映射
            int         cur_index = -1;//当前索引
    
            rel = dispatch->reldesc;//relation
            partdesc = RelationGetPartitionDesc(rel);//获取rel描述符
    
            /*
             * Convert the tuple to this parent's layout, if different from the
             * current relation.
             * 如果元组与当前关系不同，则将tuple转换为parent's layout。
             */
            myslot = dispatch->tupslot;
            if (myslot != NULL && map != NULL)
            {
                tuple = do_convert_tuple(tuple, map);
                ExecStoreTuple(tuple, myslot, InvalidBuffer, true);
                slot = myslot;
            }
    
            /*
             * Extract partition key from tuple. Expression evaluation machinery
             * that FormPartitionKeyDatum() invokes expects ecxt_scantuple to
             * point to the correct tuple slot.  The slot might have changed from
             * what was used for the parent table if the table of the current
             * partitioning level has different tuple descriptor from the parent.
             * So update ecxt_scantuple accordingly.
             * 从元组中提取分区键。
             * FormPartitionKeyDatum()调用的表达式计算机制期望ecxt_scantuple指向正确的元组slot。
             * 如果当前分区级别的表与父表具有不同的元组描述符，那么slot可能已经改变了父表使用的slot。
             * 因此相应地更新ecxt_scantuple。
             */
            ecxt->ecxt_scantuple = slot;
            FormPartitionKeyDatum(dispatch, slot, estate, values, isnull);
    
            /*
             * Nothing for get_partition_for_tuple() to do if there are no
             * partitions to begin with.
             * 如无分区,则退出(无需调用get_partition_for_tuple)
             */
            if (partdesc->nparts == 0)
            {
                result = -1;
                break;
            }
            //调用get_partition_for_tuple
            cur_index = get_partition_for_tuple(rel, values, isnull);
    
            /*
             * cur_index < 0 means we failed to find a partition of this parent.
             * cur_index >= 0 means we either found the leaf partition, or the
             * next parent to find a partition of.
             * cur_index < 0表示未能找到该父节点的分区。
             * cur_index >= 0表示要么找到叶子分区，要么找到下一个父分区。
             */
            if (cur_index < 0)
            {
                result = -1;
                break;//找不到,退出
            }
            else if (dispatch->indexes[cur_index] >= 0)
            {
                result = dispatch->indexes[cur_index];
                /* success! */
                break;//找到了,退出循环
            }
            else
            {
                /* move down one level */
                //移到下一层查找
                dispatch = pd[-dispatch->indexes[cur_index]];
    
                /*
                 * Release the dedicated slot, if it was used.  Create a copy of
                 * the tuple first, for the next iteration.
                 */
                if (slot == myslot)
                {
                    tuple = ExecCopySlotTuple(myslot);
                    ExecClearTuple(myslot);
                }
            }
        }
    
        /* Release the tuple in the lowest parent's dedicated slot. */
         //释放位于最低父级的专用的slot相对应的元组。
        if (slot == myslot)
            ExecClearTuple(myslot);
    
        /* A partition was not found. */
        //找不到partition
        if (result < 0)
        {
            char       *val_desc;
    
            val_desc = ExecBuildSlotPartitionKeyDescription(rel,
                                                            values, isnull, 64);
            Assert(OidIsValid(RelationGetRelid(rel)));
            ereport(ERROR,
                    (errcode(ERRCODE_CHECK_VIOLATION),
                     errmsg("no partition of relation \"%s\" found for row",
                            RelationGetRelationName(rel)),
                     val_desc ? errdetail("Partition key of the failing row contains %s.", val_desc) : 0));
        }
    
        MemoryContextSwitchTo(oldcxt);
        ecxt->ecxt_scantuple = ecxt_scantuple_old;
    
        return result;
    }
    
    
    
    /*
     * get_partition_for_tuple
     *      Finds partition of relation which accepts the partition key specified
     *      in values and isnull
     * get_partition_for_tuple
     *      查找参数为values和isnull中指定分区键的关系分区
     *
     * Return value is index of the partition (>= 0 and < partdesc->nparts) if one
     * found or -1 if none found.
     * 返回值是分区的索引(>= 0和< partdesc->nparts)，
     *   如果找到一个分区，则返回值;如果没有找到，则返回值为-1。
     */
    static int
    get_partition_for_tuple(Relation relation, Datum *values, bool *isnull)
    {
        int         bound_offset;
        int         part_index = -1;
        PartitionKey key = RelationGetPartitionKey(relation);
        PartitionDesc partdesc = RelationGetPartitionDesc(relation);
        PartitionBoundInfo boundinfo = partdesc->boundinfo;
    
        /* Route as appropriate based on partitioning strategy. */
        //基于分区的策略进行路由
        switch (key->strategy)
        {
            case PARTITION_STRATEGY_HASH://HASH分区
                {
                    int         greatest_modulus;
                    uint64      rowHash;
    
                    greatest_modulus = get_hash_partition_greatest_modulus(boundinfo);
                    rowHash = compute_partition_hash_value(key->partnatts,
                                                           key->partsupfunc,
                                                           values, isnull);
    
                    part_index = boundinfo->indexes[rowHash % greatest_modulus];
                }
                break;
    
            case PARTITION_STRATEGY_LIST://列表分区
                if (isnull[0])
                {
                    if (partition_bound_accepts_nulls(boundinfo))
                        part_index = boundinfo->null_index;
                }
                else
                {
                    bool        equal = false;
    
                    bound_offset = partition_list_bsearch(key->partsupfunc,
                                                          key->partcollation,
                                                          boundinfo,
                                                          values[0], &equal);
                    if (bound_offset >= 0 && equal)
                        part_index = boundinfo->indexes[bound_offset];
                }
                break;
    
            case PARTITION_STRATEGY_RANGE://范围分区
                {
                    bool        equal = false,
                                range_partkey_has_null = false;
                    int         i;
    
                    /*
                     * No range includes NULL, so this will be accepted by the
                     * default partition if there is one, and otherwise rejected.
                     * 任何范围都不包含NULL值，因此默认分区将接受该值(如果存在)，否则将拒绝该值。
                     */
                    for (i = 0; i < key->partnatts; i++)
                    {
                        if (isnull[i])
                        {
                            range_partkey_has_null = true;
                            break;
                        }
                    }
    
                    if (!range_partkey_has_null)
                    {
                        bound_offset = partition_range_datum_bsearch(key->partsupfunc,
                                                                     key->partcollation,
                                                                     boundinfo,
                                                                     key->partnatts,
                                                                     values,
                                                                     &equal);
    
                        /*
                         * The bound at bound_offset is less than or equal to the
                         * tuple value, so the bound at offset+1 is the upper
                         * bound of the partition we're looking for, if there
                         * actually exists one.
                         * bound_offset的边界小于或等于元组值，所以offset+1的边界是我们要找的分区的上界，如存在的话。
                         */
                        part_index = boundinfo->indexes[bound_offset + 1];
                    }
                }
                break;
    
            default:
                elog(ERROR, "unexpected partition strategy: %d",
                     (int) key->strategy);//暂不支持其他分区
        }
    
        /*
         * part_index < 0 means we failed to find a partition of this parent. Use
         * the default partition, if there is one.
         * part_index < 0表示没有找到这个父节点的分区。如存在分区，则使用默认分区。
         */
        if (part_index < 0)
            part_index = boundinfo->default_index;
    
        return part_index;
    }
    

_依赖的函数_

    
    
    /*
     * get_hash_partition_greatest_modulus
     *
     * Returns the greatest modulus of the hash partition bound. The greatest
     * modulus will be at the end of the datums array because hash partitions are
     * arranged in the ascending order of their moduli and remainders.
     * 返回哈希分区边界的最大模。
     * 最大模量将位于datums数组的末尾，因为哈希分区按照它们的模块和余数的升序排列。
     */
    int
    get_hash_partition_greatest_modulus(PartitionBoundInfo bound)
    {
        Assert(bound && bound->strategy == PARTITION_STRATEGY_HASH);
        Assert(bound->datums && bound->ndatums > 0);
        Assert(DatumGetInt32(bound->datums[bound->ndatums - 1][0]) > 0);
    
        return DatumGetInt32(bound->datums[bound->ndatums - 1][0]);
    }
    
    /*
     * compute_partition_hash_value
     * 
     * Compute the hash value for given partition key values.
     * 给定分区键值,计算相应的Hash值
     */
    uint64
    compute_partition_hash_value(int partnatts, FmgrInfo *partsupfunc,
                                 Datum *values, bool *isnull)
    {
        int         i;
        uint64      rowHash = 0;//返回结果
        Datum       seed = UInt64GetDatum(HASH_PARTITION_SEED);
    
        for (i = 0; i < partnatts; i++)
        {
            /* Nulls are just ignored */
            if (!isnull[i])
            {
                //不为NULL
                Datum       hash;
    
                Assert(OidIsValid(partsupfunc[i].fn_oid));
    
                /*
                 * Compute hash for each datum value by calling respective
                 * datatype-specific hash functions of each partition key
                 * attribute.
                 * 通过调用每个分区键属性的特定于数据类型的哈希函数，计算每个数据值的哈希值。
                 */
                hash = FunctionCall2(&partsupfunc[i], values[i], seed);
    
                /* Form a single 64-bit hash value */
                //组合成一个单独的64bit哈希值
                rowHash = hash_combine64(rowHash, DatumGetUInt64(hash));
            }
        }
    
        return rowHash;
    }
    
    
    /*
     * Combine two 64-bit hash values, resulting in another hash value, using the
     * same kind of technique as hash_combine().  Testing shows that this also
     * produces good bit mixing.
     * 使用与hash_combine()相同的技术组合两个64位哈希值，生成另一个哈希值。
     * 测试表明，该方法也能产生良好的混合效果。
     */
    static inline uint64
    hash_combine64(uint64 a, uint64 b)
    {
        /* 0x49a0f4dd15e5a8e3 is 64bit random data */
        a ^= b + UINT64CONST(0x49a0f4dd15e5a8e3) + (a << 54) + (a >> 7);
        return a;
    }
    
    //两个参数的函数调用宏定义
    #define FunctionCall2(flinfo, arg1, arg2) \
       FunctionCall2Coll(flinfo, InvalidOid, arg1, arg2)
    
    

### 三、跟踪分析

测试脚本如下

    
    
    -- Hash Partition
    drop table if exists t_hash_partition;
    create table t_hash_partition (c1 int not null,c2  varchar(40),c3 varchar(40)) partition by hash(c1);
    create table t_hash_partition_1 partition of t_hash_partition for values with (modulus 6,remainder 0);
    create table t_hash_partition_2 partition of t_hash_partition for values with (modulus 6,remainder 1);
    create table t_hash_partition_3 partition of t_hash_partition for values with (modulus 6,remainder 2);
    create table t_hash_partition_4 partition of t_hash_partition for values with (modulus 6,remainder 3);
    create table t_hash_partition_5 partition of t_hash_partition for values with (modulus 6,remainder 4);
    create table t_hash_partition_6 partition of t_hash_partition for values with (modulus 6,remainder 5);
    
    insert into t_hash_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    

启动gdb,设置断点,进入ExecFindPartition

    
    
    (gdb) b ExecFindPartition
    Breakpoint 1 at 0x6e19e7: file execPartition.c, line 227.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecFindPartition (resultRelInfo=0x14299a8, pd=0x142ae58, slot=0x142a140, estate=0x1429758)
        at execPartition.c:227
    227     ExprContext *ecxt = GetPerTupleExprContext(estate);
    

初始化变量,切换内存上下文

    
    
    227     ExprContext *ecxt = GetPerTupleExprContext(estate);
    (gdb) n
    228     TupleTableSlot *ecxt_scantuple_old = ecxt->ecxt_scantuple;
    (gdb) 
    229     TupleTableSlot *myslot = NULL;
    (gdb) 
    234     oldcxt = MemoryContextSwitchTo(GetPerTupleMemoryContext(estate));
    (gdb) p ecxt_scantuple_old
    $1 = (TupleTableSlot *) 0x0
    

提取tuple,获取dispatch

    
    
    (gdb) n
    244     tuple = ExecFetchSlotTuple(slot);
    (gdb) 
    245     dispatch = pd[0];
    (gdb) n
    249         TupleConversionMap *map = dispatch->tupmap;
    (gdb) p *tuple
    $2 = {t_len = 40, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 0, t_data = 0x142b158}
    (gdb) 
    

查看分发器dispatch信息

    
    
    (gdb) p *dispatch
    $3 = {reldesc = 0x7fbfa6900950, key = 0x1489860, keystate = 0x0, partdesc = 0x149b130, tupslot = 0x0, tupmap = 0x0, 
      indexes = 0x142ade8}
    (gdb) p *dispatch->reldesc
    $4 = {rd_node = {spcNode = 1663, dbNode = 16402, relNode = 16986}, rd_smgr = 0x0, rd_refcnt = 1, rd_backend = -1, 
      rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 0 '\000', rd_statvalid = false, 
      rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7fbfa6900b68, rd_att = 0x7fbfa6900c80, rd_id = 16986, 
      rd_lockInfo = {lockRelId = {relId = 16986, dbId = 16402}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, 
      rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x1489710, rd_partkey = 0x1489860, 
      rd_pdcxt = 0x149afe0, rd_partdesc = 0x149b130, rd_partcheck = 0x0, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, 
      rd_replidindex = 0, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, 
      rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, 
      rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, 
      rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x0}
    ----------------------------------------------------------------------------  
    testdb=# select relname from pg_class where oid=16986;
         relname      
    ------------------     t_hash_partition -->hash分区表
    (1 row)
    ----------------------------------------------------------------------------  
    (gdb) p *dispatch->key
    $5 = {strategy = 104 'h', partnatts = 1, partattrs = 0x14898f8, partexprs = 0x0, partopfamily = 0x1489918, 
      partopcintype = 0x1489938, partsupfunc = 0x1489958, partcollation = 0x14899b0, parttypid = 0x14899d0, 
      parttypmod = 0x14899f0, parttyplen = 0x1489a10, parttypbyval = 0x1489a30, 
      parttypalign = 0x1489a50 "i~\177\177\177\177\177\177\b", parttypcoll = 0x1489a70}
    (gdb) p *dispatch->partdesc
    $6 = {nparts = 6, oids = 0x149b168, boundinfo = 0x149b1a0}
    (gdb) p *dispatch->partdesc->boundinfo
    $8 = {strategy = 104 'h', ndatums = 6, datums = 0x149b1f8, kind = 0x0, indexes = 0x149b288, null_index = -1, 
      default_index = -1}
    (gdb) p *dispatch->partdesc->boundinfo->datums
    $9 = (Datum *) 0x149b2c0
    (gdb) p **dispatch->partdesc->boundinfo->datums
    $10 = 6
    (gdb) p *dispatch->indexes
    $15 = 0
    

分区描述符中的oids(分别对应t_hash_partition_1->6)

    
    
    (gdb) p dispatch->partdesc->oids[0]
    $11 = 16989
    (gdb) p dispatch->partdesc->oids[1]
    $12 = 16992
    ...
    (gdb) p dispatch->partdesc->oids[5]
    $13 = 17004
    

索引信息

    
    
    (gdb) p dispatch->indexes[0]
    $16 = 0
    ...
    (gdb) p dispatch->indexes[5]
    $18 = 5
    

设置当前索引(-1),获取relation信息,获取分区描述符

    
    
    (gdb) n
    250         int         cur_index = -1;
    (gdb) 
    252         rel = dispatch->reldesc;
    (gdb) 
    253         partdesc = RelationGetPartitionDesc(rel);
    (gdb) 
    259         myslot = dispatch->tupslot;
    (gdb) p *partdesc
    $19 = {nparts = 6, oids = 0x149b168, boundinfo = 0x149b1a0}
    (gdb) 
    

myslot为NULL

    
    
    (gdb) n
    260         if (myslot != NULL && map != NULL)
    (gdb) p myslot
    $20 = (TupleTableSlot *) 0x0
    

从元组中提取分区键

    
    
    (gdb) n
    275         ecxt->ecxt_scantuple = slot;
    (gdb) 
    276         FormPartitionKeyDatum(dispatch, slot, estate, values, isnull);
    (gdb) 
    282         if (partdesc->nparts == 0)
    (gdb) p *partdesc
    $21 = {nparts = 6, oids = 0x149b168, boundinfo = 0x149b1a0}
    (gdb) p *slot
    $22 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = true, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x142b140, tts_tupleDescriptor = 0x1429f28, tts_mcxt = 0x1429640, tts_buffer = 0, tts_nvalid = 1, 
      tts_values = 0x142a1a0, tts_isnull = 0x142a1b8, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 4, tts_fixedTupleDescriptor = true}
    (gdb) p values
    $23 = {0, 7152626, 21144656, 21144128, 7141053, 21143088, 21144128, 16372128, 140722434628688, 0, 0, 0, 21143872, 
      140722434628736, 140461078524324, 21141056, 21144128, 0, 21143088, 21141056, 7152279, 0, 7421941, 21141056, 21143088, 
      21614576, 140722434628800, 7422189, 21143872, 140722434628839, 21143088, 21144128}
    (gdb) p isnull
    $24 = {false, 91, 186, 126, 252, 127, false, false, 208, 166, 71, false, false, false, false, false, 2, 
      false <repeats 15 times>}
    (gdb) p *estate
    $25 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1451ee0, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x14a71c0, es_plannedstmt = 0x14a72b8, 
      es_sourceText = 0x13acec8 "insert into t_hash_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');", es_junkFilter = 0x0, 
      es_output_cid = 0, es_result_relations = 0x14299a8, es_num_result_relations = 1, es_result_relation_info = 0x14299a8, 
      es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, 
      es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x142afc0, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, 
      es_param_list_info = 0x0, es_param_exec_vals = 0x1429970, es_queryEnv = 0x0, es_query_cxt = 0x1429640, 
      es_tupleTable = 0x142a200, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, 
      es_finished = false, es_exprcontexts = 0x1429ef0, es_subplanstates = 0x0, es_auxmodifytables = 0x0, 
      es_per_tuple_exprcontext = 0x142b080, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0, es_jit_worker_instr = 0x0}
    (gdb) 
    

进入get_partition_for_tuple函数

    
    
    (gdb) n
    288         cur_index = get_partition_for_tuple(rel, values, isnull);
    (gdb) step
    get_partition_for_tuple (relation=0x7fbfa6900950, values=0x7ffc7eba5bb0, isnull=0x7ffc7eba5b90) at execPartition.c:1139
    1139        int         part_index = -1;
    (gdb) 
    

get_partition_for_tuple->获取分区键

    
    
    1139        int         part_index = -1;
    (gdb) n
    1140        PartitionKey key = RelationGetPartitionKey(relation);
    (gdb) 
    1141        PartitionDesc partdesc = RelationGetPartitionDesc(relation);
    (gdb) p key
    $26 = (PartitionKey) 0x1489860
    (gdb) p *key
    $27 = {strategy = 104 'h', partnatts = 1, partattrs = 0x14898f8, partexprs = 0x0, partopfamily = 0x1489918, 
      partopcintype = 0x1489938, partsupfunc = 0x1489958, partcollation = 0x14899b0, parttypid = 0x14899d0, 
      parttypmod = 0x14899f0, parttyplen = 0x1489a10, parttypbyval = 0x1489a30, 
      parttypalign = 0x1489a50 "i~\177\177\177\177\177\177\b", parttypcoll = 0x1489a70}
    

get_partition_for_tuple->获取分区描述符&分区边界信息

    
    
    (gdb) n
    1142        PartitionBoundInfo boundinfo = partdesc->boundinfo;
    (gdb) 
    1145        switch (key->strategy)
    (gdb) p *partdesc
    $28 = {nparts = 6, oids = 0x149b168, boundinfo = 0x149b1a0}
    (gdb) p *boundinfo
    $29 = {strategy = 104 'h', ndatums = 6, datums = 0x149b1f8, kind = 0x0, indexes = 0x149b288, null_index = -1, 
      default_index = -1}
    

get_partition_for_tuple->进入Hash分区处理分支

    
    
    (gdb) n
    1152                    greatest_modulus = get_hash_partition_greatest_modulus(boundinfo);
    (gdb) p key->strategy
    $30 = 104 'h'
    

get_partition_for_tuple->计算模块数&行hash值,获得分区编号(index)

    
    
    (gdb) n
    1153                    rowHash = compute_partition_hash_value(key->partnatts,
    (gdb) n
    1157                    part_index = boundinfo->indexes[rowHash % greatest_modulus];
    (gdb) 
    1159                break;
    (gdb) p part_index
    $31 = 2
    (gdb) 
    

get_partition_for_tuple->返回

    
    
    (gdb) n
    1228        if (part_index < 0)
    (gdb) 
    1231        return part_index;
    (gdb) 
    1232    }
    (gdb) 
    ExecFindPartition (resultRelInfo=0x14299a8, pd=0x142ae58, slot=0x142a140, estate=0x1429758) at execPartition.c:295
    295         if (cur_index < 0)
    (gdb) 
    

已取得分区信息(分区索引编号=2)

    
    
    (gdb) n
    300         else if (dispatch->indexes[cur_index] >= 0)
    (gdb) 
    302             result = dispatch->indexes[cur_index];
    (gdb) p dispatch->indexes[cur_index]
    $32 = 2
    (gdb) n
    304             break;
    (gdb) 
    324     if (slot == myslot)
    (gdb) 
    328     if (result < 0)
    (gdb) 
    342     MemoryContextSwitchTo(oldcxt);
    (gdb) 
    343     ecxt->ecxt_scantuple = ecxt_scantuple_old;
    (gdb) 
    345     return result;
    (gdb) 
    

完成函数调用

    
    
    (gdb) n
    346 }
    (gdb) 
    ExecPrepareTupleRouting (mtstate=0x1429ac0, estate=0x1429758, proute=0x142a7a8, targetRelInfo=0x14299a8, slot=0x142a140)
        at nodeModifyTable.c:1716
    1716        Assert(partidx >= 0 && partidx < proute->num_partitions);
    

DONE!

### 四、参考资料

PG 11.1 Source Code.  
注: [doxygen](https://doxygen.postgresql.org)上的源代码与PG 11.1源代码并不一致,本节基于11.1进行分析.

