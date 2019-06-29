本文简单介绍了PG插入数据部分的源码，主要内容包括ExecInsert函数的实现逻辑，该函数位于nodeModifyTable.c文件中。

### 一、基础信息

按惯例，首先看看ExecInsert函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、ModifyTableState_

    
    
     /* ----------------      *      PlanState node
      *
      * We never actually instantiate any PlanState nodes; this is just the common
      * abstract superclass for all PlanState-type nodes.
      * ----------------      */
     typedef struct PlanState
     {
         NodeTag     type;
     
         Plan       *plan;           /* associated Plan node */
     
         EState     *state;          /* at execution time, states of individual
                                      * nodes point to one EState for the whole
                                      * top-level plan */
     
         ExecProcNodeMtd ExecProcNode;   /* function to return next tuple */
         ExecProcNodeMtd ExecProcNodeReal;   /* actual function, if above is a
                                              * wrapper */
     
         Instrumentation *instrument;    /* Optional runtime stats for this node */
         WorkerInstrumentation *worker_instrument;   /* per-worker instrumentation */
     
         /*
          * Common structural data for all Plan types.  These links to subsidiary
          * state trees parallel links in the associated plan tree (except for the
          * subPlan list, which does not exist in the plan tree).
          */
         ExprState  *qual;           /* boolean qual condition */
         struct PlanState *lefttree; /* input plan tree(s) */
         struct PlanState *righttree;
     
         List       *initPlan;       /* Init SubPlanState nodes (un-correlated expr
                                      * subselects) */
         List       *subPlan;        /* SubPlanState nodes in my expressions */
     
         /*
          * State for management of parameter-change-driven rescanning
          */
         Bitmapset  *chgParam;       /* set of IDs of changed Params */
     
         /*
          * Other run-time state needed by most if not all node types.
          */
         TupleTableSlot *ps_ResultTupleSlot; /* slot for my result tuples */
         ExprContext *ps_ExprContext;    /* node's expression-evaluation context */
         ProjectionInfo *ps_ProjInfo;    /* info for doing tuple projection */
     
         /*
          * Scanslot's descriptor if known. This is a bit of a hack, but otherwise
          * it's hard for expression compilation to optimize based on the
          * descriptor, without encoding knowledge about all executor nodes.
          */
         TupleDesc   scandesc;
     } PlanState;
    
    /* ----------------      *   ModifyTableState information
      * ----------------      */
     typedef struct ModifyTableState
     {
         PlanState   ps;             /* its first field is NodeTag */
         CmdType     operation;      /* INSERT, UPDATE, or DELETE */
         bool        canSetTag;      /* do we set the command tag/es_processed? */
         bool        mt_done;        /* are we done? */
         PlanState **mt_plans;       /* subplans (one per target rel) */
         int         mt_nplans;      /* number of plans in the array */
         int         mt_whichplan;   /* which one is being executed (0..n-1) */
         ResultRelInfo *resultRelInfo;   /* per-subplan target relations */
         ResultRelInfo *rootResultRelInfo;   /* root target relation (partitioned
                                              * table root) */
         List      **mt_arowmarks;   /* per-subplan ExecAuxRowMark lists */
         EPQState    mt_epqstate;    /* for evaluating EvalPlanQual rechecks */
         bool        fireBSTriggers; /* do we need to fire stmt triggers? */
         TupleTableSlot *mt_existing;    /* slot to store existing target tuple in */
         List       *mt_excludedtlist;   /* the excluded pseudo relation's tlist  */
         TupleTableSlot *mt_conflproj;   /* CONFLICT ... SET ... projection target */
     
         /* Tuple-routing support info */
         struct PartitionTupleRouting *mt_partition_tuple_routing;
     
         /* controls transition table population for specified operation */
         struct TransitionCaptureState *mt_transition_capture;
     
         /* controls transition table population for INSERT...ON CONFLICT UPDATE */
         struct TransitionCaptureState *mt_oc_transition_capture;
     
         /* Per plan map for tuple conversion from child to root */
         TupleConversionMap **mt_per_subplan_tupconv_maps;
     } ModifyTableState;
     
     
    

_2、TupleTableSlot_

    
    
    /*----------      * The executor stores tuples in a "tuple table" which is a List of
      * independent TupleTableSlots.  There are several cases we need to handle:
      *      1. physical tuple in a disk buffer page
      *      2. physical tuple constructed in palloc'ed memory
      *      3. "minimal" physical tuple constructed in palloc'ed memory
      *      4. "virtual" tuple consisting of Datum/isnull arrays
      *
      * The first two cases are similar in that they both deal with "materialized"
      * tuples, but resource management is different.  For a tuple in a disk page
      * we need to hold a pin on the buffer until the TupleTableSlot's reference
      * to the tuple is dropped; while for a palloc'd tuple we usually want the
      * tuple pfree'd when the TupleTableSlot's reference is dropped.
      *
      * A "minimal" tuple is handled similarly to a palloc'd regular tuple.
      * At present, minimal tuples never are stored in buffers, so there is no
      * parallel to case 1.  Note that a minimal tuple has no "system columns".
      * (Actually, it could have an OID, but we have no need to access the OID.)
      *
      * A "virtual" tuple is an optimization used to minimize physical data
      * copying in a nest of plan nodes.  Any pass-by-reference Datums in the
      * tuple point to storage that is not directly associated with the
      * TupleTableSlot; generally they will point to part of a tuple stored in
      * a lower plan node's output TupleTableSlot, or to a function result
      * constructed in a plan node's per-tuple econtext.  It is the responsibility
      * of the generating plan node to be sure these resources are not released
      * for as long as the virtual tuple needs to be valid.  We only use virtual
      * tuples in the result slots of plan nodes --- tuples to be copied anywhere
      * else need to be "materialized" into physical tuples.  Note also that a
      * virtual tuple does not have any "system columns".
      *
      * It is also possible for a TupleTableSlot to hold both physical and minimal
      * copies of a tuple.  This is done when the slot is requested to provide
      * the format other than the one it currently holds.  (Originally we attempted
      * to handle such requests by replacing one format with the other, but that
      * had the fatal defect of invalidating any pass-by-reference Datums pointing
      * into the existing slot contents.)  Both copies must contain identical data
      * payloads when this is the case.
      *
      * The Datum/isnull arrays of a TupleTableSlot serve double duty.  When the
      * slot contains a virtual tuple, they are the authoritative data.  When the
      * slot contains a physical tuple, the arrays contain data extracted from
      * the tuple.  (In this state, any pass-by-reference Datums point into
      * the physical tuple.)  The extracted information is built "lazily",
      * ie, only as needed.  This serves to avoid repeated extraction of data
      * from the physical tuple.
      *
      * A TupleTableSlot can also be "empty", holding no valid data.  This is
      * the only valid state for a freshly-created slot that has not yet had a
      * tuple descriptor assigned to it.  In this state, tts_isempty must be
      * true, tts_shouldFree false, tts_tuple NULL, tts_buffer InvalidBuffer,
      * and tts_nvalid zero.
      *
      * The tupleDescriptor is simply referenced, not copied, by the TupleTableSlot
      * code.  The caller of ExecSetSlotDescriptor() is responsible for providing
      * a descriptor that will live as long as the slot does.  (Typically, both
      * slots and descriptors are in per-query memory and are freed by memory
      * context deallocation at query end; so it's not worth providing any extra
      * mechanism to do more.  However, the slot will increment the tupdesc
      * reference count if a reference-counted tupdesc is supplied.)
      *
      * When tts_shouldFree is true, the physical tuple is "owned" by the slot
      * and should be freed when the slot's reference to the tuple is dropped.
      *
      * If tts_buffer is not InvalidBuffer, then the slot is holding a pin
      * on the indicated buffer page; drop the pin when we release the
      * slot's reference to that buffer.  (tts_shouldFree should always be
      * false in such a case, since presumably tts_tuple is pointing at the
      * buffer page.)
      *
      * tts_nvalid indicates the number of valid columns in the tts_values/isnull
      * arrays.  When the slot is holding a "virtual" tuple this must be equal
      * to the descriptor's natts.  When the slot is holding a physical tuple
      * this is equal to the number of columns we have extracted (we always
      * extract columns from left to right, so there are no holes).
      *
      * tts_values/tts_isnull are allocated when a descriptor is assigned to the
      * slot; they are of length equal to the descriptor's natts.
      *
      * tts_mintuple must always be NULL if the slot does not hold a "minimal"
      * tuple.  When it does, tts_mintuple points to the actual MinimalTupleData
      * object (the thing to be pfree'd if tts_shouldFreeMin is true).  If the slot
      * has only a minimal and not also a regular physical tuple, then tts_tuple
      * points at tts_minhdr and the fields of that struct are set correctly
      * for access to the minimal tuple; in particular, tts_minhdr.t_data points
      * MINIMAL_TUPLE_OFFSET bytes before tts_mintuple.  This allows column
      * extraction to treat the case identically to regular physical tuples.
      *
      * tts_slow/tts_off are saved state for slot_deform_tuple, and should not
      * be touched by any other code.
      *----------      */
     typedef struct TupleTableSlot
     {
         NodeTag     type;
         bool        tts_isempty;    /* true = slot is empty */
         bool        tts_shouldFree; /* should pfree tts_tuple? */
         bool        tts_shouldFreeMin;  /* should pfree tts_mintuple? */
     #define FIELDNO_TUPLETABLESLOT_SLOW 4
         bool        tts_slow;       /* saved state for slot_deform_tuple */
     #define FIELDNO_TUPLETABLESLOT_TUPLE 5
         HeapTuple   tts_tuple;      /* physical tuple, or NULL if virtual */
     #define FIELDNO_TUPLETABLESLOT_TUPLEDESCRIPTOR 6
         TupleDesc   tts_tupleDescriptor;    /* slot's tuple descriptor */
         MemoryContext tts_mcxt;     /* slot itself is in this context */
         Buffer      tts_buffer;     /* tuple's buffer, or InvalidBuffer */
     #define FIELDNO_TUPLETABLESLOT_NVALID 9
         int         tts_nvalid;     /* # of valid values in tts_values */
     #define FIELDNO_TUPLETABLESLOT_VALUES 10
         Datum      *tts_values;     /* current per-attribute values */
     #define FIELDNO_TUPLETABLESLOT_ISNULL 11
         bool       *tts_isnull;     /* current per-attribute isnull flags */
         MinimalTuple tts_mintuple;  /* minimal tuple, or NULL if none */
         HeapTupleData tts_minhdr;   /* workspace for minimal-tuple-only case */
     #define FIELDNO_TUPLETABLESLOT_OFF 14
         uint32      tts_off;        /* saved state for slot_deform_tuple */
         bool        tts_fixedTupleDescriptor;   /* descriptor can't be changed */
     } TupleTableSlot;
     
    

_3、EState_

    
    
     /* ----------------      *    EState information
      *
      * Master working state for an Executor invocation
      * ----------------      */
     typedef struct EState
     {
         NodeTag     type;
     
         /* Basic state for all query types: */
         ScanDirection es_direction; /* current scan direction */
         Snapshot    es_snapshot;    /* time qual to use */
         Snapshot    es_crosscheck_snapshot; /* crosscheck time qual for RI */
         List       *es_range_table; /* List of RangeTblEntry */
         PlannedStmt *es_plannedstmt;    /* link to top of plan tree */
         const char *es_sourceText;  /* Source text from QueryDesc */
     
         JunkFilter *es_junkFilter;  /* top-level junk filter, if any */
     
         /* If query can insert/delete tuples, the command ID to mark them with */
         CommandId   es_output_cid;
     
         /* Info about target table(s) for insert/update/delete queries: */
         ResultRelInfo *es_result_relations; /* array of ResultRelInfos */
         int         es_num_result_relations;    /* length of array */
         ResultRelInfo *es_result_relation_info; /* currently active array elt */
     
         /*
          * Info about the target partitioned target table root(s) for
          * update/delete queries.  They required only to fire any per-statement
          * triggers defined on the table.  It exists separately from
          * es_result_relations, because partitioned tables don't appear in the
          * plan tree for the update/delete cases.
          */
         ResultRelInfo *es_root_result_relations;    /* array of ResultRelInfos */
         int         es_num_root_result_relations;   /* length of the array */
     
         /*
          * The following list contains ResultRelInfos created by the tuple routing
          * code for partitions that don't already have one.
          */
         List       *es_tuple_routing_result_relations;
     
         /* Stuff used for firing triggers: */
         List       *es_trig_target_relations;   /* trigger-only ResultRelInfos */
         TupleTableSlot *es_trig_tuple_slot; /* for trigger output tuples */
         TupleTableSlot *es_trig_oldtup_slot;    /* for TriggerEnabled */
         TupleTableSlot *es_trig_newtup_slot;    /* for TriggerEnabled */
     
         /* Parameter info: */
         ParamListInfo es_param_list_info;   /* values of external params */
         ParamExecData *es_param_exec_vals;  /* values of internal params */
     
         QueryEnvironment *es_queryEnv;  /* query environment */
     
         /* Other working state: */
         MemoryContext es_query_cxt; /* per-query context in which EState lives */
     
         List       *es_tupleTable;  /* List of TupleTableSlots */
     
         List       *es_rowMarks;    /* List of ExecRowMarks */
     
         uint64      es_processed;   /* # of tuples processed */
         Oid         es_lastoid;     /* last oid processed (by INSERT) */
     
         int         es_top_eflags;  /* eflags passed to ExecutorStart */
         int         es_instrument;  /* OR of InstrumentOption flags */
         bool        es_finished;    /* true when ExecutorFinish is done */
     
         List       *es_exprcontexts;    /* List of ExprContexts within EState */
     
         List       *es_subplanstates;   /* List of PlanState for SubPlans */
     
         List       *es_auxmodifytables; /* List of secondary ModifyTableStates */
     
         /*
          * this ExprContext is for per-output-tuple operations, such as constraint
          * checks and index-value computations.  It will be reset for each output
          * tuple.  Note that it will be created only if needed.
          */
         ExprContext *es_per_tuple_exprcontext;
     
         /*
          * These fields are for re-evaluating plan quals when an updated tuple is
          * substituted in READ COMMITTED mode.  es_epqTuple[] contains tuples that
          * scan plan nodes should return instead of whatever they'd normally
          * return, or NULL if nothing to return; es_epqTupleSet[] is true if a
          * particular array entry is valid; and es_epqScanDone[] is state to
          * remember if the tuple has been returned already.  Arrays are of size
          * list_length(es_range_table) and are indexed by scan node scanrelid - 1.
          */
         HeapTuple  *es_epqTuple;    /* array of EPQ substitute tuples */
         bool       *es_epqTupleSet; /* true if EPQ tuple is provided */
         bool       *es_epqScanDone; /* true if EPQ tuple has been fetched */
     
         bool        es_use_parallel_mode;   /* can we use parallel workers? */
     
         /* The per-query shared memory area to use for parallel execution. */
         struct dsa_area *es_query_dsa;
     
         /*
          * JIT information. es_jit_flags indicates whether JIT should be performed
          * and with which options.  es_jit is created on-demand when JITing is
          * performed.
          */
         int         es_jit_flags;
         struct JitContext *es_jit;
     } EState;
     
    

_4、ResultRelInfo_

    
    
     /*
      * ResultRelInfo
      *
      * Whenever we update an existing relation, we have to update indexes on the
      * relation, and perhaps also fire triggers.  ResultRelInfo holds all the
      * information needed about a result relation, including indexes.
      */
     typedef struct ResultRelInfo
     {
         NodeTag     type;
     
         /* result relation's range table index */
         Index       ri_RangeTableIndex;
     
         /* relation descriptor for result relation */
         Relation    ri_RelationDesc;
     
         /* # of indices existing on result relation */
         int         ri_NumIndices;
     
         /* array of relation descriptors for indices */
         RelationPtr ri_IndexRelationDescs;
     
         /* array of key/attr info for indices */
         IndexInfo **ri_IndexRelationInfo;
     
         /* triggers to be fired, if any */
         TriggerDesc *ri_TrigDesc;
     
         /* cached lookup info for trigger functions */
         FmgrInfo   *ri_TrigFunctions;
     
         /* array of trigger WHEN expr states */
         ExprState **ri_TrigWhenExprs;
     
         /* optional runtime measurements for triggers */
         Instrumentation *ri_TrigInstrument;
     
         /* FDW callback functions, if foreign table */
         struct FdwRoutine *ri_FdwRoutine;
     
         /* available to save private state of FDW */
         void       *ri_FdwState;
     
         /* true when modifying foreign table directly */
         bool        ri_usesFdwDirectModify;
     
         /* list of WithCheckOption's to be checked */
         List       *ri_WithCheckOptions;
     
         /* list of WithCheckOption expr states */
         List       *ri_WithCheckOptionExprs;
     
         /* array of constraint-checking expr states */
         ExprState **ri_ConstraintExprs;
     
         /* for removing junk attributes from tuples */
         JunkFilter *ri_junkFilter;
     
         /* list of RETURNING expressions */
         List       *ri_returningList;
     
         /* for computing a RETURNING list */
         ProjectionInfo *ri_projectReturning;
     
         /* list of arbiter indexes to use to check conflicts */
         List       *ri_onConflictArbiterIndexes;
     
         /* ON CONFLICT evaluation state */
         OnConflictSetState *ri_onConflict;
     
         /* partition check expression */
         List       *ri_PartitionCheck;
     
         /* partition check expression state */
         ExprState  *ri_PartitionCheckExpr;
     
         /* relation descriptor for root partitioned table */
         Relation    ri_PartitionRoot;
     
         /* true if ready for tuple routing */
         bool        ri_PartitionReadyForRouting;
     } ResultRelInfo;
     
    

_5、List_

    
    
     typedef struct ListCell ListCell;
     
     typedef struct List
     {
         NodeTag     type;           /* T_List, T_IntList, or T_OidList */
         int         length;
         ListCell   *head;
         ListCell   *tail;
     } List;
     
     struct ListCell
     {
         union
         {
             void       *ptr_value;
             int         int_value;
             Oid         oid_value;
         }           data;
         ListCell   *next;
     };
     
    

_6、TransitionCaptureState_

    
    
     
     typedef struct TransitionCaptureState
     {
         /*
          * Is there at least one trigger specifying each transition relation on
          * the relation explicitly named in the DML statement or COPY command?
          * Note: in current usage, these flags could be part of the private state,
          * but it seems possibly useful to let callers see them.
          */
         bool        tcs_delete_old_table;
         bool        tcs_update_old_table;
         bool        tcs_update_new_table;
         bool        tcs_insert_new_table;
     
         /*
          * For UPDATE and DELETE, AfterTriggerSaveEvent may need to convert the
          * new and old tuples from a child table's format to the format of the
          * relation named in a query so that it is compatible with the transition
          * tuplestores.  The caller must store the conversion map here if so.
          */
         TupleConversionMap *tcs_map;
     
         /*
          * For INSERT and COPY, it would be wasteful to convert tuples from child
          * format to parent format after they have already been converted in the
          * opposite direction during routing.  In that case we bypass conversion
          * and allow the inserting code (copy.c and nodeModifyTable.c) to provide
          * the original tuple directly.
          */
         HeapTuple   tcs_original_insert_tuple;
     
         /*
          * Private data including the tuplestore(s) into which to insert tuples.
          */
         struct AfterTriggersTableData *tcs_private;
     } TransitionCaptureState;
    

*7、ModifyTable *
    
    
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
    

_8、OnConflictAction_

    
    
    /*
      * OnConflictAction -      *    "ON CONFLICT" clause type of query
      *
      * This is needed in both parsenodes.h and plannodes.h, so put it here...
      */
     typedef enum OnConflictAction
     {
         ONCONFLICT_NONE,            /* No "ON CONFLICT" clause */
         ONCONFLICT_NOTHING,         /* ON CONFLICT ... DO NOTHING */
         ONCONFLICT_UPDATE           /* ON CONFLICT ... DO UPDATE */
     } OnConflictAction;
    

_8、MemoryContext_

    
    
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
    

**依赖的函数**  
_1、ExecMaterializeSlot_

    
    
     /* --------------------------------      *      ExecMaterializeSlot
      *          Force a slot into the "materialized" state.
      *
      *      This causes the slot's tuple to be a local copy not dependent on
      *      any external storage.  A pointer to the contained tuple is returned.
      *
      *      A typical use for this operation is to prepare a computed tuple
      *      for being stored on disk.  The original data may or may not be
      *      virtual, but in any case we need a private copy for heap_insert
      *      to scribble on.
      * --------------------------------      */
     HeapTuple
     ExecMaterializeSlot(TupleTableSlot *slot)
     {
         MemoryContext oldContext;
     
         /*
          * sanity checks
          */
         Assert(slot != NULL);
         Assert(!slot->tts_isempty);
     
         /*
          * If we have a regular physical tuple, and it's locally palloc'd, we have
          * nothing to do.
          */
         if (slot->tts_tuple && slot->tts_shouldFree)
             return slot->tts_tuple;
     
         /*
          * Otherwise, copy or build a physical tuple, and store it into the slot.
          *
          * We may be called in a context that is shorter-lived than the tuple
          * slot, but we have to ensure that the materialized tuple will survive
          * anyway.
          */
         oldContext = MemoryContextSwitchTo(slot->tts_mcxt);//内存上下文切换至slot->tts_mcxt
         slot->tts_tuple = ExecCopySlotTuple(slot);
         slot->tts_shouldFree = true;
         MemoryContextSwitchTo(oldContext);//内存上下文切换回原来
     
         /*
          * Drop the pin on the referenced buffer, if there is one.
          */
         if (BufferIsValid(slot->tts_buffer))
             ReleaseBuffer(slot->tts_buffer);
     
         slot->tts_buffer = InvalidBuffer;
     
         /*
          * Mark extracted state invalid.  This is important because the slot is
          * not supposed to depend any more on the previous external data; we
          * mustn't leave any dangling pass-by-reference datums in tts_values.
          * However, we have not actually invalidated any such datums, if there
          * happen to be any previously fetched from the slot.  (Note in particular
          * that we have not pfree'd tts_mintuple, if there is one.)
          */
         slot->tts_nvalid = 0;
     
         /*
          * On the same principle of not depending on previous remote storage,
          * forget the mintuple if it's not local storage.  (If it is local
          * storage, we must not pfree it now, since callers might have already
          * fetched datum pointers referencing it.)
          */
         if (!slot->tts_shouldFreeMin)
             slot->tts_mintuple = NULL;
     
         return slot->tts_tuple;
     }
    
     #ifndef FRONTEND
     static inline MemoryContext
     MemoryContextSwitchTo(MemoryContext context)
     {
         MemoryContext old = CurrentMemoryContext;
     
         CurrentMemoryContext = context;
         return old;
     }
     #endif                          /* FRONTEND */
    
    

_2、HeapTupleSetOid_

    
    
     #define HeapTupleSetOid(tuple, oid) \
      HeapTupleHeaderSetOid((tuple)->t_data, (oid))
    

_3、ExecBRInsertTriggers_

    
    
     TupleTableSlot *
     ExecBRInsertTriggers(EState *estate, ResultRelInfo *relinfo,
                          TupleTableSlot *slot)
     {
         TriggerDesc *trigdesc = relinfo->ri_TrigDesc;
         HeapTuple   slottuple = ExecMaterializeSlot(slot);
         HeapTuple   newtuple = slottuple;
         HeapTuple   oldtuple;
         TriggerData LocTriggerData;
         int         i;
     
         LocTriggerData.type = T_TriggerData;
         LocTriggerData.tg_event = TRIGGER_EVENT_INSERT |
             TRIGGER_EVENT_ROW |
             TRIGGER_EVENT_BEFORE;
         LocTriggerData.tg_relation = relinfo->ri_RelationDesc;
         LocTriggerData.tg_newtuple = NULL;
         LocTriggerData.tg_oldtable = NULL;
         LocTriggerData.tg_newtable = NULL;
         LocTriggerData.tg_newtuplebuf = InvalidBuffer;
         for (i = 0; i < trigdesc->numtriggers; i++)
         {
             Trigger    *trigger = &trigdesc->triggers[i];
     
             if (!TRIGGER_TYPE_MATCHES(trigger->tgtype,
                                       TRIGGER_TYPE_ROW,
                                       TRIGGER_TYPE_BEFORE,
                                       TRIGGER_TYPE_INSERT))
                 continue;
             if (!TriggerEnabled(estate, relinfo, trigger, LocTriggerData.tg_event,
                                 NULL, NULL, newtuple))
                 continue;
     
             LocTriggerData.tg_trigtuple = oldtuple = newtuple;
             LocTriggerData.tg_trigtuplebuf = InvalidBuffer;
             LocTriggerData.tg_trigger = trigger;
             newtuple = ExecCallTriggerFunc(&LocTriggerData,
                                            i,
                                            relinfo->ri_TrigFunctions,
                                            relinfo->ri_TrigInstrument,
                                            GetPerTupleMemoryContext(estate));
             if (oldtuple != newtuple && oldtuple != slottuple)
                 heap_freetuple(oldtuple);
             if (newtuple == NULL)
                 return NULL;        /* "do nothing" */
         }
     
         if (newtuple != slottuple)
         {
             /*
              * Return the modified tuple using the es_trig_tuple_slot.  We assume
              * the tuple was allocated in per-tuple memory context, and therefore
              * will go away by itself. The tuple table slot should not try to
              * clear it.
              */
             TupleTableSlot *newslot = estate->es_trig_tuple_slot;
             TupleDesc   tupdesc = RelationGetDescr(relinfo->ri_RelationDesc);
     
             if (newslot->tts_tupleDescriptor != tupdesc)
                 ExecSetSlotDescriptor(newslot, tupdesc);
             ExecStoreTuple(newtuple, newslot, InvalidBuffer, false);
             slot = newslot;
         }
         return slot;
     }
    

_4、ExecIRInsertTriggers_

    
    
     TupleTableSlot *
     ExecIRInsertTriggers(EState *estate, ResultRelInfo *relinfo,
                          TupleTableSlot *slot)
     {
         TriggerDesc *trigdesc = relinfo->ri_TrigDesc;
         HeapTuple   slottuple = ExecMaterializeSlot(slot);
         HeapTuple   newtuple = slottuple;
         HeapTuple   oldtuple;
         TriggerData LocTriggerData;
         int         i;
     
         LocTriggerData.type = T_TriggerData;
         LocTriggerData.tg_event = TRIGGER_EVENT_INSERT |
             TRIGGER_EVENT_ROW |
             TRIGGER_EVENT_INSTEAD;
         LocTriggerData.tg_relation = relinfo->ri_RelationDesc;
         LocTriggerData.tg_newtuple = NULL;
         LocTriggerData.tg_oldtable = NULL;
         LocTriggerData.tg_newtable = NULL;
         LocTriggerData.tg_newtuplebuf = InvalidBuffer;
         for (i = 0; i < trigdesc->numtriggers; i++)
         {
             Trigger    *trigger = &trigdesc->triggers[i];
     
             if (!TRIGGER_TYPE_MATCHES(trigger->tgtype,
                                       TRIGGER_TYPE_ROW,
                                       TRIGGER_TYPE_INSTEAD,
                                       TRIGGER_TYPE_INSERT))
                 continue;
             if (!TriggerEnabled(estate, relinfo, trigger, LocTriggerData.tg_event,
                                 NULL, NULL, newtuple))
                 continue;
     
             LocTriggerData.tg_trigtuple = oldtuple = newtuple;
             LocTriggerData.tg_trigtuplebuf = InvalidBuffer;
             LocTriggerData.tg_trigger = trigger;
             newtuple = ExecCallTriggerFunc(&LocTriggerData,
                                            i,
                                            relinfo->ri_TrigFunctions,
                                            relinfo->ri_TrigInstrument,
                                            GetPerTupleMemoryContext(estate));
             if (oldtuple != newtuple && oldtuple != slottuple)
                 heap_freetuple(oldtuple);
             if (newtuple == NULL)
                 return NULL;        /* "do nothing" */
         }
     
         if (newtuple != slottuple)
         {
             /*
              * Return the modified tuple using the es_trig_tuple_slot.  We assume
              * the tuple was allocated in per-tuple memory context, and therefore
              * will go away by itself. The tuple table slot should not try to
              * clear it.
              */
             TupleTableSlot *newslot = estate->es_trig_tuple_slot;
             TupleDesc   tupdesc = RelationGetDescr(relinfo->ri_RelationDesc);
     
             if (newslot->tts_tupleDescriptor != tupdesc)
                 ExecSetSlotDescriptor(newslot, tupdesc);
             ExecStoreTuple(newtuple, newslot, InvalidBuffer, false);
             slot = newslot;
         }
         return slot;
     }
     
    

_5、ExecForeignInsert_

    
    
    -- 函数指针
      typedef TupleTableSlot *(*ExecForeignInsert_function) (EState *estate,
                                                            ResultRelInfo *rinfo,
                                                            TupleTableSlot *slot,
                                                            TupleTableSlot *planSlot);
    
      ExecForeignInsert_function ExecForeignInsert;
    
    

_6、RelationGetRelid_

    
    
     /*
      * RelationGetRelid
      *      Returns the OID of the relation
      */
     #define RelationGetRelid(relation) ((relation)->rd_id)
    

_7、ExecWithCheckOptions_

    
    
    /*
      * ExecWithCheckOptions -- check that tuple satisfies any WITH CHECK OPTIONs
      * of the specified kind.
      *
      * Note that this needs to be called multiple times to ensure that all kinds of
      * WITH CHECK OPTIONs are handled (both those from views which have the WITH
      * CHECK OPTION set and from row level security policies).  See ExecInsert()
      * and ExecUpdate().
      */
     void
     ExecWithCheckOptions(WCOKind kind, ResultRelInfo *resultRelInfo,
                          TupleTableSlot *slot, EState *estate)
     {
         Relation    rel = resultRelInfo->ri_RelationDesc;
         TupleDesc   tupdesc = RelationGetDescr(rel);
         ExprContext *econtext;
         ListCell   *l1,
                    *l2;
     
         /*
          * We will use the EState's per-tuple context for evaluating constraint
          * expressions (creating it if it's not already there).
          */
         econtext = GetPerTupleExprContext(estate);
     
         /* Arrange for econtext's scan tuple to be the tuple under test */
         econtext->ecxt_scantuple = slot;
     
         /* Check each of the constraints */
         forboth(l1, resultRelInfo->ri_WithCheckOptions,
                 l2, resultRelInfo->ri_WithCheckOptionExprs)
         {
             WithCheckOption *wco = (WithCheckOption *) lfirst(l1);
             ExprState  *wcoExpr = (ExprState *) lfirst(l2);
     
             /*
              * Skip any WCOs which are not the kind we are looking for at this
              * time.
              */
             if (wco->kind != kind)
                 continue;
     
             /*
              * WITH CHECK OPTION checks are intended to ensure that the new tuple
              * is visible (in the case of a view) or that it passes the
              * 'with-check' policy (in the case of row security). If the qual
              * evaluates to NULL or FALSE, then the new tuple won't be included in
              * the view or doesn't pass the 'with-check' policy for the table.
              */
             if (!ExecQual(wcoExpr, econtext))
             {
                 char       *val_desc;
                 Bitmapset  *modifiedCols;
                 Bitmapset  *insertedCols;
                 Bitmapset  *updatedCols;
     
                 switch (wco->kind)
                 {
                         /*
                          * For WITH CHECK OPTIONs coming from views, we might be
                          * able to provide the details on the row, depending on
                          * the permissions on the relation (that is, if the user
                          * could view it directly anyway).  For RLS violations, we
                          * don't include the data since we don't know if the user
                          * should be able to view the tuple as that depends on the
                          * USING policy.
                          */
                     case WCO_VIEW_CHECK:
                         /* See the comment in ExecConstraints(). */
                         if (resultRelInfo->ri_PartitionRoot)
                         {
                             HeapTuple   tuple = ExecFetchSlotTuple(slot);
                             TupleDesc   old_tupdesc = RelationGetDescr(rel);
                             TupleConversionMap *map;
     
                             rel = resultRelInfo->ri_PartitionRoot;
                             tupdesc = RelationGetDescr(rel);
                             /* a reverse map */
                             map = convert_tuples_by_name(old_tupdesc, tupdesc,
                                                          gettext_noop("could not convert row type"));
                             if (map != NULL)
                             {
                                 tuple = do_convert_tuple(tuple, map);
                                 ExecSetSlotDescriptor(slot, tupdesc);
                                 ExecStoreTuple(tuple, slot, InvalidBuffer, false);
                             }
                         }
     
                         insertedCols = GetInsertedColumns(resultRelInfo, estate);
                         updatedCols = GetUpdatedColumns(resultRelInfo, estate);
                         modifiedCols = bms_union(insertedCols, updatedCols);
                         val_desc = ExecBuildSlotValueDescription(RelationGetRelid(rel),
                                                                  slot,
                                                                  tupdesc,
                                                                  modifiedCols,
                                                                  64);
     
                         ereport(ERROR,
                                 (errcode(ERRCODE_WITH_CHECK_OPTION_VIOLATION),
                                  errmsg("new row violates check option for view \"%s\"",
                                         wco->relname),
                                  val_desc ? errdetail("Failing row contains %s.",
                                                       val_desc) : 0));
                         break;
                     case WCO_RLS_INSERT_CHECK:
                     case WCO_RLS_UPDATE_CHECK:
                         if (wco->polname != NULL)
                             ereport(ERROR,
                                     (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                                      errmsg("new row violates row-level security policy \"%s\" for table \"%s\"",
                                             wco->polname, wco->relname)));
                         else
                             ereport(ERROR,
                                     (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                                      errmsg("new row violates row-level security policy for table \"%s\"",
                                             wco->relname)));
                         break;
                     case WCO_RLS_CONFLICT_CHECK:
                         if (wco->polname != NULL)
                             ereport(ERROR,
                                     (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                                      errmsg("new row violates row-level security policy \"%s\" (USING expression) for table \"%s\"",
                                             wco->polname, wco->relname)));
                         else
                             ereport(ERROR,
                                     (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                                      errmsg("new row violates row-level security policy (USING expression) for table \"%s\"",
                                             wco->relname)));
                         break;
                     default:
                         elog(ERROR, "unrecognized WCO kind: %u", wco->kind);
                         break;
                 }
             }
         }
     }
    

_8、ExecConstraints_

    
    
     /*
      * ExecConstraints - check constraints of the tuple in 'slot'
      *
      * This checks the traditional NOT NULL and check constraints.
      *
      * The partition constraint is *NOT* checked.
      *
      * Note: 'slot' contains the tuple to check the constraints of, which may
      * have been converted from the original input tuple after tuple routing.
      * 'resultRelInfo' is the final result relation, after tuple routing.
      */
     void
     ExecConstraints(ResultRelInfo *resultRelInfo,
                     TupleTableSlot *slot, EState *estate)
     {
         Relation    rel = resultRelInfo->ri_RelationDesc;
         TupleDesc   tupdesc = RelationGetDescr(rel);
         TupleConstr *constr = tupdesc->constr;
         Bitmapset  *modifiedCols;
         Bitmapset  *insertedCols;
         Bitmapset  *updatedCols;
     
         Assert(constr || resultRelInfo->ri_PartitionCheck);
     
         if (constr && constr->has_not_null)
         {
             int         natts = tupdesc->natts;
             int         attrChk;
     
             for (attrChk = 1; attrChk <= natts; attrChk++)
             {
                 Form_pg_attribute att = TupleDescAttr(tupdesc, attrChk - 1);
     
                 if (att->attnotnull && slot_attisnull(slot, attrChk))
                 {
                     char       *val_desc;
                     Relation    orig_rel = rel;
                     TupleDesc   orig_tupdesc = RelationGetDescr(rel);
     
                     /*
                      * If the tuple has been routed, it's been converted to the
                      * partition's rowtype, which might differ from the root
                      * table's.  We must convert it back to the root table's
                      * rowtype so that val_desc shown error message matches the
                      * input tuple.
                      */
                     if (resultRelInfo->ri_PartitionRoot)
                     {
                         HeapTuple   tuple = ExecFetchSlotTuple(slot);
                         TupleConversionMap *map;
     
                         rel = resultRelInfo->ri_PartitionRoot;
                         tupdesc = RelationGetDescr(rel);
                         /* a reverse map */
                         map = convert_tuples_by_name(orig_tupdesc, tupdesc,
                                                      gettext_noop("could not convert row type"));
                         if (map != NULL)
                         {
                             tuple = do_convert_tuple(tuple, map);
                             ExecSetSlotDescriptor(slot, tupdesc);
                             ExecStoreTuple(tuple, slot, InvalidBuffer, false);
                         }
                     }
     
                     insertedCols = GetInsertedColumns(resultRelInfo, estate);
                     updatedCols = GetUpdatedColumns(resultRelInfo, estate);
                     modifiedCols = bms_union(insertedCols, updatedCols);
                     val_desc = ExecBuildSlotValueDescription(RelationGetRelid(rel),
                                                              slot,
                                                              tupdesc,
                                                              modifiedCols,
                                                              64);
     
                     ereport(ERROR,
                             (errcode(ERRCODE_NOT_NULL_VIOLATION),
                              errmsg("null value in column \"%s\" violates not-null constraint",
                                     NameStr(att->attname)),
                              val_desc ? errdetail("Failing row contains %s.", val_desc) : 0,
                              errtablecol(orig_rel, attrChk)));
                 }
             }
         }
     
         if (constr && constr->num_check > 0)
         {
             const char *failed;
     
             if ((failed = ExecRelCheck(resultRelInfo, slot, estate)) != NULL)
             {
                 char       *val_desc;
                 Relation    orig_rel = rel;
     
                 /* See the comment above. */
                 if (resultRelInfo->ri_PartitionRoot)
                 {
                     HeapTuple   tuple = ExecFetchSlotTuple(slot);
                     TupleDesc   old_tupdesc = RelationGetDescr(rel);
                     TupleConversionMap *map;
     
                     rel = resultRelInfo->ri_PartitionRoot;
                     tupdesc = RelationGetDescr(rel);
                     /* a reverse map */
                     map = convert_tuples_by_name(old_tupdesc, tupdesc,
                                                  gettext_noop("could not convert row type"));
                     if (map != NULL)
                     {
                         tuple = do_convert_tuple(tuple, map);
                         ExecSetSlotDescriptor(slot, tupdesc);
                         ExecStoreTuple(tuple, slot, InvalidBuffer, false);
                     }
                 }
     
                 insertedCols = GetInsertedColumns(resultRelInfo, estate);
                 updatedCols = GetUpdatedColumns(resultRelInfo, estate);
                 modifiedCols = bms_union(insertedCols, updatedCols);
                 val_desc = ExecBuildSlotValueDescription(RelationGetRelid(rel),
                                                          slot,
                                                          tupdesc,
                                                          modifiedCols,
                                                          64);
                 ereport(ERROR,
                         (errcode(ERRCODE_CHECK_VIOLATION),
                          errmsg("new row for relation \"%s\" violates check constraint \"%s\"",
                                 RelationGetRelationName(orig_rel), failed),
                          val_desc ? errdetail("Failing row contains %s.", val_desc) : 0,
                          errtableconstraint(orig_rel, failed)));
             }
         }
     }
    

_9、ExecPartitionCheck_

    
    
     /*
      * ExecPartitionCheck --- check that tuple meets the partition constraint.
      *
      * Returns true if it meets the partition constraint.  If the constraint
      * fails and we're asked to emit to error, do so and don't return; otherwise
      * return false.
      */
     bool
     ExecPartitionCheck(ResultRelInfo *resultRelInfo, TupleTableSlot *slot,
                        EState *estate, bool emitError)
     {
         ExprContext *econtext;
         bool        success;
     
         /*
          * If first time through, build expression state tree for the partition
          * check expression.  Keep it in the per-query memory context so they'll
          * survive throughout the query.
          */
         if (resultRelInfo->ri_PartitionCheckExpr == NULL)
         {
             List       *qual = resultRelInfo->ri_PartitionCheck;
     
             resultRelInfo->ri_PartitionCheckExpr = ExecPrepareCheck(qual, estate);
         }
     
         /*
          * We will use the EState's per-tuple context for evaluating constraint
          * expressions (creating it if it's not already there).
          */
         econtext = GetPerTupleExprContext(estate);
     
         /* Arrange for econtext's scan tuple to be the tuple under test */
         econtext->ecxt_scantuple = slot;
     
         /*
          * As in case of the catalogued constraints, we treat a NULL result as
          * success here, not a failure.
          */
         success = ExecCheck(resultRelInfo->ri_PartitionCheckExpr, econtext);
     
         /* if asked to emit error, don't actually return on failure */
         if (!success && emitError)
             ExecPartitionCheckEmitError(resultRelInfo, slot, estate);
     
         return success;
     }
     
    

_10、ExecCheckIndexConstraints_

    
    
     /* ----------------------------------------------------------------      *      ExecCheckIndexConstraints
      *
      *      This routine checks if a tuple violates any unique or
      *      exclusion constraints.  Returns true if there is no conflict.
      *      Otherwise returns false, and the TID of the conflicting
      *      tuple is returned in *conflictTid.
      *
      *      If 'arbiterIndexes' is given, only those indexes are checked.
      *      NIL means all indexes.
      *
      *      Note that this doesn't lock the values in any way, so it's
      *      possible that a conflicting tuple is inserted immediately
      *      after this returns.  But this can be used for a pre-check
      *      before insertion.
      * ----------------------------------------------------------------      */
     bool
     ExecCheckIndexConstraints(TupleTableSlot *slot,
                               EState *estate, ItemPointer conflictTid,
                               List *arbiterIndexes)
     {
         ResultRelInfo *resultRelInfo;
         int         i;
         int         numIndices;
         RelationPtr relationDescs;
         Relation    heapRelation;
         IndexInfo **indexInfoArray;
         ExprContext *econtext;
         Datum       values[INDEX_MAX_KEYS];
         bool        isnull[INDEX_MAX_KEYS];
         ItemPointerData invalidItemPtr;
         bool        checkedIndex = false;
     
         ItemPointerSetInvalid(conflictTid);
         ItemPointerSetInvalid(&invalidItemPtr);
     
         /*
          * Get information from the result relation info structure.
          */
         resultRelInfo = estate->es_result_relation_info;
         numIndices = resultRelInfo->ri_NumIndices;
         relationDescs = resultRelInfo->ri_IndexRelationDescs;
         indexInfoArray = resultRelInfo->ri_IndexRelationInfo;
         heapRelation = resultRelInfo->ri_RelationDesc;
     
         /*
          * We will use the EState's per-tuple context for evaluating predicates
          * and index expressions (creating it if it's not already there).
          */
         econtext = GetPerTupleExprContext(estate);
     
         /* Arrange for econtext's scan tuple to be the tuple under test */
         econtext->ecxt_scantuple = slot;
     
         /*
          * For each index, form index tuple and check if it satisfies the
          * constraint.
          */
         for (i = 0; i < numIndices; i++)
         {
             Relation    indexRelation = relationDescs[i];
             IndexInfo  *indexInfo;
             bool        satisfiesConstraint;
     
             if (indexRelation == NULL)
                 continue;
     
             indexInfo = indexInfoArray[i];
     
             if (!indexInfo->ii_Unique && !indexInfo->ii_ExclusionOps)
                 continue;
     
             /* If the index is marked as read-only, ignore it */
             if (!indexInfo->ii_ReadyForInserts)
                 continue;
     
             /* When specific arbiter indexes requested, only examine them */
             if (arbiterIndexes != NIL &&
                 !list_member_oid(arbiterIndexes,
                                  indexRelation->rd_index->indexrelid))
                 continue;
     
             if (!indexRelation->rd_index->indimmediate)
                 ereport(ERROR,
                         (errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
                          errmsg("ON CONFLICT does not support deferrable unique constraints/exclusion constraints as arbiters"),
                          errtableconstraint(heapRelation,
                                             RelationGetRelationName(indexRelation))));
     
             checkedIndex = true;
     
             /* Check for partial index */
             if (indexInfo->ii_Predicate != NIL)
             {
                 ExprState  *predicate;
     
                 /*
                  * If predicate state not set up yet, create it (in the estate's
                  * per-query context)
                  */
                 predicate = indexInfo->ii_PredicateState;
                 if (predicate == NULL)
                 {
                     predicate = ExecPrepareQual(indexInfo->ii_Predicate, estate);
                     indexInfo->ii_PredicateState = predicate;
                 }
     
                 /* Skip this index-update if the predicate isn't satisfied */
                 if (!ExecQual(predicate, econtext))
                     continue;
             }
     
             /*
              * FormIndexDatum fills in its values and isnull parameters with the
              * appropriate values for the column(s) of the index.
              */
             FormIndexDatum(indexInfo,
                            slot,
                            estate,
                            values,
                            isnull);
     
             satisfiesConstraint =
                 check_exclusion_or_unique_constraint(heapRelation, indexRelation,
                                                      indexInfo, &invalidItemPtr,
                                                      values, isnull, estate, false,
                                                      CEOUC_WAIT, true,
                                                      conflictTid);
             if (!satisfiesConstraint)
                 return false;
         }
     
         if (arbiterIndexes != NIL && !checkedIndex)
             elog(ERROR, "unexpected failure to find arbiter index");
     
         return true;
     }
    

_11、ExecOnConflictUpdate_

    
    
    /*
      * ExecOnConflictUpdate --- execute UPDATE of INSERT ON CONFLICT DO UPDATE
      *
      * Try to lock tuple for update as part of speculative insertion.  If
      * a qual originating from ON CONFLICT DO UPDATE is satisfied, update
      * (but still lock row, even though it may not satisfy estate's
      * snapshot).
      *
      * Returns true if we're done (with or without an update), or false if
      * the caller must retry the INSERT from scratch.
      */
     static bool
     ExecOnConflictUpdate(ModifyTableState *mtstate,
                          ResultRelInfo *resultRelInfo,
                          ItemPointer conflictTid,
                          TupleTableSlot *planSlot,
                          TupleTableSlot *excludedSlot,
                          EState *estate,
                          bool canSetTag,
                          TupleTableSlot **returning)
     {
         ExprContext *econtext = mtstate->ps.ps_ExprContext;
         Relation    relation = resultRelInfo->ri_RelationDesc;
         ExprState  *onConflictSetWhere = resultRelInfo->ri_onConflict->oc_WhereClause;
         HeapTupleData tuple;
         HeapUpdateFailureData hufd;
         LockTupleMode lockmode;
         HTSU_Result test;
         Buffer      buffer;
     
         /* Determine lock mode to use */
         lockmode = ExecUpdateLockMode(estate, resultRelInfo);
     
         /*
          * Lock tuple for update.  Don't follow updates when tuple cannot be
          * locked without doing so.  A row locking conflict here means our
          * previous conclusion that the tuple is conclusively committed is not
          * true anymore.
          */
         tuple.t_self = *conflictTid;
         test = heap_lock_tuple(relation, &tuple, estate->es_output_cid,
                                lockmode, LockWaitBlock, false, &buffer,
                                &hufd);
         switch (test)
         {
             case HeapTupleMayBeUpdated:
                 /* success! */
                 break;
     
             case HeapTupleInvisible:
     
                 /*
                  * This can occur when a just inserted tuple is updated again in
                  * the same command. E.g. because multiple rows with the same
                  * conflicting key values are inserted.
                  *
                  * This is somewhat similar to the ExecUpdate()
                  * HeapTupleSelfUpdated case.  We do not want to proceed because
                  * it would lead to the same row being updated a second time in
                  * some unspecified order, and in contrast to plain UPDATEs
                  * there's no historical behavior to break.
                  *
                  * It is the user's responsibility to prevent this situation from
                  * occurring.  These problems are why SQL-2003 similarly specifies
                  * that for SQL MERGE, an exception must be raised in the event of
                  * an attempt to update the same row twice.
                  */
                 if (TransactionIdIsCurrentTransactionId(HeapTupleHeaderGetXmin(tuple.t_data)))
                     ereport(ERROR,
                             (errcode(ERRCODE_CARDINALITY_VIOLATION),
                              errmsg("ON CONFLICT DO UPDATE command cannot affect row a second time"),
                              errhint("Ensure that no rows proposed for insertion within the same command have duplicate constrained values.")));
     
                 /* This shouldn't happen */
                 elog(ERROR, "attempted to lock invisible tuple");
                 break;
     
             case HeapTupleSelfUpdated:
     
                 /*
                  * This state should never be reached. As a dirty snapshot is used
                  * to find conflicting tuples, speculative insertion wouldn't have
                  * seen this row to conflict with.
                  */
                 elog(ERROR, "unexpected self-updated tuple");
                 break;
     
             case HeapTupleUpdated:
                 if (IsolationUsesXactSnapshot())
                     ereport(ERROR,
                             (errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),
                              errmsg("could not serialize access due to concurrent update")));
     
                 /*
                  * As long as we don't support an UPDATE of INSERT ON CONFLICT for
                  * a partitioned table we shouldn't reach to a case where tuple to
                  * be lock is moved to another partition due to concurrent update
                  * of the partition key.
                  */
                 Assert(!ItemPointerIndicatesMovedPartitions(&hufd.ctid));
     
                 /*
                  * Tell caller to try again from the very start.
                  *
                  * It does not make sense to use the usual EvalPlanQual() style
                  * loop here, as the new version of the row might not conflict
                  * anymore, or the conflicting tuple has actually been deleted.
                  */
                 ReleaseBuffer(buffer);
                 return false;
     
             default:
                 elog(ERROR, "unrecognized heap_lock_tuple status: %u", test);
         }
     
         /*
          * Success, the tuple is locked.
          *
          * Reset per-tuple memory context to free any expression evaluation
          * storage allocated in the previous cycle.
          */
         ResetExprContext(econtext);
     
         /*
          * Verify that the tuple is visible to our MVCC snapshot if the current
          * isolation level mandates that.
          *
          * It's not sufficient to rely on the check within ExecUpdate() as e.g.
          * CONFLICT ... WHERE clause may prevent us from reaching that.
          *
          * This means we only ever continue when a new command in the current
          * transaction could see the row, even though in READ COMMITTED mode the
          * tuple will not be visible according to the current statement's
          * snapshot.  This is in line with the way UPDATE deals with newer tuple
          * versions.
          */
         ExecCheckHeapTupleVisible(estate, &tuple, buffer);
     
         /* Store target's existing tuple in the state's dedicated slot */
         ExecStoreTuple(&tuple, mtstate->mt_existing, buffer, false);
     
         /*
          * Make tuple and any needed join variables available to ExecQual and
          * ExecProject.  The EXCLUDED tuple is installed in ecxt_innertuple, while
          * the target's existing tuple is installed in the scantuple.  EXCLUDED
          * has been made to reference INNER_VAR in setrefs.c, but there is no
          * other redirection.
          */
         econtext->ecxt_scantuple = mtstate->mt_existing;
         econtext->ecxt_innertuple = excludedSlot;
         econtext->ecxt_outertuple = NULL;
     
         if (!ExecQual(onConflictSetWhere, econtext))
         {
             ReleaseBuffer(buffer);
             InstrCountFiltered1(&mtstate->ps, 1);
             return true;            /* done with the tuple */
         }
     
         if (resultRelInfo->ri_WithCheckOptions != NIL)
         {
             /*
              * Check target's existing tuple against UPDATE-applicable USING
              * security barrier quals (if any), enforced here as RLS checks/WCOs.
              *
              * The rewriter creates UPDATE RLS checks/WCOs for UPDATE security
              * quals, and stores them as WCOs of "kind" WCO_RLS_CONFLICT_CHECK,
              * but that's almost the extent of its special handling for ON
              * CONFLICT DO UPDATE.
              *
              * The rewriter will also have associated UPDATE applicable straight
              * RLS checks/WCOs for the benefit of the ExecUpdate() call that
              * follows.  INSERTs and UPDATEs naturally have mutually exclusive WCO
              * kinds, so there is no danger of spurious over-enforcement in the
              * INSERT or UPDATE path.
              */
             ExecWithCheckOptions(WCO_RLS_CONFLICT_CHECK, resultRelInfo,
                                  mtstate->mt_existing,
                                  mtstate->ps.state);
         }
     
         /* Project the new tuple version */
         ExecProject(resultRelInfo->ri_onConflict->oc_ProjInfo);
     
         /*
          * Note that it is possible that the target tuple has been modified in
          * this session, after the above heap_lock_tuple. We choose to not error
          * out in that case, in line with ExecUpdate's treatment of similar cases.
          * This can happen if an UPDATE is triggered from within ExecQual(),
          * ExecWithCheckOptions() or ExecProject() above, e.g. by selecting from a
          * wCTE in the ON CONFLICT's SET.
          */
     
         /* Execute UPDATE with projection */
         *returning = ExecUpdate(mtstate, &tuple.t_self, NULL,
                                 mtstate->mt_conflproj, planSlot,
                                 &mtstate->mt_epqstate, mtstate->ps.state,
                                 canSetTag);
     
         ReleaseBuffer(buffer);
         return true;
     }
    

_12、InstrCountTuples2_

    
    
     /* Macros for inline access to certain instrumentation counters */
     #define InstrCountTuples2(node, delta) \
         do { \
             if (((PlanState *)(node))->instrument) \
                 ((PlanState *)(node))->instrument->ntuples2 += (delta); \
         } while (0)
    

_13、ExecCheckTIDVisible_

    
    
     /*
      * ExecCheckTIDVisible -- convenience variant of ExecCheckHeapTupleVisible()
      */
     static void
     ExecCheckTIDVisible(EState *estate,
                         ResultRelInfo *relinfo,
                         ItemPointer tid)
     {
         Relation    rel = relinfo->ri_RelationDesc;
         Buffer      buffer;
         HeapTupleData tuple;
     
         /* Redundantly check isolation level */
         if (!IsolationUsesXactSnapshot())
             return;
     
         tuple.t_self = *tid;
         if (!heap_fetch(rel, SnapshotAny, &tuple, &buffer, false, NULL))
             elog(ERROR, "failed to fetch conflicting tuple for ON CONFLICT");
         ExecCheckHeapTupleVisible(estate, &tuple, buffer);
         ReleaseBuffer(buffer);
     }
     
    

_14、SpeculativeInsertionLockAcquire_

    
    
    /*
      *      SpeculativeInsertionLockAcquire
      *
      * Insert a lock showing that the given transaction ID is inserting a tuple,
      * but hasn't yet decided whether it's going to keep it.  The lock can then be
      * used to wait for the decision to go ahead with the insertion, or aborting
      * it.
      *
      * The token is used to distinguish multiple insertions by the same
      * transaction.  It is returned to caller.
      */
     uint32
     SpeculativeInsertionLockAcquire(TransactionId xid)
     {
         LOCKTAG     tag;
     
         speculativeInsertionToken++;
     
         /*
          * Check for wrap-around. Zero means no token is held, so don't use that.
          */
         if (speculativeInsertionToken == 0)
             speculativeInsertionToken = 1;
     
         SET_LOCKTAG_SPECULATIVE_INSERTION(tag, xid, speculativeInsertionToken);
     
         (void) LockAcquire(&tag, ExclusiveLock, false, false);
     
         return speculativeInsertionToken;
     }
    

_15、HeapTupleHeaderSetSpeculativeToken_

    
    
     #define HeapTupleHeaderSetSpeculativeToken(tup, token)  \
     ( \
         ItemPointerSet(&(tup)->t_ctid, token, SpecTokenOffsetNumber) \
     )
    

_16、heap_insert_

    
    
    //上一节已介绍
    

_17、ExecInsertIndexTuples_

    
    
     /* ----------------------------------------------------------------      *      ExecInsertIndexTuples
      *
      *      This routine takes care of inserting index tuples
      *      into all the relations indexing the result relation
      *      when a heap tuple is inserted into the result relation.
      *
      *      Unique and exclusion constraints are enforced at the same
      *      time.  This returns a list of index OIDs for any unique or
      *      exclusion constraints that are deferred and that had
      *      potential (unconfirmed) conflicts.  (if noDupErr == true,
      *      the same is done for non-deferred constraints, but report
      *      if conflict was speculative or deferred conflict to caller)
      *
      *      If 'arbiterIndexes' is nonempty, noDupErr applies only to
      *      those indexes.  NIL means noDupErr applies to all indexes.
      *
      *      CAUTION: this must not be called for a HOT update.
      *      We can't defend against that here for lack of info.
      *      Should we change the API to make it safer?
      * ----------------------------------------------------------------      */
     List *
     ExecInsertIndexTuples(TupleTableSlot *slot,
                           ItemPointer tupleid,
                           EState *estate,
                           bool noDupErr,
                           bool *specConflict,
                           List *arbiterIndexes)
     {
         List       *result = NIL;
         ResultRelInfo *resultRelInfo;
         int         i;
         int         numIndices;
         RelationPtr relationDescs;
         Relation    heapRelation;
         IndexInfo **indexInfoArray;
         ExprContext *econtext;
         Datum       values[INDEX_MAX_KEYS];
         bool        isnull[INDEX_MAX_KEYS];
     
         /*
          * Get information from the result relation info structure.
          */
         resultRelInfo = estate->es_result_relation_info;
         numIndices = resultRelInfo->ri_NumIndices;
         relationDescs = resultRelInfo->ri_IndexRelationDescs;
         indexInfoArray = resultRelInfo->ri_IndexRelationInfo;
         heapRelation = resultRelInfo->ri_RelationDesc;
     
         /*
          * We will use the EState's per-tuple context for evaluating predicates
          * and index expressions (creating it if it's not already there).
          */
         econtext = GetPerTupleExprContext(estate);
     
         /* Arrange for econtext's scan tuple to be the tuple under test */
         econtext->ecxt_scantuple = slot;
     
         /*
          * for each index, form and insert the index tuple
          */
         for (i = 0; i < numIndices; i++)
         {
             Relation    indexRelation = relationDescs[i];
             IndexInfo  *indexInfo;
             bool        applyNoDupErr;
             IndexUniqueCheck checkUnique;
             bool        satisfiesConstraint;
     
             if (indexRelation == NULL)
                 continue;
     
             indexInfo = indexInfoArray[i];
     
             /* If the index is marked as read-only, ignore it */
             if (!indexInfo->ii_ReadyForInserts)
                 continue;
     
             /* Check for partial index */
             if (indexInfo->ii_Predicate != NIL)
             {
                 ExprState  *predicate;
     
                 /*
                  * If predicate state not set up yet, create it (in the estate's
                  * per-query context)
                  */
                 predicate = indexInfo->ii_PredicateState;
                 if (predicate == NULL)
                 {
                     predicate = ExecPrepareQual(indexInfo->ii_Predicate, estate);
                     indexInfo->ii_PredicateState = predicate;
                 }
     
                 /* Skip this index-update if the predicate isn't satisfied */
                 if (!ExecQual(predicate, econtext))
                     continue;
             }
     
             /*
              * FormIndexDatum fills in its values and isnull parameters with the
              * appropriate values for the column(s) of the index.
              */
             FormIndexDatum(indexInfo,
                            slot,
                            estate,
                            values,
                            isnull);
     
             /* Check whether to apply noDupErr to this index */
             applyNoDupErr = noDupErr &&
                 (arbiterIndexes == NIL ||
                  list_member_oid(arbiterIndexes,
                                  indexRelation->rd_index->indexrelid));
     
             /*
              * The index AM does the actual insertion, plus uniqueness checking.
              *
              * For an immediate-mode unique index, we just tell the index AM to
              * throw error if not unique.
              *
              * For a deferrable unique index, we tell the index AM to just detect
              * possible non-uniqueness, and we add the index OID to the result
              * list if further checking is needed.
              *
              * For a speculative insertion (used by INSERT ... ON CONFLICT), do
              * the same as for a deferrable unique index.
              */
             if (!indexRelation->rd_index->indisunique)
                 checkUnique = UNIQUE_CHECK_NO;
             else if (applyNoDupErr)
                 checkUnique = UNIQUE_CHECK_PARTIAL;
             else if (indexRelation->rd_index->indimmediate)
                 checkUnique = UNIQUE_CHECK_YES;
             else
                 checkUnique = UNIQUE_CHECK_PARTIAL;
     
             satisfiesConstraint =
                 index_insert(indexRelation, /* index relation */
                              values,    /* array of index Datums */
                              isnull,    /* null flags */
                              tupleid,   /* tid of heap tuple */
                              heapRelation,  /* heap relation */
                              checkUnique,   /* type of uniqueness check to do */
                              indexInfo);    /* index AM may need this */
     
             /*
              * If the index has an associated exclusion constraint, check that.
              * This is simpler than the process for uniqueness checks since we
              * always insert first and then check.  If the constraint is deferred,
              * we check now anyway, but don't throw error on violation or wait for
              * a conclusive outcome from a concurrent insertion; instead we'll
              * queue a recheck event.  Similarly, noDupErr callers (speculative
              * inserters) will recheck later, and wait for a conclusive outcome
              * then.
              *
              * An index for an exclusion constraint can't also be UNIQUE (not an
              * essential property, we just don't allow it in the grammar), so no
              * need to preserve the prior state of satisfiesConstraint.
              */
             if (indexInfo->ii_ExclusionOps != NULL)
             {
                 bool        violationOK;
                 CEOUC_WAIT_MODE waitMode;
     
                 if (applyNoDupErr)
                 {
                     violationOK = true;
                     waitMode = CEOUC_LIVELOCK_PREVENTING_WAIT;
                 }
                 else if (!indexRelation->rd_index->indimmediate)
                 {
                     violationOK = true;
                     waitMode = CEOUC_NOWAIT;
                 }
                 else
                 {
                     violationOK = false;
                     waitMode = CEOUC_WAIT;
                 }
     
                 satisfiesConstraint =
                     check_exclusion_or_unique_constraint(heapRelation,
                                                          indexRelation, indexInfo,
                                                          tupleid, values, isnull,
                                                          estate, false,
                                                          waitMode, violationOK, NULL);
             }
     
             if ((checkUnique == UNIQUE_CHECK_PARTIAL ||
                  indexInfo->ii_ExclusionOps != NULL) &&
                 !satisfiesConstraint)
             {
                 /*
                  * The tuple potentially violates the uniqueness or exclusion
                  * constraint, so make a note of the index so that we can re-check
                  * it later.  Speculative inserters are told if there was a
                  * speculative conflict, since that always requires a restart.
                  */
                 result = lappend_oid(result, RelationGetRelid(indexRelation));
                 if (indexRelation->rd_index->indimmediate && specConflict)
                     *specConflict = true;
             }
         }
     
         return result;
     }
     
    

_18、heap_finish_speculative_

    
    
     /*
      *  heap_finish_speculative - mark speculative insertion as successful
      *
      * To successfully finish a speculative insertion we have to clear speculative
      * token from tuple.  To do so the t_ctid field, which will contain a
      * speculative token value, is modified in place to point to the tuple itself,
      * which is characteristic of a newly inserted ordinary tuple.
      *
      * NB: It is not ok to commit without either finishing or aborting a
      * speculative insertion.  We could treat speculative tuples of committed
      * transactions implicitly as completed, but then we would have to be prepared
      * to deal with speculative tokens on committed tuples.  That wouldn't be
      * difficult - no-one looks at the ctid field of a tuple with invalid xmax -      * but clearing the token at completion isn't very expensive either.
      * An explicit confirmation WAL record also makes logical decoding simpler.
      */
     void
     heap_finish_speculative(Relation relation, HeapTuple tuple)
     {
         Buffer      buffer;
         Page        page;
         OffsetNumber offnum;
         ItemId      lp = NULL;
         HeapTupleHeader htup;
     
         buffer = ReadBuffer(relation, ItemPointerGetBlockNumber(&(tuple->t_self)));
         LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
         page = (Page) BufferGetPage(buffer);
     
         offnum = ItemPointerGetOffsetNumber(&(tuple->t_self));
         if (PageGetMaxOffsetNumber(page) >= offnum)
             lp = PageGetItemId(page, offnum);
     
         if (PageGetMaxOffsetNumber(page) < offnum || !ItemIdIsNormal(lp))
             elog(ERROR, "invalid lp");
     
         htup = (HeapTupleHeader) PageGetItem(page, lp);
     
         /* SpecTokenOffsetNumber should be distinguishable from any real offset */
         StaticAssertStmt(MaxOffsetNumber < SpecTokenOffsetNumber,
                          "invalid speculative token constant");
     
         /* NO EREPORT(ERROR) from here till changes are logged */
         START_CRIT_SECTION();
     
         Assert(HeapTupleHeaderIsSpeculative(tuple->t_data));
     
         MarkBufferDirty(buffer);
     
         /*
          * Replace the speculative insertion token with a real t_ctid, pointing to
          * itself like it does on regular tuples.
          */
         htup->t_ctid = tuple->t_self;
     
         /* XLOG stuff */
         if (RelationNeedsWAL(relation))
         {
             xl_heap_confirm xlrec;
             XLogRecPtr  recptr;
     
             xlrec.offnum = ItemPointerGetOffsetNumber(&tuple->t_self);
     
             XLogBeginInsert();
     
             /* We want the same filtering on this as on a plain insert */
             XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
     
             XLogRegisterData((char *) &xlrec, SizeOfHeapConfirm);
             XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
     
             recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_CONFIRM);
     
             PageSetLSN(page, recptr);
         }
     
         END_CRIT_SECTION();
     
         UnlockReleaseBuffer(buffer);
     }
     
    

_19、heap_abort_speculative_

    
    
     /*
      *  heap_abort_speculative - kill a speculatively inserted tuple
      *
      * Marks a tuple that was speculatively inserted in the same command as dead,
      * by setting its xmin as invalid.  That makes it immediately appear as dead
      * to all transactions, including our own.  In particular, it makes
      * HeapTupleSatisfiesDirty() regard the tuple as dead, so that another backend
      * inserting a duplicate key value won't unnecessarily wait for our whole
      * transaction to finish (it'll just wait for our speculative insertion to
      * finish).
      *
      * Killing the tuple prevents "unprincipled deadlocks", which are deadlocks
      * that arise due to a mutual dependency that is not user visible.  By
      * definition, unprincipled deadlocks cannot be prevented by the user
      * reordering lock acquisition in client code, because the implementation level
      * lock acquisitions are not under the user's direct control.  If speculative
      * inserters did not take this precaution, then under high concurrency they
      * could deadlock with each other, which would not be acceptable.
      *
      * This is somewhat redundant with heap_delete, but we prefer to have a
      * dedicated routine with stripped down requirements.  Note that this is also
      * used to delete the TOAST tuples created during speculative insertion.
      *
      * This routine does not affect logical decoding as it only looks at
      * confirmation records.
      */
     void
     heap_abort_speculative(Relation relation, HeapTuple tuple)
     {
         TransactionId xid = GetCurrentTransactionId();
         ItemPointer tid = &(tuple->t_self);
         ItemId      lp;
         HeapTupleData tp;
         Page        page;
         BlockNumber block;
         Buffer      buffer;
     
         Assert(ItemPointerIsValid(tid));
     
         block = ItemPointerGetBlockNumber(tid);
         buffer = ReadBuffer(relation, block);
         page = BufferGetPage(buffer);
     
         LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
     
         /*
          * Page can't be all visible, we just inserted into it, and are still
          * running.
          */
         Assert(!PageIsAllVisible(page));
     
         lp = PageGetItemId(page, ItemPointerGetOffsetNumber(tid));
         Assert(ItemIdIsNormal(lp));
     
         tp.t_tableOid = RelationGetRelid(relation);
         tp.t_data = (HeapTupleHeader) PageGetItem(page, lp);
         tp.t_len = ItemIdGetLength(lp);
         tp.t_self = *tid;
     
         /*
          * Sanity check that the tuple really is a speculatively inserted tuple,
          * inserted by us.
          */
         if (tp.t_data->t_choice.t_heap.t_xmin != xid)
             elog(ERROR, "attempted to kill a tuple inserted by another transaction");
         if (!(IsToastRelation(relation) || HeapTupleHeaderIsSpeculative(tp.t_data)))
             elog(ERROR, "attempted to kill a non-speculative tuple");
         Assert(!HeapTupleHeaderIsHeapOnly(tp.t_data));
     
         /*
          * No need to check for serializable conflicts here.  There is never a
          * need for a combocid, either.  No need to extract replica identity, or
          * do anything special with infomask bits.
          */
     
         START_CRIT_SECTION();
     
         /*
          * The tuple will become DEAD immediately.  Flag that this page
          * immediately is a candidate for pruning by setting xmin to
          * RecentGlobalXmin.  That's not pretty, but it doesn't seem worth
          * inventing a nicer API for this.
          */
         Assert(TransactionIdIsValid(RecentGlobalXmin));
         PageSetPrunable(page, RecentGlobalXmin);
     
         /* store transaction information of xact deleting the tuple */
         tp.t_data->t_infomask &= ~(HEAP_XMAX_BITS | HEAP_MOVED);
         tp.t_data->t_infomask2 &= ~HEAP_KEYS_UPDATED;
     
         /*
          * Set the tuple header xmin to InvalidTransactionId.  This makes the
          * tuple immediately invisible everyone.  (In particular, to any
          * transactions waiting on the speculative token, woken up later.)
          */
         HeapTupleHeaderSetXmin(tp.t_data, InvalidTransactionId);
     
         /* Clear the speculative insertion token too */
         tp.t_data->t_ctid = tp.t_self;
     
         MarkBufferDirty(buffer);
     
         /*
          * XLOG stuff
          *
          * The WAL records generated here match heap_delete().  The same recovery
          * routines are used.
          */
         if (RelationNeedsWAL(relation))
         {
             xl_heap_delete xlrec;
             XLogRecPtr  recptr;
     
             xlrec.flags = XLH_DELETE_IS_SUPER;
             xlrec.infobits_set = compute_infobits(tp.t_data->t_infomask,
                                                   tp.t_data->t_infomask2);
             xlrec.offnum = ItemPointerGetOffsetNumber(&tp.t_self);
             xlrec.xmax = xid;
     
             XLogBeginInsert();
             XLogRegisterData((char *) &xlrec, SizeOfHeapDelete);
             XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
     
             /* No replica identity & replication origin logged */
     
             recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_DELETE);
     
             PageSetLSN(page, recptr);
         }
     
         END_CRIT_SECTION();
     
         LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
     
         if (HeapTupleHasExternal(&tp))
         {
             Assert(!IsToastRelation(relation));
             toast_delete(relation, &tp, true);
         }
     
         /*
          * Never need to mark tuple for invalidation, since catalogs don't support
          * speculative insertion
          */
     
         /* Now we can release the buffer */
         ReleaseBuffer(buffer);
     
         /* count deletion, as we counted the insertion too */
         pgstat_count_heap_delete(relation);
     }
    

_20、SpeculativeInsertionLockRelease_

    
    
    /*
      *      SpeculativeInsertionLockRelease
      *
      * Delete the lock showing that the given transaction is speculatively
      * inserting a tuple.
      */
     void
     SpeculativeInsertionLockRelease(TransactionId xid)
     {
         LOCKTAG     tag;
     
         SET_LOCKTAG_SPECULATIVE_INSERTION(tag, xid, speculativeInsertionToken);
     
         LockRelease(&tag, ExclusiveLock, false);
     }
     
    

_21、list_free_

    
    
     /*
      * Free all the cells of the list, as well as the list itself. Any
      * objects that are pointed-to by the cells of the list are NOT
      * free'd.
      *
      * On return, the argument to this function has been freed, so the
      * caller would be wise to set it to NIL for safety's sake.
      */
     void
     list_free(List *list)
     {
         list_free_private(list, false);
     }
    
     /*
      * Free all storage in a list, and optionally the pointed-to elements
      */
     static void
     list_free_private(List *list, bool deep)
     {
         ListCell   *cell;
     
         check_list_invariants(list);
     
         cell = list_head(list);
         while (cell != NULL)
         {
             ListCell   *tmp = cell;
     
             cell = lnext(cell);
             if (deep)
                 pfree(lfirst(tmp));
             pfree(tmp);
         }
     
         if (list)
             pfree(list);
     }
    

_22、setLastTid_

    
    
     void
     setLastTid(const ItemPointer tid)
     {
         Current_last_tid = *tid;
     }
     
    

_23、ExecARUpdateTriggers_

    
    
     void
     ExecARUpdateTriggers(EState *estate, ResultRelInfo *relinfo,
                          ItemPointer tupleid,
                          HeapTuple fdw_trigtuple,
                          HeapTuple newtuple,
                          List *recheckIndexes,
                          TransitionCaptureState *transition_capture)
     {
         TriggerDesc *trigdesc = relinfo->ri_TrigDesc;
     
         if ((trigdesc && trigdesc->trig_update_after_row) ||
             (transition_capture &&
              (transition_capture->tcs_update_old_table ||
               transition_capture->tcs_update_new_table)))
         {
             HeapTuple   trigtuple;
     
             /*
              * Note: if the UPDATE is converted into a DELETE+INSERT as part of
              * update-partition-key operation, then this function is also called
              * separately for DELETE and INSERT to capture transition table rows.
              * In such case, either old tuple or new tuple can be NULL.
              */
             if (fdw_trigtuple == NULL && ItemPointerIsValid(tupleid))
                 trigtuple = GetTupleForTrigger(estate,
                                                NULL,
                                                relinfo,
                                                tupleid,
                                                LockTupleExclusive,
                                                NULL);
             else
                 trigtuple = fdw_trigtuple;
     
             AfterTriggerSaveEvent(estate, relinfo, TRIGGER_EVENT_UPDATE,
                                   true, trigtuple, newtuple, recheckIndexes,
                                   GetUpdatedColumns(relinfo, estate),
                                   transition_capture);
             if (trigtuple != fdw_trigtuple)
                 heap_freetuple(trigtuple);
         }
     }
     
    

_24、ExecARInsertTriggers_

    
    
     void
     ExecARInsertTriggers(EState *estate, ResultRelInfo *relinfo,
                          HeapTuple trigtuple, List *recheckIndexes,
                          TransitionCaptureState *transition_capture)
     {
         TriggerDesc *trigdesc = relinfo->ri_TrigDesc;
     
         if ((trigdesc && trigdesc->trig_insert_after_row) ||
             (transition_capture && transition_capture->tcs_insert_new_table))
             AfterTriggerSaveEvent(estate, relinfo, TRIGGER_EVENT_INSERT,
                                   true, NULL, trigtuple,
                                   recheckIndexes, NULL,
                                   transition_capture);
     }
    

_25、ExecProcessReturning_

    
    
    /*
      * ExecProcessReturning --- evaluate a RETURNING list
      *
      * resultRelInfo: current result rel
      * tupleSlot: slot holding tuple actually inserted/updated/deleted
      * planSlot: slot holding tuple returned by top subplan node
      *
      * Note: If tupleSlot is NULL, the FDW should have already provided econtext's
      * scan tuple.
      *
      * Returns a slot holding the result tuple
      */
     static TupleTableSlot *
     ExecProcessReturning(ResultRelInfo *resultRelInfo,
                          TupleTableSlot *tupleSlot,
                          TupleTableSlot *planSlot)
     {
         ProjectionInfo *projectReturning = resultRelInfo->ri_projectReturning;
         ExprContext *econtext = projectReturning->pi_exprContext;
     
         /*
          * Reset per-tuple memory context to free any expression evaluation
          * storage allocated in the previous cycle.
          */
         ResetExprContext(econtext);
     
         /* Make tuple and any needed join variables available to ExecProject */
         if (tupleSlot)
             econtext->ecxt_scantuple = tupleSlot;
         else
         {
             HeapTuple   tuple;
     
             /*
              * RETURNING expressions might reference the tableoid column, so
              * initialize t_tableOid before evaluating them.
              */
             Assert(!TupIsNull(econtext->ecxt_scantuple));
             tuple = ExecMaterializeSlot(econtext->ecxt_scantuple);
             tuple->t_tableOid = RelationGetRelid(resultRelInfo->ri_RelationDesc);
         }
         econtext->ecxt_outertuple = planSlot;
     
         /* Compute the RETURNING expressions */
         return ExecProject(projectReturning);
     }
    

### 二、源码解读

_这部分源码需要二次解读（2018-8-6 Mark）_

    
    
    /* ----------------------------------------------------------------     *      ExecInsert
     *
     *      For INSERT, we have to insert the tuple into the target relation
     *      and insert appropriate tuples into the index relations.
     *
     *      Returns RETURNING result if any, otherwise NULL.
     * ----------------------------------------------------------------     */
    /*
    输入：
        mtstate-存储“更新”数据表（包括INSERT, UPDATE, or DELETE）时的状态
        slot-执行器使用Tuple Table存储Tuples，每个Tuple存储在该Table的一个Tuple Slot中
        planSlot-TODO
        estate-管理Executor调用的工作状态
        canSetTag-TODO
    输出：
        TupleTableSlot-相应的Tuple的Slot
    */
    static TupleTableSlot *
    ExecInsert(ModifyTableState *mtstate,
               TupleTableSlot *slot,
               TupleTableSlot *planSlot,
               EState *estate,
               bool canSetTag)
    {
        HeapTuple   tuple;//要插入的tuple
        ResultRelInfo *resultRelInfo;//执行更新操作时Relation相关的所有信息
        Relation    resultRelationDesc;//目标Relation信息
        Oid         newId;//数据表Oid
        List       *recheckIndexes = NIL;//需要检查的Index的列表
        TupleTableSlot *result = NULL;//结果Tuple对应的Slot
        TransitionCaptureState *ar_insert_trig_tcs;//TODO
        ModifyTable *node = (ModifyTable *) mtstate->ps.plan;//更新表Node
        OnConflictAction onconflict = node->onConflictAction;//冲突处理
    
        /*
         * get the heap tuple out of the tuple table slot, making sure we have a
         * writable copy
         */
        tuple = ExecMaterializeSlot(slot);//"物化"Tuple
    
        /*
         * get information on the (current) result relation
         */
        //获取Relation相关信息
        resultRelInfo = estate->es_result_relation_info;
        resultRelationDesc = resultRelInfo->ri_RelationDesc;
    
        /*
         * If the result relation has OIDs, force the tuple's OID to zero so that
         * heap_insert will assign a fresh OID.  Usually the OID already will be
         * zero at this point, but there are corner cases where the plan tree can
         * return a tuple extracted literally from some table with the same
         * rowtype.
         *
         * XXX if we ever wanted to allow users to assign their own OIDs to new
         * rows, this'd be the place to do it.  For the moment, we make a point of
         * doing this before calling triggers, so that a user-supplied trigger
         * could hack the OID if desired.
         */
        //设置Oid为InvalidOid(0)，在heap_insert时会更新Oid
        if (resultRelationDesc->rd_rel->relhasoids)
            HeapTupleSetOid(tuple, InvalidOid);
    
        /*
         * BEFORE ROW INSERT Triggers.
         *
         * Note: We fire BEFORE ROW TRIGGERS for every attempted insertion in an
         * INSERT ... ON CONFLICT statement.  We cannot check for constraint
         * violations before firing these triggers, because they can change the
         * values to insert.  Also, they can run arbitrary user-defined code with
         * side-effects that we can't cancel by just not inserting the tuple.
         */
        //触发器-TODO
        if (resultRelInfo->ri_TrigDesc &&
            resultRelInfo->ri_TrigDesc->trig_insert_before_row)
        {
            slot = ExecBRInsertTriggers(estate, resultRelInfo, slot);
    
            if (slot == NULL)       /* "do nothing" */
                return NULL;
    
            /* trigger might have changed tuple */
            tuple = ExecMaterializeSlot(slot);
        }
    
        //触发器-TODO
        /* INSTEAD OF ROW INSERT Triggers */
        if (resultRelInfo->ri_TrigDesc &&
            resultRelInfo->ri_TrigDesc->trig_insert_instead_row)
        {
            slot = ExecIRInsertTriggers(estate, resultRelInfo, slot);
    
            if (slot == NULL)       /* "do nothing" */
                return NULL;
    
            /* trigger might have changed tuple */
            tuple = ExecMaterializeSlot(slot);
    
            newId = InvalidOid;
        }
        else if (resultRelInfo->ri_FdwRoutine)//FDW-TODO
        {
            /*
             * insert into foreign table: let the FDW do it
             */
            slot = resultRelInfo->ri_FdwRoutine->ExecForeignInsert(estate,
                                                                   resultRelInfo,
                                                                   slot,
                                                                   planSlot);
    
            if (slot == NULL)       /* "do nothing" */
                return NULL;
    
            /* FDW might have changed tuple */
            tuple = ExecMaterializeSlot(slot);
    
            /*
             * AFTER ROW Triggers or RETURNING expressions might reference the
             * tableoid column, so initialize t_tableOid before evaluating them.
             */
            tuple->t_tableOid = RelationGetRelid(resultRelationDesc);
    
            newId = InvalidOid;
        }
        else//正常数据表
        {
            WCOKind     wco_kind;//With Check Option类型
    
            /*
             * Constraints might reference the tableoid column, so initialize
             * t_tableOid before evaluating them.
             */
            //获取RelationID
            tuple->t_tableOid = RelationGetRelid(resultRelationDesc);
    
            /*
             * Check any RLS WITH CHECK policies.
             *
             * Normally we should check INSERT policies. But if the insert is the
             * result of a partition key update that moved the tuple to a new
             * partition, we should instead check UPDATE policies, because we are
             * executing policies defined on the target table, and not those
             * defined on the child partitions.
             */
            //With Check Option类型：Update OR Insert？
            wco_kind = (mtstate->operation == CMD_UPDATE) ?
                WCO_RLS_UPDATE_CHECK : WCO_RLS_INSERT_CHECK;
    
            /*
             * ExecWithCheckOptions() will skip any WCOs which are not of the kind
             * we are looking for at this point.
             */
            //执行检查
            if (resultRelInfo->ri_WithCheckOptions != NIL)
                ExecWithCheckOptions(wco_kind, resultRelInfo, slot, estate);
    
            /*
             * Check the constraints of the tuple.
             */
            //检查约束
            if (resultRelationDesc->rd_att->constr)
                ExecConstraints(resultRelInfo, slot, estate);
    
            /*
             * Also check the tuple against the partition constraint, if there is
             * one; except that if we got here via tuple-routing, we don't need to
             * if there's no BR trigger defined on the partition.
             */
            //分区表
            if (resultRelInfo->ri_PartitionCheck &&
                (resultRelInfo->ri_PartitionRoot == NULL ||
                 (resultRelInfo->ri_TrigDesc &&
                  resultRelInfo->ri_TrigDesc->trig_insert_before_row)))
                ExecPartitionCheck(resultRelInfo, slot, estate, true);
            //索引检查
            if (onconflict != ONCONFLICT_NONE && resultRelInfo->ri_NumIndices > 0)
            {
                /* Perform a speculative insertion. */
                uint32      specToken;//Token
                ItemPointerData conflictTid;//数据行指针
                bool        specConflict;//冲突声明
                List       *arbiterIndexes;//相关索引
    
                arbiterIndexes = resultRelInfo->ri_onConflictArbiterIndexes;
    
                /*
                 * Do a non-conclusive check for conflicts first.
                 *
                 * We're not holding any locks yet, so this doesn't guarantee that
                 * the later insert won't conflict.  But it avoids leaving behind
                 * a lot of canceled speculative insertions, if you run a lot of
                 * INSERT ON CONFLICT statements that do conflict.
                 *
                 * We loop back here if we find a conflict below, either during
                 * the pre-check, or when we re-check after inserting the tuple
                 * speculatively.
                 */
        vlock:
                specConflict = false;
                if (!ExecCheckIndexConstraints(slot, estate, &conflictTid,
                                               arbiterIndexes))
                {
                    /* committed conflict tuple found */
                    if (onconflict == ONCONFLICT_UPDATE)
                    {
                        /*
                         * In case of ON CONFLICT DO UPDATE, execute the UPDATE
                         * part.  Be prepared to retry if the UPDATE fails because
                         * of another concurrent UPDATE/DELETE to the conflict
                         * tuple.
                         */
                        TupleTableSlot *returning = NULL;
    
                        if (ExecOnConflictUpdate(mtstate, resultRelInfo,
                                                 &conflictTid, planSlot, slot,
                                                 estate, canSetTag, &returning))
                        {
                            InstrCountTuples2(&mtstate->ps, 1);
                            return returning;
                        }
                        else
                            goto vlock;
                    }
                    else
                    {
                        /*
                         * In case of ON CONFLICT DO NOTHING, do nothing. However,
                         * verify that the tuple is visible to the executor's MVCC
                         * snapshot at higher isolation levels.
                         */
                        Assert(onconflict == ONCONFLICT_NOTHING);
                        ExecCheckTIDVisible(estate, resultRelInfo, &conflictTid);
                        InstrCountTuples2(&mtstate->ps, 1);
                        return NULL;
                    }
                }
    
                /*
                 * Before we start insertion proper, acquire our "speculative
                 * insertion lock".  Others can use that to wait for us to decide
                 * if we're going to go ahead with the insertion, instead of
                 * waiting for the whole transaction to complete.
                 */
                specToken = SpeculativeInsertionLockAcquire(GetCurrentTransactionId());
                HeapTupleHeaderSetSpeculativeToken(tuple->t_data, specToken);
    
                /* insert the tuple, with the speculative token */
                newId = heap_insert(resultRelationDesc, tuple,
                                    estate->es_output_cid,
                                    HEAP_INSERT_SPECULATIVE,
                                    NULL);
    
                /* insert index entries for tuple */
                recheckIndexes = ExecInsertIndexTuples(slot, &(tuple->t_self),
                                                       estate, true, &specConflict,
                                                       arbiterIndexes);
    
                /* adjust the tuple's state accordingly */
                if (!specConflict)
                    heap_finish_speculative(resultRelationDesc, tuple);
                else
                    heap_abort_speculative(resultRelationDesc, tuple);
    
                /*
                 * Wake up anyone waiting for our decision.  They will re-check
                 * the tuple, see that it's no longer speculative, and wait on our
                 * XID as if this was a regularly inserted tuple all along.  Or if
                 * we killed the tuple, they will see it's dead, and proceed as if
                 * the tuple never existed.
                 */
                SpeculativeInsertionLockRelease(GetCurrentTransactionId());
    
                /*
                 * If there was a conflict, start from the beginning.  We'll do
                 * the pre-check again, which will now find the conflicting tuple
                 * (unless it aborts before we get there).
                 */
                if (specConflict)
                {
                    list_free(recheckIndexes);
                    goto vlock;
                }
    
                /* Since there was no insertion conflict, we're done */
            }
            else//无需Index检查
            {
                /*
                 * insert the tuple normally.
                 *
                 * Note: heap_insert returns the tid (location) of the new tuple
                 * in the t_self field.
                 */
                newId = heap_insert(resultRelationDesc, tuple,
                                    estate->es_output_cid,
                                    0, NULL);//插入数据
    
                /* insert index entries for tuple */
                if (resultRelInfo->ri_NumIndices > 0)
                    recheckIndexes = ExecInsertIndexTuples(slot, &(tuple->t_self),
                                                           estate, false, NULL,
                                                           NIL);//索引
            }
        }
    
        if (canSetTag)
        {
            (estate->es_processed)++;
            estate->es_lastoid = newId;
            setLastTid(&(tuple->t_self));
        }
    
        /*
         * If this insert is the result of a partition key update that moved the
         * tuple to a new partition, put this row into the transition NEW TABLE,
         * if there is one. We need to do this separately for DELETE and INSERT
         * because they happen on different tables.
         */
        //触发器-TODO
        ar_insert_trig_tcs = mtstate->mt_transition_capture;
        if (mtstate->operation == CMD_UPDATE && mtstate->mt_transition_capture
            && mtstate->mt_transition_capture->tcs_update_new_table)
        {
            ExecARUpdateTriggers(estate, resultRelInfo, NULL,
                                 NULL,
                                 tuple,
                                 NULL,
                                 mtstate->mt_transition_capture);
    
            /*
             * We've already captured the NEW TABLE row, so make sure any AR
             * INSERT trigger fired below doesn't capture it again.
             */
            ar_insert_trig_tcs = NULL;
        }
    
        /* AFTER ROW INSERT Triggers */
        ExecARInsertTriggers(estate, resultRelInfo, tuple, recheckIndexes,
                             ar_insert_trig_tcs);
    
        list_free(recheckIndexes);
    
        /*
         * Check any WITH CHECK OPTION constraints from parent views.  We are
         * required to do this after testing all constraints and uniqueness
         * violations per the SQL spec, so we do it after actually inserting the
         * record into the heap and all indexes.
         *
         * ExecWithCheckOptions will elog(ERROR) if a violation is found, so the
         * tuple will never be seen, if it violates the WITH CHECK OPTION.
         *
         * ExecWithCheckOptions() will skip any WCOs which are not of the kind we
         * are looking for at this point.
         */
        if (resultRelInfo->ri_WithCheckOptions != NIL)
            ExecWithCheckOptions(WCO_VIEW_CHECK, resultRelInfo, slot, estate);
    
        /* Process RETURNING if present */
        if (resultRelInfo->ri_projectReturning)
            result = ExecProcessReturning(resultRelInfo, slot, planSlot);
    
        return result;
    }
    
    

### 三、跟踪分析

添加主键，插入测试数据：

    
    
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1527
    (1 row)
    
    testdb=# 
    testdb=# -- 添加主键
    testdb=# alter table t_insert add constraint pk_t_insert primary key(id);
    ALTER TABLE
    testdb=# insert into t_insert values(12,'12','12','12');
    （挂起）
    

启动gdb

    
    
    [root@localhost ~]# gdb -p 1527
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b ExecInsert
    Breakpoint 1 at 0x6c0227: file nodeModifyTable.c, line 273.
    #查看输入参数
    (gdb) p *mtstate
    $1 = {ps = {type = T_ModifyTableState, plan = 0x1257fa0, state = 0x130bf80, ExecProcNode = 0x6c2485 <ExecModifyTable>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
        worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x130d1e0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
        scandesc = 0x0}, operation = CMD_INSERT, canSetTag = true, mt_done = false, mt_plans = 0x130c4e0, mt_nplans = 1, mt_whichplan = 0, resultRelInfo = 0x130c1c0, rootResultRelInfo = 0x0, 
      mt_arowmarks = 0x130c4f8, mt_epqstate = {estate = 0x0, planstate = 0x0, origslot = 0x130c950, plan = 0x132de88, arowMarks = 0x0, epqParam = 0}, fireBSTriggers = false, mt_existing = 0x0, 
      mt_excludedtlist = 0x0, mt_conflproj = 0x0, mt_partition_tuple_routing = 0x0, mt_transition_capture = 0x0, mt_oc_transition_capture = 0x0, mt_per_subplan_tupconv_maps = 0x0}
    (gdb) p *(mtstate->ps->plan)
    $2 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 136, parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *(mtstate->ps->state)
    $3 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1309fa0, es_crosscheck_snapshot = 0x0, es_range_table = 0x132e1f8, es_plannedstmt = 0x1258420, 
      es_sourceText = 0x1256ef0 "insert into t_insert values(12,'12','12','12');", es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x130c1c0, es_num_result_relations = 1, 
      es_result_relation_info = 0x130c1c0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x130d290, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x130c190, es_queryEnv = 0x0, es_query_cxt = 0x130be70, 
      es_tupleTable = 0x130c810, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x130c7b0, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0}
    (gdb) p *(mtstate->ps->ps_ResultTupleSlot)
    $4 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x130d1b0, tts_mcxt = 0x130be70, 
      tts_buffer = 0, tts_nvalid = 0, tts_values = 0x130d240, tts_isnull = 0x130d240, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) p *(mtstate->ps->ps_ResultTupleSlot->tts_tupleDescriptor)
    $5 = {natts = 0, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x130d1d0}
    (gdb) p *(mtstate->ps->ps_ResultTupleSlot->tts_tupleDescriptor->attrs)
    $6 = {attrelid = 128, attname = {
        data = "\000\000\000\000p\276\060\001\000\000\000\000\b\000\000\000\001", '\000' <repeats 11 times>, "\260\321\060\001\000\000\000\000p\276\060\001", '\000' <repeats 12 times>, "@\322\060\001\000\000\000\000@\322\060\001"}, atttypid = 0, attstattarget = 0, attlen = 0, attnum = 0, attndims = 0, attcacheoff = 0, atttypmod = 0, attbyval = false, attstorage = 0 '\000', attalign = 0 '\000', 
      attnotnull = false, atthasdef = false, atthasmissing = false, attidentity = 0 '\000', attisdropped = false, attislocal = false, attinhcount = 0, attcollation = 1}
    $7 = (PlanState *) 0x130c560
    (gdb) p **(mtstate->mt_plans)
    $8 = {type = T_ResultState, plan = 0x132de88, state = 0x130bf80, ExecProcNode = 0x6c5094 <ExecResult>, ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, worker_instrument = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x130c950, ps_ExprContext = 0x130c670, ps_ProjInfo = 0x130c700, scandesc = 0x0}
    (gdb) p **(mtstate->resultRelInfo)
    Structure has no component named operator*.
    (gdb) p *(mtstate->resultRelInfo)
    $9 = {type = T_ResultRelInfo, ri_RangeTableIndex = 1, ri_RelationDesc = 0x7f2af957d9e8, ri_NumIndices = 1, ri_IndexRelationDescs = 0x130c7e0, ri_IndexRelationInfo = 0x130c7f8, ri_TrigDesc = 0x0, 
      ri_TrigFunctions = 0x0, ri_TrigWhenExprs = 0x0, ri_TrigInstrument = 0x0, ri_FdwRoutine = 0x0, ri_FdwState = 0x0, ri_usesFdwDirectModify = false, ri_WithCheckOptions = 0x0, 
      ri_WithCheckOptionExprs = 0x0, ri_ConstraintExprs = 0x0, ri_junkFilter = 0x0, ri_returningList = 0x0, ri_projectReturning = 0x0, ri_onConflictArbiterIndexes = 0x0, ri_onConflict = 0x0, 
      ri_PartitionCheck = 0x0, ri_PartitionCheckExpr = 0x0, ri_PartitionRoot = 0x0, ri_PartitionReadyForRouting = false}
    (gdb) p *(mtstate->resultRelInfo->ri_RelationDesc)
    $10 = {rd_node = {spcNode = 1663, dbNode = 16477, relNode = 26731}, rd_smgr = 0x0, rd_refcnt = 1, rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, 
      rd_indexvalid = 1 '\001', rd_statvalid = false, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f2af94c31a8, rd_att = 0x7f2af957f6e8, rd_id = 26731, rd_lockInfo = {lockRelId = {
          relId = 26731, dbId = 16477}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x7f2af94c2818, rd_oidindex = 0, rd_pkindex = 26737, rd_replidindex = 26737, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, 
      rd_keyattr = 0x0, rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, 
      rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x12d5850}
    (gdb) 
    #第2个参数slot
    (gdb) p *slot
    $11 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x130cb70, tts_mcxt = 0x130be70, 
      tts_buffer = 0, tts_nvalid = 4, tts_values = 0x130c9b0, tts_isnull = 0x130c9d0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) p *(slot->tts_tupleDescriptor)
    $12 = {natts = 4, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x130cb90}
    (gdb) p *(slot->tts_values)
    $13 = 12
    (gdb) p *(slot->tts_isnull)
    $14 = false
    #参数planSlot
    (gdb) p *planSlot
    $15 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x130cb70, tts_mcxt = 0x130be70, 
      tts_buffer = 0, tts_nvalid = 4, tts_values = 0x130c9b0, tts_isnull = 0x130c9d0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) p *(planSlot->tts_mcxt)
    $16 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0x131f150, firstchild = 0x130fe90, prevchild = 0x0, nextchild = 0x0, 
      name = 0xb1a840 "ExecutorState", ident = 0x0, reset_cbs = 0x0}
    #参数estate
    (gdb) p *estate
    $17 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1309fa0, es_crosscheck_snapshot = 0x0, es_range_table = 0x132e1f8, es_plannedstmt = 0x1258420, 
      es_sourceText = 0x1256ef0 "insert into t_insert values(12,'12','12','12');", es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x130c1c0, es_num_result_relations = 1, 
      es_result_relation_info = 0x130c1c0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x130d290, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x130c190, es_queryEnv = 0x0, es_query_cxt = 0x130be70, 
      es_tupleTable = 0x130c810, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x130c7b0, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0}
    (gdb) p *(estate->es_snapshot)
    $18 = {satisfies = 0x9f73fc <HeapTupleSatisfiesMVCC>, xmin = 1612861, xmax = 1612861, xip = 0x0, xcnt = 0, subxip = 0x0, subxcnt = 0, suboverflowed = false, takenDuringRecovery = false, copied = true, 
      curcid = 0, speculativeToken = 0, active_count = 1, regd_count = 2, ph_node = {first_child = 0xe7bac0 <CatalogSnapshotData+64>, next_sibling = 0x0, prev_or_parent = 0x0}, whenTaken = 0, lsn = 0}
    (gdb) 
    #参数canSetTag
    (gdb) p canSetTag
    $19 = true
    
    #进入函数内部
    (gdb) next
    274     TupleTableSlot *result = NULL;
    (gdb) 
    276     ModifyTable *node = (ModifyTable *) mtstate->ps.plan;
    (gdb) p *node
    $20 = {plan = {type = 1005513616, startup_cost = 3.502247076882698e-317, total_cost = 9.8678822363083824e-317, plan_rows = 4.1888128009757547e-314, plan_width = 3, parallel_aware = 253, 
        parallel_safe = 127, plan_node_id = 10076823, targetlist = 0x686b0000405d, qual = 0x130c2d0, lefttree = 0x0, righttree = 0x130c1c0, initPlan = 0x6c2485 <ExecModifyTable>, extParam = 0x0, 
        allParam = 0x7ffd3beeeb50}, operation = 6923989, canSetTag = false, nominalRelation = 4183284200, partitioned_rels = 0x130c1c0, partColsUpdated = 128, resultRelations = 0x1257fa0, 
      resultRelIndex = 19974480, rootResultRelIndex = 0, plans = 0x0, withCheckOptionLists = 0x33beeeb50, returningLists = 0x130bf80, fdwPrivLists = 0x0, fdwDirectModifyPlans = 0x130c2d0, rowMarks = 0x0, 
      epqParam = 0, onConflictAction = ONCONFLICT_NONE, arbiterIndexes = 0x130c950, onConflictSet = 0x0, onConflictWhere = 0x130c560, exclRelRTI = 19972544, exclRelTlist = 0x0}
    (gdb) 
    (gdb) p onconflict
    $21 = ONCONFLICT_NONE
    ...
    (gdb) next
    289     resultRelationDesc = resultRelInfo->ri_RelationDesc;
    (gdb) p *resultRelInfo
    $26 = {type = T_ResultRelInfo, ri_RangeTableIndex = 1, ri_RelationDesc = 0x7f2af957d9e8, ri_NumIndices = 1, ri_IndexRelationDescs = 0x130c7e0, ri_IndexRelationInfo = 0x130c7f8, ri_TrigDesc = 0x0, 
      ri_TrigFunctions = 0x0, ri_TrigWhenExprs = 0x0, ri_TrigInstrument = 0x0, ri_FdwRoutine = 0x0, ri_FdwState = 0x0, ri_usesFdwDirectModify = false, ri_WithCheckOptions = 0x0, 
      ri_WithCheckOptionExprs = 0x0, ri_ConstraintExprs = 0x0, ri_junkFilter = 0x0, ri_returningList = 0x0, ri_projectReturning = 0x0, ri_onConflictArbiterIndexes = 0x0, ri_onConflict = 0x0, 
      ri_PartitionCheck = 0x0, ri_PartitionCheckExpr = 0x0, ri_PartitionRoot = 0x0, ri_PartitionReadyForRouting = false}
    (gdb) p *(tuple->t_data)
    $27 = {t_choice = {t_heap = {t_xmin = 244, t_xmax = 4294967295, t_field3 = {t_cid = 2249, t_xvac = 2249}}, t_datum = {datum_len_ = 244, datum_typmod = -1, datum_typeid = 2249}}, t_ctid = {ip_blkid = {
          bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_infomask2 = 4, t_infomask = 2, t_hoff = 24 '\030', t_bits = 0x130d36f ""}
    (gdb) p *(tuple)
    $28 = {t_len = 61, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 0, t_data = 0x130d358}
    (gdb) 
    (gdb) p *resultRelationDesc
    $29 = {rd_node = {spcNode = 1663, dbNode = 16477, relNode = 26731}, rd_smgr = 0x0, rd_refcnt = 1, rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, 
      rd_indexvalid = 1 '\001', rd_statvalid = false, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f2af94c31a8, rd_att = 0x7f2af957f6e8, rd_id = 26731, rd_lockInfo = {lockRelId = {
          relId = 26731, dbId = 16477}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x7f2af94c2818, rd_oidindex = 0, rd_pkindex = 26737, rd_replidindex = 26737, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, 
      rd_keyattr = 0x0, rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, 
      rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x12d5850}
    #以上，获取了Relation相关的信息
    #没有触发器，无需执行触发器相关逻辑
    #进入正常数据表的处理逻辑 line 373行
    (gdb) next
    315     if (resultRelInfo->ri_TrigDesc &&
    (gdb) next
    328     if (resultRelInfo->ri_TrigDesc &&
    (gdb) 
    341     else if (resultRelInfo->ri_FdwRoutine)
    (gdb) 
    373         tuple->t_tableOid = RelationGetRelid(resultRelationDesc);
    (gdb) next
    384         wco_kind = (mtstate->operation == CMD_UPDATE) ?
    (gdb) next
    391         if (resultRelInfo->ri_WithCheckOptions != NIL)
    (gdb) p tuple->t_tableOid
    $31 = 26731
    (gdb) p wco_kind
    $32 = WCO_RLS_INSERT_CHECK
    #检查约束
    (gdb) next
    397         if (resultRelationDesc->rd_att->constr)
    (gdb) 
    398             ExecConstraints(resultRelInfo, slot, estate);
    (gdb) 
    #非分区表，无需检查
    #进入正常数据插入逻辑
    411         if (onconflict != ONCONFLICT_NONE && resultRelInfo->ri_NumIndices > 0)
    (gdb) 
    529             newId = heap_insert(resultRelationDesc, tuple,
    (gdb) p resultRelInfo->ri_NumIndices
    $33 = 1
    (gdb) p onconflict
    $34 = ONCONFLICT_NONE
    (gdb) 
    (gdb) next
    534             if (resultRelInfo->ri_NumIndices > 0)
    (gdb) p newId
    $35 = 0
    (gdb) next
    535                 recheckIndexes = ExecInsertIndexTuples(slot, &(tuple->t_self),
    (gdb) 
    541     if (canSetTag)
    (gdb) 
    543         (estate->es_processed)++;
    (gdb) 
    544         estate->es_lastoid = newId;
    (gdb) 
    545         setLastTid(&(tuple->t_self));
    (gdb) 
    554     ar_insert_trig_tcs = mtstate->mt_transition_capture;
    (gdb) 
    555     if (mtstate->operation == CMD_UPDATE && mtstate->mt_transition_capture
    (gdb) 
    572     ExecARInsertTriggers(estate, resultRelInfo, tuple, recheckIndexes,
    (gdb) 
    575     list_free(recheckIndexes);
    (gdb) 
    589     if (resultRelInfo->ri_WithCheckOptions != NIL)
    (gdb) 
    593     if (resultRelInfo->ri_projectReturning)
    (gdb) 
    596     return result;
    (gdb) 
    597 }
    #DONE!
    
    

### 四、小结

1、其他相关知识：插入数据涉及执行计划等相关的数据结构，需要进一步的了解和理解；  
2、触发器：插入前后的触发器调用在此函数中处理；  
3、索引：索引Tuple的插入部分暂未深入解读。

**最后：涉及的数据结构越来越多，数据结构的结构越来越复杂，慢慢进入了深水区，需做好潜水准备**

