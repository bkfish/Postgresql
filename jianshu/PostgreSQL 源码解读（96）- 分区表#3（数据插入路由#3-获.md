本节介绍了ExecPrepareTupleRouting->ExecFindPartition->FormPartitionKeyDatum函数，该函数获取Tuple的分区键值。

### 一、数据结构

**ModifyTable**  
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

FormPartitionKeyDatum函数获取Tuple的分区键值,返回键值values[]数组和是否为null标记isnull[]数组.

    
    
    /* ----------------     *      FormPartitionKeyDatum
     *          Construct values[] and isnull[] arrays for the partition key
     *          of a tuple.
     *          构造values[]数组和isnull[]数组
     *
     *  pd              Partition dispatch object of the partitioned table
     *  pd              分区表的分区分发器(dispatch)对象
     *
     *  slot            Heap tuple from which to extract partition key
     *  slot            从其中提前分区键的heap tuple
     *
     *  estate          executor state for evaluating any partition key
     *                  expressions (must be non-NULL)
     *  estate          解析分区键表达式(必须非NULL)的执行器状态
     *
     *  values          Array of partition key Datums (output area)
     *                  分区键Datums数组(输出参数)
     *  isnull          Array of is-null indicators (output area)
     *                  is-null标记数组(输出参数)
     *
     * the ecxt_scantuple slot of estate's per-tuple expr context must point to
     * the heap tuple passed in.
     * estate的per-tuple上下文的ecxt_scantuple必须指向传入的heap tuple
     * ----------------     */
    static void
    FormPartitionKeyDatum(PartitionDispatch pd,
                          TupleTableSlot *slot,
                          EState *estate,
                          Datum *values,
                          bool *isnull)
    {
        ListCell   *partexpr_item;
        int         i;
    
        if (pd->key->partexprs != NIL && pd->keystate == NIL)
        {
            /* Check caller has set up context correctly */
            //检查调用者是否已正确配置内存上下文
            Assert(estate != NULL &&
                   GetPerTupleExprContext(estate)->ecxt_scantuple == slot);
    
            /* First time through, set up expression evaluation state */
            //第一次进入,配置表达式解析器状态
            pd->keystate = ExecPrepareExprList(pd->key->partexprs, estate);
        }
    
        partexpr_item = list_head(pd->keystate);//获取分区键表达式状态
        for (i = 0; i < pd->key->partnatts; i++)//循环遍历分区键
        {
            AttrNumber  keycol = pd->key->partattrs[i];//分区键属性编号
            Datum       datum;// typedef uintptr_t Datum;sizeof(Datum) == sizeof(void *) == 4 or 8
            bool        isNull;//是否null
    
            if (keycol != 0)//编号不为0
            {
                /* Plain column; get the value directly from the heap tuple */
                //扁平列,直接从堆元组中提取值
                datum = slot_getattr(slot, keycol, &isNull);
            }
            else
            {
                /* Expression; need to evaluate it */
                //表达式,需要解析
                if (partexpr_item == NULL)//分区键表达式状态为NULL,报错
                    elog(ERROR, "wrong number of partition key expressions");
                //获取表达式值
                datum = ExecEvalExprSwitchContext((ExprState *) lfirst(partexpr_item),
                                                  GetPerTupleExprContext(estate),
                                                  &isNull);
                //切换至下一个
                partexpr_item = lnext(partexpr_item);
            }
            values[i] = datum;//赋值
            isnull[i] = isNull;
        }
    
        if (partexpr_item != NULL)//参数设置有误?报错
            elog(ERROR, "wrong number of partition key expressions");
    }
    
    
    
    /*
     * slot_getattr - fetch one attribute of the slot's contents.
     * slot_getattr - 提取slot中的某个属性值
     */
    static inline Datum
    slot_getattr(TupleTableSlot *slot, int attnum,
                 bool *isnull)
    {
        AssertArg(attnum > 0);
    
        if (attnum > slot->tts_nvalid)
            slot_getsomeattrs(slot, attnum);
    
        *isnull = slot->tts_isnull[attnum - 1];
    
        return slot->tts_values[attnum - 1];
    }
    
    
    /*
     * This function forces the entries of the slot's Datum/isnull arrays to be
     * valid at least up through the attnum'th entry.
     * 这个函数强制slot的Datum/isnull数组的条目至少在attnum的第一个条目上是有效的。
     */
    static inline void
    slot_getsomeattrs(TupleTableSlot *slot, int attnum)
    {
        if (slot->tts_nvalid < attnum)
            slot_getsomeattrs_int(slot, attnum);
    }
    
    
    /*
     * slot_getsomeattrs_int - workhorse for slot_getsomeattrs()
     * slot_getsomeattrs_int - slot_getsomeattrs()函数的实际实现
     */
    void
    slot_getsomeattrs_int(TupleTableSlot *slot, int attnum)
    {
        /* Check for caller errors */
        //检查调用者输入参数是否有误
        Assert(slot->tts_nvalid < attnum); /* slot_getsomeattr checked */
        Assert(attnum > 0);
        //attnum参数判断
        if (unlikely(attnum > slot->tts_tupleDescriptor->natts))
            elog(ERROR, "invalid attribute number %d", attnum);
    
        /* Fetch as many attributes as possible from the underlying tuple. */
        //从元组中获取尽可能多的属性。
        slot->tts_ops->getsomeattrs(slot, attnum);
    
        /*
         * If the underlying tuple doesn't have enough attributes, tuple descriptor
         * must have the missing attributes.
         * 如果底层元组没有足够的属性，那么元组描述符必须具有缺少的属性。
         */
        if (unlikely(slot->tts_nvalid < attnum))
        {
            slot_getmissingattrs(slot, slot->tts_nvalid, attnum);
            slot->tts_nvalid = attnum;
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
    
    insert into t_hash_partition(c1,c2,c3) VALUES(20,'HASH0','HAHS0');
    

启动gdb,设置断点

    
    
    (gdb) b FormPartitionKeyDatum
    Breakpoint 5 at 0x6e30d2: file execPartition.c, line 1087.
    (gdb) b slot_getattr
    Breakpoint 6 at 0x489d9b: file heaptuple.c, line 1510.
    (gdb) c
    Continuing.
    
    Breakpoint 5, FormPartitionKeyDatum (pd=0x2e1bfa0, slot=0x2e1b8a0, estate=0x2e1aeb8, values=0x7fff4e2407a0, 
        isnull=0x7fff4e240780) at execPartition.c:1087
    1087        if (pd->key->partexprs != NIL && pd->keystate == NIL)
    

循环,根据分区键获取相应的键值

    
    
    1087        if (pd->key->partexprs != NIL && pd->keystate == NIL)
    (gdb) n
    1097        partexpr_item = list_head(pd->keystate);
    (gdb) 
    1098        for (i = 0; i < pd->key->partnatts; i++)
    (gdb) 
    1100            AttrNumber  keycol = pd->key->partattrs[i];
    (gdb) 
    1104            if (keycol != 0)
    (gdb) 
    1107                datum = slot_getattr(slot, keycol, &isNull);
    

进入函数slot_getattr

    
    
    (gdb) step
    
    Breakpoint 6, slot_getattr (slot=0x2e1b8a0, attnum=1, isnull=0x7fff4e240735) at heaptuple.c:1510
    1510        HeapTuple   tuple = slot->tts_tuple;
    

获取结果,分区键值为20

    
    
    ...
    (gdb) p *isnull
    $31 = false
    (gdb) p slot->tts_values[attnum - 1]
    $32 = 20
    

返回到FormPartitionKeyDatum函数中

    
    
    (gdb) n
    1593    }
    (gdb) 
    FormPartitionKeyDatum (pd=0x2e1bfa0, slot=0x2e1b8a0, estate=0x2e1aeb8, values=0x7fff4e2407a0, isnull=0x7fff4e240780)
        at execPartition.c:1119
    1119            values[i] = datum;
    

完成调用

    
    
    1119            values[i] = datum;
    (gdb) n
    1120            isnull[i] = isNull;
    (gdb) 
    1098        for (i = 0; i < pd->key->partnatts; i++)
    (gdb) 
    1123        if (partexpr_item != NULL)
    (gdb) 
    1125    }
    (gdb) 
    ExecFindPartition (resultRelInfo=0x2e1b108, pd=0x2e1c5b8, slot=0x2e1b8a0, estate=0x2e1aeb8) at execPartition.c:282
    282         if (partdesc->nparts == 0)
    

DONE!

### 四、参考资料

PG 11.1 Source Code.  
注: [doxygen](https://doxygen.postgresql.org)上的源代码与PG 11.1源代码并不一致,本节基于11.1进行分析.

