本文简单介绍了PG插入数据部分的源码，主要内容包括ProcessQuery函数的实现逻辑，该函数位于文件pquery.c中。

### 一、基础信息

ProcessQuery函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、NodeTag_

    
    
    //节点标记，枚举类型
     /*
      * The first field of every node is NodeTag. Each node created (with makeNode)
      * will have one of the following tags as the value of its first field.
      *
      * Note that inserting or deleting node types changes the numbers of other
      * node types later in the list.  This is no problem during development, since
      * the node numbers are never stored on disk.  But don't do it in a released
      * branch, because that would represent an ABI break for extensions.
      */
     typedef enum NodeTag
     {
         T_Invalid = 0,
     
         /*
          * TAGS FOR EXECUTOR NODES (execnodes.h)
          */
         T_IndexInfo,
         T_ExprContext,
         T_ProjectionInfo,
         T_JunkFilter,
         T_OnConflictSetState,
         T_ResultRelInfo,
         T_EState,
         T_TupleTableSlot,
     
         /*
          * TAGS FOR PLAN NODES (plannodes.h)
          */
         T_Plan,
         T_Result,
         T_ProjectSet,
         T_ModifyTable,
         T_Append,
         T_MergeAppend,
         T_RecursiveUnion,
         T_BitmapAnd,
         T_BitmapOr,
         T_Scan,
         T_SeqScan,
         T_SampleScan,
         T_IndexScan,
         T_IndexOnlyScan,
         T_BitmapIndexScan,
         T_BitmapHeapScan,
         T_TidScan,
         T_SubqueryScan,
         T_FunctionScan,
         T_ValuesScan,
         T_TableFuncScan,
         T_CteScan,
         T_NamedTuplestoreScan,
         T_WorkTableScan,
         T_ForeignScan,
         T_CustomScan,
         T_Join,
         T_NestLoop,
         T_MergeJoin,
         T_HashJoin,
         T_Material,
         T_Sort,
         T_Group,
         T_Agg,
         T_WindowAgg,
         T_Unique,
         T_Gather,
         T_GatherMerge,
         T_Hash,
         T_SetOp,
         T_LockRows,
         T_Limit,
         /* these aren't subclasses of Plan: */
         T_NestLoopParam,
         T_PlanRowMark,
         T_PartitionPruneInfo,
         T_PartitionedRelPruneInfo,
         T_PartitionPruneStepOp,
         T_PartitionPruneStepCombine,
         T_PlanInvalItem,
     
         /*
          * TAGS FOR PLAN STATE NODES (execnodes.h)
          *
          * These should correspond one-to-one with Plan node types.
          */
         T_PlanState,
         T_ResultState,
         T_ProjectSetState,
         T_ModifyTableState,
         T_AppendState,
         T_MergeAppendState,
         T_RecursiveUnionState,
         T_BitmapAndState,
         T_BitmapOrState,
         T_ScanState,
         T_SeqScanState,
         T_SampleScanState,
         T_IndexScanState,
         T_IndexOnlyScanState,
         T_BitmapIndexScanState,
         T_BitmapHeapScanState,
         T_TidScanState,
         T_SubqueryScanState,
         T_FunctionScanState,
         T_TableFuncScanState,
         T_ValuesScanState,
         T_CteScanState,
         T_NamedTuplestoreScanState,
         T_WorkTableScanState,
         T_ForeignScanState,
         T_CustomScanState,
         T_JoinState,
         T_NestLoopState,
         T_MergeJoinState,
         T_HashJoinState,
         T_MaterialState,
         T_SortState,
         T_GroupState,
         T_AggState,
         T_WindowAggState,
         T_UniqueState,
         T_GatherState,
         T_GatherMergeState,
         T_HashState,
         T_SetOpState,
         T_LockRowsState,
         T_LimitState,
     
         /*
          * TAGS FOR PRIMITIVE NODES (primnodes.h)
          */
         T_Alias,
         T_RangeVar,
         T_TableFunc,
         T_Expr,
         T_Var,
         T_Const,
         T_Param,
         T_Aggref,
         T_GroupingFunc,
         T_WindowFunc,
         T_ArrayRef,
         T_FuncExpr,
         T_NamedArgExpr,
         T_OpExpr,
         T_DistinctExpr,
         T_NullIfExpr,
         T_ScalarArrayOpExpr,
         T_BoolExpr,
         T_SubLink,
         T_SubPlan,
         T_AlternativeSubPlan,
         T_FieldSelect,
         T_FieldStore,
         T_RelabelType,
         T_CoerceViaIO,
         T_ArrayCoerceExpr,
         T_ConvertRowtypeExpr,
         T_CollateExpr,
         T_CaseExpr,
         T_CaseWhen,
         T_CaseTestExpr,
         T_ArrayExpr,
         T_RowExpr,
         T_RowCompareExpr,
         T_CoalesceExpr,
         T_MinMaxExpr,
         T_SQLValueFunction,
         T_XmlExpr,
         T_NullTest,
         T_BooleanTest,
         T_CoerceToDomain,
         T_CoerceToDomainValue,
         T_SetToDefault,
         T_CurrentOfExpr,
         T_NextValueExpr,
         T_InferenceElem,
         T_TargetEntry,
         T_RangeTblRef,
         T_JoinExpr,
         T_FromExpr,
         T_OnConflictExpr,
         T_IntoClause,
     
         /*
          * TAGS FOR EXPRESSION STATE NODES (execnodes.h)
          *
          * ExprState represents the evaluation state for a whole expression tree.
          * Most Expr-based plan nodes do not have a corresponding expression state
          * node, they're fully handled within execExpr* - but sometimes the state
          * needs to be shared with other parts of the executor, as for example
          * with AggrefExprState, which nodeAgg.c has to modify.
          */
         T_ExprState,
         T_AggrefExprState,
         T_WindowFuncExprState,
         T_SetExprState,
         T_SubPlanState,
         T_AlternativeSubPlanState,
         T_DomainConstraintState,
     
         /*
          * TAGS FOR PLANNER NODES (relation.h)
          */
         T_PlannerInfo,
         T_PlannerGlobal,
         T_RelOptInfo,
         T_IndexOptInfo,
         T_ForeignKeyOptInfo,
         T_ParamPathInfo,
         T_Path,
         T_IndexPath,
         T_BitmapHeapPath,
         T_BitmapAndPath,
         T_BitmapOrPath,
         T_TidPath,
         T_SubqueryScanPath,
         T_ForeignPath,
         T_CustomPath,
         T_NestPath,
         T_MergePath,
         T_HashPath,
         T_AppendPath,
         T_MergeAppendPath,
         T_ResultPath,
         T_MaterialPath,
         T_UniquePath,
         T_GatherPath,
         T_GatherMergePath,
         T_ProjectionPath,
         T_ProjectSetPath,
         T_SortPath,
         T_GroupPath,
         T_UpperUniquePath,
         T_AggPath,
         T_GroupingSetsPath,
         T_MinMaxAggPath,
         T_WindowAggPath,
         T_SetOpPath,
         T_RecursiveUnionPath,
         T_LockRowsPath,
         T_ModifyTablePath,
         T_LimitPath,
         /* these aren't subclasses of Path: */
         T_EquivalenceClass,
         T_EquivalenceMember,
         T_PathKey,
         T_PathTarget,
         T_RestrictInfo,
         T_PlaceHolderVar,
         T_SpecialJoinInfo,
         T_AppendRelInfo,
         T_PlaceHolderInfo,
         T_MinMaxAggInfo,
         T_PlannerParamItem,
         T_RollupData,
         T_GroupingSetData,
         T_StatisticExtInfo,
     
         /*
          * TAGS FOR MEMORY NODES (memnodes.h)
          */
         T_MemoryContext,
         T_AllocSetContext,
         T_SlabContext,
         T_GenerationContext,
     
         /*
          * TAGS FOR VALUE NODES (value.h)
          */
         T_Value,
         T_Integer,
         T_Float,
         T_String,
         T_BitString,
         T_Null,
     
         /*
          * TAGS FOR LIST NODES (pg_list.h)
          */
         T_List,
         T_IntList,
         T_OidList,
     
         /*
          * TAGS FOR EXTENSIBLE NODES (extensible.h)
          */
         T_ExtensibleNode,
     
         /*
          * TAGS FOR STATEMENT NODES (mostly in parsenodes.h)
          */
         T_RawStmt,
         T_Query,
         T_PlannedStmt,
         T_InsertStmt,
         T_DeleteStmt,
         T_UpdateStmt,
         T_SelectStmt,
         T_AlterTableStmt,
         T_AlterTableCmd,
         T_AlterDomainStmt,
         T_SetOperationStmt,
         T_GrantStmt,
         T_GrantRoleStmt,
         T_AlterDefaultPrivilegesStmt,
         T_ClosePortalStmt,
         T_ClusterStmt,
         T_CopyStmt,
         T_CreateStmt,
         T_DefineStmt,
         T_DropStmt,
         T_TruncateStmt,
         T_CommentStmt,
         T_FetchStmt,
         T_IndexStmt,
         T_CreateFunctionStmt,
         T_AlterFunctionStmt,
         T_DoStmt,
         T_RenameStmt,
         T_RuleStmt,
         T_NotifyStmt,
         T_ListenStmt,
         T_UnlistenStmt,
         T_TransactionStmt,
         T_ViewStmt,
         T_LoadStmt,
         T_CreateDomainStmt,
         T_CreatedbStmt,
         T_DropdbStmt,
         T_VacuumStmt,
         T_ExplainStmt,
         T_CreateTableAsStmt,
         T_CreateSeqStmt,
         T_AlterSeqStmt,
         T_VariableSetStmt,
         T_VariableShowStmt,
         T_DiscardStmt,
         T_CreateTrigStmt,
         T_CreatePLangStmt,
         T_CreateRoleStmt,
         T_AlterRoleStmt,
         T_DropRoleStmt,
         T_LockStmt,
         T_ConstraintsSetStmt,
         T_ReindexStmt,
         T_CheckPointStmt,
         T_CreateSchemaStmt,
         T_AlterDatabaseStmt,
         T_AlterDatabaseSetStmt,
         T_AlterRoleSetStmt,
         T_CreateConversionStmt,
         T_CreateCastStmt,
         T_CreateOpClassStmt,
         T_CreateOpFamilyStmt,
         T_AlterOpFamilyStmt,
         T_PrepareStmt,
         T_ExecuteStmt,
         T_DeallocateStmt,
         T_DeclareCursorStmt,
         T_CreateTableSpaceStmt,
         T_DropTableSpaceStmt,
         T_AlterObjectDependsStmt,
         T_AlterObjectSchemaStmt,
         T_AlterOwnerStmt,
         T_AlterOperatorStmt,
         T_DropOwnedStmt,
         T_ReassignOwnedStmt,
         T_CompositeTypeStmt,
         T_CreateEnumStmt,
         T_CreateRangeStmt,
         T_AlterEnumStmt,
         T_AlterTSDictionaryStmt,
         T_AlterTSConfigurationStmt,
         T_CreateFdwStmt,
         T_AlterFdwStmt,
         T_CreateForeignServerStmt,
         T_AlterForeignServerStmt,
         T_CreateUserMappingStmt,
         T_AlterUserMappingStmt,
         T_DropUserMappingStmt,
         T_AlterTableSpaceOptionsStmt,
         T_AlterTableMoveAllStmt,
         T_SecLabelStmt,
         T_CreateForeignTableStmt,
         T_ImportForeignSchemaStmt,
         T_CreateExtensionStmt,
         T_AlterExtensionStmt,
         T_AlterExtensionContentsStmt,
         T_CreateEventTrigStmt,
         T_AlterEventTrigStmt,
         T_RefreshMatViewStmt,
         T_ReplicaIdentityStmt,
         T_AlterSystemStmt,
         T_CreatePolicyStmt,
         T_AlterPolicyStmt,
         T_CreateTransformStmt,
         T_CreateAmStmt,
         T_CreatePublicationStmt,
         T_AlterPublicationStmt,
         T_CreateSubscriptionStmt,
         T_AlterSubscriptionStmt,
         T_DropSubscriptionStmt,
         T_CreateStatsStmt,
         T_AlterCollationStmt,
         T_CallStmt,
     
         /*
          * TAGS FOR PARSE TREE NODES (parsenodes.h)
          */
         T_A_Expr,
         T_ColumnRef,
         T_ParamRef,
         T_A_Const,
         T_FuncCall,
         T_A_Star,
         T_A_Indices,
         T_A_Indirection,
         T_A_ArrayExpr,
         T_ResTarget,
         T_MultiAssignRef,
         T_TypeCast,
         T_CollateClause,
         T_SortBy,
         T_WindowDef,
         T_RangeSubselect,
         T_RangeFunction,
         T_RangeTableSample,
         T_RangeTableFunc,
         T_RangeTableFuncCol,
         T_TypeName,
         T_ColumnDef,
         T_IndexElem,
         T_Constraint,
         T_DefElem,
         T_RangeTblEntry,
         T_RangeTblFunction,
         T_TableSampleClause,
         T_WithCheckOption,
         T_SortGroupClause,
         T_GroupingSet,
         T_WindowClause,
         T_ObjectWithArgs,
         T_AccessPriv,
         T_CreateOpClassItem,
         T_TableLikeClause,
         T_FunctionParameter,
         T_LockingClause,
         T_RowMarkClause,
         T_XmlSerialize,
         T_WithClause,
         T_InferClause,
         T_OnConflictClause,
         T_CommonTableExpr,
         T_RoleSpec,
         T_TriggerTransition,
         T_PartitionElem,
         T_PartitionSpec,
         T_PartitionBoundSpec,
         T_PartitionRangeDatum,
         T_PartitionCmd,
         T_VacuumRelation,
     
         /*
          * TAGS FOR REPLICATION GRAMMAR PARSE NODES (replnodes.h)
          */
         T_IdentifySystemCmd,
         T_BaseBackupCmd,
         T_CreateReplicationSlotCmd,
         T_DropReplicationSlotCmd,
         T_StartReplicationCmd,
         T_TimeLineHistoryCmd,
         T_SQLCmd,
     
         /*
          * TAGS FOR RANDOM OTHER STUFF
          *
          * These are objects that aren't part of parse/plan/execute node tree
          * structures, but we give them NodeTags anyway for identification
          * purposes (usually because they are involved in APIs where we want to
          * pass multiple object types through the same pointer).
          */
         T_TriggerData,              /* in commands/trigger.h */
         T_EventTriggerData,         /* in commands/event_trigger.h */
         T_ReturnSetInfo,            /* in nodes/execnodes.h */
         T_WindowObjectData,         /* private in nodeWindowAgg.c */
         T_TIDBitmap,                /* in nodes/tidbitmap.h */
         T_InlineCodeBlock,          /* in nodes/parsenodes.h */
         T_FdwRoutine,               /* in foreign/fdwapi.h */
         T_IndexAmRoutine,           /* in access/amapi.h */
         T_TsmRoutine,               /* in access/tsmapi.h */
         T_ForeignKeyCacheInfo,      /* in utils/rel.h */
         T_CallContext               /* in nodes/parsenodes.h */
     } NodeTag;
     
     /*
      * The first field of a node of any type is guaranteed to be the NodeTag.
      * Hence the type of any node can be gotten by casting it to Node. Declaring
      * a variable to be of Node * (instead of void *) can also facilitate
      * debugging.
      */
     typedef struct Node
     {
         NodeTag     type;
     } Node;
     
     #define nodeTag(nodeptr)        (((const Node*)(nodeptr))->type)
    

_2、MemoryContext_

    
    
    //内存上下文
    //AllocSetContext结构体的MemoryContextData与其共享
    //
     typedef struct MemoryContextData
     {
         NodeTag     type;           /* identifies exact kind of context */
         /* these two fields are placed here to minimize alignment wastage: */
         bool        isReset;        /* T = no space alloced since last reset */
         bool        allowInCritSection; /* allow palloc in critical section */
         const MemoryContextMethods *methods;    /* virtual function table */
         MemoryContext parent;       /* NULL if no parent (toplevel context) */
         MemoryContext firstchild;   /* head of linked list of children */
         MemoryContext prevchild;    /* previous child of same parent */
         MemoryContext nextchild;    /* next child of same parent */
         const char *name;           /* context name (just for debugging) */
         const char *ident;          /* context ID if any (just for debugging) */
         MemoryContextCallback *reset_cbs;   /* list of reset/delete callbacks */
     } MemoryContextData;
     
     /* utils/palloc.h contains typedef struct MemoryContextData *MemoryContext */
    
     /*
      * Type MemoryContextData is declared in nodes/memnodes.h.  Most users
      * of memory allocation should just treat it as an abstract type, so we
      * do not provide the struct contents here.
      */
     typedef struct MemoryContextData *MemoryContext;
     
    

_3、AllocSet_

    
    
    /*
      * AllocSetContext is our standard implementation of MemoryContext.
      *
      * Note: header.isReset means there is nothing for AllocSetReset to do.
      * This is different from the aset being physically empty (empty blocks list)
      * because we will still have a keeper block.  It's also different from the set
      * being logically empty, because we don't attempt to detect pfree'ing the
      * last active chunk.
      */
     typedef struct AllocSetContext
     {
         MemoryContextData header;   /* Standard memory-context fields */
         /* Info about storage allocated in this context: */
         AllocBlock  blocks;         /* head of list of blocks in this set */
         AllocChunk  freelist[ALLOCSET_NUM_FREELISTS];   /* free chunk lists */
         /* Allocation parameters for this context: */
         Size        initBlockSize;  /* initial block size */
         Size        maxBlockSize;   /* maximum block size */
         Size        nextBlockSize;  /* next block size to allocate */
         Size        allocChunkLimit;    /* effective chunk size limit */
         AllocBlock  keeper;         /* keep this block over resets */
         /* freelist this context could be put in, or -1 if not a candidate: */
         int         freeListIndex;  /* index in context_freelists[], or -1 */
     } AllocSetContext;
     
     typedef AllocSetContext *AllocSet;
     
    

_4、AllocBlock_

    
    
     /*
      * AllocBlock
      *      An AllocBlock is the unit of memory that is obtained by aset.c
      *      from malloc().  It contains one or more AllocChunks, which are
      *      the units requested by palloc() and freed by pfree().  AllocChunks
      *      cannot be returned to malloc() individually, instead they are put
      *      on freelists by pfree() and re-used by the next palloc() that has
      *      a matching request size.
      *
      *      AllocBlockData is the header data for a block --- the usable space
      *      within the block begins at the next alignment boundary.
      */
     typedef struct AllocBlockData
     {
         AllocSet    aset;           /* aset that owns this block */
         AllocBlock  prev;           /* prev block in aset's blocks list, if any */
         AllocBlock  next;           /* next block in aset's blocks list, if any */
         char       *freeptr;        /* start of free space in this block */
         char       *endptr;         /* end of space in this block */
     }           AllocBlockData;
     
     typedef struct AllocBlockData *AllocBlock; /* forward reference */
    
    

_5、AllocChunk_

    
    
     /*
      * AllocChunk
      *      The prefix of each piece of memory in an AllocBlock
      *
      * Note: to meet the memory context APIs, the payload area of the chunk must
      * be maxaligned, and the "aset" link must be immediately adjacent to the
      * payload area (cf. GetMemoryChunkContext).  We simplify matters for this
      * module by requiring sizeof(AllocChunkData) to be maxaligned, and then
      * we can ensure things work by adding any required alignment padding before
      * the "aset" field.  There is a static assertion below that the alignment
      * is done correctly.
      */
     typedef struct AllocChunkData
     {
         /* size is always the size of the usable space in the chunk */
         Size        size;
     #ifdef MEMORY_CONTEXT_CHECKING
         /* when debugging memory usage, also store actual requested size */
         /* this is zero in a free chunk */
         Size        requested_size;
     
     #define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T * 2 + SIZEOF_VOID_P)
     #else
     #define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T + SIZEOF_VOID_P)
     #endif                          /* MEMORY_CONTEXT_CHECKING */
     
         /* ensure proper alignment by adding padding if needed */
     #if (ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF) != 0
         char        padding[MAXIMUM_ALIGNOF - ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF];
     #endif
     
         /* aset is the owning aset if allocated, or the freelist link if free */
         void       *aset;
         /* there must not be any padding to reach a MAXALIGN boundary here! */
     }           AllocChunkData;
     
     typedef struct AllocChunkData *AllocChunk;
    
    

_6、AllocSetFreeList_

    
    
     
    typedef struct AllocSetFreeList
    {
        int         num_free;       /* current list length */
        AllocSetContext *first_free;    /* list header */
    } AllocSetFreeList;
    

_7、context_freelists_

    
    
    /* context_freelists[0] is for default params, [1] for small params */
    static AllocSetFreeList context_freelists[2] =
    {
        {
            0, NULL
        },
        {
            0, NULL
        }
    };
    

_8、AllocSetMethods_

    
    
    //内存管理方法
     typedef struct MemoryContextMethods
     {
         void       *(*alloc) (MemoryContext context, Size size);
         /* call this free_p in case someone #define's free() */
         void        (*free_p) (MemoryContext context, void *pointer);
         void       *(*realloc) (MemoryContext context, void *pointer, Size size);
         void        (*reset) (MemoryContext context);
         void        (*delete_context) (MemoryContext context);
         Size        (*get_chunk_space) (MemoryContext context, void *pointer);
         bool        (*is_empty) (MemoryContext context);
         void        (*stats) (MemoryContext context,
                               MemoryStatsPrintFunc printfunc, void *passthru,
                               MemoryContextCounters *totals);
     #ifdef MEMORY_CONTEXT_CHECKING
         void        (*check) (MemoryContext context);
     #endif
     } MemoryContextMethods;
    
    //预定义
     /*
      * This is the virtual function table for AllocSet contexts.
      */
     static const MemoryContextMethods AllocSetMethods = {
         AllocSetAlloc,
         AllocSetFree,
         AllocSetRealloc,
         AllocSetReset,
         AllocSetDelete,
         AllocSetGetChunkSpace,
         AllocSetIsEmpty,
         AllocSetStats
     #ifdef MEMORY_CONTEXT_CHECKING
         ,AllocSetCheck
     #endif
     };
    

_9、宏定义_

    
    
     #define ALLOC_BLOCKHDRSZ    MAXALIGN(sizeof(AllocBlockData))
     #define ALLOC_CHUNKHDRSZ    sizeof(struct AllocChunkData)
    
     /*
      * Recommended default alloc parameters, suitable for "ordinary" contexts
      * that might hold quite a lot of data.
      */
     #define ALLOCSET_DEFAULT_MINSIZE   0
     #define ALLOCSET_DEFAULT_INITSIZE  (8 * 1024)
     #define ALLOCSET_DEFAULT_MAXSIZE   (8 * 1024 * 1024)
     #define ALLOCSET_DEFAULT_SIZES \
         ALLOCSET_DEFAULT_MINSIZE, ALLOCSET_DEFAULT_INITSIZE, ALLOCSET_DEFAULT_MAXSIZE
     #define VALGRIND_MAKE_MEM_NOACCESS(addr, size) do {} while (0)
    
     #define ALLOC_MINBITS 3 /* smallest chunk size is 8 bytes */
     #define ALLOCSET_NUM_FREELISTS 11
     #define ALLOC_CHUNK_LIMIT (1 << (ALLOCSET_NUM_FREELISTS-1+ALLOC_MINBITS))
     #define ALLOCSET_SEPARATE_THRESHOLD 8192
    #define ALLOC_CHUNK_FRACTION 4
     /* We allow chunks to be at most 1/4 of maxBlockSize (less overhead) */
    

**依赖的函数**  
_1、CreateQueryDesc_

    
    
    //根据输入的参数构造QueryDesc
     /*
      * CreateQueryDesc
      */
     QueryDesc *
     CreateQueryDesc(PlannedStmt *plannedstmt,
                     const char *sourceText,
                     Snapshot snapshot,
                     Snapshot crosscheck_snapshot,
                     DestReceiver *dest,
                     ParamListInfo params,
                     QueryEnvironment *queryEnv,
                     int instrument_options)
     {
         QueryDesc  *qd = (QueryDesc *) palloc(sizeof(QueryDesc));
     
         qd->operation = plannedstmt->commandType;   /* operation */
         qd->plannedstmt = plannedstmt;  /* plan */
         qd->sourceText = sourceText;    /* query text */
         qd->snapshot = RegisterSnapshot(snapshot);  /* snapshot */
         /* RI check snapshot */
         qd->crosscheck_snapshot = RegisterSnapshot(crosscheck_snapshot);
         qd->dest = dest;            /* output dest */
         qd->params = params;        /* parameter values passed into query */
         qd->queryEnv = queryEnv;
         qd->instrument_options = instrument_options;    /* instrumentation wanted? */
     
         /* null these fields until set by ExecutorStart */
         qd->tupDesc = NULL;
         qd->estate = NULL;
         qd->planstate = NULL;
         qd->totaltime = NULL;
     
         /* not yet executed */
         qd->already_executed = false;
     
         return qd;
     }
     
    

_2、CreateExecutorState_

    
    
     /* ----------------      *      CreateExecutorState
      *
      *      Create and initialize an EState node, which is the root of
      *      working storage for an entire Executor invocation.
      *
      * Principally, this creates the per-query memory context that will be
      * used to hold all working data that lives till the end of the query.
      * Note that the per-query context will become a child of the caller's
      * CurrentMemoryContext.
      * ----------------      */
     EState *
     CreateExecutorState(void)//构建EState
     {
         EState     *estate;//EState指针
         MemoryContext qcontext;
         MemoryContext oldcontext;
     
         /*
          * Create the per-query context for this Executor run.
          */
         qcontext = AllocSetContextCreate(CurrentMemoryContext,
                                          "ExecutorState",
                                          ALLOCSET_DEFAULT_SIZES);//创建MemoryContext
     
         /*
          * Make the EState node within the per-query context.  This way, we don't
          * need a separate pfree() operation for it at shutdown.
          */
         oldcontext = MemoryContextSwitchTo(qcontext);//切换至新创建的MemoryContext
     
         estate = makeNode(EState);//创建Node
     
         /*
          * Initialize all fields of the Executor State structure
          */
         //初始化Executor State structure
         estate->es_direction = ForwardScanDirection;
         estate->es_snapshot = InvalidSnapshot;  /* caller must initialize this */
         estate->es_crosscheck_snapshot = InvalidSnapshot;   /* no crosscheck */
         estate->es_range_table = NIL;
         estate->es_plannedstmt = NULL;
     
         estate->es_junkFilter = NULL;
     
         estate->es_output_cid = (CommandId) 0;
     
         estate->es_result_relations = NULL;
         estate->es_num_result_relations = 0;
         estate->es_result_relation_info = NULL;
     
         estate->es_root_result_relations = NULL;
         estate->es_num_root_result_relations = 0;
     
         estate->es_tuple_routing_result_relations = NIL;
     
         estate->es_trig_target_relations = NIL;
         estate->es_trig_tuple_slot = NULL;
         estate->es_trig_oldtup_slot = NULL;
         estate->es_trig_newtup_slot = NULL;
     
         estate->es_param_list_info = NULL;
         estate->es_param_exec_vals = NULL;
     
         estate->es_queryEnv = NULL;
     
         estate->es_query_cxt = qcontext;
     
         estate->es_tupleTable = NIL;
     
         estate->es_rowMarks = NIL;
     
         estate->es_processed = 0;
         estate->es_lastoid = InvalidOid;
     
         estate->es_top_eflags = 0;
         estate->es_instrument = 0;
         estate->es_finished = false;
     
         estate->es_exprcontexts = NIL;
     
         estate->es_subplanstates = NIL;
     
         estate->es_auxmodifytables = NIL;
     
         estate->es_per_tuple_exprcontext = NULL;
     
         estate->es_epqTuple = NULL;
         estate->es_epqTupleSet = NULL;
         estate->es_epqScanDone = NULL;
         estate->es_sourceText = NULL;
     
         estate->es_use_parallel_mode = false;
     
         estate->es_jit_flags = 0;
         estate->es_jit = NULL;
     
         /*
          * Return the executor state structure
          */
         MemoryContextSwitchTo(oldcontext);
     
         return estate;
     }
     /*------------------ makeNode --------------------*/
     #define makeNode(_type_)        ((_type_ *) newNode(sizeof(_type_),T_##_type_))
     /*
      * newNode -      *    create a new node of the specified size and tag the node with the
      *    specified tag.
      *
      * !WARNING!: Avoid using newNode directly. You should be using the
      *    macro makeNode.  eg. to create a Query node, use makeNode(Query)
      *
      * Note: the size argument should always be a compile-time constant, so the
      * apparent risk of multiple evaluation doesn't matter in practice.
      */
     #ifdef __GNUC__
     
     /* With GCC, we can use a compound statement within an expression */
     #define newNode(size, tag) \
     ({  Node   *_result; \
         AssertMacro((size) >= sizeof(Node));        /* need the tag, at least */ \
         _result = (Node *) palloc0fast(size); \
         _result->type = (tag); \
         _result; \
     })
     #else
     /*
      *  There is no way to dereference the palloc'ed pointer to assign the
      *  tag, and also return the pointer itself, so we need a holder variable.
      *  Fortunately, this macro isn't recursive so we just define
      *  a global variable for this purpose.
      */
     extern PGDLLIMPORT Node *newNodeMacroHolder;
     
     #define newNode(size, tag) \
     ( \
         AssertMacro((size) >= sizeof(Node)),        /* need the tag, at least */ \
         newNodeMacroHolder = (Node *) palloc0fast(size), \
         newNodeMacroHolder->type = (tag), \
         newNodeMacroHolder \
     )
     #endif                          /* __GNUC__ */
    
     /*------------------ AllocSetContextCreate --------------------*/
    
     #define AllocSetContextCreate(parent, name, allocparams) \
         AllocSetContextCreateExtended(parent, name, allocparams)
     #endif
    
     /*
      * AllocSetContextCreateExtended
      *      Create a new AllocSet context.
      *
      * parent: parent context, or NULL if top-level context
      * name: name of context (must be statically allocated)
      * minContextSize: minimum context size
      * initBlockSize: initial allocation block size
      * maxBlockSize: maximum allocation block size
      *
      * Most callers should abstract the context size parameters using a macro
      * such as ALLOCSET_DEFAULT_SIZES.  (This is now *required* when going
      * through the AllocSetContextCreate macro.)
      */
     MemoryContext
     AllocSetContextCreateExtended(MemoryContext parent,//上一级的内存上下文
                                   const char *name,//MemoryContext名称
                                   Size minContextSize,//最小大小
                                   Size initBlockSize,//初始化大小
                                   Size maxBlockSize)//最大大小
     {
         int         freeListIndex;//空闲列表索引
         Size        firstBlockSize;//第一个Block的Size
         AllocSet    set;//AllocSetContext指针
         AllocBlock  block;//Context中的Block
     
         /* Assert we padded AllocChunkData properly */
         StaticAssertStmt(ALLOC_CHUNKHDRSZ == MAXALIGN(ALLOC_CHUNKHDRSZ),
                          "sizeof(AllocChunkData) is not maxaligned");//对齐
         StaticAssertStmt(offsetof(AllocChunkData, aset) + sizeof(MemoryContext) ==
                          ALLOC_CHUNKHDRSZ,
                          "padding calculation in AllocChunkData is wrong");//对齐
     
         /*
          * First, validate allocation parameters.  Once these were regular runtime
          * test and elog's, but in practice Asserts seem sufficient because nobody
          * varies their parameters at runtime.  We somewhat arbitrarily enforce a
          * minimum 1K block size.
          */
         //验证参数
         Assert(initBlockSize == MAXALIGN(initBlockSize) &&
                initBlockSize >= 1024);
         Assert(maxBlockSize == MAXALIGN(maxBlockSize) &&
                maxBlockSize >= initBlockSize &&
                AllocHugeSizeIsValid(maxBlockSize)); /* must be safe to double */
         Assert(minContextSize == 0 ||
                (minContextSize == MAXALIGN(minContextSize) &&
                 minContextSize >= 1024 &&
                 minContextSize <= maxBlockSize));
     
         /*
          * Check whether the parameters match either available freelist.  We do
          * not need to demand a match of maxBlockSize.
          */
         if (minContextSize == ALLOCSET_DEFAULT_MINSIZE &&
             initBlockSize == ALLOCSET_DEFAULT_INITSIZE)
             freeListIndex = 0;
         else if (minContextSize == ALLOCSET_SMALL_MINSIZE &&
                  initBlockSize == ALLOCSET_SMALL_INITSIZE)
             freeListIndex = 1;
         else
             freeListIndex = -1;//默认为-1
     
         /*
          * If a suitable freelist entry exists, just recycle that context.
          */
         if (freeListIndex >= 0)//SMALL/DEFAULT值
         {
             AllocSetFreeList *freelist = &context_freelists[freeListIndex];//使用预定义的freelist
     
             if (freelist->first_free != NULL)
             {
                 /* Remove entry from freelist */
                 set = freelist->first_free;//使用第一个空闲的AllocSetContext
                 freelist->first_free = (AllocSet) set->header.nextchild;//指针指向下一个空闲的AllocSetContext
                 freelist->num_free--;//free计数器减一
     
                 /* Update its maxBlockSize; everything else should be OK */
                 set->maxBlockSize = maxBlockSize;//更新AllocSetContext的相关信息
     
                 /* Reinitialize its header, installing correct name and parent */
                 MemoryContextCreate((MemoryContext) set,
                                     T_AllocSetContext,
                                     &AllocSetMethods,
                                     parent,
                                     name);//创建MemoryContext
     
                 return (MemoryContext) set;//返回
             }
         }
    
         //freeListIndex = -1，定制化自己的MemoryContext
         /* Determine size of initial block */
         firstBlockSize = MAXALIGN(sizeof(AllocSetContext)) +
             ALLOC_BLOCKHDRSZ + ALLOC_CHUNKHDRSZ;//申请内存：AllocSetContext大小+Block头部大小+Chunk头部大小
         if (minContextSize != 0)
             firstBlockSize = Max(firstBlockSize, minContextSize);
         else
             firstBlockSize = Max(firstBlockSize, initBlockSize);
     
         /*
          * Allocate the initial block.  Unlike other aset.c blocks, it starts with
          * the context header and its block header follows that.
          */
         set = (AllocSet) malloc(firstBlockSize);//分配内存
         if (set == NULL)//OOM？
         {
             if (TopMemoryContext)
                 MemoryContextStats(TopMemoryContext);
             ereport(ERROR,
                     (errcode(ERRCODE_OUT_OF_MEMORY),
                      errmsg("out of memory"),
                      errdetail("Failed while creating memory context \"%s\".",
                                name)));
         }
     
         /*
          * Avoid writing code that can fail between here and MemoryContextCreate;
          * we'd leak the header/initial block if we ereport in this stretch.
          */
     
         /* Fill in the initial block's block header */
         //获取Block头部指针，开始填充Block头部信息
         block = (AllocBlock) (((char *) set) + MAXALIGN(sizeof(AllocSetContext)));
         block->aset = set;
         block->freeptr = ((char *) block) + ALLOC_BLOCKHDRSZ;
         block->endptr = ((char *) set) + firstBlockSize;
         block->prev = NULL;
         block->next = NULL;
     
         /* Mark unallocated space NOACCESS; leave the block header alone. */
         VALGRIND_MAKE_MEM_NOACCESS(block->freeptr, block->endptr - block->freeptr);
     
         /* Remember block as part of block list */
         set->blocks = block;
         /* Mark block as not to be released at reset time */
         set->keeper = block;
     
         /* Finish filling in aset-specific parts of the context header */
         MemSetAligned(set->freelist, 0, sizeof(set->freelist));//对齐
     
         set->initBlockSize = initBlockSize;//初始化set的各项属性
         set->maxBlockSize = maxBlockSize;
         set->nextBlockSize = initBlockSize;
         set->freeListIndex = freeListIndex;
     
         /*
          * Compute the allocation chunk size limit for this context.  It can't be
          * more than ALLOC_CHUNK_LIMIT because of the fixed number of freelists.
          * If maxBlockSize is small then requests exceeding the maxBlockSize, or
          * even a significant fraction of it, should be treated as large chunks
          * too.  For the typical case of maxBlockSize a power of 2, the chunk size
          * limit will be at most 1/8th maxBlockSize, so that given a stream of
          * requests that are all the maximum chunk size we will waste at most
          * 1/8th of the allocated space.
          *
          * We have to have allocChunkLimit a power of two, because the requested
          * and actually-allocated sizes of any chunk must be on the same side of
          * the limit, else we get confused about whether the chunk is "big".
          *
          * Also, allocChunkLimit must not exceed ALLOCSET_SEPARATE_THRESHOLD.
          */
         StaticAssertStmt(ALLOC_CHUNK_LIMIT == ALLOCSET_SEPARATE_THRESHOLD,//ALLOCSET_SEPARATE_THRESHOLD,8192
                          "ALLOC_CHUNK_LIMIT != ALLOCSET_SEPARATE_THRESHOLD");
     
         set->allocChunkLimit = ALLOC_CHUNK_LIMIT;
         while ((Size) (set->allocChunkLimit + ALLOC_CHUNKHDRSZ) >
                (Size) ((maxBlockSize - ALLOC_BLOCKHDRSZ) / ALLOC_CHUNK_FRACTION))//ALLOC_CHUNK_FRACTION-4
             set->allocChunkLimit >>= 1;//计算ChunkLimit上限，每次/2
     
         /* Finally, do the type-independent part of context creation */
         MemoryContextCreate((MemoryContext) set,
                             T_AllocSetContext,
                             &AllocSetMethods,
                             parent,
                             name);//创建MemoryContext
     
         return (MemoryContext) set;
     }
    
     /*
      * MemoryContextCreate
      *      Context-type-independent part of context creation.
      *
      * This is only intended to be called by context-type-specific
      * context creation routines, not by the unwashed masses.
      *
      * The memory context creation procedure goes like this:
      *  1.  Context-type-specific routine makes some initial space allocation,
      *      including enough space for the context header.  If it fails,
      *      it can ereport() with no damage done.
      *  2.  Context-type-specific routine sets up all type-specific fields of
      *      the header (those beyond MemoryContextData proper), as well as any
      *      other management fields it needs to have a fully valid context.
      *      Usually, failure in this step is impossible, but if it's possible
      *      the initial space allocation should be freed before ereport'ing.
      *  3.  Context-type-specific routine calls MemoryContextCreate() to fill in
      *      the generic header fields and link the context into the context tree.
      *  4.  We return to the context-type-specific routine, which finishes
      *      up type-specific initialization.  This routine can now do things
      *      that might fail (like allocate more memory), so long as it's
      *      sure the node is left in a state that delete will handle.
      *
      * node: the as-yet-uninitialized common part of the context header node.
      * tag: NodeTag code identifying the memory context type.
      * methods: context-type-specific methods (usually statically allocated).
      * parent: parent context, or NULL if this will be a top-level context.
      * name: name of context (must be statically allocated).
      *
      * Context routines generally assume that MemoryContextCreate can't fail,
      * so this can contain Assert but not elog/ereport.
      */
     void
     MemoryContextCreate(MemoryContext node,
                         NodeTag tag,
                         const MemoryContextMethods *methods,
                         MemoryContext parent,
                         const char *name)
     {
         /* Creating new memory contexts is not allowed in a critical section */
         Assert(CritSectionCount == 0);
     
         /* Initialize all standard fields of memory context header */
         node->type = tag;
         node->isReset = true;
         node->methods = methods;
         node->parent = parent;
         node->firstchild = NULL;
         node->prevchild = NULL;
         node->name = name;
         node->ident = NULL;
         node->reset_cbs = NULL;
     
         /* OK to link node into context tree */
         if (parent)
         {
             node->nextchild = parent->firstchild;
             if (parent->firstchild != NULL)
                 parent->firstchild->prevchild = node;
             parent->firstchild = node;
             /* inherit allowInCritSection flag from parent */
             node->allowInCritSection = parent->allowInCritSection;
         }
         else
         {
             node->nextchild = NULL;
             node->allowInCritSection = false;
         }
     
         VALGRIND_CREATE_MEMPOOL(node, 0, false);
     }
     
    

_3、InitPlan_

    
    
     /* ----------------------------------------------------------------      *      InitPlan
      *
      *      Initializes the query plan: open files, allocate storage
      *      and start up the rule manager
      * ----------------------------------------------------------------      */
     static void
     InitPlan(QueryDesc *queryDesc, int eflags)
     {
         CmdType     operation = queryDesc->operation;//操作类型
         PlannedStmt *plannedstmt = queryDesc->plannedstmt;//已规划的Statement
         Plan       *plan = plannedstmt->planTree;//执行计划
         List       *rangeTable = plannedstmt->rtable;//本次执行涉及的Table
         EState     *estate = queryDesc->estate;//执行状态
         PlanState  *planstate;//计划状态
         TupleDesc   tupType;//Tuple信息
         ListCell   *l;//
         int         i;//
     
         /*
          * Do permissions checks
          */
         ExecCheckRTPerms(rangeTable, true);//权限检查
     
         /*
          * initialize the node's execution state
          */
         estate->es_range_table = rangeTable;
         estate->es_plannedstmt = plannedstmt;
     
         /*
          * initialize result relation stuff, and open/lock the result rels.
          *
          * We must do this before initializing the plan tree, else we might try to
          * do a lock upgrade if a result rel is also a source rel.
          */
         //初始化结果Relation
         if (plannedstmt->resultRelations)
         {
             List       *resultRelations = plannedstmt->resultRelations;
             int         numResultRelations = list_length(resultRelations);
             ResultRelInfo *resultRelInfos;
             ResultRelInfo *resultRelInfo;
     
             resultRelInfos = (ResultRelInfo *)
                 palloc(numResultRelations * sizeof(ResultRelInfo));
             resultRelInfo = resultRelInfos;
             foreach(l, resultRelations)
             {
                 Index       resultRelationIndex = lfirst_int(l);
                 Oid         resultRelationOid;
                 Relation    resultRelation;
     
                 resultRelationOid = getrelid(resultRelationIndex, rangeTable);
                 resultRelation = heap_open(resultRelationOid, RowExclusiveLock);
     
                 InitResultRelInfo(resultRelInfo,
                                   resultRelation,
                                   resultRelationIndex,
                                   NULL,
                                   estate->es_instrument);
                 resultRelInfo++;
             }
             estate->es_result_relations = resultRelInfos;
             estate->es_num_result_relations = numResultRelations;
             /* es_result_relation_info is NULL except when within ModifyTable */
             estate->es_result_relation_info = NULL;
     
             /*
              * In the partitioned result relation case, lock the non-leaf result
              * relations too.  A subset of these are the roots of respective
              * partitioned tables, for which we also allocate ResultRelInfos.
              */
             estate->es_root_result_relations = NULL;
             estate->es_num_root_result_relations = 0;
             if (plannedstmt->nonleafResultRelations)
             {
                 int         num_roots = list_length(plannedstmt->rootResultRelations);
     
                 /*
                  * Firstly, build ResultRelInfos for all the partitioned table
                  * roots, because we will need them to fire the statement-level
                  * triggers, if any.
                  */
                 resultRelInfos = (ResultRelInfo *)
                     palloc(num_roots * sizeof(ResultRelInfo));
                 resultRelInfo = resultRelInfos;
                 foreach(l, plannedstmt->rootResultRelations)
                 {
                     Index       resultRelIndex = lfirst_int(l);
                     Oid         resultRelOid;
                     Relation    resultRelDesc;
     
                     resultRelOid = getrelid(resultRelIndex, rangeTable);
                     resultRelDesc = heap_open(resultRelOid, RowExclusiveLock);
                     InitResultRelInfo(resultRelInfo,
                                       resultRelDesc,
                                       lfirst_int(l),
                                       NULL,
                                       estate->es_instrument);
                     resultRelInfo++;
                 }
     
                 estate->es_root_result_relations = resultRelInfos;
                 estate->es_num_root_result_relations = num_roots;
     
                 /* Simply lock the rest of them. */
                 foreach(l, plannedstmt->nonleafResultRelations)
                 {
                     Index       resultRelIndex = lfirst_int(l);
     
                     /* We locked the roots above. */
                     if (!list_member_int(plannedstmt->rootResultRelations,
                                          resultRelIndex))
                         LockRelationOid(getrelid(resultRelIndex, rangeTable),
                                         RowExclusiveLock);
                 }
             }
         }
         else
         {
             /*
              * if no result relation, then set state appropriately
              */
             estate->es_result_relations = NULL;
             estate->es_num_result_relations = 0;
             estate->es_result_relation_info = NULL;
             estate->es_root_result_relations = NULL;
             estate->es_num_root_result_relations = 0;
         }
     
         /*
          * Similarly, we have to lock relations selected FOR [KEY] UPDATE/SHARE
          * before we initialize the plan tree, else we'd be risking lock upgrades.
          * While we are at it, build the ExecRowMark list.  Any partitioned child
          * tables are ignored here (because isParent=true) and will be locked by
          * the first Append or MergeAppend node that references them.  (Note that
          * the RowMarks corresponding to partitioned child tables are present in
          * the same list as the rest, i.e., plannedstmt->rowMarks.)
          */
         estate->es_rowMarks = NIL;
         foreach(l, plannedstmt->rowMarks)
         {
             PlanRowMark *rc = (PlanRowMark *) lfirst(l);
             Oid         relid;
             Relation    relation;
             ExecRowMark *erm;
     
             /* ignore "parent" rowmarks; they are irrelevant at runtime */
             if (rc->isParent)
                 continue;
     
             /* get relation's OID (will produce InvalidOid if subquery) */
             relid = getrelid(rc->rti, rangeTable);
     
             /*
              * If you change the conditions under which rel locks are acquired
              * here, be sure to adjust ExecOpenScanRelation to match.
              */
             switch (rc->markType)
             {
                 case ROW_MARK_EXCLUSIVE:
                 case ROW_MARK_NOKEYEXCLUSIVE:
                 case ROW_MARK_SHARE:
                 case ROW_MARK_KEYSHARE:
                     relation = heap_open(relid, RowShareLock);
                     break;
                 case ROW_MARK_REFERENCE:
                     relation = heap_open(relid, AccessShareLock);
                     break;
                 case ROW_MARK_COPY:
                     /* no physical table access is required */
                     relation = NULL;
                     break;
                 default:
                     elog(ERROR, "unrecognized markType: %d", rc->markType);
                     relation = NULL;    /* keep compiler quiet */
                     break;
             }
     
             /* Check that relation is a legal target for marking */
             if (relation)
                 CheckValidRowMarkRel(relation, rc->markType);
     
             erm = (ExecRowMark *) palloc(sizeof(ExecRowMark));
             erm->relation = relation;
             erm->relid = relid;
             erm->rti = rc->rti;
             erm->prti = rc->prti;
             erm->rowmarkId = rc->rowmarkId;
             erm->markType = rc->markType;
             erm->strength = rc->strength;
             erm->waitPolicy = rc->waitPolicy;
             erm->ermActive = false;
             ItemPointerSetInvalid(&(erm->curCtid));
             erm->ermExtra = NULL;
             estate->es_rowMarks = lappend(estate->es_rowMarks, erm);
         }
     
         /*
          * Initialize the executor's tuple table to empty.
          */
         estate->es_tupleTable = NIL;
         estate->es_trig_tuple_slot = NULL;
         estate->es_trig_oldtup_slot = NULL;
         estate->es_trig_newtup_slot = NULL;
     
         /* mark EvalPlanQual not active */
         estate->es_epqTuple = NULL;
         estate->es_epqTupleSet = NULL;
         estate->es_epqScanDone = NULL;
     
         /*
          * Initialize private state information for each SubPlan.  We must do this
          * before running ExecInitNode on the main query tree, since
          * ExecInitSubPlan expects to be able to find these entries.
          */
         Assert(estate->es_subplanstates == NIL);
         i = 1;                      /* subplan indices count from 1 */
         //初始化子Plan
         foreach(l, plannedstmt->subplans)
         {
             Plan       *subplan = (Plan *) lfirst(l);
             PlanState  *subplanstate;
             int         sp_eflags;
     
             /*
              * A subplan will never need to do BACKWARD scan nor MARK/RESTORE. If
              * it is a parameterless subplan (not initplan), we suggest that it be
              * prepared to handle REWIND efficiently; otherwise there is no need.
              */
             sp_eflags = eflags
                 & (EXEC_FLAG_EXPLAIN_ONLY | EXEC_FLAG_WITH_NO_DATA);
             if (bms_is_member(i, plannedstmt->rewindPlanIDs))
                 sp_eflags |= EXEC_FLAG_REWIND;
     
             subplanstate = ExecInitNode(subplan, estate, sp_eflags);
     
             estate->es_subplanstates = lappend(estate->es_subplanstates,
                                                subplanstate);
     
             i++;
         }
     
         /*
          * Initialize the private state information for all the nodes in the query
          * tree.  This opens files, allocates storage and leaves us ready to start
          * processing tuples.
          */
         planstate = ExecInitNode(plan, estate, eflags);
     
         /*
          * Get the tuple descriptor describing the type of tuples to return.
          */
         tupType = ExecGetResultType(planstate);
     
         /*
          * Initialize the junk filter if needed.  SELECT queries need a filter if
          * there are any junk attrs in the top-level tlist.
          */
         if (operation == CMD_SELECT)
         {
             bool        junk_filter_needed = false;
             ListCell   *tlist;
     
             foreach(tlist, plan->targetlist)
             {
                 TargetEntry *tle = (TargetEntry *) lfirst(tlist);
     
                 if (tle->resjunk)
                 {
                     junk_filter_needed = true;
                     break;
                 }
             }
     
             if (junk_filter_needed)
             {
                 JunkFilter *j;
     
                 j = ExecInitJunkFilter(planstate->plan->targetlist,
                                        tupType->tdhasoid,
                                        ExecInitExtraTupleSlot(estate, NULL));
                 estate->es_junkFilter = j;
     
                 /* Want to return the cleaned tuple type */
                 tupType = j->jf_cleanTupType;
             }
         }
     
         queryDesc->tupDesc = tupType;
         queryDesc->planstate = planstate;
     }
    
    /*
      * ExecCheckRTPerms
      *      Check access permissions for all relations listed in a range table.
      *
      * Returns true if permissions are adequate.  Otherwise, throws an appropriate
      * error if ereport_on_violation is true, or simply returns false otherwise.
      *
      * Note that this does NOT address row level security policies (aka: RLS).  If
      * rows will be returned to the user as a result of this permission check
      * passing, then RLS also needs to be consulted (and check_enable_rls()).
      *
      * See rewrite/rowsecurity.c.
      */
     bool
     ExecCheckRTPerms(List *rangeTable, bool ereport_on_violation)
     {
         ListCell   *l;
         bool        result = true;
     
         foreach(l, rangeTable)
         {
             RangeTblEntry *rte = (RangeTblEntry *) lfirst(l);
     
             result = ExecCheckRTEPerms(rte);//基于ACL Mode的权限检查
             if (!result)
             {
                 Assert(rte->rtekind == RTE_RELATION);
                 if (ereport_on_violation)
                     aclcheck_error(ACLCHECK_NO_PRIV, get_relkind_objtype(get_rel_relkind(rte->relid)),
                                    get_rel_name(rte->relid));
                 return false;
             }
         }
     
         if (ExecutorCheckPerms_hook)
             result = (*ExecutorCheckPerms_hook) (rangeTable,
                                                  ereport_on_violation);
         return result;
     }
     
     /* ------------------------------------------------------------------------      *      ExecInitNode
      *
      *      Recursively initializes all the nodes in the plan tree rooted
      *      at 'node'.
      *
      *      Inputs:
      *        'node' is the current node of the plan produced by the query planner
      *        'estate' is the shared execution state for the plan tree
      *        'eflags' is a bitwise OR of flag bits described in executor.h
      *
      *      Returns a PlanState node corresponding to the given Plan node.
      * ------------------------------------------------------------------------      */
     //初始化节点，返回Plan状态
     PlanState *
     ExecInitNode(Plan *node, EState *estate, int eflags)
     {
         PlanState  *result;
         List       *subps;
         ListCell   *l;
     
         /*
          * do nothing when we get to the end of a leaf on tree.
          */
         if (node == NULL)
             return NULL;
     
         /*
          * Make sure there's enough stack available. Need to check here, in
          * addition to ExecProcNode() (via ExecProcNodeFirst()), to ensure the
          * stack isn't overrun while initializing the node tree.
          */
         check_stack_depth();
     
         switch (nodeTag(node))
         {
                 /*
                  * control nodes
                  */
             case T_Result:
                 result = (PlanState *) ExecInitResult((Result *) node,
                                                       estate, eflags);
                 break;
     
             case T_ProjectSet:
                 result = (PlanState *) ExecInitProjectSet((ProjectSet *) node,
                                                           estate, eflags);
                 break;
     
             case T_ModifyTable://插入数据
                 result = (PlanState *) ExecInitModifyTable((ModifyTable *) node,
                                                            estate, eflags);
                 break;
     
             case T_Append:
                 result = (PlanState *) ExecInitAppend((Append *) node,
                                                       estate, eflags);
                 break;
     
             case T_MergeAppend:
                 result = (PlanState *) ExecInitMergeAppend((MergeAppend *) node,
                                                            estate, eflags);
                 break;
     
             case T_RecursiveUnion:
                 result = (PlanState *) ExecInitRecursiveUnion((RecursiveUnion *) node,
                                                               estate, eflags);
                 break;
     
             case T_BitmapAnd:
                 result = (PlanState *) ExecInitBitmapAnd((BitmapAnd *) node,
                                                          estate, eflags);
                 break;
     
             case T_BitmapOr:
                 result = (PlanState *) ExecInitBitmapOr((BitmapOr *) node,
                                                         estate, eflags);
                 break;
     
                 /*
                  * scan nodes
                  */
             case T_SeqScan:
                 result = (PlanState *) ExecInitSeqScan((SeqScan *) node,
                                                        estate, eflags);
                 break;
     
             case T_SampleScan:
                 result = (PlanState *) ExecInitSampleScan((SampleScan *) node,
                                                           estate, eflags);
                 break;
     
             case T_IndexScan:
                 result = (PlanState *) ExecInitIndexScan((IndexScan *) node,
                                                          estate, eflags);
                 break;
     
             case T_IndexOnlyScan:
                 result = (PlanState *) ExecInitIndexOnlyScan((IndexOnlyScan *) node,
                                                              estate, eflags);
                 break;
     
             case T_BitmapIndexScan:
                 result = (PlanState *) ExecInitBitmapIndexScan((BitmapIndexScan *) node,
                                                                estate, eflags);
                 break;
     
             case T_BitmapHeapScan:
                 result = (PlanState *) ExecInitBitmapHeapScan((BitmapHeapScan *) node,
                                                               estate, eflags);
                 break;
     
             case T_TidScan:
                 result = (PlanState *) ExecInitTidScan((TidScan *) node,
                                                        estate, eflags);
                 break;
     
             case T_SubqueryScan:
                 result = (PlanState *) ExecInitSubqueryScan((SubqueryScan *) node,
                                                             estate, eflags);
                 break;
     
             case T_FunctionScan:
                 result = (PlanState *) ExecInitFunctionScan((FunctionScan *) node,
                                                             estate, eflags);
                 break;
     
             case T_TableFuncScan:
                 result = (PlanState *) ExecInitTableFuncScan((TableFuncScan *) node,
                                                              estate, eflags);
                 break;
     
             case T_ValuesScan:
                 result = (PlanState *) ExecInitValuesScan((ValuesScan *) node,
                                                           estate, eflags);
                 break;
     
             case T_CteScan:
                 result = (PlanState *) ExecInitCteScan((CteScan *) node,
                                                        estate, eflags);
                 break;
     
             case T_NamedTuplestoreScan:
                 result = (PlanState *) ExecInitNamedTuplestoreScan((NamedTuplestoreScan *) node,
                                                                    estate, eflags);
                 break;
     
             case T_WorkTableScan:
                 result = (PlanState *) ExecInitWorkTableScan((WorkTableScan *) node,
                                                              estate, eflags);
                 break;
     
             case T_ForeignScan:
                 result = (PlanState *) ExecInitForeignScan((ForeignScan *) node,
                                                            estate, eflags);
                 break;
     
             case T_CustomScan:
                 result = (PlanState *) ExecInitCustomScan((CustomScan *) node,
                                                           estate, eflags);
                 break;
     
                 /*
                  * join nodes
                  */
             case T_NestLoop:
                 result = (PlanState *) ExecInitNestLoop((NestLoop *) node,
                                                         estate, eflags);
                 break;
     
             case T_MergeJoin:
                 result = (PlanState *) ExecInitMergeJoin((MergeJoin *) node,
                                                          estate, eflags);
                 break;
     
             case T_HashJoin:
                 result = (PlanState *) ExecInitHashJoin((HashJoin *) node,
                                                         estate, eflags);
                 break;
     
                 /*
                  * materialization nodes
                  */
             case T_Material:
                 result = (PlanState *) ExecInitMaterial((Material *) node,
                                                         estate, eflags);
                 break;
     
             case T_Sort:
                 result = (PlanState *) ExecInitSort((Sort *) node,
                                                     estate, eflags);
                 break;
     
             case T_Group:
                 result = (PlanState *) ExecInitGroup((Group *) node,
                                                      estate, eflags);
                 break;
     
             case T_Agg:
                 result = (PlanState *) ExecInitAgg((Agg *) node,
                                                    estate, eflags);
                 break;
     
             case T_WindowAgg:
                 result = (PlanState *) ExecInitWindowAgg((WindowAgg *) node,
                                                          estate, eflags);
                 break;
     
             case T_Unique:
                 result = (PlanState *) ExecInitUnique((Unique *) node,
                                                       estate, eflags);
                 break;
     
             case T_Gather:
                 result = (PlanState *) ExecInitGather((Gather *) node,
                                                       estate, eflags);
                 break;
     
             case T_GatherMerge:
                 result = (PlanState *) ExecInitGatherMerge((GatherMerge *) node,
                                                            estate, eflags);
                 break;
     
             case T_Hash:
                 result = (PlanState *) ExecInitHash((Hash *) node,
                                                     estate, eflags);
                 break;
     
             case T_SetOp:
                 result = (PlanState *) ExecInitSetOp((SetOp *) node,
                                                      estate, eflags);
                 break;
     
             case T_LockRows:
                 result = (PlanState *) ExecInitLockRows((LockRows *) node,
                                                         estate, eflags);
                 break;
     
             case T_Limit:
                 result = (PlanState *) ExecInitLimit((Limit *) node,
                                                      estate, eflags);
                 break;
     
             default:
                 elog(ERROR, "unrecognized node type: %d", (int) nodeTag(node));
                 result = NULL;      /* keep compiler quiet */
                 break;
         }
     
         ExecSetExecProcNode(result, result->ExecProcNode);
     
         /*
          * Initialize any initPlans present in this node.  The planner put them in
          * a separate list for us.
          */
         subps = NIL;
         foreach(l, node->initPlan)
         {
             SubPlan    *subplan = (SubPlan *) lfirst(l);
             SubPlanState *sstate;
     
             Assert(IsA(subplan, SubPlan));
             sstate = ExecInitSubPlan(subplan, result);
             subps = lappend(subps, sstate);
         }
         result->initPlan = subps;
     
         /* Set up instrumentation for this node if requested */
         if (estate->es_instrument)
             result->instrument = InstrAlloc(1, estate->es_instrument);
     
         return result;
     }
     
    /* ----------------------------------------------------------------      *      ExecInitModifyTable
      * ----------------------------------------------------------------      */
     ModifyTableState *
     ExecInitModifyTable(ModifyTable *node, EState *estate, int eflags)
     {
         ModifyTableState *mtstate;//返回结果
         CmdType     operation = node->operation;//操作类型
         int         nplans = list_length(node->plans);//节点中的plan个数
         ResultRelInfo *saved_resultRelInfo;
         ResultRelInfo *resultRelInfo;//结果Relation信息
         Plan       *subplan;//子Plan
         ListCell   *l;//临时变量
         int         i;
         Relation    rel;
         bool        update_tuple_routing_needed = node->partColsUpdated;
     
         /* check for unsupported flags */
         Assert(!(eflags & (EXEC_FLAG_BACKWARD | EXEC_FLAG_MARK)));
     
         /*
          * create state structure
          */
         mtstate = makeNode(ModifyTableState);//构建节点
         mtstate->ps.plan = (Plan *) node;//设置Plan
         mtstate->ps.state = estate;//设置执行状态
         mtstate->ps.ExecProcNode = ExecModifyTable;//设置处理函数为ExecModifyTable
     
         mtstate->operation = operation;//操作类型
         mtstate->canSetTag = node->canSetTag;
         mtstate->mt_done = false;
     
         mtstate->mt_plans = (PlanState **) palloc0(sizeof(PlanState *) * nplans);//分配内存
         mtstate->resultRelInfo = estate->es_result_relations + node->resultRelIndex;//结果Relation信息
     
         /* If modifying a partitioned table, initialize the root table info */
         if (node->rootResultRelIndex >= 0)
             mtstate->rootResultRelInfo = estate->es_root_result_relations +
                 node->rootResultRelIndex;
     
         mtstate->mt_arowmarks = (List **) palloc0(sizeof(List *) * nplans);
         mtstate->mt_nplans = nplans;
     
         /* set up epqstate with dummy subplan data for the moment */
         EvalPlanQualInit(&mtstate->mt_epqstate, estate, NULL, NIL, node->epqParam);
         mtstate->fireBSTriggers = true;
     
         /*
          * call ExecInitNode on each of the plans to be executed and save the
          * results into the array "mt_plans".  This is also a convenient place to
          * verify that the proposed target relations are valid and open their
          * indexes for insertion of new index entries.  Note we *must* set
          * estate->es_result_relation_info correctly while we initialize each
          * sub-plan; ExecContextForcesOids depends on that!
          */
         saved_resultRelInfo = estate->es_result_relation_info;
     
         resultRelInfo = mtstate->resultRelInfo;
         i = 0;
         //初始化每个子Plan，保存在mt_plans数组中
         foreach(l, node->plans)
         {
             subplan = (Plan *) lfirst(l);
     
             /* Initialize the usesFdwDirectModify flag */
             resultRelInfo->ri_usesFdwDirectModify = bms_is_member(i,
                                                                   node->fdwDirectModifyPlans);
     
             /*
              * Verify result relation is a valid target for the current operation
              */
             CheckValidResultRel(resultRelInfo, operation);
     
             /*
              * If there are indices on the result relation, open them and save
              * descriptors in the result relation info, so that we can add new
              * index entries for the tuples we add/update.  We need not do this
              * for a DELETE, however, since deletion doesn't affect indexes. Also,
              * inside an EvalPlanQual operation, the indexes might be open
              * already, since we share the resultrel state with the original
              * query.
              */
             if (resultRelInfo->ri_RelationDesc->rd_rel->relhasindex &&
                 operation != CMD_DELETE &&
                 resultRelInfo->ri_IndexRelationDescs == NULL)
                 ExecOpenIndices(resultRelInfo,
                                 node->onConflictAction != ONCONFLICT_NONE);//初始化Index
     
             /*
              * If this is an UPDATE and a BEFORE UPDATE trigger is present, the
              * trigger itself might modify the partition-key values. So arrange
              * for tuple routing.
              */
             if (resultRelInfo->ri_TrigDesc &&
                 resultRelInfo->ri_TrigDesc->trig_update_before_row &&
                 operation == CMD_UPDATE)
                 update_tuple_routing_needed = true;
     
             /* Now init the plan for this result rel */
             estate->es_result_relation_info = resultRelInfo;
             mtstate->mt_plans[i] = ExecInitNode(subplan, estate, eflags);//初始化子节点
     
             /* Also let FDWs init themselves for foreign-table result rels */
             if (!resultRelInfo->ri_usesFdwDirectModify &&
                 resultRelInfo->ri_FdwRoutine != NULL &&
                 resultRelInfo->ri_FdwRoutine->BeginForeignModify != NULL)
             {
                 List       *fdw_private = (List *) list_nth(node->fdwPrivLists, i);
     
                 resultRelInfo->ri_FdwRoutine->BeginForeignModify(mtstate,
                                                                  resultRelInfo,
                                                                  fdw_private,
                                                                  i,
                                                                  eflags);
             }
     
             resultRelInfo++;
             i++;
         }
     
         estate->es_result_relation_info = saved_resultRelInfo;
     
         /* Get the target relation */
         rel = (getTargetResultRelInfo(mtstate))->ri_RelationDesc;
     
         /*
          * If it's not a partitioned table after all, UPDATE tuple routing should
          * not be attempted.
          */
         if (rel->rd_rel->relkind != RELKIND_PARTITIONED_TABLE)
             update_tuple_routing_needed = false;
     
         /*
          * Build state for tuple routing if it's an INSERT or if it's an UPDATE of
          * partition key.
          */
         if (rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE &&
             (operation == CMD_INSERT || update_tuple_routing_needed))
             mtstate->mt_partition_tuple_routing =
                 ExecSetupPartitionTupleRouting(mtstate, rel);
     
         /*
          * Build state for collecting transition tuples.  This requires having a
          * valid trigger query context, so skip it in explain-only mode.
          */
         if (!(eflags & EXEC_FLAG_EXPLAIN_ONLY))
             ExecSetupTransitionCaptureState(mtstate, estate);
     
         /*
          * Construct mapping from each of the per-subplan partition attnos to the
          * root attno.  This is required when during update row movement the tuple
          * descriptor of a source partition does not match the root partitioned
          * table descriptor.  In such a case we need to convert tuples to the root
          * tuple descriptor, because the search for destination partition starts
          * from the root.  Skip this setup if it's not a partition key update.
          */
         if (update_tuple_routing_needed)
             ExecSetupChildParentMapForSubplan(mtstate);
     
         /*
          * Initialize any WITH CHECK OPTION constraints if needed.
          */
         resultRelInfo = mtstate->resultRelInfo;
         i = 0;
         //设置Check选项
         foreach(l, node->withCheckOptionLists)
         {
             List       *wcoList = (List *) lfirst(l);
             List       *wcoExprs = NIL;
             ListCell   *ll;
     
             foreach(ll, wcoList)
             {
                 WithCheckOption *wco = (WithCheckOption *) lfirst(ll);
                 ExprState  *wcoExpr = ExecInitQual((List *) wco->qual,
                                                    mtstate->mt_plans[i]);
     
                 wcoExprs = lappend(wcoExprs, wcoExpr);
             }
     
             resultRelInfo->ri_WithCheckOptions = wcoList;
             resultRelInfo->ri_WithCheckOptionExprs = wcoExprs;
             resultRelInfo++;
             i++;
         }
     
         /*
          * Initialize RETURNING projections if needed.
          */
         if (node->returningLists)
         {
             TupleTableSlot *slot;
             ExprContext *econtext;
     
             /*
              * Initialize result tuple slot and assign its rowtype using the first
              * RETURNING list.  We assume the rest will look the same.
              */
             mtstate->ps.plan->targetlist = (List *) linitial(node->returningLists);
     
             /* Set up a slot for the output of the RETURNING projection(s) */
             ExecInitResultTupleSlotTL(estate, &mtstate->ps);
             slot = mtstate->ps.ps_ResultTupleSlot;
     
             /* Need an econtext too */
             if (mtstate->ps.ps_ExprContext == NULL)
                 ExecAssignExprContext(estate, &mtstate->ps);
             econtext = mtstate->ps.ps_ExprContext;
     
             /*
              * Build a projection for each result rel.
              */
             resultRelInfo = mtstate->resultRelInfo;
             foreach(l, node->returningLists)
             {
                 List       *rlist = (List *) lfirst(l);
     
                 resultRelInfo->ri_returningList = rlist;
                 resultRelInfo->ri_projectReturning =
                     ExecBuildProjectionInfo(rlist, econtext, slot, &mtstate->ps,
                                             resultRelInfo->ri_RelationDesc->rd_att);
                 resultRelInfo++;
             }
         }
         else
         {
             /*
              * We still must construct a dummy result tuple type, because InitPlan
              * expects one (maybe should change that?).
              */
             mtstate->ps.plan->targetlist = NIL;
             ExecInitResultTupleSlotTL(estate, &mtstate->ps);
     
             mtstate->ps.ps_ExprContext = NULL;
         }
     
         /* Set the list of arbiter indexes if needed for ON CONFLICT */
         resultRelInfo = mtstate->resultRelInfo;
         if (node->onConflictAction != ONCONFLICT_NONE)
             resultRelInfo->ri_onConflictArbiterIndexes = node->arbiterIndexes;
     
         /*
          * If needed, Initialize target list, projection and qual for ON CONFLICT
          * DO UPDATE.
          */
         if (node->onConflictAction == ONCONFLICT_UPDATE)
         {
             ExprContext *econtext;
             TupleDesc   relationDesc;
             TupleDesc   tupDesc;
     
             /* insert may only have one plan, inheritance is not expanded */
             Assert(nplans == 1);
     
             /* already exists if created by RETURNING processing above */
             if (mtstate->ps.ps_ExprContext == NULL)
                 ExecAssignExprContext(estate, &mtstate->ps);
     
             econtext = mtstate->ps.ps_ExprContext;
             relationDesc = resultRelInfo->ri_RelationDesc->rd_att;
     
             /*
              * Initialize slot for the existing tuple.  If we'll be performing
              * tuple routing, the tuple descriptor to use for this will be
              * determined based on which relation the update is actually applied
              * to, so we don't set its tuple descriptor here.
              */
             mtstate->mt_existing =
                 ExecInitExtraTupleSlot(mtstate->ps.state,
                                        mtstate->mt_partition_tuple_routing ?
                                        NULL : relationDesc);
     
             /* carried forward solely for the benefit of explain */
             mtstate->mt_excludedtlist = node->exclRelTlist;
     
             /* create state for DO UPDATE SET operation */
             resultRelInfo->ri_onConflict = makeNode(OnConflictSetState);
     
             /*
              * Create the tuple slot for the UPDATE SET projection.
              *
              * Just like mt_existing above, we leave it without a tuple descriptor
              * in the case of partitioning tuple routing, so that it can be
              * changed by ExecPrepareTupleRouting.  In that case, we still save
              * the tupdesc in the parent's state: it can be reused by partitions
              * with an identical descriptor to the parent.
              */
             tupDesc = ExecTypeFromTL((List *) node->onConflictSet,
                                      relationDesc->tdhasoid);
             mtstate->mt_conflproj =
                 ExecInitExtraTupleSlot(mtstate->ps.state,
                                        mtstate->mt_partition_tuple_routing ?
                                        NULL : tupDesc);
             resultRelInfo->ri_onConflict->oc_ProjTupdesc = tupDesc;
     
             /* build UPDATE SET projection state */
             resultRelInfo->ri_onConflict->oc_ProjInfo =
                 ExecBuildProjectionInfo(node->onConflictSet, econtext,
                                         mtstate->mt_conflproj, &mtstate->ps,
                                         relationDesc);
     
             /* initialize state to evaluate the WHERE clause, if any */
             if (node->onConflictWhere)
             {
                 ExprState  *qualexpr;
     
                 qualexpr = ExecInitQual((List *) node->onConflictWhere,
                                         &mtstate->ps);
                 resultRelInfo->ri_onConflict->oc_WhereClause = qualexpr;
             }
         }
     
         /*
          * If we have any secondary relations in an UPDATE or DELETE, they need to
          * be treated like non-locked relations in SELECT FOR UPDATE, ie, the
          * EvalPlanQual mechanism needs to be told about them.  Locate the
          * relevant ExecRowMarks.
          */
         foreach(l, node->rowMarks)
         {
             PlanRowMark *rc = lfirst_node(PlanRowMark, l);
             ExecRowMark *erm;
     
             /* ignore "parent" rowmarks; they are irrelevant at runtime */
             if (rc->isParent)
                 continue;
     
             /* find ExecRowMark (same for all subplans) */
             erm = ExecFindRowMark(estate, rc->rti, false);
     
             /* build ExecAuxRowMark for each subplan */
             for (i = 0; i < nplans; i++)
             {
                 ExecAuxRowMark *aerm;
     
                 subplan = mtstate->mt_plans[i]->plan;
                 aerm = ExecBuildAuxRowMark(erm, subplan->targetlist);
                 mtstate->mt_arowmarks[i] = lappend(mtstate->mt_arowmarks[i], aerm);
             }
         }
     
         /* select first subplan */
         mtstate->mt_whichplan = 0;
         subplan = (Plan *) linitial(node->plans);
         EvalPlanQualSetPlan(&mtstate->mt_epqstate, subplan,
                             mtstate->mt_arowmarks[0]);
     
         /*
          * Initialize the junk filter(s) if needed.  INSERT queries need a filter
          * if there are any junk attrs in the tlist.  UPDATE and DELETE always
          * need a filter, since there's always at least one junk attribute present
          * --- no need to look first.  Typically, this will be a 'ctid' or
          * 'wholerow' attribute, but in the case of a foreign data wrapper it
          * might be a set of junk attributes sufficient to identify the remote
          * row.
          *
          * If there are multiple result relations, each one needs its own junk
          * filter.  Note multiple rels are only possible for UPDATE/DELETE, so we
          * can't be fooled by some needing a filter and some not.
          *
          * This section of code is also a convenient place to verify that the
          * output of an INSERT or UPDATE matches the target table(s).
          */
         {
             bool        junk_filter_needed = false;
     
             switch (operation)
             {
                 case CMD_INSERT:
                     foreach(l, subplan->targetlist)
                     {
                         TargetEntry *tle = (TargetEntry *) lfirst(l);
     
                         if (tle->resjunk)
                         {
                             junk_filter_needed = true;
                             break;
                         }
                     }
                     break;
                 case CMD_UPDATE:
                 case CMD_DELETE:
                     junk_filter_needed = true;
                     break;
                 default:
                     elog(ERROR, "unknown operation");
                     break;
             }
     
             if (junk_filter_needed)
             {
                 resultRelInfo = mtstate->resultRelInfo;
                 for (i = 0; i < nplans; i++)
                 {
                     JunkFilter *j;
     
                     subplan = mtstate->mt_plans[i]->plan;
                     if (operation == CMD_INSERT || operation == CMD_UPDATE)
                         ExecCheckPlanOutput(resultRelInfo->ri_RelationDesc,
                                             subplan->targetlist);
     
                     j = ExecInitJunkFilter(subplan->targetlist,
                                            resultRelInfo->ri_RelationDesc->rd_att->tdhasoid,
                                            ExecInitExtraTupleSlot(estate, NULL));
     
                     if (operation == CMD_UPDATE || operation == CMD_DELETE)
                     {
                         /* For UPDATE/DELETE, find the appropriate junk attr now */
                         char        relkind;
     
                         relkind = resultRelInfo->ri_RelationDesc->rd_rel->relkind;
                         if (relkind == RELKIND_RELATION ||
                             relkind == RELKIND_MATVIEW ||
                             relkind == RELKIND_PARTITIONED_TABLE)
                         {
                             j->jf_junkAttNo = ExecFindJunkAttribute(j, "ctid");
                             if (!AttributeNumberIsValid(j->jf_junkAttNo))
                                 elog(ERROR, "could not find junk ctid column");
                         }
                         else if (relkind == RELKIND_FOREIGN_TABLE)
                         {
                             /*
                              * When there is a row-level trigger, there should be
                              * a wholerow attribute.
                              */
                             j->jf_junkAttNo = ExecFindJunkAttribute(j, "wholerow");
                         }
                         else
                         {
                             j->jf_junkAttNo = ExecFindJunkAttribute(j, "wholerow");
                             if (!AttributeNumberIsValid(j->jf_junkAttNo))
                                 elog(ERROR, "could not find junk wholerow column");
                         }
                     }
     
                     resultRelInfo->ri_junkFilter = j;
                     resultRelInfo++;
                 }
             }
             else
             {
                 if (operation == CMD_INSERT)
                     ExecCheckPlanOutput(mtstate->resultRelInfo->ri_RelationDesc,
                                         subplan->targetlist);
             }
         }
     
         /*
          * Set up a tuple table slot for use for trigger output tuples. In a plan
          * containing multiple ModifyTable nodes, all can share one such slot, so
          * we keep it in the estate.
          */
         if (estate->es_trig_tuple_slot == NULL)
             estate->es_trig_tuple_slot = ExecInitExtraTupleSlot(estate, NULL);
     
         /*
          * Lastly, if this is not the primary (canSetTag) ModifyTable node, add it
          * to estate->es_auxmodifytables so that it will be run to completion by
          * ExecPostprocessPlan.  (It'd actually work fine to add the primary
          * ModifyTable node too, but there's no need.)  Note the use of lcons not
          * lappend: we need later-initialized ModifyTable nodes to be shut down
          * before earlier ones.  This ensures that we don't throw away RETURNING
          * rows that need to be seen by a later CTE subplan.
          */
         if (!mtstate->canSetTag)
             estate->es_auxmodifytables = lcons(mtstate,
                                                estate->es_auxmodifytables);
     
         return mtstate;
     }
    
     /* ----------------      *   ModifyTable node -      *      Apply rows produced by subplan(s) to result table(s),
      *      by inserting, updating, or deleting.
      *
      * Note that rowMarks and epqParam are presumed to be valid for all the
      * subplan(s); they can't contain any info that varies across subplans.
      * ----------------      */
     typedef struct ModifyTable
     {
         Plan        plan;
         CmdType     operation;      /* INSERT, UPDATE, or DELETE */
         bool        canSetTag;      /* do we set the command tag/es_processed? */
         Index       nominalRelation;    /* Parent RT index for use of EXPLAIN */
         /* RT indexes of non-leaf tables in a partition tree */
         List       *partitioned_rels;
         bool        partColsUpdated;    /* some part key in hierarchy updated */
         List       *resultRelations;    /* integer list of RT indexes */
         int         resultRelIndex; /* index of first resultRel in plan's list */
         int         rootResultRelIndex; /* index of the partitioned table root */
         List       *plans;          /* plan(s) producing source data */
         List       *withCheckOptionLists;   /* per-target-table WCO lists */
         List       *returningLists; /* per-target-table RETURNING tlists */
         List       *fdwPrivLists;   /* per-target-table FDW private data lists */
         Bitmapset  *fdwDirectModifyPlans;   /* indices of FDW DM plans */
         List       *rowMarks;       /* PlanRowMarks (non-locking only) */
         int         epqParam;       /* ID of Param for EvalPlanQual re-eval */
         OnConflictAction onConflictAction;  /* ON CONFLICT action */
         List       *arbiterIndexes; /* List of ON CONFLICT arbiter index OIDs  */
         List       *onConflictSet;  /* SET for INSERT ON CONFLICT DO UPDATE */
         Node       *onConflictWhere;    /* WHERE for ON CONFLICT UPDATE */
         Index       exclRelRTI;     /* RTI of the EXCLUDED pseudo relation */
         List       *exclRelTlist;   /* tlist of the EXCLUDED pseudo relation */
     } ModifyTable;
    
    

_4、ExecutorStart_

    
    
     /* ----------------------------------------------------------------      *      ExecutorStart
      *
      *      This routine must be called at the beginning of any execution of any
      *      query plan
      *
      * Takes a QueryDesc previously created by CreateQueryDesc (which is separate
      * only because some places use QueryDescs for utility commands).  The tupDesc
      * field of the QueryDesc is filled in to describe the tuples that will be
      * returned, and the internal fields (estate and planstate) are set up.
      *
      * eflags contains flag bits as described in executor.h.
      *
      * NB: the CurrentMemoryContext when this is called will become the parent
      * of the per-query context used for this Executor invocation.
      *
      * We provide a function hook variable that lets loadable plugins
      * get control when ExecutorStart is called.  Such a plugin would
      * normally call standard_ExecutorStart().
      *
      * ----------------------------------------------------------------      */
     void
     ExecutorStart(QueryDesc *queryDesc, int eflags)//eflags见后
     {
         if (ExecutorStart_hook)
             (*ExecutorStart_hook) (queryDesc, eflags);//提供了钩子函数
         else
             standard_ExecutorStart(queryDesc, eflags);//标准函数
     }
     
     void
     standard_ExecutorStart(QueryDesc *queryDesc, int eflags)//标准函数
     {
         EState     *estate;//执行器状态信息
         MemoryContext oldcontext;//原内存上下文
     
         /* sanity checks: queryDesc must not be started already */
         Assert(queryDesc != NULL);
         Assert(queryDesc->estate == NULL);
     
         /*
          * If the transaction is read-only, we need to check if any writes are
          * planned to non-temporary tables.  EXPLAIN is considered read-only.
          *
          * Don't allow writes in parallel mode.  Supporting UPDATE and DELETE
          * would require (a) storing the combocid hash in shared memory, rather
          * than synchronizing it just once at the start of parallelism, and (b) an
          * alternative to heap_update()'s reliance on xmax for mutual exclusion.
          * INSERT may have no such troubles, but we forbid it to simplify the
          * checks.
          *
          * We have lower-level defenses in CommandCounterIncrement and elsewhere
          * against performing unsafe operations in parallel mode, but this gives a
          * more user-friendly error message.
          */
         if ((XactReadOnly || IsInParallelMode()) &&
             !(eflags & EXEC_FLAG_EXPLAIN_ONLY))
             ExecCheckXactReadOnly(queryDesc->plannedstmt);//ReadOnly？
     
         /*
          * Build EState, switch into per-query memory context for startup.
          */
         estate = CreateExecutorState();//创建执行器状态信息
         queryDesc->estate = estate;//赋值
     
         oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);//切换上下文
     
         /*
          * Fill in external parameters, if any, from queryDesc; and allocate
          * workspace for internal parameters
          */
         estate->es_param_list_info = queryDesc->params;//设置参数
     
         if (queryDesc->plannedstmt->paramExecTypes != NIL)//TODO
         {
             int         nParamExec;
     
             nParamExec = list_length(queryDesc->plannedstmt->paramExecTypes);
             estate->es_param_exec_vals = (ParamExecData *)
                 palloc0(nParamExec * sizeof(ParamExecData));
         }
     
         estate->es_sourceText = queryDesc->sourceText;//源SQL语句
     
         /*
          * Fill in the query environment, if any, from queryDesc.
          */
         estate->es_queryEnv = queryDesc->queryEnv;//查询环境
     
         /*
          * If non-read-only query, set the command ID to mark output tuples with
          */
         switch (queryDesc->operation)
         {
             case CMD_SELECT://查询语句 TODO
     
                 /*
                  * SELECT FOR [KEY] UPDATE/SHARE and modifying CTEs need to mark
                  * tuples
                  */
                 if (queryDesc->plannedstmt->rowMarks != NIL ||
                     queryDesc->plannedstmt->hasModifyingCTE)
                     estate->es_output_cid = GetCurrentCommandId(true);
     
                 /*
                  * A SELECT without modifying CTEs can't possibly queue triggers,
                  * so force skip-triggers mode. This is just a marginal efficiency
                  * hack, since AfterTriggerBeginQuery/AfterTriggerEndQuery aren't
                  * all that expensive, but we might as well do it.
                  */
                 if (!queryDesc->plannedstmt->hasModifyingCTE)
                     eflags |= EXEC_FLAG_SKIP_TRIGGERS;
                 break;
     
             case CMD_INSERT://插入语句
             case CMD_DELETE:
             case CMD_UPDATE:
                 estate->es_output_cid = GetCurrentCommandId(true);
                 break;
     
             default:
                 elog(ERROR, "unrecognized operation code: %d",
                      (int) queryDesc->operation);
                 break;
         }
     
         /*
          * Copy other important information into the EState
          */
         estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot);
         estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot);
         estate->es_top_eflags = eflags;
         estate->es_instrument = queryDesc->instrument_options;
         estate->es_jit_flags = queryDesc->plannedstmt->jitFlags;
     
         /*
          * Set up an AFTER-trigger statement context, unless told not to, or
          * unless it's EXPLAIN-only mode (when ExecutorFinish won't be called).
          */
         if (!(eflags & (EXEC_FLAG_SKIP_TRIGGERS | EXEC_FLAG_EXPLAIN_ONLY)))
             AfterTriggerBeginQuery();
         
         /*
          * Initialize the plan state tree
          */
         InitPlan(queryDesc, eflags);//初始化Plan State tree
     
         MemoryContextSwitchTo(oldcontext);
     }
     
     /*
      *  GetCurrentCommandId
      *
      * "used" must be true if the caller intends to use the command ID to mark
      * inserted/updated/deleted tuples.  false means the ID is being fetched
      * for read-only purposes (ie, as a snapshot validity cutoff).  See
      * CommandCounterIncrement() for discussion.
      */
     CommandId
     GetCurrentCommandId(bool used)
     {
         /* this is global to a transaction, not subtransaction-local */
         if (used)
         {
             /*
              * Forbid setting currentCommandIdUsed in a parallel worker, because
              * we have no provision for communicating this back to the master.  We
              * could relax this restriction when currentCommandIdUsed was already
              * true at the start of the parallel operation.
              */
             Assert(!IsParallelWorker());
             currentCommandIdUsed = true;
         }
         return currentCommandId;
     }
     
    /*
     * The "eflags" argument to ExecutorStart and the various ExecInitNode
     * routines is a bitwise OR of the following flag bits, which tell the
     * called plan node what to expect.  Note that the flags will get modified
     * as they are passed down the plan tree, since an upper node may require
     * functionality in its subnode not demanded of the plan as a whole
     * (example: MergeJoin requires mark/restore capability in its inner input),
     * or an upper node may shield its input from some functionality requirement
     * (example: Materialize shields its input from needing to do backward scan).
     *
     * EXPLAIN_ONLY indicates that the plan tree is being initialized just so
     * EXPLAIN can print it out; it will not be run.  Hence, no side-effects
     * of startup should occur.  However, error checks (such as permission checks)
     * should be performed.
     *
     * REWIND indicates that the plan node should try to efficiently support
     * rescans without parameter changes.  (Nodes must support ExecReScan calls
     * in any case, but if this flag was not given, they are at liberty to do it
     * through complete recalculation.  Note that a parameter change forces a
     * full recalculation in any case.)
     *
     * BACKWARD indicates that the plan node must respect the es_direction flag.
     * When this is not passed, the plan node will only be run forwards.
     *
     * MARK indicates that the plan node must support Mark/Restore calls.
     * When this is not passed, no Mark/Restore will occur.
     *
     * SKIP_TRIGGERS tells ExecutorStart/ExecutorFinish to skip calling
     * AfterTriggerBeginQuery/AfterTriggerEndQuery.  This does not necessarily
     * mean that the plan can't queue any AFTER triggers; just that the caller
     * is responsible for there being a trigger context for them to be queued in.
     *
     * WITH/WITHOUT_OIDS tell the executor to emit tuples with or without space
     * for OIDs, respectively.  These are currently used only for CREATE TABLE AS.
     * If neither is set, the plan may or may not produce tuples including OIDs.
     */
    #define EXEC_FLAG_EXPLAIN_ONLY  0x0001  /* EXPLAIN, no ANALYZE */
    #define EXEC_FLAG_REWIND        0x0002  /* need efficient rescan */
    #define EXEC_FLAG_BACKWARD      0x0004  /* need backward scan */
    #define EXEC_FLAG_MARK          0x0008  /* need mark/restore */
    #define EXEC_FLAG_SKIP_TRIGGERS 0x0010  /* skip AfterTrigger calls */
    #define EXEC_FLAG_WITH_OIDS     0x0020  /* force OIDs in returned tuples */
    #define EXEC_FLAG_WITHOUT_OIDS  0x0040  /* force no OIDs in returned tuples */
    #define EXEC_FLAG_WITH_NO_DATA  0x0080  /* rel scannability doesn't matter */
    
    

_5、ExecutorRun_

    
    
    //上一节已介绍
    

_6、ExecutorFinish_

    
    
    /* ----------------------------------------------------------------      *      ExecutorFinish
      *
      *      This routine must be called after the last ExecutorRun call.
      *      It performs cleanup such as firing AFTER triggers.  It is
      *      separate from ExecutorEnd because EXPLAIN ANALYZE needs to
      *      include these actions in the total runtime.
      *
      *      We provide a function hook variable that lets loadable plugins
      *      get control when ExecutorFinish is called.  Such a plugin would
      *      normally call standard_ExecutorFinish().
      *
      * ----------------------------------------------------------------      */
     void
     ExecutorFinish(QueryDesc *queryDesc)
     {
         if (ExecutorFinish_hook)
             (*ExecutorFinish_hook) (queryDesc);
         else
             standard_ExecutorFinish(queryDesc);
     }
     
     void
     standard_ExecutorFinish(QueryDesc *queryDesc)
     {
         EState     *estate;
         MemoryContext oldcontext;
     
         /* sanity checks */
         Assert(queryDesc != NULL);
     
         estate = queryDesc->estate;
     
         Assert(estate != NULL);
         Assert(!(estate->es_top_eflags & EXEC_FLAG_EXPLAIN_ONLY));
     
         /* This should be run once and only once per Executor instance */
         Assert(!estate->es_finished);
     
         /* Switch into per-query memory context */
         oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
     
         /* Allow instrumentation of Executor overall runtime */
         if (queryDesc->totaltime)
             InstrStartNode(queryDesc->totaltime);
     
         /* Run ModifyTable nodes to completion */
         ExecPostprocessPlan(estate);
     
         /* Execute queued AFTER triggers, unless told not to */
         if (!(estate->es_top_eflags & EXEC_FLAG_SKIP_TRIGGERS))
             AfterTriggerEndQuery(estate);
     
         if (queryDesc->totaltime)
             InstrStopNode(queryDesc->totaltime, 0);
     
         MemoryContextSwitchTo(oldcontext);
     
         estate->es_finished = true;
     }
     
     /* ----------------------------------------------------------------      *      ExecPostprocessPlan
      *
      *      Give plan nodes a final chance to execute before shutdown
      * ----------------------------------------------------------------      */
     static void
     ExecPostprocessPlan(EState *estate)
     {
         ListCell   *lc;
     
         /*
          * Make sure nodes run forward.
          */
         estate->es_direction = ForwardScanDirection;
     
         /*
          * Run any secondary ModifyTable nodes to completion, in case the main
          * query did not fetch all rows from them.  (We do this to ensure that
          * such nodes have predictable results.)
          */
         foreach(lc, estate->es_auxmodifytables)
         {
             PlanState  *ps = (PlanState *) lfirst(lc);
     
             for (;;)
             {
                 TupleTableSlot *slot;
     
                 /* Reset the per-output-tuple exprcontext each time */
                 ResetPerTupleExprContext(estate);
     
                 slot = ExecProcNode(ps);
     
                 if (TupIsNull(slot))
                     break;
             }
      }
    
    

_7、ExecutorEnd_

    
    
     /* ----------------------------------------------------------------      *      ExecutorEnd
      *
      *      This routine must be called at the end of execution of any
      *      query plan
      *
      *      We provide a function hook variable that lets loadable plugins
      *      get control when ExecutorEnd is called.  Such a plugin would
      *      normally call standard_ExecutorEnd().
      *
      * ----------------------------------------------------------------      */
     void
     ExecutorEnd(QueryDesc *queryDesc)
     {
         if (ExecutorEnd_hook)
             (*ExecutorEnd_hook) (queryDesc);
         else
             standard_ExecutorEnd(queryDesc);
     }
     
     void
     standard_ExecutorEnd(QueryDesc *queryDesc)
     {
         EState     *estate;
         MemoryContext oldcontext;
     
         /* sanity checks */
         Assert(queryDesc != NULL);
     
         estate = queryDesc->estate;
     
         Assert(estate != NULL);
     
         /*
          * Check that ExecutorFinish was called, unless in EXPLAIN-only mode. This
          * Assert is needed because ExecutorFinish is new as of 9.1, and callers
          * might forget to call it.
          */
         Assert(estate->es_finished ||
                (estate->es_top_eflags & EXEC_FLAG_EXPLAIN_ONLY));
     
         /*
          * Switch into per-query memory context to run ExecEndPlan
          */
         oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
     
         ExecEndPlan(queryDesc->planstate, estate);
     
         /* do away with our snapshots */
         UnregisterSnapshot(estate->es_snapshot);
         UnregisterSnapshot(estate->es_crosscheck_snapshot);
     
         /*
          * Must switch out of context before destroying it
          */
         MemoryContextSwitchTo(oldcontext);
     
         /*
          * Release EState and per-query memory context.  This should release
          * everything the executor has allocated.
          */
         FreeExecutorState(estate);
     
         /* Reset queryDesc fields that no longer point to anything */
         queryDesc->tupDesc = NULL;
         queryDesc->estate = NULL;
         queryDesc->planstate = NULL;
         queryDesc->totaltime = NULL;
     }
    
     /* ----------------------------------------------------------------      *      ExecEndPlan
      *
      *      Cleans up the query plan -- closes files and frees up storage
      *
      * NOTE: we are no longer very worried about freeing storage per se
      * in this code; FreeExecutorState should be guaranteed to release all
      * memory that needs to be released.  What we are worried about doing
      * is closing relations and dropping buffer pins.  Thus, for example,
      * tuple tables must be cleared or dropped to ensure pins are released.
      * ----------------------------------------------------------------      */
     static void
     ExecEndPlan(PlanState *planstate, EState *estate)
     {
         ResultRelInfo *resultRelInfo;
         int         i;
         ListCell   *l;
     
         /*
          * shut down the node-type-specific query processing
          */
         ExecEndNode(planstate);
     
         /*
          * for subplans too
          */
         foreach(l, estate->es_subplanstates)
         {
             PlanState  *subplanstate = (PlanState *) lfirst(l);
     
             ExecEndNode(subplanstate);
         }
     
         /*
          * destroy the executor's tuple table.  Actually we only care about
          * releasing buffer pins and tupdesc refcounts; there's no need to pfree
          * the TupleTableSlots, since the containing memory context is about to go
          * away anyway.
          */
         ExecResetTupleTable(estate->es_tupleTable, false);
     
         /*
          * close the result relation(s) if any, but hold locks until xact commit.
          */
         resultRelInfo = estate->es_result_relations;
         for (i = estate->es_num_result_relations; i > 0; i--)
         {
             /* Close indices and then the relation itself */
             ExecCloseIndices(resultRelInfo);
             heap_close(resultRelInfo->ri_RelationDesc, NoLock);
             resultRelInfo++;
         }
     
         /* Close the root target relation(s). */
         resultRelInfo = estate->es_root_result_relations;
         for (i = estate->es_num_root_result_relations; i > 0; i--)
         {
             heap_close(resultRelInfo->ri_RelationDesc, NoLock);
             resultRelInfo++;
         }
     
         /* likewise close any trigger target relations */
         ExecCleanUpTriggerState(estate);
     
         /*
          * close any relations selected FOR [KEY] UPDATE/SHARE, again keeping
          * locks
          */
         foreach(l, estate->es_rowMarks)
         {
             ExecRowMark *erm = (ExecRowMark *) lfirst(l);
     
             if (erm->relation)
                 heap_close(erm->relation, NoLock);
         }
     }
    

_8、FreeQueryDesc_

    
    
    //释放资源
     /*
      * FreeQueryDesc
      */
     void
     FreeQueryDesc(QueryDesc *qdesc)
     {
         /* Can't be a live query */
         Assert(qdesc->estate == NULL);
     
         /* forget our snapshots */
         UnregisterSnapshot(qdesc->snapshot);
         UnregisterSnapshot(qdesc->crosscheck_snapshot);
     
         /* Only the QueryDesc itself need be freed */
         pfree(qdesc);
     }
     
    

### 二、源码解读

    
    
    /*
     * ProcessQuery
     *      Execute a single plannable query within a PORTAL_MULTI_QUERY,
     *      PORTAL_ONE_RETURNING, or PORTAL_ONE_MOD_WITH portal
     *
     *  plan: the plan tree for the query
     *  sourceText: the source text of the query
     *  params: any parameters needed
     *  dest: where to send results
     *  completionTag: points to a buffer of size COMPLETION_TAG_BUFSIZE
     *      in which to store a command completion status string.
     *
     * completionTag may be NULL if caller doesn't want a status string.
     *
     * Must be called in a memory context that will be reset or deleted on
     * error; otherwise the executor's memory usage will be leaked.
     */
    /*
    输入：
        plan-已生成执行计划的语句
        sourceText-源SQL语句
        params-TODO
        queryEnv-查询执行的环境
        dest-目标接收器
        completionTag-完成标记
    输出：
        无
    */
    static void
    ProcessQuery(PlannedStmt *plan,
                 const char *sourceText,
                 ParamListInfo params,
                 QueryEnvironment *queryEnv,
                 DestReceiver *dest,
                 char *completionTag)
    {
        QueryDesc  *queryDesc;//查询描述符
    
        /*
         * Create the QueryDesc object
         */
        queryDesc = CreateQueryDesc(plan, sourceText,
                                    GetActiveSnapshot(), InvalidSnapshot,
                                    dest, params, queryEnv, 0);//构造查询描述符
    
        /*
         * Call ExecutorStart to prepare the plan for execution
         */
        ExecutorStart(queryDesc, 0);//启动执行器
    
        /*
         * Run the plan to completion.
         */
        ExecutorRun(queryDesc, ForwardScanDirection, 0L, true);//执行
    
        /*
         * Build command completion status string, if caller wants one.
         */
        if (completionTag)//如果需要完成标记
        {
            Oid         lastOid;
    
            switch (queryDesc->operation)
            {
                case CMD_SELECT:
                    snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
                             "SELECT " UINT64_FORMAT,
                             queryDesc->estate->es_processed);
                    break;
                case CMD_INSERT://插入语句
                    if (queryDesc->estate->es_processed == 1)
                        lastOid = queryDesc->estate->es_lastoid;
                    else
                        lastOid = InvalidOid;
                    snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
                             "INSERT %u " UINT64_FORMAT,
                             lastOid, queryDesc->estate->es_processed);
                    break;
                case CMD_UPDATE:
                    snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
                             "UPDATE " UINT64_FORMAT,
                             queryDesc->estate->es_processed);
                    break;
                case CMD_DELETE:
                    snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
                             "DELETE " UINT64_FORMAT,
                             queryDesc->estate->es_processed);
                    break;
                default:
                    strcpy(completionTag, "???");
                    break;
            }
        }
    
        /*
         * Now, we close down all the scans and free allocated resources.
         */
        ExecutorFinish(queryDesc);//完成
        ExecutorEnd(queryDesc);//结束
    
        FreeQueryDesc(queryDesc);//释放资源
    }
    
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- #9.1 ProcessQuery
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               2551
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(17,'ProcessQuery','ProcessQuery','ProcessQuery');
    (挂起)
    

启动gdb，跟踪调试：

    
    
    [root@localhost ~]# gdb -p 2551
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b ProcessQuery
    Breakpoint 1 at 0x851d19: file pquery.c, line 149.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ProcessQuery (plan=0x2ccb378, sourceText=0x2c09ef0 "insert into t_insert values(17,'ProcessQuery','ProcessQuery','ProcessQuery');", params=0x0, queryEnv=0x0, dest=0x2ccb4d8, 
        completionTag=0x7ffe94ba4940 "") at pquery.c:149
    149     queryDesc = CreateQueryDesc(plan, sourceText,
    #查看参数
    #1、plan
    (gdb) p *plan
    $1 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x2ccafe8, rtable = 0x2ccb2a8, resultRelations = 0x2ccb348, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x2ccb2f8, invalItems = 0x0, paramExecTypes = 0x2c31370, utilityStmt = 0x0, stmt_location = 0, stmt_len = 76}
    (gdb) p *(plan->planTree)
    #执行树，左右均无兄弟节点
    $2 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *(plan->rtable)
    $3 = {type = T_List, length = 1, head = 0x2ccb288, tail = 0x2ccb288}
    (gdb) p *(plan->rtable->head)
    #Oid=46969208，可使用pg_class查询
    $4 = {data = {ptr_value = 0x2ccb178, int_value = 46969208, oid_value = 46969208}, next = 0x0}
    (gdb) p *(plan->resultRelations)
    $5 = {type = T_IntList, length = 1, head = 0x2ccb328, tail = 0x2ccb328}
    (gdb) p *(plan->resultRelations->head)
    #Oid=1？，可使用pg_class查询
    $6 = {data = {ptr_value = 0x1, int_value = 1, oid_value = 1}, next = 0x0}
    (gdb) p *(plan->relationOids)
    $7 = {type = T_OidList, length = 1, head = 0x2ccb2d8, tail = 0x2ccb2d8}
    (gdb) p *(plan->relationOids->head)
    #Oid=26731，可使用pg_class查询
    $8 = {data = {ptr_value = 0x686b, int_value = 26731, oid_value = 26731}, next = 0x0}
    #2、sourceText
    (gdb) p sourceText
    $11 = 0x2c09ef0 "insert into t_insert values(17,'ProcessQuery','ProcessQuery','ProcessQuery');"
    #3、params
    (gdb) p params
    #NULL
    $12 = (ParamListInfo) 0x0
    #4、queryEnv
    (gdb) p queryEnv
    #NULL
    $13 = (QueryEnvironment *) 0x0
    #5、dest
    (gdb) p dest
    $14 = (DestReceiver *) 0x2ccb4d8
    (gdb) p *dest
    $15 = {receiveSlot = 0x4857ad <printtup>, rStartup = 0x485196 <printtup_startup>, rShutdown = 0x485bad <printtup_shutdown>, rDestroy = 0x485c21 <printtup_destroy>, mydest = DestRemote}
    (gdb) 
    #6、completionTag
    (gdb) p completionTag
    #空字符串
    $16 = 0x7ffe94ba4940 ""
    (gdb) next
    156     ExecutorStart(queryDesc, 0);
    (gdb) 
    161     ExecutorRun(queryDesc, ForwardScanDirection, 0L, true);
    (gdb) 
    166     if (completionTag)
    (gdb) 
    170         switch (queryDesc->operation)
    (gdb) 
    178                 if (queryDesc->estate->es_processed == 1)
    (gdb) 
    179                     lastOid = queryDesc->estate->es_lastoid;
    (gdb) p queryDesc->estate->es_lastoid
    $18 = 0
    (gdb) 
    $19 = 0
    (gdb) next
    184                          lastOid, queryDesc->estate->es_processed);
    (gdb) p lastOid
    $20 = 0
    (gdb) next
    182                 snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
    (gdb) 
    185                 break;
    (gdb) p completionTag
    #返回标记，在psql中输出的信息
    $21 = 0x7ffe94ba4940 "INSERT 0 1"
    (gdb) 
    $22 = 0x7ffe94ba4940 "INSERT 0 1"
    (gdb) next
    205     ExecutorFinish(queryDesc);
    (gdb) 
    206     ExecutorEnd(queryDesc);
    (gdb) 
    208     FreeQueryDesc(queryDesc);
    (gdb) 
    209 }
    (gdb) 
    PortalRunMulti (portal=0x2c6f490, isTopLevel=true, setHoldSnapshot=false, dest=0x2ccb4d8, altdest=0x2ccb4d8, completionTag=0x7ffe94ba4940 "INSERT 0 1") at pquery.c:1302
    1302                if (log_executor_stats)
    (gdb) 
    #DONE！
    

为了更深入理解整个执行过程中最重要的两个数据结构PlanState和EState，对子函数InitPlan和CreateExecutorState作进一步的跟踪分析：  
**InitPlan**

    
    
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(18,'ProcessQuery.InitPlan','ProcessQuery.InitPlan','ProcessQuery.InitPlan');
    （挂起）
    
    (gdb) b InitPlan
    Breakpoint 1 at 0x691560: file execMain.c, line 811.
    #查看参数
    #1、queryDesc
    (gdb) p *queryDesc
    $8 = {operation = CMD_INSERT, plannedstmt = 0x2ccb408, sourceText = 0x2c09ef0 "insert into t_insert values(18,'ProcessQuery.InitPlan','ProcessQuery.InitPlan','ProcessQuery.InitPlan');", 
      snapshot = 0x2c2d920, crosscheck_snapshot = 0x0, dest = 0x2ccb568, params = 0x0, queryEnv = 0x0, instrument_options = 0, tupDesc = 0x0, estate = 0x2cbcc70, planstate = 0x0, already_executed = false, 
      totaltime = 0x0}
    #2、eflags
    (gdb) p eflags
    $9 = 0
    (gdb) next
    812     PlannedStmt *plannedstmt = queryDesc->plannedstmt;
    (gdb) 
    813     Plan       *plan = plannedstmt->planTree;
    (gdb) 
    814     List       *rangeTable = plannedstmt->rtable;
    (gdb) p *(queryDesc->plannedstmt)
    $10 = {type = T_PlannedStmt, commandType = CMD_INSERT, queryId = 0, hasReturning = false, hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x2ccb078, rtable = 0x2ccb338, resultRelations = 0x2ccb3d8, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x2ccb388, invalItems = 0x0, paramExecTypes = 0x2c313f8, utilityStmt = 0x0, stmt_location = 0, stmt_len = 103}
    (gdb) next
    815     EState     *estate = queryDesc->estate;
    (gdb) 
    824     ExecCheckRTPerms(rangeTable, true);
    (gdb) 
    829     estate->es_range_table = rangeTable;
    (gdb) 
    830     estate->es_plannedstmt = plannedstmt;
    (gdb) p *rangeTable
    $11 = {type = T_List, length = 1, head = 0x2ccb318, tail = 0x2ccb318}
    (gdb) p *(rangeTable->head)
    $12 = {data = {ptr_value = 0x2ccb208, int_value = 46969352, oid_value = 46969352}, next = 0x0}
    (gdb) next
    838     if (plannedstmt->resultRelations)
    (gdb) 
    840         List       *resultRelations = plannedstmt->resultRelations;
    (gdb) 
    841         int         numResultRelations = list_length(resultRelations);
    (gdb) 
    846             palloc(numResultRelations * sizeof(ResultRelInfo));
    (gdb) p numResultRelations
    $13 = 1
    (gdb) p *(resultRelations->head)
    $14 = {data = {ptr_value = 0x1, int_value = 1, oid_value = 1}, next = 0x0}
    (gdb) 
    (gdb) next
    845         resultRelInfos = (ResultRelInfo *)
    (gdb) 
    847         resultRelInfo = resultRelInfos;
    (gdb) 
    848         foreach(l, resultRelations)
    (gdb) 
    850             Index       resultRelationIndex = lfirst_int(l);
    (gdb) 
    854             resultRelationOid = getrelid(resultRelationIndex, rangeTable);
    (gdb) 
    855             resultRelation = heap_open(resultRelationOid, RowExclusiveLock);
    (gdb) 
    857             InitResultRelInfo(resultRelInfo,
    (gdb) p resultRelationOid
    $15 = 26731
    (gdb) p resultRelation
    $16 = (Relation) 0x7f3a64247b78
    #目标Relation，t_insert
    (gdb) p *resultRelation
    $17 = {rd_node = {spcNode = 1663, dbNode = 16477, relNode = 26747}, rd_smgr = 0x2c99328, rd_refcnt = 1, rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, 
      rd_indexvalid = 1 '\001', rd_statvalid = true, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f3a64247d88, rd_att = 0x7f3a64247e98, rd_id = 26731, rd_lockInfo = {lockRelId = {
          relId = 26731, dbId = 16477}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x7f3a64249cf0, rd_oidindex = 0, rd_pkindex = 26737, rd_replidindex = 26737, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, 
      rd_keyattr = 0x0, rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, 
      rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x2c8ae98}
    (gdb) 
    gdb) next
    848         foreach(l, resultRelations)
    (gdb) 
    864         estate->es_result_relations = resultRelInfos;
    (gdb) 
    865         estate->es_num_result_relations = numResultRelations;
    (gdb) 
    867         estate->es_result_relation_info = NULL;
    (gdb) 
    874         estate->es_root_result_relations = NULL;
    (gdb) 
    875         estate->es_num_root_result_relations = 0;
    (gdb) 
    876         if (plannedstmt->nonleafResultRelations)
    (gdb) p *plannedstmt->nonleafResultRelations
    Cannot access memory at address 0x0
    (gdb) next
    941     estate->es_rowMarks = NIL;
    942     foreach(l, plannedstmt->rowMarks)
    (gdb) p plannedstmt->rowMarks
    $19 = (List *) 0x0
    (gdb) next
    1003        estate->es_tupleTable = NIL;
    (gdb) 
    1004        estate->es_trig_tuple_slot = NULL;
    (gdb) 
    1005        estate->es_trig_oldtup_slot = NULL;
    (gdb) 
    1006        estate->es_trig_newtup_slot = NULL;
    (gdb) 
    1009        estate->es_epqTuple = NULL;
    (gdb) 
    1010        estate->es_epqTupleSet = NULL;
    (gdb) 
    1011        estate->es_epqScanDone = NULL;
    (gdb) 
    1019        i = 1;                      /* subplan indices count from 1 */
    (gdb) 
    1020        foreach(l, plannedstmt->subplans)
    (gdb) p *plannedstmt->subplans
    Cannot access memory at address 0x0
    (gdb) next
    1049        planstate = ExecInitNode(plan, estate, eflags);
    (gdb) step
    ExecInitNode (node=0x2ccb078, estate=0x2cbcc70, eflags=0) at execProcnode.c:148
    148     if (node == NULL)
    (gdb) next
    156     check_stack_depth();
    (gdb) 
    158     switch (nodeTag(node))
    (gdb) 
    174             result = (PlanState *) ExecInitModifyTable((ModifyTable *) node,
    (gdb) step
    ExecInitModifyTable (node=0x2ccb078, estate=0x2cbcc70, eflags=0) at nodeModifyTable.c:2179
    2179        CmdType     operation = node->operation;
    (gdb) next
    2180        int         nplans = list_length(node->plans);
    (gdb) 
    2187        bool        update_tuple_routing_needed = node->partColsUpdated;
    (gdb) p node->plans
    $20 = (List *) 0x2c317c8
    (gdb) p *(node->plans)
    $21 = {type = T_List, length = 1, head = 0x2ccb058, tail = 0x2ccb058}
    (gdb) p *(node->plans->head)
    $22 = {data = {ptr_value = 0x2c315b8, int_value = 46339512, oid_value = 46339512}, next = 0x0}
    (gdb) next
    2195        mtstate = makeNode(ModifyTableState);
    (gdb) 
    2196        mtstate->ps.plan = (Plan *) node;
    (gdb) p *mtstate
    $23 = {ps = {type = T_ModifyTableState, plan = 0x0, state = 0x0, ExecProcNode = 0x0, ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, 
        initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, operation = CMD_UNKNOWN, canSetTag = false, mt_done = false, 
      mt_plans = 0x0, mt_nplans = 0, mt_whichplan = 0, resultRelInfo = 0x0, rootResultRelInfo = 0x0, mt_arowmarks = 0x0, mt_epqstate = {estate = 0x0, planstate = 0x0, origslot = 0x0, plan = 0x0, 
        arowMarks = 0x0, epqParam = 0}, fireBSTriggers = false, mt_existing = 0x0, mt_excludedtlist = 0x0, mt_conflproj = 0x0, mt_partition_tuple_routing = 0x0, mt_transition_capture = 0x0, 
      mt_oc_transition_capture = 0x0, mt_per_subplan_tupconv_maps = 0x0}
    (gdb) next
    2197        mtstate->ps.state = estate;
    (gdb) 
    2198        mtstate->ps.ExecProcNode = ExecModifyTable;
    (gdb) 
    2200        mtstate->operation = operation;
    (gdb) 
    2201        mtstate->canSetTag = node->canSetTag;
    (gdb) 
    2202        mtstate->mt_done = false;
    (gdb) 
    2204        mtstate->mt_plans = (PlanState **) palloc0(sizeof(PlanState *) * nplans);
    (gdb) 
    2205        mtstate->resultRelInfo = estate->es_result_relations + node->resultRelIndex;
    (gdb) 
    2208        if (node->rootResultRelIndex >= 0)
    (gdb) 
    2212        mtstate->mt_arowmarks = (List **) palloc0(sizeof(List *) * nplans);
    (gdb) 
    2213        mtstate->mt_nplans = nplans;
    (gdb) 
    2216        EvalPlanQualInit(&mtstate->mt_epqstate, estate, NULL, NIL, node->epqParam);
    (gdb) 
    2217        mtstate->fireBSTriggers = true;
    (gdb) 
    2227        saved_resultRelInfo = estate->es_result_relation_info;
    (gdb) p *(mtstate->mt_epqstate)
    Structure has no component named operator*.
    (gdb) p mtstate->mt_epqstate
    $24 = {estate = 0x0, planstate = 0x0, origslot = 0x0, plan = 0x0, arowMarks = 0x0, epqParam = 0}
    (gdb) next
    2229        resultRelInfo = mtstate->resultRelInfo;
    (gdb) 
    2230        i = 0;
    (gdb) 
    2231        foreach(l, node->plans)
    (gdb) 
    2233            subplan = (Plan *) lfirst(l);
    (gdb) 
    2237                                                                  node->fdwDirectModifyPlans);
    (gdb) 
    2236            resultRelInfo->ri_usesFdwDirectModify = bms_is_member(i,
    (gdb) 
    2242            CheckValidResultRel(resultRelInfo, operation);
    (gdb) 
    2253            if (resultRelInfo->ri_RelationDesc->rd_rel->relhasindex &&
    (gdb) 
    2255                resultRelInfo->ri_IndexRelationDescs == NULL)
    (gdb) 
    2254                operation != CMD_DELETE &&
    (gdb) 
    2257                                node->onConflictAction != ONCONFLICT_NONE);
    (gdb) 
    2256                ExecOpenIndices(resultRelInfo,
    (gdb) 
    2264            if (resultRelInfo->ri_TrigDesc &&
    (gdb) 
    2270            estate->es_result_relation_info = resultRelInfo;
    (gdb) 
    2271            mtstate->mt_plans[i] = ExecInitNode(subplan, estate, eflags);
    (gdb) finish
    Run till exit from #0  ExecInitModifyTable (node=0x2ccb078, estate=0x2cbcc70, eflags=0) at nodeModifyTable.c:2294
    0x000000000069a21a in ExecInitNode (node=0x2ccb078, estate=0x2cbcc70, eflags=0) at execProcnode.c:174
    174             result = (PlanState *) ExecInitModifyTable((ModifyTable *) node,
    Value returned is $25 = (ModifyTableState *) 0x2cbcfc0
    (gdb) next
    176             break;
    (gdb) 
    373     ExecSetExecProcNode(result, result->ExecProcNode);
    (gdb) p result->ExecProcNode
    $26 = (ExecProcNodeMtd) 0x6c2485 <ExecModifyTable>
    (gdb) finish
    Run till exit from #0  ExecInitNode (node=0x2ccb078, estate=0x2cbcc70, eflags=0) at execProcnode.c:392
    0x0000000000691c2f in InitPlan (queryDesc=0x2cc1580, eflags=0) at execMain.c:1049
    1049        planstate = ExecInitNode(plan, estate, eflags);
    Value returned is $27 = (PlanState *) 0x2cbcfc0
    (gdb) next
    1054        tupType = ExecGetResultType(planstate);
    (gdb) 
    1060        if (operation == CMD_SELECT)
    (gdb) p tupType
    $28 = (TupleDesc) 0x2cbdd40
    (gdb) p *tupType
    $29 = {natts = 0, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x2cbdd60}
    (gdb) next
    1090        queryDesc->tupDesc = tupType;
    (gdb) 
    1091        queryDesc->planstate = planstate;
    (gdb) 
    1092    }
    (gdb) 
    standard_ExecutorStart (queryDesc=0x2cc1580, eflags=0) at execMain.c:266
    266     MemoryContextSwitchTo(oldcontext);
    (gdb) 
    #DONE!
    
    

**CreateExecutorState**

    
    
    #gdb
    (gdb) b CreateExecutorState
    Breakpoint 1 at 0x69f2c5: file execUtils.c, line 89.
    #psql
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(19,'ProcessQuery.CreateExecutorState','ProcessQuery.CreateExecutorState','ProcessQuery.CreateExecutorState');
    （挂起）
    #gdb
    (gdb) c
    Continuing.
    
    Breakpoint 1, CreateExecutorState () at execUtils.c:89
    89      qcontext = AllocSetContextCreate(CurrentMemoryContext,
    #查看输入参数
    #该函数无输入参数
    (gdb) step #进入AllocSetContextCreate函数内部
    AllocSetContextCreateExtended (parent=0x2c09de0, name=0xb1a840 "ExecutorState", minContextSize=0, initBlockSize=8192, maxBlockSize=8388608) at aset.c:426
    426     if (minContextSize == ALLOCSET_DEFAULT_MINSIZE &&
    (gdb) next
    428         freeListIndex = 0;
    #查看AllocSetContextCreate的输入参数
    #1、parent
    (gdb) p *parent
    $3 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c04ba0, firstchild = 0x0, prevchild = 0x2c7fc60, nextchild = 0x2cb6b30, 
      name = 0xb4e87c "MessageContext", ident = 0x0, reset_cbs = 0x0}
    #顶层Context
    (gdb) p *(parent->parent)
    $4 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x0, firstchild = 0x2c2d7e0, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb8d050 "TopMemoryContext", ident = 0x0, reset_cbs = 0x0}
    #2、name
    (gdb) p name
    $5 = 0xb1a840 "ExecutorState"
    #3、minContextSize
    (gdb) p minContextSize
    $6 = 0
    #4、initBlockSize
    (gdb) p initBlockSize
    $7 = 8192 #8KB
    #5、maxBlockSize
    (gdb) p maxBlockSize
    $8 = 8388608 #8MB
    (gdb) next
    440         AllocSetFreeList *freelist = &context_freelists[freeListIndex];
    (gdb) p freeListIndex 
    $9 = 0
    (gdb) 
    $10 = 0
    (gdb) next
    442         if (freelist->first_free != NULL)
    (gdb) p *freelist
    $11 = {num_free = 4, first_free = 0x2cbcb60}
    (gdb) p *(freelist->first_free)
    $12 = {header = {type = T_AllocSetContext, isReset = true, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x0, firstchild = 0x0, prevchild = 0x0, nextchild = 0x2cbeb70, 
        name = 0xb1a840 "ExecutorState", ident = 0x0, reset_cbs = 0x0}, blocks = 0x2cbcc38, freelist = {0x0 <repeats 11 times>}, initBlockSize = 8192, maxBlockSize = 8388608, nextBlockSize = 8192, 
      allocChunkLimit = 8192, keeper = 0x2cbcc38, freeListIndex = 0}
    (gdb) next
    445             set = freelist->first_free;
    (gdb) 
    446             freelist->first_free = (AllocSet) set->header.nextchild;
    (gdb) 
    447             freelist->num_free--;
    (gdb) 
    450             set->maxBlockSize = maxBlockSize;
    (gdb) 
    453             MemoryContextCreate((MemoryContext) set,
    (gdb) 
    459             return (MemoryContext) set;
    (gdb) p *set
    $13 = {header = {type = T_AllocSetContext, isReset = true, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x2c09de0, firstchild = 0x0, prevchild = 0x0, nextchild = 0x0, 
        name = 0xb1a840 "ExecutorState", ident = 0x0, reset_cbs = 0x0}, blocks = 0x2cbcc38, freelist = {0x0 <repeats 11 times>}, initBlockSize = 8192, maxBlockSize = 8388608, nextBlockSize = 8192, 
      allocChunkLimit = 8192, keeper = 0x2cbcc38, freeListIndex = 0}
    (gdb) next
    548 }
    (gdb) 
    CreateExecutorState () at execUtils.c:97
    97      oldcontext = MemoryContextSwitchTo(qcontext);
    (gdb) 
    99      estate = makeNode(EState);
    (gdb) 
    104     estate->es_direction = ForwardScanDirection;
    (gdb) finish
    Run till exit from #0  CreateExecutorState () at execUtils.c:104
    0x000000000078cc2f in evaluate_expr (expr=0x2c30520, result_type=1043, result_typmod=44, result_collation=100) at clauses.c:4858
    4858        estate = CreateExecutorState();
    Value returned is $14 = (EState *) 0x2cbcc70
    
    

### 四、小结

1、TODO：更进一步的理解，在执行查询语句时再进一步解读；  
2、EState&PlanState数据结构：在此函数中构造，需进一步理解；  
3、List数据结构：PG广泛的使用List这样的数据结构对各种信息进行管理。

