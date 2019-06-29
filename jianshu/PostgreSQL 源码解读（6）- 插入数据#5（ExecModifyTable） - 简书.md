本文简单介绍了PG插入数据部分的源码，主要内容包括ExecModifyTable函数的实现逻辑，该函数位于nodeModifyTable.c文件中。

### 一、基础信息

ExecModifyTable函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、PartitionTupleRouting_

    
    
    //分区表相关的数据结构，再续解读分区表处理时再行更新
     /*-----------------------      * PartitionTupleRouting - Encapsulates all information required to execute
      * tuple-routing between partitions.
      *
      * partition_dispatch_info      Array of PartitionDispatch objects with one
      *                              entry for every partitioned table in the
      *                              partition tree.
      * num_dispatch                 number of partitioned tables in the partition
      *                              tree (= length of partition_dispatch_info[])
      * partition_oids               Array of leaf partitions OIDs with one entry
      *                              for every leaf partition in the partition tree,
      *                              initialized in full by
      *                              ExecSetupPartitionTupleRouting.
      * partitions                   Array of ResultRelInfo* objects with one entry
      *                              for every leaf partition in the partition tree,
      *                              initialized lazily by ExecInitPartitionInfo.
      * num_partitions               Number of leaf partitions in the partition tree
      *                              (= 'partitions_oid'/'partitions' array length)
      * parent_child_tupconv_maps    Array of TupleConversionMap objects with one
      *                              entry for every leaf partition (required to
      *                              convert tuple from the root table's rowtype to
      *                              a leaf partition's rowtype after tuple routing
      *                              is done)
      * child_parent_tupconv_maps    Array of TupleConversionMap objects with one
      *                              entry for every leaf partition (required to
      *                              convert an updated tuple from the leaf
      *                              partition's rowtype to the root table's rowtype
      *                              so that tuple routing can be done)
      * child_parent_map_not_required  Array of bool. True value means that a map is
      *                              determined to be not required for the given
      *                              partition. False means either we haven't yet
      *                              checked if a map is required, or it was
      *                              determined to be required.
      * subplan_partition_offsets    Integer array ordered by UPDATE subplans. Each
      *                              element of this array has the index into the
      *                              corresponding partition in partitions array.
      * num_subplan_partition_offsets  Length of 'subplan_partition_offsets' array
      * partition_tuple_slot         TupleTableSlot to be used to manipulate any
      *                              given leaf partition's rowtype after that
      *                              partition is chosen for insertion by
      *                              tuple-routing.
      * root_tuple_slot              TupleTableSlot to be used to transiently hold
      *                              copy of a tuple that's being moved across
      *                              partitions in the root partitioned table's
      *                              rowtype
      *-----------------------      */
     typedef struct PartitionTupleRouting
     {
         PartitionDispatch *partition_dispatch_info;
         int         num_dispatch;
         Oid        *partition_oids;
         ResultRelInfo **partitions;
         int         num_partitions;
         TupleConversionMap **parent_child_tupconv_maps;
         TupleConversionMap **child_parent_tupconv_maps;
         bool       *child_parent_map_not_required;
         int        *subplan_partition_offsets;
         int         num_subplan_partition_offsets;
         TupleTableSlot *partition_tuple_slot;
         TupleTableSlot *root_tuple_slot;
     } PartitionTupleRouting;
    

_2、CmdType_

    
    
    //命令类型，DDL命令都归到CMD_UTILITY里面了
     /*
      * CmdType -      *    enums for type of operation represented by a Query or PlannedStmt
      *
      * This is needed in both parsenodes.h and plannodes.h, so put it here...
      */
     typedef enum CmdType
     {
         CMD_UNKNOWN,
         CMD_SELECT,                 /* select stmt */
         CMD_UPDATE,                 /* update stmt */
         CMD_INSERT,                 /* insert stmt */
         CMD_DELETE,
         CMD_UTILITY,                /* cmds like create, destroy, copy, vacuum,
                                      * etc. */
         CMD_NOTHING                 /* dummy command for instead nothing rules
                                      * with qual */
     } CmdType;
     
    

_3、JunkFilter_

    
    
     //运行期需要的Attribute成为junk attribute，实际的Tuple需要使用JunkFilter过滤这些属性
     /* ----------------      *    JunkFilter
      *
      *    This class is used to store information regarding junk attributes.
      *    A junk attribute is an attribute in a tuple that is needed only for
      *    storing intermediate information in the executor, and does not belong
      *    in emitted tuples.  For example, when we do an UPDATE query,
      *    the planner adds a "junk" entry to the targetlist so that the tuples
      *    returned to ExecutePlan() contain an extra attribute: the ctid of
      *    the tuple to be updated.  This is needed to do the update, but we
      *    don't want the ctid to be part of the stored new tuple!  So, we
      *    apply a "junk filter" to remove the junk attributes and form the
      *    real output tuple.  The junkfilter code also provides routines to
      *    extract the values of the junk attribute(s) from the input tuple.
      *
      *    targetList:       the original target list (including junk attributes).
      *    cleanTupType:     the tuple descriptor for the "clean" tuple (with
      *                      junk attributes removed).
      *    cleanMap:         A map with the correspondence between the non-junk
      *                      attribute numbers of the "original" tuple and the
      *                      attribute numbers of the "clean" tuple.
      *    resultSlot:       tuple slot used to hold cleaned tuple.
      *    junkAttNo:        not used by junkfilter code.  Can be used by caller
      *                      to remember the attno of a specific junk attribute
      *                      (nodeModifyTable.c keeps the "ctid" or "wholerow"
      *                      attno here).
      * ----------------      */
     typedef struct JunkFilter
     {
         NodeTag     type;
         List       *jf_targetList;
         TupleDesc   jf_cleanTupType;
         AttrNumber *jf_cleanMap;
         TupleTableSlot *jf_resultSlot;
         AttrNumber  jf_junkAttNo;
     } JunkFilter;
    

_4、Datum_

    
    
    //实际类型为unsigned long int    ，无符号长整型整数
     /*
      * A Datum contains either a value of a pass-by-value type or a pointer to a
      * value of a pass-by-reference type.  Therefore, we require:
      *
      * sizeof(Datum) == sizeof(void *) == 4 or 8
      *
      * The macros below and the analogous macros for other types should be used to
      * convert between a Datum and the appropriate C type.
      */
     
     typedef uintptr_t Datum;
    
    uintptr_t   unsigned integer type capable of holding a pointer,defined in header <stdint.h> 
    typedef unsigned long int   uintptr_t;
    
    

_5、CHECK_FOR_INTERRUPTS_

    
    
    / * The CHECK_FOR_INTERRUPTS() macro is called at strategically located spots
      * where it is normally safe to accept a cancel or die interrupt.  In some
      * cases, we invoke CHECK_FOR_INTERRUPTS() inside low-level subroutines that
      * might sometimes be called in contexts that do *not* want to allow a cancel
      * or die interrupt.  The HOLD_INTERRUPTS() and RESUME_INTERRUPTS() macros
      * allow code to ensure that no cancel or die interrupt will be accepted,
      * even if CHECK_FOR_INTERRUPTS() gets called in a subroutine.  The interrupt
      * will be held off until CHECK_FOR_INTERRUPTS() is done outside any
      * HOLD_INTERRUPTS() ... RESUME_INTERRUPTS() section.
      */
    
    #define CHECK_FOR_INTERRUPTS() \
     do { \
         if (InterruptPending) \
             ProcessInterrupts(); \
     } while(0)
    

**依赖的函数**  
_1、fireBSTriggers_

    
    
    //触发语句级的触发器
     /*
      * Process BEFORE EACH STATEMENT triggers
      */
     static void
     fireBSTriggers(ModifyTableState *node)
     {
         ModifyTable *plan = (ModifyTable *) node->ps.plan;
         ResultRelInfo *resultRelInfo = node->resultRelInfo;
     
         /*
          * If the node modifies a partitioned table, we must fire its triggers.
          * Note that in that case, node->resultRelInfo points to the first leaf
          * partition, not the root table.
          */
         if (node->rootResultRelInfo != NULL)
             resultRelInfo = node->rootResultRelInfo;
     
         switch (node->operation)
         {
             case CMD_INSERT:
                 ExecBSInsertTriggers(node->ps.state, resultRelInfo);
                 if (plan->onConflictAction == ONCONFLICT_UPDATE)
                     ExecBSUpdateTriggers(node->ps.state,
                                          resultRelInfo);
                 break;
             case CMD_UPDATE:
                 ExecBSUpdateTriggers(node->ps.state, resultRelInfo);
                 break;
             case CMD_DELETE:
                 ExecBSDeleteTriggers(node->ps.state, resultRelInfo);
                 break;
             default:
                 elog(ERROR, "unknown operation");
                 break;
         }
     }
     
    

_2、ResetPerTupleExprContext_

    
    
     /* Reset an EState's per-output-tuple exprcontext, if one's been created */
     #define ResetPerTupleExprContext(estate) \
         do { \
             if ((estate)->es_per_tuple_exprcontext) \
                 ResetExprContext((estate)->es_per_tuple_exprcontext); \
         } while (0)
     
    

_3、ExecProcNode_

    
    
    //执行子计划时使用，这个函数同时也是高N级的函数，后续再行解读
    

_4、TupIsNull_

    
    
    //判断TupleTableSlot 是否为Null（包括Empty）
     /*
      * TupIsNull -- is a TupleTableSlot empty?
      */
     #define TupIsNull(slot) \
         ((slot) == NULL || (slot)->tts_isempty)
    

_5、EvalPlanQualSetPlan_

    
    
    //TODO
     /*
      * EvalPlanQualSetPlan -- set or change subplan of an EPQState.
      *
      * We need this so that ModifyTable can deal with multiple subplans.
      */
     void
     EvalPlanQualSetPlan(EPQState *epqstate, Plan *subplan, List *auxrowmarks)
     {
         /* If we have a live EPQ query, shut it down */
         EvalPlanQualEnd(epqstate);
         /* And set/change the plan pointer */
         epqstate->plan = subplan;
         /* The rowmarks depend on the plan, too */
         epqstate->arowMarks = auxrowmarks;
     }
    

_6、tupconv_map_for_subplan_

    
    
    //分区表使用，后续再行解析
    //TODO
     /*
      * For a given subplan index, get the tuple conversion map.
      */
     static TupleConversionMap *
     tupconv_map_for_subplan(ModifyTableState *mtstate, int whichplan)
     {
         /*
          * If a partition-index tuple conversion map array is allocated, we need
          * to first get the index into the partition array. Exactly *one* of the
          * two arrays is allocated. This is because if there is a partition array
          * required, we don't require subplan-indexed array since we can translate
          * subplan index into partition index. And, we create a subplan-indexed
          * array *only* if partition-indexed array is not required.
          */
         if (mtstate->mt_per_subplan_tupconv_maps == NULL)
         {
             int         leaf_index;
             PartitionTupleRouting *proute = mtstate->mt_partition_tuple_routing;
     
             /*
              * If subplan-indexed array is NULL, things should have been arranged
              * to convert the subplan index to partition index.
              */
             Assert(proute && proute->subplan_partition_offsets != NULL &&
                    whichplan < proute->num_subplan_partition_offsets);
     
             leaf_index = proute->subplan_partition_offsets[whichplan];
     
             return TupConvMapForLeaf(proute, getTargetResultRelInfo(mtstate),
                                      leaf_index);
         }
         else
         {
             Assert(whichplan >= 0 && whichplan < mtstate->mt_nplans);
             return mtstate->mt_per_subplan_tupconv_maps[whichplan];
         }
     }
    

_7、ExecProcessReturning_

    
    
    //返回结果Tuple
    //更详细的解读有待执行查询语句时的分析解读
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
    
     /* ----------------      *      ProjectionInfo node information
      *
      *      This is all the information needed to perform projections ---      *      that is, form new tuples by evaluation of targetlist expressions.
      *      Nodes which need to do projections create one of these.
      *
      *      The target tuple slot is kept in ProjectionInfo->pi_state.resultslot.
      *      ExecProject() evaluates the tlist, forms a tuple, and stores it
      *      in the given slot.  Note that the result will be a "virtual" tuple
      *      unless ExecMaterializeSlot() is then called to force it to be
      *      converted to a physical tuple.  The slot must have a tupledesc
      *      that matches the output of the tlist!
      * ----------------      */
     typedef struct ProjectionInfo
     {
         NodeTag     type;
         /* instructions to evaluate projection */
         ExprState   pi_state;
         /* expression context in which to evaluate expression */
         ExprContext *pi_exprContext;
     } ProjectionInfo;
     
     /* ----------------      *    ExprContext
      *
      *      This class holds the "current context" information
      *      needed to evaluate expressions for doing tuple qualifications
      *      and tuple projections.  For example, if an expression refers
      *      to an attribute in the current inner tuple then we need to know
      *      what the current inner tuple is and so we look at the expression
      *      context.
      *
      *  There are two memory contexts associated with an ExprContext:
      *  * ecxt_per_query_memory is a query-lifespan context, typically the same
      *    context the ExprContext node itself is allocated in.  This context
      *    can be used for purposes such as storing function call cache info.
      *  * ecxt_per_tuple_memory is a short-term context for expression results.
      *    As the name suggests, it will typically be reset once per tuple,
      *    before we begin to evaluate expressions for that tuple.  Each
      *    ExprContext normally has its very own per-tuple memory context.
      *
      *  CurrentMemoryContext should be set to ecxt_per_tuple_memory before
      *  calling ExecEvalExpr() --- see ExecEvalExprSwitchContext().
      * ----------------      */
     typedef struct ExprContext
     {
         NodeTag     type;
     
         /* Tuples that Var nodes in expression may refer to */
     #define FIELDNO_EXPRCONTEXT_SCANTUPLE 1
         TupleTableSlot *ecxt_scantuple;
     #define FIELDNO_EXPRCONTEXT_INNERTUPLE 2
         TupleTableSlot *ecxt_innertuple;
     #define FIELDNO_EXPRCONTEXT_OUTERTUPLE 3
         TupleTableSlot *ecxt_outertuple;
     
         /* Memory contexts for expression evaluation --- see notes above */
         MemoryContext ecxt_per_query_memory;
         MemoryContext ecxt_per_tuple_memory;
     
         /* Values to substitute for Param nodes in expression */
         ParamExecData *ecxt_param_exec_vals;    /* for PARAM_EXEC params */
         ParamListInfo ecxt_param_list_info; /* for other param types */
     
         /*
          * Values to substitute for Aggref nodes in the expressions of an Agg
          * node, or for WindowFunc nodes within a WindowAgg node.
          */
     #define FIELDNO_EXPRCONTEXT_AGGVALUES 8
         Datum      *ecxt_aggvalues; /* precomputed values for aggs/windowfuncs */
     #define FIELDNO_EXPRCONTEXT_AGGNULLS 9
         bool       *ecxt_aggnulls;  /* null flags for aggs/windowfuncs */
     
         /* Value to substitute for CaseTestExpr nodes in expression */
     #define FIELDNO_EXPRCONTEXT_CASEDATUM 10
         Datum       caseValue_datum;
     #define FIELDNO_EXPRCONTEXT_CASENULL 11
         bool        caseValue_isNull;
     
         /* Value to substitute for CoerceToDomainValue nodes in expression */
     #define FIELDNO_EXPRCONTEXT_DOMAINDATUM 12
         Datum       domainValue_datum;
     #define FIELDNO_EXPRCONTEXT_DOMAINNULL 13
         bool        domainValue_isNull;
     
         /* Link to containing EState (NULL if a standalone ExprContext) */
         struct EState *ecxt_estate;
     
         /* Functions to call back when ExprContext is shut down or rescanned */
         ExprContext_CB *ecxt_callbacks;
     } ExprContext;
    
     /*
      * ExecProject
      *
      * Projects a tuple based on projection info and stores it in the slot passed
      * to ExecBuildProjectInfo().
      *
      * Note: the result is always a virtual tuple; therefore it may reference
      * the contents of the exprContext's scan tuples and/or temporary results
      * constructed in the exprContext.  If the caller wishes the result to be
      * valid longer than that data will be valid, he must call ExecMaterializeSlot
      * on the result slot.
      */
     #ifndef FRONTEND
     static inline TupleTableSlot *
     ExecProject(ProjectionInfo *projInfo)
     {
         ExprContext *econtext = projInfo->pi_exprContext;
         ExprState  *state = &projInfo->pi_state;
         TupleTableSlot *slot = state->resultslot;
         bool        isnull;
     
         /*
          * Clear any former contents of the result slot.  This makes it safe for
          * us to use the slot's Datum/isnull arrays as workspace.
          */
         ExecClearTuple(slot);
     
         /* Run the expression, discarding scalar result from the last column. */
         (void) ExecEvalExprSwitchContext(state, econtext, &isnull);
     
         /*
          * Successfully formed a result row.  Mark the result slot as containing a
          * valid virtual tuple (inlined version of ExecStoreVirtualTuple()).
          */
         slot->tts_isempty = false;
         slot->tts_nvalid = slot->tts_tupleDescriptor->natts;
     
         return slot;
     }
     #endif
     
    

_8、EvalPlanQualSetSlot_

    
    
     #define EvalPlanQualSetSlot(epqstate, slot)  ((epqstate)->origslot = (slot))
    

_9、ExecGetJunkAttribute_

    
    
    //获取Junk中的某个属性值
    /*
      * ExecGetJunkAttribute
      *
      * Given a junk filter's input tuple (slot) and a junk attribute's number
      * previously found by ExecFindJunkAttribute, extract & return the value and
      * isNull flag of the attribute.
      */
     Datum
     ExecGetJunkAttribute(TupleTableSlot *slot, AttrNumber attno,
                          bool *isNull)
     {
         Assert(attno > 0);
     
         return slot_getattr(slot, attno, isNull);
     }
     
    

_10、DatumGetPointer_

    
    
    // 将无符号长整型转换为指针然后进行传递
     /*
      * DatumGetPointer
      *      Returns pointer value of a datum.
      */
     
     #define DatumGetPointer(X) ((Pointer) (X))
    

_11、AttributeNumberIsValid_

    
    
    //判断属性（号）是否合法
     /* ----------------      *      support macros
      * ----------------      */
     /*
      * AttributeNumberIsValid
      *      True iff the attribute number is valid.
      */
     #define AttributeNumberIsValid(attributeNumber) \
         ((bool) ((attributeNumber) != InvalidAttrNumber))
    

_12、DatumGetHeapTupleHeader_

    
    
     #define DatumGetHeapTupleHeader(X)  ((HeapTupleHeader) PG_DETOAST_DATUM(X))
    

_13、HeapTupleHeaderGetDatumLength_

    
    
     #define HeapTupleHeaderGetDatumLength(tup) \
         VARSIZE(tup)
    

_14、ItemPointerSetInvalid_

    
    
    //块号&块内偏移设置为Invalid
     /*
      * ItemPointerSetInvalid
      *      Sets a disk item pointer to be invalid.
      */
     #define ItemPointerSetInvalid(pointer) \
     ( \
         AssertMacro(PointerIsValid(pointer)), \
         BlockIdSet(&((pointer)->ip_blkid), InvalidBlockNumber), \
         (pointer)->ip_posid = InvalidOffsetNumber \
     )
    

_15、ExecFilterJunk_

    
    
    //去掉所有的Junk属性
    /*
      * ExecFilterJunk
      *
      * Construct and return a slot with all the junk attributes removed.
      */
     TupleTableSlot *
     ExecFilterJunk(JunkFilter *junkfilter, TupleTableSlot *slot)
     {
         TupleTableSlot *resultSlot;
         AttrNumber *cleanMap;
         TupleDesc   cleanTupType;
         int         cleanLength;
         int         i;
         Datum      *values;
         bool       *isnull;
         Datum      *old_values;
         bool       *old_isnull;
     
         /*
          * Extract all the values of the old tuple.
          */
         slot_getallattrs(slot);
         old_values = slot->tts_values;
         old_isnull = slot->tts_isnull;
     
         /*
          * get info from the junk filter
          */
         cleanTupType = junkfilter->jf_cleanTupType;
         cleanLength = cleanTupType->natts;
         cleanMap = junkfilter->jf_cleanMap;
         resultSlot = junkfilter->jf_resultSlot;
     
         /*
          * Prepare to build a virtual result tuple.
          */
         ExecClearTuple(resultSlot);
         values = resultSlot->tts_values;
         isnull = resultSlot->tts_isnull;
     
         /*
          * Transpose data into proper fields of the new tuple.
          */
         for (i = 0; i < cleanLength; i++)
         {
             int         j = cleanMap[i];
     
             if (j == 0)
             {
                 values[i] = (Datum) 0;
                 isnull[i] = true;
             }
             else
             {
                 values[i] = old_values[j - 1];
                 isnull[i] = old_isnull[j - 1];
             }
         }
     
         /*
          * And return the virtual tuple.
          */
         return ExecStoreVirtualTuple(resultSlot);
     }
    

_16、ExecPrepareTupleRouting_

    
    
    //分区表使用，后续再行解析
     /*
      * ExecPrepareTupleRouting --- prepare for routing one tuple
      *
      * Determine the partition in which the tuple in slot is to be inserted,
      * and modify mtstate and estate to prepare for it.
      *
      * Caller must revert the estate changes after executing the insertion!
      * In mtstate, transition capture changes may also need to be reverted.
      *
      * Returns a slot holding the tuple of the partition rowtype.
      */
     static TupleTableSlot *
     ExecPrepareTupleRouting(ModifyTableState *mtstate,
                             EState *estate,
                             PartitionTupleRouting *proute,
                             ResultRelInfo *targetRelInfo,
                             TupleTableSlot *slot)
     {
         ModifyTable *node;
         int         partidx;
         ResultRelInfo *partrel;
         HeapTuple   tuple;
     
         /*
          * Determine the target partition.  If ExecFindPartition does not find a
          * partition after all, it doesn't return here; otherwise, the returned
          * value is to be used as an index into the arrays for the ResultRelInfo
          * and TupleConversionMap for the partition.
          */
         partidx = ExecFindPartition(targetRelInfo,
                                     proute->partition_dispatch_info,
                                     slot,
                                     estate);
         Assert(partidx >= 0 && partidx < proute->num_partitions);
     
         /*
          * Get the ResultRelInfo corresponding to the selected partition; if not
          * yet there, initialize it.
          */
         partrel = proute->partitions[partidx];
         if (partrel == NULL)
             partrel = ExecInitPartitionInfo(mtstate, targetRelInfo,
                                             proute, estate,
                                             partidx);
     
         /*
          * Check whether the partition is routable if we didn't yet
          *
          * Note: an UPDATE of a partition key invokes an INSERT that moves the
          * tuple to a new partition.  This check would be applied to a subplan
          * partition of such an UPDATE that is chosen as the partition to route
          * the tuple to.  The reason we do this check here rather than in
          * ExecSetupPartitionTupleRouting is to avoid aborting such an UPDATE
          * unnecessarily due to non-routable subplan partitions that may not be
          * chosen for update tuple movement after all.
          */
         if (!partrel->ri_PartitionReadyForRouting)
         {
             /* Verify the partition is a valid target for INSERT. */
             CheckValidResultRel(partrel, CMD_INSERT);
     
             /* Set up information needed for routing tuples to the partition. */
             ExecInitRoutingInfo(mtstate, estate, proute, partrel, partidx);
         }
     
         /*
          * Make it look like we are inserting into the partition.
          */
         estate->es_result_relation_info = partrel;
     
         /* Get the heap tuple out of the given slot. */
         tuple = ExecMaterializeSlot(slot);
     
         /*
          * If we're capturing transition tuples, we might need to convert from the
          * partition rowtype to parent rowtype.
          */
         if (mtstate->mt_transition_capture != NULL)
         {
             if (partrel->ri_TrigDesc &&
                 partrel->ri_TrigDesc->trig_insert_before_row)
             {
                 /*
                  * If there are any BEFORE triggers on the partition, we'll have
                  * to be ready to convert their result back to tuplestore format.
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
          */
         ConvertPartitionTupleSlot(proute->parent_child_tupconv_maps[partidx],
                                   tuple,
                                   proute->partition_tuple_slot,
                                   &slot,
                                   true);
     
         /* Initialize information needed to handle ON CONFLICT DO UPDATE. */
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
    

_17、ExecInsert_

    
    
    //上一节已解读
    

_18、ExecUpdate/ExecDelete_

    
    
    //执行更新/删除，在后续实验Update/Delete时再行解析
    

_19、fireASTriggers_

    
    
    //触发执行语句后的触发器（语句级）
     /*
      * Process AFTER EACH STATEMENT triggers
      */
     static void
     fireASTriggers(ModifyTableState *node)
     {
         ModifyTable *plan = (ModifyTable *) node->ps.plan;
         ResultRelInfo *resultRelInfo = getTargetResultRelInfo(node);
     
         switch (node->operation)
         {
             case CMD_INSERT:
                 if (plan->onConflictAction == ONCONFLICT_UPDATE)
                     ExecASUpdateTriggers(node->ps.state,
                                          resultRelInfo,
                                          node->mt_oc_transition_capture);
                 ExecASInsertTriggers(node->ps.state, resultRelInfo,
                                      node->mt_transition_capture);
                 break;
             case CMD_UPDATE:
                 ExecASUpdateTriggers(node->ps.state, resultRelInfo,
                                      node->mt_transition_capture);
                 break;
             case CMD_DELETE:
                 ExecASDeleteTriggers(node->ps.state, resultRelInfo,
                                      node->mt_transition_capture);
                 break;
             default:
                 elog(ERROR, "unknown operation");
                 break;
         }
     }
    

### 二、源码解读

    
    
    /* ----------------------------------------------------------------     *     ExecModifyTable
     *
     *      Perform table modifications as required, and return RETURNING results
     *      if needed.
     * ----------------------------------------------------------------     */
    /*
    输入：
        pstate-SQL语句执行计划的State（各种状态信息）
    输出：
        TupleTableSlot指针，存储执行后Tuple的Slot指针
    */
    static TupleTableSlot *
    ExecModifyTable(PlanState *pstate)
    {
        ModifyTableState *node = castNode(ModifyTableState, pstate);//强制类型转换为ModifyTableState类型
        PartitionTupleRouting *proute = node->mt_partition_tuple_routing;//分区表执行
        EState     *estate = node->ps.state;//执行器Executor状态
        CmdType     operation = node->operation;//命令类型
        ResultRelInfo *saved_resultRelInfo;//结果RelationInfo
        ResultRelInfo *resultRelInfo;//同上
        PlanState  *subplanstate;//子执行计划状态
        JunkFilter *junkfilter;//无用属性过滤器
        TupleTableSlot *slot;//Tuple存储的Slot指针
        TupleTableSlot *planSlot;//Plan存储的Slot指针
        ItemPointer tupleid;//Tuple行指针
        ItemPointerData tuple_ctid;//Tuple行指针数据（Block号、偏移等）
        HeapTupleData oldtupdata;//原Tuple数据
        HeapTuple   oldtuple;//原Tuple数据指针
    
        CHECK_FOR_INTERRUPTS();//检查中断信号
    
        /*
         * This should NOT get called during EvalPlanQual; we should have passed a
         * subplan tree to EvalPlanQual, instead.  Use a runtime test not just
         * Assert because this condition is easy to miss in testing.  (Note:
         * although ModifyTable should not get executed within an EvalPlanQual
         * operation, we do have to allow it to be initialized and shut down in
         * case it is within a CTE subplan.  Hence this test must be here, not in
         * ExecInitModifyTable.)
         */
        if (estate->es_epqTuple != NULL)
            elog(ERROR, "ModifyTable should not be called during EvalPlanQual");
    
        /*
         * If we've already completed processing, don't try to do more.  We need
         * this test because ExecPostprocessPlan might call us an extra time, and
         * our subplan's nodes aren't necessarily robust against being called
         * extra times.
         */
        //已经完成，返回NULL
        if (node->mt_done)
            return NULL;
    
        /*
         * On first call, fire BEFORE STATEMENT triggers before proceeding.
         */
        //语句级触发器触发
        if (node->fireBSTriggers)
        {
            fireBSTriggers(node);
            node->fireBSTriggers = false;
        }
    
        /* Preload local variables */
        //预先构造本地变量
        resultRelInfo = node->resultRelInfo + node->mt_whichplan;//设置正在操作的Relation，mt_whichplan是偏移
        subplanstate = node->mt_plans[node->mt_whichplan];//执行计划的State
        junkfilter = resultRelInfo->ri_junkFilter;//额外（无价值）的信息过滤器
    
        /*
         * es_result_relation_info must point to the currently active result
         * relation while we are within this ModifyTable node.  Even though
         * ModifyTable nodes can't be nested statically, they can be nested
         * dynamically (since our subplan could include a reference to a modifying
         * CTE).  So we have to save and restore the caller's value.
         */
        //切换当前活跃的Relation
        saved_resultRelInfo = estate->es_result_relation_info;
    
        estate->es_result_relation_info = resultRelInfo;
    
        /*
         * Fetch rows from subplan(s), and execute the required table modification
         * for each row.
         */
        for (;;)
        {
            /*
             * Reset the per-output-tuple exprcontext.  This is needed because
             * triggers expect to use that context as workspace.  It's a bit ugly
             * to do this below the top level of the plan, however.  We might need
             * to rethink this later.
             */
            ResetPerTupleExprContext(estate);//设置上下文
    
            planSlot = ExecProcNode(subplanstate);//获取子Plan的Slot（插入数据操作返回值应为NULL）
    
            if (TupIsNull(planSlot))//没有执行计划（执行计划已全部完成）
            {
                /* advance to next subplan if any */
                node->mt_whichplan++;//移至下一个执行Plan
                if (node->mt_whichplan < node->mt_nplans)
                {
                    resultRelInfo++;//切换至相应的Relation，如果是插入操作，只有一个
                    subplanstate = node->mt_plans[node->mt_whichplan];//切换至相应的子Plan状态
                    junkfilter = resultRelInfo->ri_junkFilter;//切换至相应的junkfilter
                    estate->es_result_relation_info = resultRelInfo;//切换
                    EvalPlanQualSetPlan(&node->mt_epqstate, subplanstate->plan,
                                        node->mt_arowmarks[node->mt_whichplan]);//设置子Plan
                    /* Prepare to convert transition tuples from this child. */
                    if (node->mt_transition_capture != NULL)
                    {
                        node->mt_transition_capture->tcs_map =
                            tupconv_map_for_subplan(node, node->mt_whichplan);//插入数据暂不使用
                    }
                    if (node->mt_oc_transition_capture != NULL)
                    {
                        node->mt_oc_transition_capture->tcs_map =
                            tupconv_map_for_subplan(node, node->mt_whichplan);//插入数据暂不使用
                    }
                    continue;
                }
                else
                    break;//所有的Plan均已执行，跳出循环
            }
    
            //存在子Plan
            /*
             * If resultRelInfo->ri_usesFdwDirectModify is true, all we need to do
             * here is compute the RETURNING expressions.
             */
            if (resultRelInfo->ri_usesFdwDirectModify)
            {
                Assert(resultRelInfo->ri_projectReturning);
    
                /*
                 * A scan slot containing the data that was actually inserted,
                 * updated or deleted has already been made available to
                 * ExecProcessReturning by IterateDirectModify, so no need to
                 * provide it here.
                 */
                slot = ExecProcessReturning(resultRelInfo, NULL, planSlot);//FDW，直接返回
    
                estate->es_result_relation_info = saved_resultRelInfo;
                return slot;
            }
    
            EvalPlanQualSetSlot(&node->mt_epqstate, planSlot);//设置相应的Slot（子Plan Slot）
            slot = planSlot;
    
            tupleid = NULL;//Tuple ID
            oldtuple = NULL;//原Tuple
            if (junkfilter != NULL)//去掉废弃的属性
            {
                /*
                 * extract the 'ctid' or 'wholerow' junk attribute.
                 */
                if (operation == CMD_UPDATE || operation == CMD_DELETE)//更新或者删除操作
                {
                    char        relkind;
                    Datum       datum;
                    bool        isNull;
    
                    relkind = resultRelInfo->ri_RelationDesc->rd_rel->relkind;
                    if (relkind == RELKIND_RELATION || relkind == RELKIND_MATVIEW)
                    {
                        datum = ExecGetJunkAttribute(slot,
                                                     junkfilter->jf_junkAttNo,
                                                     &isNull);
                        /* shouldn't ever get a null result... */
                        if (isNull)
                            elog(ERROR, "ctid is NULL");
    
                        tupleid = (ItemPointer) DatumGetPointer(datum);
                        tuple_ctid = *tupleid;  /* be sure we don't free ctid!! */
                        tupleid = &tuple_ctid;
                    }
    
                    /*
                     * Use the wholerow attribute, when available, to reconstruct
                     * the old relation tuple.
                     *
                     * Foreign table updates have a wholerow attribute when the
                     * relation has a row-level trigger.  Note that the wholerow
                     * attribute does not carry system columns.  Foreign table
                     * triggers miss seeing those, except that we know enough here
                     * to set t_tableOid.  Quite separately from this, the FDW may
                     * fetch its own junk attrs to identify the row.
                     *
                     * Other relevant relkinds, currently limited to views, always
                     * have a wholerow attribute.
                     */
                    else if (AttributeNumberIsValid(junkfilter->jf_junkAttNo))
                    {
                        datum = ExecGetJunkAttribute(slot,
                                                     junkfilter->jf_junkAttNo,
                                                     &isNull);
                        /* shouldn't ever get a null result... */
                        if (isNull)
                            elog(ERROR, "wholerow is NULL");
    
                        oldtupdata.t_data = DatumGetHeapTupleHeader(datum);
                        oldtupdata.t_len =
                            HeapTupleHeaderGetDatumLength(oldtupdata.t_data);
                        ItemPointerSetInvalid(&(oldtupdata.t_self));
                        /* Historically, view triggers see invalid t_tableOid. */
                        oldtupdata.t_tableOid =
                            (relkind == RELKIND_VIEW) ? InvalidOid :
                            RelationGetRelid(resultRelInfo->ri_RelationDesc);
    
                        oldtuple = &oldtupdata;
                    }
                    else
                        Assert(relkind == RELKIND_FOREIGN_TABLE);
                }
    
                /*
                 * apply the junkfilter if needed.
                 */
                if (operation != CMD_DELETE)
                    slot = ExecFilterJunk(junkfilter, slot);//去掉废弃的属性
            }
    
            switch (operation)//根据操作类型，执行相应的操作
            {
                case CMD_INSERT://插入
                    /* Prepare for tuple routing if needed. */
                    if (proute)
                        slot = ExecPrepareTupleRouting(node, estate, proute,
                                                       resultRelInfo, slot);//准备相应的Tuple Slot
                    slot = ExecInsert(node, slot, planSlot,
                                      estate, node->canSetTag);//执行插入
                    /* Revert ExecPrepareTupleRouting's state change. */
                    if (proute)
                        estate->es_result_relation_info = resultRelInfo;
                    break;
                case CMD_UPDATE:
                    slot = ExecUpdate(node, tupleid, oldtuple, slot, planSlot,
                                      &node->mt_epqstate, estate, node->canSetTag);
                    break;
                case CMD_DELETE:
                    slot = ExecDelete(node, tupleid, oldtuple, planSlot,
                                      &node->mt_epqstate, estate,
                                      NULL, true, node->canSetTag,
                                      false /* changingPart */ );
                    break;
                default:
                    elog(ERROR, "unknown operation");
                    break;
            }
    
            /*
             * If we got a RETURNING result, return it to caller.  We'll continue
             * the work on next call.
             */
            if (slot)
            {
                estate->es_result_relation_info = saved_resultRelInfo;
                return slot;
            }
        }
        /* Restore es_result_relation_info before exiting */
        estate->es_result_relation_info = saved_resultRelInfo;
    
        /*
         * We're done, but fire AFTER STATEMENT triggers before exiting.
         */
        fireASTriggers(node);
    
        node->mt_done = true;
        //Insert语句，返回NULL
        return NULL;
    }
    

### 三、跟踪分析

执行测试脚本：

    
    
    alter table t_insert alter column c1 type varchar(40);
    alter table t_insert alter column c2 type varchar(40);
    alter table t_insert alter column c3 type varchar(40);
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1570
    (1 row)
    testdb=# insert into t_insert values(13,'ExecModifyTable','ExecModifyTable','ExecModifyTable');
    (挂起)
    
    

启动gdb跟踪：

    
    
    [root@localhost ~]# gdb -p 1570
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b ExecModifyTable
    Breakpoint 1 at 0x6c2498: file nodeModifyTable.c, line 1915.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecModifyTable (pstate=0x2cde640) at nodeModifyTable.c:1915
    1915        ModifyTableState *node = castNode(ModifyTableState, pstate);
    #查看参数
    (gdb) p *pstate
    #PlanState
    $1 = {type = T_ModifyTableState, plan = 0x2ce66c8, state = 0x2cde2f0, ExecProcNode = 0x6c2485 <ExecModifyTable>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
      worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2cdf3f0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    #执行计划Plan
    (gdb) p *(pstate->plan)
    $2 = {type = T_ModifyTable, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, parallel_aware = false, parallel_safe = false, plan_node_id = 0, targetlist = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    #执行计划对应的执行环境（State）
    (gdb) p *(pstate->state)
    $3 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x2ccac80, es_crosscheck_snapshot = 0x0, es_range_table = 0x2ce6970, es_plannedstmt = 0x2ce6a40, 
      es_sourceText = 0x2c1bef0 "insert into t_insert values(13,'ExecModifyTable','ExecModifyTable','ExecModifyTable');", es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x2cde530, 
      es_num_result_relations = 1, es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, 
      es_trig_tuple_slot = 0x2cdf4a0, es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x2cde500, es_queryEnv = 0x0, es_query_cxt = 0x2cde1e0, 
      es_tupleTable = 0x2cdeef0, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, es_top_eflags = 0, es_instrument = 0, es_finished = false, es_exprcontexts = 0x2cde8a0, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0}
    #结果Tuple Slot
    (gdb) p *(pstate->ps_ResultTupleSlot)
    $6 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x2cdf3c0, tts_mcxt = 0x2cde1e0, 
      tts_buffer = 0, tts_nvalid = 0, tts_values = 0x2cdf450, tts_isnull = 0x2cdf450, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    #继续执行
    #proute应为NULL
    (gdb) p proute
    $7 = (PartitionTupleRouting *) 0x0
    #插入操作
    (gdb) p operation
    $9 = CMD_INSERT
    #查看本地变量
    (gdb) p resultRelInfo
    $11 = (ResultRelInfo *) 0x2cde530
    (gdb) p *resultRelInfo
    $12 = {type = T_ResultRelInfo, ri_RangeTableIndex = 1, ri_RelationDesc = 0x7f3c13f56150, ri_NumIndices = 1, ri_IndexRelationDescs = 0x2cde8d0, ri_IndexRelationInfo = 0x2cde8e8, ri_TrigDesc = 0x0, 
      ri_TrigFunctions = 0x0, ri_TrigWhenExprs = 0x0, ri_TrigInstrument = 0x0, ri_FdwRoutine = 0x0, ri_FdwState = 0x0, ri_usesFdwDirectModify = false, ri_WithCheckOptions = 0x0, 
      ri_WithCheckOptionExprs = 0x0, ri_ConstraintExprs = 0x0, ri_junkFilter = 0x0, ri_returningList = 0x0, ri_projectReturning = 0x0, ri_onConflictArbiterIndexes = 0x0, ri_onConflict = 0x0, 
      ri_PartitionCheck = 0x0, ri_PartitionCheckExpr = 0x0, ri_PartitionRoot = 0x0, ri_PartitionReadyForRouting = false}
    (gdb) p subplanstate
    $13 = (PlanState *) 0x2cdea10
    (gdb) p *subplanstate
    $14 = {type = T_ResultState, plan = 0x2cd21f8, state = 0x2cde2f0, ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, worker_instrument = 0x0, 
      qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2cdedc0, ps_ExprContext = 0x2cdeb20, ps_ProjInfo = 0x2cdef20, scandesc = 0x0}
    (gdb) p *(subplanstate->plan)
    $15 = {type = T_Result, startup_cost = 0, total_cost = 0.01, plan_rows = 1, plan_width = 298, parallel_aware = false, parallel_safe = false, plan_node_id = 1, targetlist = 0x2cd22f8, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p junkfilter
    $18 = (JunkFilter *) 0x830
    (gdb) p *junkfilter
    Cannot access memory at address 0x830
    gdb) next
    1974        saved_resultRelInfo = estate->es_result_relation_info;
    (gdb) 
    1976        estate->es_result_relation_info = resultRelInfo;
    (gdb) 
    1990            ResetPerTupleExprContext(estate);
    (gdb) 
    1992            planSlot = ExecProcNode(subplanstate);
    (gdb) p *planSlot
    $19 = {type = T_Invalid, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x3000000000064, tts_tupleDescriptor = 0x2c1dcb8, tts_mcxt = 0x2c, 
      tts_buffer = 0, tts_nvalid = 0, tts_values = 0x0, tts_isnull = 0x0, tts_mintuple = 0x0, tts_minhdr = {t_len = 512, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 57824}, t_tableOid = 0, 
        t_data = 0x3c}, tts_off = 47081160, tts_fixedTupleDescriptor = false}
    (gdb) next
    1994            if (TupIsNull(planSlot))
    (gdb) p *planSlot
    $20 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x0, tts_tupleDescriptor = 0x2cdebb0, tts_mcxt = 0x2cde1e0, 
      tts_buffer = 0, tts_nvalid = 4, tts_values = 0x2cdee20, tts_isnull = 0x2cdee40, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) next
    2027            if (resultRelInfo->ri_usesFdwDirectModify)
    (gdb) 
    2043            EvalPlanQualSetSlot(&node->mt_epqstate, planSlot);
    (gdb) 
    2044            slot = planSlot;
    (gdb) 
    2046            tupleid = NULL;
    (gdb) 
    2047            oldtuple = NULL;
    (gdb) 
    2048            if (junkfilter != NULL)
    (gdb) 
    2119            switch (operation)
    (gdb) p junkfilter
    $21 = (JunkFilter *) 0x0
    (gdb) next
    2123                    if (proute)
    (gdb) 
    2127                                      estate, node->canSetTag);
    (gdb) 
    2126                    slot = ExecInsert(node, slot, planSlot,
    (gdb) 
    2129                    if (proute)
    (gdb) p *slot
    Cannot access memory at address 0x0
    (gdb) next
    2151            if (slot)
    (gdb) 
    2156        }
    #进入第2轮循环
    (gdb) 
    1990            ResetPerTupleExprContext(estate);
    (gdb) 
    1992            planSlot = ExecProcNode(subplanstate);
    (gdb) p *planSlot
    $22 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = true, tts_shouldFreeMin = false, tts_slow = false, tts_tuple = 0x2cdf550, tts_tupleDescriptor = 0x2cdebb0, tts_mcxt = 0x2cde1e0, 
      tts_buffer = 0, tts_nvalid = 1, tts_values = 0x2cdee20, tts_isnull = 0x2cdee40, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, 
        t_tableOid = 0, t_data = 0x0}, tts_off = 4, tts_fixedTupleDescriptor = true}
    (gdb) next
    1994            if (TupIsNull(planSlot))
    (gdb) 
    1997                node->mt_whichplan++;
    (gdb) next
    1998                if (node->mt_whichplan < node->mt_nplans)
    (gdb) p *node
    $23 = {ps = {type = T_ModifyTableState, plan = 0x2ce66c8, state = 0x2cde2f0, ExecProcNode = 0x6c2485 <ExecModifyTable>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
        worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2cdf3f0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
        scandesc = 0x0}, operation = CMD_INSERT, canSetTag = true, mt_done = false, mt_plans = 0x2cde850, mt_nplans = 1, mt_whichplan = 1, resultRelInfo = 0x2cde530, rootResultRelInfo = 0x0, 
      mt_arowmarks = 0x2cde868, mt_epqstate = {estate = 0x0, planstate = 0x0, origslot = 0x2cdedc0, plan = 0x2cd21f8, arowMarks = 0x0, epqParam = 0}, fireBSTriggers = false, mt_existing = 0x0, 
      mt_excludedtlist = 0x0, mt_conflproj = 0x0, mt_partition_tuple_routing = 0x0, mt_transition_capture = 0x0, mt_oc_transition_capture = 0x0, mt_per_subplan_tupconv_maps = 0x0}
    //执行完毕，跳出循环
    (gdb) next
    2020                    break;
    (gdb) p *slot
    Cannot access memory at address 0x0
    (gdb) next
    2159        estate->es_result_relation_info = saved_resultRelInfo;
    (gdb) p saved_resultRelInfo
    $24 = (ResultRelInfo *) 0x0
    (gdb) next
    2164        fireASTriggers(node);
    (gdb) next
    2166        node->mt_done = true;
    (gdb) 
    #执行成功，返回NULL（Insert语句）
    2168        return NULL;
    

### 四、小结

1、执行模型：INSERT/UPDATE/DELETE语句与SELECT使用了同一套的执行模型，如何区分和处理，需要进一步的深入理解；  
2、执行状态：PG在执行某个Statement时，维护了一系列的状态，这些状态的数据结构需要进一步的了解。

