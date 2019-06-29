在PG中,分区表通过"继承"的方式实现,这里就会存在一个问题,就是在插入数据时,PG如何确定数据应该插入到哪个目标分区?在PG中,通过函数ExecPrepareTupleRouting为路由待插入的元组做准备,主要的目的是确定元组所在的分区。

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

ExecPrepareTupleRouting函数确定要插入slot中的tuple所属的分区，同时修改mtstate和estate等相关信息,为后续实际的插入作准备。

    
    
    /*
     * ExecPrepareTupleRouting --- prepare for routing one tuple
     * ExecPrepareTupleRouting --- 为路由一个元组做准备
     * 
     * Determine the partition in which the tuple in slot is to be inserted,
     * and modify mtstate and estate to prepare for it.
     * 确定要插入slot中tuple的分区，并修改mtstate和estate以为插入作准备。
     *
     * Caller must revert the estate changes after executing the insertion!
     * In mtstate, transition capture changes may also need to be reverted.
     * 调用方必须在执行插入之后恢复estate中被修改的属性值!
     * 在mtstate中，转换捕获更改也可能需要恢复。
     *
     * Returns a slot holding the tuple of the partition rowtype.
     * 返回包含分区rowtype元组的槽位。
     */
    static TupleTableSlot *
    ExecPrepareTupleRouting(ModifyTableState *mtstate,
                            EState *estate,
                            PartitionTupleRouting *proute,
                            ResultRelInfo *targetRelInfo,
                            TupleTableSlot *slot)
    {
        ModifyTable *node;//ModifyTable节点
        int         partidx;//分区索引
        ResultRelInfo *partrel;//ResultRelInfo结构体指针(数组)
        HeapTuple   tuple;//元组
    
        /*
         * Determine the target partition.  If ExecFindPartition does not find a
         * partition after all, it doesn't return here; otherwise, the returned
         * value is to be used as an index into the arrays for the ResultRelInfo
         * and TupleConversionMap for the partition.
         * 确定目标分区。
         * 如果ExecFindPartition最终没有找到分区，它不会在这里返回;
         * 否则，返回值将用作分区的ResultRelInfo和TupleConversionMap数组的索引。
         */
        partidx = ExecFindPartition(targetRelInfo,
                                    proute->partition_dispatch_info,
                                    slot,
                                    estate);
        Assert(partidx >= 0 && partidx < proute->num_partitions);
    
        /*
         * Get the ResultRelInfo corresponding to the selected partition; if not
         * yet there, initialize it.
         * 获取与所选分区对应的ResultRelInfo;如果还没有，则初始化。
         */
        partrel = proute->partitions[partidx];
        if (partrel == NULL)
            partrel = ExecInitPartitionInfo(mtstate, targetRelInfo,
                                            proute, estate,
                                            partidx);
    
        /*
         * Check whether the partition is routable if we didn't yet
         * 检查分区是否可路由
         * 
         * Note: an UPDATE of a partition key invokes an INSERT that moves the
         * tuple to a new partition.  This check would be applied to a subplan
         * partition of such an UPDATE that is chosen as the partition to route
         * the tuple to.  The reason we do this check here rather than in
         * ExecSetupPartitionTupleRouting is to avoid aborting such an UPDATE
         * unnecessarily due to non-routable subplan partitions that may not be
         * chosen for update tuple movement after all.
         * 注意:分区键的更新调用将元组移动到新分区的插入。
         * 此检查将应用于此类更新的子计划分区，该分区被选择为将元组路由到的分区。
         * 在这里而不是在ExecSetupPartitionTupleRouting中执行此检查的原因是
             为了避免由于无法路由的子计划分区而不必要地中止这样的更新，这些分区可能最终不会被选择用于更新元组移动。
         */
        if (!partrel->ri_PartitionReadyForRouting)
        {
            /* Verify the partition is a valid target for INSERT. */
            //验证分区是否可用于INSERT
            CheckValidResultRel(partrel, CMD_INSERT);
    
            /* Set up information needed for routing tuples to the partition. */
            //设置将元组路由到分区所需的信息。
            ExecInitRoutingInfo(mtstate, estate, proute, partrel, partidx);
        }
    
        /*
         * Make it look like we are inserting into the partition.
         * 让它看起来像是插入到分区中。
         */
        estate->es_result_relation_info = partrel;
    
        /* Get the heap tuple out of the given slot. */
        //从给定的slot中获取heap tuple
        tuple = ExecMaterializeSlot(slot);
    
        /*
         * If we're capturing transition tuples, we might need to convert from the
         * partition rowtype to parent rowtype.
         * 如果正在捕获转换元组，可能需要将分区行类型转换为根分区表的行类型。
         */
        if (mtstate->mt_transition_capture != NULL)
        {
            if (partrel->ri_TrigDesc &&
                partrel->ri_TrigDesc->trig_insert_before_row)
            {
                /*
                 * If there are any BEFORE triggers on the partition, we'll have
                 * to be ready to convert their result back to tuplestore format.
                 * 如果分区上有BEFORE触发器，必须准备将它们的结果转换回tuplestore格式。
                 */
                mtstate->mt_transition_capture->tcs_original_insert_tuple = NULL;
                mtstate->mt_transition_capture->tcs_map =
                    TupConvMapForLeaf(proute, targetRelInfo, partidx);
            }
            else
            {
                /*
                 * Otherwise, just remember the original unconverted tuple, to
                 * avoid a needless round trip conversion.
                 * 否则，只需记住原始的未转换元组，以避免不必要的来回转换。
                 */
                mtstate->mt_transition_capture->tcs_original_insert_tuple = tuple;
                mtstate->mt_transition_capture->tcs_map = NULL;
            }
        }
        if (mtstate->mt_oc_transition_capture != NULL)
        {
            mtstate->mt_oc_transition_capture->tcs_map =
                TupConvMapForLeaf(proute, targetRelInfo, partidx);
        }
    
        /*
         * Convert the tuple, if necessary.
         * 如需要,转换元组
         */
        ConvertPartitionTupleSlot(proute->parent_child_tupconv_maps[partidx],
                                  tuple,
                                  proute->partition_tuple_slot,
                                  &slot);
    
        /* Initialize information needed to handle ON CONFLICT DO UPDATE. */
        //如为ON CONFLICT DO UPDATE模式,则初始化相关信息
        Assert(mtstate != NULL);
        node = (ModifyTable *) mtstate->ps.plan;
        if (node->onConflictAction == ONCONFLICT_UPDATE)
        {
            Assert(mtstate->mt_existing != NULL);
            ExecSetSlotDescriptor(mtstate->mt_existing,
                                  RelationGetDescr(partrel->ri_RelationDesc));
            Assert(mtstate->mt_conflproj != NULL);
            ExecSetSlotDescriptor(mtstate->mt_conflproj,
                                  partrel->ri_onConflict->oc_ProjTupdesc);
        }
    
        return slot;
    }
    
    /*
     * ExecFetchSlotHeapTuple - fetch HeapTuple representing the slot's content
     * ExecFetchSlotHeapTuple - 根据slot提取HeapTuple
     *
     * The returned HeapTuple represents the slot's content as closely as
     * possible.
     * 返回的HeapTuple尽可能就是slot的内容。
     * 
     * If materialize is true, the contents of the slots will be made independent
     * from the underlying storage (i.e. all buffer pins are release, memory is
     * allocated in the slot's context).
     * 如果materialize为T，slot的内容将独立于底层存储(即释放所有缓冲区pin，在slot的上下文中分配内存)。
     *
     * If shouldFree is not-NULL it'll be set to true if the returned tuple has
     * been allocated in the calling memory context, and must be freed by the
     * caller (via explicit pfree() or a memory context reset).
     * 如果shouldFree not-NULL，那么如果返回的元组已经在调用内存上下文中分配，
     *   并且必须由调用方释放(通过显式pfree()或内存上下文重置)。
     *
     * NB: If materialize is true, modifications of the returned tuple are
     * allowed. But it depends on the type of the slot whether such modifications
     * will also affect the slot's contents. While that is not the nicest
     * behaviour, all such modifcations are in the process of being removed.
     * 注意:如果materialize为T，则允许修改返回的元组。
     * 但这取决于slot的类型，这种修改是否也会影响slot的内容。
     * 虽然这不是最好的行为，但所有这些修改都在被移除的过程中。
     */
    HeapTuple
    ExecFetchSlotHeapTuple(TupleTableSlot *slot, bool materialize, bool *shouldFree)
    {
        /*
         * sanity checks
         * 安全检查
         */
        Assert(slot != NULL);
        Assert(!TTS_EMPTY(slot));
    
        /* Materialize the tuple so that the slot "owns" it, if requested. */
        //物化元组，以便slot“拥有”它(如要求)。
        if (materialize)
            slot->tts_ops->materialize(slot);
    
        if (slot->tts_ops->get_heap_tuple == NULL)
        {
            if (shouldFree)
                *shouldFree = true;
            return slot->tts_ops->copy_heap_tuple(slot);//返回slot拷贝
        }
        else
        {
            if (shouldFree)
                *shouldFree = false;
            return slot->tts_ops->get_heap_tuple(slot);//直接返回slot
        }
    }
    
    
    

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
    
    -- delete from t_hash_partition where c1 = 0;
    insert into t_hash_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    

启动gdb,设置断点,进入ExecPrepareTupleRouting

    
    
    (gdb) b ExecPrepareTupleRouting
    Breakpoint 1 at 0x710b1e: file nodeModifyTable.c, line 1712.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecPrepareTupleRouting (mtstate=0x1e4de60, estate=0x1e4daf8, proute=0x1e4eb48, targetRelInfo=0x1e4dd48, 
        slot=0x1e4e4e0) at nodeModifyTable.c:1712
    1712        partidx = ExecFindPartition(targetRelInfo,
    

查看函数调用栈  
ExecPrepareTupleRouting在ExecModifyTable Node中被调用,为后续的插入作准备.

    
    
    (gdb) bt
    #0  ExecPrepareTupleRouting (mtstate=0x1e4de60, estate=0x1e4daf8, proute=0x1e4eb48, targetRelInfo=0x1e4dd48, slot=0x1e4e4e0)
        at nodeModifyTable.c:1712
    #1  0x0000000000711602 in ExecModifyTable (pstate=0x1e4de60) at nodeModifyTable.c:2157
    #2  0x00000000006e4c30 in ExecProcNodeFirst (node=0x1e4de60) at execProcnode.c:445
    #3  0x00000000006d9974 in ExecProcNode (node=0x1e4de60) at ../../../src/include/executor/executor.h:237
    #4  0x00000000006dc22d in ExecutePlan (estate=0x1e4daf8, planstate=0x1e4de60, use_parallel_mode=false, 
        operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, dest=0x1e67e90, 
        execute_once=true) at execMain.c:1723
    #5  0x00000000006d9f5c in standard_ExecutorRun (queryDesc=0x1e39d68, direction=ForwardScanDirection, count=0, 
        execute_once=true) at execMain.c:364
    #6  0x00000000006d9d7f in ExecutorRun (queryDesc=0x1e39d68, direction=ForwardScanDirection, count=0, execute_once=true)
        at execMain.c:307
    #7  0x00000000008cbdb3 in ProcessQuery (plan=0x1e67d18, 
        sourceText=0x1d60ec8 "insert into t_hash_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');", params=0x0, queryEnv=0x0, 
        dest=0x1e67e90, completionTag=0x7ffdcf148b20 "") at pquery.c:161
    #8  0x00000000008cd6f9 in PortalRunMulti (portal=0x1dc6538, isTopLevel=true, setHoldSnapshot=false, dest=0x1e67e90, 
        altdest=0x1e67e90, completionTag=0x7ffdcf148b20 "") at pquery.c:1286
    #9  0x00000000008cccb9 in PortalRun (portal=0x1dc6538, count=9223372036854775807, isTopLevel=true, run_once=true, 
        dest=0x1e67e90, altdest=0x1e67e90, completionTag=0x7ffdcf148b20 "") at pquery.c:799
    #10 0x00000000008c6b1e in exec_simple_query (
        query_string=0x1d60ec8 "insert into t_hash_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');") at postgres.c:1145
    #11 0x00000000008cae70 in PostgresMain (argc=1, argv=0x1d8aba8, dbname=0x1d8aa10 "testdb", username=0x1d5dba8 "xdb")
        at postgres.c:4182
    

找到该元组所在的分区

    
    
    (gdb) n
    1716        Assert(partidx >= 0 && partidx < proute->num_partitions);
    (gdb) p partidx
    $1 = 2
    

获取与所选分区对应的ResultRelInfo;如果还没有，则初始化

    
    
    (gdb) n
    1722        partrel = proute->partitions[partidx];
    (gdb) 
    1723        if (partrel == NULL)
    (gdb) p *partrel
    Cannot access memory at address 0x0
    (gdb) n
    1724            partrel = ExecInitPartitionInfo(mtstate, targetRelInfo,    
    

初始化后的partrel

    
    
    (gdb) p *partrel
    $2 = {type = T_ResultRelInfo, ri_RangeTableIndex = 1, ri_RelationDesc = 0x1e7c940, ri_NumIndices = 0, 
      ri_IndexRelationDescs = 0x0, ri_IndexRelationInfo = 0x0, ri_TrigDesc = 0x0, ri_TrigFunctions = 0x0, 
      ri_TrigWhenExprs = 0x0, ri_TrigInstrument = 0x0, ri_FdwRoutine = 0x0, ri_FdwState = 0x0, ri_usesFdwDirectModify = false, 
      ri_WithCheckOptions = 0x0, ri_WithCheckOptionExprs = 0x0, ri_ConstraintExprs = 0x0, ri_junkFilter = 0x0, 
      ri_returningList = 0x0, ri_projectReturning = 0x0, ri_onConflictArbiterIndexes = 0x0, ri_onConflict = 0x0, 
      ri_PartitionCheck = 0x1e4f538, ri_PartitionCheckExpr = 0x0, ri_PartitionRoot = 0x1e7c2f8, 
      ri_PartitionReadyForRouting = true}
    

目标分区描述符-->t_hash_partition_3

    
    
    (gdb) p *partrel->ri_RelationDesc
    $3 = {rd_node = {spcNode = 1663, dbNode = 16402, relNode = 16995}, rd_smgr = 0x1e34510, rd_refcnt = 1, rd_backend = -1, 
      rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 0 '\000', rd_statvalid = false, 
      rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x1e7c1e0, rd_att = 0x1e7cb58, rd_id = 16995, rd_lockInfo = {
        lockRelId = {relId = 16995, dbId = 16402}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, 
      rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, rd_partdesc = 0x0, 
      rd_partcheck = 0x1e7aa30, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, rd_replidindex = 0, rd_statlist = 0x0, 
      rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, 
      rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, 
      rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, 
      rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, rd_exclstrats = 0x0, rd_amcache = 0x0, 
      rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x1de40b0}
    ------------------    testdb=# select oid,relname from pg_class where oid=16995;
      oid  |      relname       
    -------+--------------------     16995 | t_hash_partition_3
    (1 row)  
    ----------------- 
    

该分区是可路由的

    
    
    (gdb) p partrel->ri_PartitionReadyForRouting
    $4 = true
    

设置estate变量(让它看起来像是插入到分区中)/物化tuple

    
    
    (gdb) n
    1751        estate->es_result_relation_info = partrel;
    (gdb) 
    1754        tuple = ExecMaterializeSlot(slot);
    (gdb) 
    1760        if (mtstate->mt_transition_capture != NULL)
    (gdb) p tuple
    $5 = (HeapTuple) 0x1e4f4e0
    (gdb) p *tuple
    $6 = {t_len = 40, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 0, t_data = 0x1e4f4f8}
    (gdb) 
    (gdb) p *tuple->t_data
    $7 = {t_choice = {t_heap = {t_xmin = 160, t_xmax = 4294967295, t_field3 = {t_cid = 2249, t_xvac = 2249}}, t_datum = {
          datum_len_ = 160, datum_typmod = -1, datum_typeid = 2249}}, t_ctid = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, 
        ip_posid = 0}, t_infomask2 = 3, t_infomask = 2, t_hoff = 24 '\030', t_bits = 0x1e4f50f ""}
    

mtstate->mt_transition_capture 为NULL,无需处理相关信息

    
    
    (gdb) p mtstate->mt_transition_capture 
    $8 = (struct TransitionCaptureState *) 0x0
    1783        if (mtstate->mt_oc_transition_capture != NULL)
    (gdb) 
    

如需要,转换元组

    
    
    1792        ConvertPartitionTupleSlot(proute->parent_child_tupconv_maps[partidx],
    (gdb) 
    1798        Assert(mtstate != NULL);
    (gdb) 
    1799        node = (ModifyTable *) mtstate->ps.plan;
    (gdb) p *mtstate
    $9 = {ps = {type = T_ModifyTableState, plan = 0x1e59838, state = 0x1e4daf8, ExecProcNode = 0x711056 <ExecModifyTable>, 
        ExecProcNodeReal = 0x711056 <ExecModifyTable>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
        qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
        ps_ResultTupleSlot = 0x1e4ede8, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, operation = CMD_INSERT, 
      canSetTag = true, mt_done = false, mt_plans = 0x1e4e078, mt_nplans = 1, mt_whichplan = 0, resultRelInfo = 0x1e4dd48, 
      rootResultRelInfo = 0x0, mt_arowmarks = 0x1e4e098, mt_epqstate = {estate = 0x0, planstate = 0x0, origslot = 0x1e4e4e0, 
        plan = 0x1e59588, arowMarks = 0x0, epqParam = 0}, fireBSTriggers = false, mt_existing = 0x0, mt_excludedtlist = 0x0, 
      mt_conflproj = 0x0, mt_partition_tuple_routing = 0x1e4eb48, mt_transition_capture = 0x0, mt_oc_transition_capture = 0x0, 
      mt_per_subplan_tupconv_maps = 0x0}  
    

返回slot,完成调用

    
    
    (gdb) n
    1800        if (node->onConflictAction == ONCONFLICT_UPDATE)
    (gdb) 
    1810        return slot;
    (gdb) 
    1811    }
    

DONE!  
**ExecFindPartition函数是主要的实现函数,下节再行介绍**

### 四、参考资料

PG 11.1 Source Code.  
注: [doxygen](https://doxygen.postgresql.org)上的源代码与PG 11.1源代码并不一致,本节基于11.1进行分析.

