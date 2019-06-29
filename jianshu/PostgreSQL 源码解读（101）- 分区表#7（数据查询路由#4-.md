本节是PG在查询分区表的时候如何确定查询的是哪个分区逻辑介绍的第二部分。在规划阶段,函数set_rel_size中，如RTE为分区表（rte->inh=T），则调用set_append_rel_size函数，在set_append_rel_size中通过prune_append_rel_partitions函数获取“可保留”的分区。  
本节的内容是介绍prune_append_rel_partitions->get_matching_partitions函数。

### 一、数据结构

**PartitionScheme**  
分区方案,根据设计，分区方案只包含分区方法的一般属性(列表与范围、分区列的数量和每个分区列的类型信息)，而不包含特定的分区边界信息。

    
    
    /*
     * If multiple relations are partitioned the same way, all such partitions
     * will have a pointer to the same PartitionScheme.  A list of PartitionScheme
     * objects is attached to the PlannerInfo.  By design, the partition scheme
     * incorporates only the general properties of the partition method (LIST vs.
     * RANGE, number of partitioning columns and the type information for each)
     * and not the specific bounds.
     * 如果多个关系以相同的方式分区，那么所有这些分区都将具有指向相同PartitionScheme的指针。
     * PartitionScheme对象的链表附加到PlannerInfo中。
     * 根据设计，分区方案只包含分区方法的一般属性(列表与范围、分区列的数量和每个分区列的类型信息)，
     *   而不包含特定的界限。
     *
     * We store the opclass-declared input data types instead of the partition key
     * datatypes since the former rather than the latter are used to compare
     * partition bounds. Since partition key data types and the opclass declared
     * input data types are expected to be binary compatible (per ResolveOpClass),
     * both of those should have same byval and length properties.
     * 我们存储opclass-declared的输入数据类型，而不是分区键数据类型，
     *   因为前者用于比较分区边界，而不是后者。
     * 由于分区键数据类型和opclass-declared的输入数据类型预期是二进制兼容的(每个ResolveOpClass)，
     *   所以它们应该具有相同的byval和length属性。
     */
    typedef struct PartitionSchemeData
    {
        char        strategy;       /* 分区策略;partition strategy */
        int16       partnatts;      /* 分区属性个数;number of partition attributes */
        Oid        *partopfamily;   /* 操作符族OIDs;OIDs of operator families */
        Oid        *partopcintype;  /* opclass声明的输入数据类型的OIDs;OIDs of opclass declared input data types */
        Oid        *partcollation;  /* 分区排序规则OIDs;OIDs of partitioning collations */
    
        /* Cached information about partition key data types. */
        //缓存有关分区键数据类型的信息。
        int16      *parttyplen;
        bool       *parttypbyval;
    
        /* Cached information about partition comparison functions. */
        //缓存有关分区比较函数的信息。
        FmgrInfo   *partsupfunc;
    }           PartitionSchemeData;
    
    typedef struct PartitionSchemeData *PartitionScheme;
    
    

**PartitionPruneXXX**  
执行Prune期间需要使用的数据结构,包括PartitionPruneStep/PartitionPruneStepOp/PartitionPruneCombineOp/PartitionPruneStepCombine

    
    
    /*
     * Abstract Node type for partition pruning steps (there are no concrete
     * Nodes of this type).
     * 用于分区修剪步骤pruning的抽象节点类型(没有这种类型的具体节点)。
     * 
     * step_id is the global identifier of the step within its pruning context.
     * step_id是步骤在其修剪pruning上下文中的全局标识符。
     */
    typedef struct PartitionPruneStep
    {
        NodeTag     type;
        int         step_id;
    } PartitionPruneStep;
    
     /*
     * PartitionPruneStepOp - Information to prune using a set of mutually AND'd
     *                          OpExpr clauses
     * PartitionPruneStepOp - 使用一组AND操作的OpExpr条件子句进行修剪prune的信息
     *
     * This contains information extracted from up to partnatts OpExpr clauses,
     * where partnatts is the number of partition key columns.  'opstrategy' is the
     * strategy of the operator in the clause matched to the last partition key.
     * 'exprs' contains expressions which comprise the lookup key to be passed to
     * the partition bound search function.  'cmpfns' contains the OIDs of
     * comparison functions used to compare aforementioned expressions with
     * partition bounds.  Both 'exprs' and 'cmpfns' contain the same number of
     * items, up to partnatts items.
     * 它包含从partnatts OpExpr子句中提取的信息，
     *   其中partnatts是分区键列的数量。
     * “opstrategy”是子句中与最后一个分区键匹配的操作符的策略。
     * 'exprs'包含一些表达式，这些表达式包含要传递给分区绑定搜索函数的查找键。
     * “cmpfns”包含用于比较上述表达式与分区边界的比较函数的OIDs。
     * “exprs”和“cmpfns”包含相同数量的条目，最多包含partnatts个条目。
     *
     * Once we find the offset of a partition bound using the lookup key, we
     * determine which partitions to include in the result based on the value of
     * 'opstrategy'.  For example, if it were equality, we'd return just the
     * partition that would contain that key or a set of partitions if the key
     * didn't consist of all partitioning columns.  For non-equality strategies,
     * we'd need to include other partitions as appropriate.
     * 一旦我们使用查找键找到分区绑定的偏移量，
     *   我们将根据“opstrategy”的值确定在结果中包含哪些分区。
     * 例如，如果它是相等的，我们只返回包含该键的分区，或者如果该键不包含所有分区列，
     *  则返回一组分区。
     * 对于非等值的情况，需要适当地包括其他分区。
     *
     * 'nullkeys' is the set containing the offset of the partition keys (0 to
     * partnatts - 1) that were matched to an IS NULL clause.  This is only
     * considered for hash partitioning as we need to pass which keys are null
     * to the hash partition bound search function.  It is never possible to
     * have an expression be present in 'exprs' for a given partition key and
     * the corresponding bit set in 'nullkeys'.
     * 'nullkeys'是包含与is NULL子句匹配的分区键(0到partnatts - 1)偏移量的集合。
     * 这只适用于哈希分区，因为我们需要将哪些键为null传递给哈希分区绑定搜索函数。
     * 对于给定的分区键和“nullkeys”中设置的相应bit，不可能在“exprs”中出现表达式。
     */
    typedef struct PartitionPruneStepOp
    {
        PartitionPruneStep step;
    
        StrategyNumber opstrategy;
        List       *exprs;
        List       *cmpfns;
        Bitmapset  *nullkeys;
    } PartitionPruneStepOp;
    
    /*
     * PartitionPruneStepCombine - Information to prune using a BoolExpr clause
     * PartitionPruneStepCombine - 使用BoolExpr条件prune的信息
     *
     * For BoolExpr clauses, we combine the set of partitions determined for each
     * of the argument clauses.
     * 对于BoolExpr子句，我们为每个参数子句确定的分区集进行组合。
     */
    typedef enum PartitionPruneCombineOp
    {
        PARTPRUNE_COMBINE_UNION,
        PARTPRUNE_COMBINE_INTERSECT
    } PartitionPruneCombineOp;
    
    typedef struct PartitionPruneStepCombine
    {
        PartitionPruneStep step;
    
        PartitionPruneCombineOp combineOp;
        List       *source_stepids;
    } PartitionPruneStepCombine;
    
    
    /* The result of performing one PartitionPruneStep */
    //执行PartitionPruneStep步骤后的结果
    typedef struct PruneStepResult
    {
        /*
         * The offsets of bounds (in a table's boundinfo) whose partition is
         * selected by the pruning step.
         * 被pruning步骤选中的分区边界(在数据表boundinfo中)偏移
         */
        Bitmapset  *bound_offsets;
    
        bool        scan_default;   /* 是否扫描默认分区? Scan the default partition? */
        bool        scan_null;      /* 是否为NULL值扫描分区? Scan the partition for NULL values? */
    } PruneStepResult;
     
    

### 二、源码解读

get_matching_partitions函数确定在分区pruning后仍然"存活"的分区。

    
    
     /*
     * get_matching_partitions
     *      Determine partitions that survive partition pruning
     *      确定在分区修剪pruning后仍然存在的分区.
     *
     * Returns a Bitmapset of the RelOptInfo->part_rels indexes of the surviving
     * partitions.
     * 返回pruning后仍存在的分区的RelOptInfo->part_rels索引位图集。
     */
    Bitmapset *
    get_matching_partitions(PartitionPruneContext *context, List *pruning_steps)
    {
        Bitmapset  *result;
        int         num_steps = list_length(pruning_steps),
                    i;
        PruneStepResult **results,
                   *final_result;
        ListCell   *lc;
    
        /* If there are no pruning steps then all partitions match. */
        //没有pruning步骤,则视为保留所有分区
        if (num_steps == 0)
        {
            Assert(context->nparts > 0);
            return bms_add_range(NULL, 0, context->nparts - 1);
        }
    
        /*
         * Allocate space for individual pruning steps to store its result.  Each
         * slot will hold a PruneStepResult after performing a given pruning step.
         * Later steps may use the result of one or more earlier steps.  The
         * result of applying all pruning steps is the value contained in the slot
         * of the last pruning step.
         * 为单个修剪步骤分配空间来存储结果。
         * 每个slot将持有pruning后，执行一个给定的pruning步骤。
         * 后面的步骤可以使用前面一个或多个步骤的结果。
         * 应用所有步骤的结果是最后一个步骤的slot中包含的值。
         */
        results = (PruneStepResult **)
            palloc0(num_steps * sizeof(PruneStepResult *));
        foreach(lc, pruning_steps)//遍历步骤
        {
            PartitionPruneStep *step = lfirst(lc);
    
            switch (nodeTag(step))
            {
                case T_PartitionPruneStepOp:
                    results[step->step_id] =
                        perform_pruning_base_step(context,
                                                  (PartitionPruneStepOp *) step);//执行pruning基础步骤
                    break;
    
                case T_PartitionPruneStepCombine:
                    results[step->step_id] =
                        perform_pruning_combine_step(context,
                                                     (PartitionPruneStepCombine *) step,
                                                     results);//执行pruning组合步骤
                    break;
    
                default:
                    elog(ERROR, "invalid pruning step type: %d",
                         (int) nodeTag(step));
            }
        }
    
        /*
         * At this point we know the offsets of all the datums whose corresponding
         * partitions need to be in the result, including special null-accepting
         * and default partitions.  Collect the actual partition indexes now.
         * 到目前为止,我们已经知道结果中需要的相应分区的所有数据的偏移量，
         *   包括特殊的接受null的分区和默认分区。
         * 现在收集实际的分区索引。
         */
        final_result = results[num_steps - 1];//最终结果
        Assert(final_result != NULL);
        i = -1;
        result = NULL;
        while ((i = bms_next_member(final_result->bound_offsets, i)) >= 0)
        {
            int         partindex = context->boundinfo->indexes[i];//分区编号
    
            /*
             * In range and hash partitioning cases, some slots may contain -1,
             * indicating that no partition has been defined to accept a given
             * range of data or for a given remainder, respectively. The default
             * partition, if any, in case of range partitioning, will be added to
             * the result, because the specified range still satisfies the query's
             * conditions.
             * 范围分区和散列分区，一些slot可能包含-1，
             *   这表示没有定义接受给定范围的数据或给定余数的分区。
             * 在范围分区的情况下，默认分区(如果有的话)将被添加到结果中，
             *   因为指定的范围仍然满足查询的条件。
             */
            if (partindex >= 0)
                result = bms_add_member(result, partindex);
        }
    
        /* Add the null and/or default partition if needed and if present. */
        //如果需要，添加NULL和/或默认分区。
        if (final_result->scan_null)
        {
            Assert(context->strategy == PARTITION_STRATEGY_LIST);
            Assert(partition_bound_accepts_nulls(context->boundinfo));
            result = bms_add_member(result, context->boundinfo->null_index);
        }
        if (final_result->scan_default)
        {
            Assert(context->strategy == PARTITION_STRATEGY_LIST ||
                   context->strategy == PARTITION_STRATEGY_RANGE);
            Assert(partition_bound_has_default(context->boundinfo));
            result = bms_add_member(result, context->boundinfo->default_index);
        }
    
        return result;
    }
    
    
     /*
     * perform_pruning_base_step
     *      Determines the indexes of datums that satisfy conditions specified in
     *      'opstep'.
     *      确定满足“opstep”中指定条件的数据索引。
     *
     * Result also contains whether special null-accepting and/or default
     * partition need to be scanned.
     * 结果还包含是否需要扫描特殊的可接受null和/或默认分区。
     */
    static PruneStepResult *
    perform_pruning_base_step(PartitionPruneContext *context,
                              PartitionPruneStepOp *opstep)
    {
        ListCell   *lc1,
                   *lc2;
        int         keyno,
                    nvalues;
        Datum       values[PARTITION_MAX_KEYS];
        FmgrInfo   *partsupfunc;
        int         stateidx;
    
        /*
         * There better be the same number of expressions and compare functions.
         * 最好有相同数量的表达式和比较函数。
         */
        Assert(list_length(opstep->exprs) == list_length(opstep->cmpfns));
    
        nvalues = 0;
        lc1 = list_head(opstep->exprs);
        lc2 = list_head(opstep->cmpfns);
    
        /*
         * Generate the partition lookup key that will be used by one of the
         * get_matching_*_bounds functions called below.
         * 生成将由下面调用的get_matching_*_bounds函数使用的分区查找键。
         */
        for (keyno = 0; keyno < context->partnatts; keyno++)
        {
            /*
             * For hash partitioning, it is possible that values of some keys are
             * not provided in operator clauses, but instead the planner found
             * that they appeared in a IS NULL clause.
             * 对于哈希分区，操作符子句中可能没有提供某些键的值，
             *   但是计划器发现它们出现在is NULL子句中。
             */
            if (bms_is_member(keyno, opstep->nullkeys))
                continue;
    
            /*
             * For range partitioning, we must only perform pruning with values
             * for either all partition keys or a prefix thereof.
             * 对于范围分区，我们必须只对所有分区键或其前缀执行值修剪pruning。
             */
            if (keyno > nvalues && context->strategy == PARTITION_STRATEGY_RANGE)
                break;
    
            if (lc1 != NULL)//步骤的条件表达式不为NULL
            {
                Expr       *expr;
                Datum       datum;
                bool        isnull;
    
                expr = lfirst(lc1);
                stateidx = PruneCxtStateIdx(context->partnatts,
                                            opstep->step.step_id, keyno);
                if (partkey_datum_from_expr(context, expr, stateidx,
                                            &datum, &isnull))
                {
                    Oid         cmpfn;
    
                    /*
                     * Since we only allow strict operators in pruning steps, any
                     * null-valued comparison value must cause the comparison to
                     * fail, so that no partitions could match.
                     * 由于我们只允许在修剪pruning步骤中使用严格的操作符，
                     *   任何空值比较值都必须导致比较失败，这样就没有分区能够匹配。
                     */
                    if (isnull)
                    {
                        PruneStepResult *result;
    
                        result = (PruneStepResult *) palloc(sizeof(PruneStepResult));
                        result->bound_offsets = NULL;
                        result->scan_default = false;
                        result->scan_null = false;
    
                        return result;
                    }
    
                    /* Set up the stepcmpfuncs entry, unless we already did */
                    //配置stepcmpfuncs(步骤比较函数)入口
                    cmpfn = lfirst_oid(lc2);
                    Assert(OidIsValid(cmpfn));
                    if (cmpfn != context->stepcmpfuncs[stateidx].fn_oid)
                    {
                        /*
                         * If the needed support function is the same one cached
                         * in the relation's partition key, copy the cached
                         * FmgrInfo.  Otherwise (i.e., when we have a cross-type
                         * comparison), an actual lookup is required.
                         * 如果所需的支持函数与关系分区键缓存的支持函数相同，
                         *   则复制缓存的FmgrInfo。
                         * 否则(比如存在一个跨类型比较时)，需要实际的查找。
                         */
                        if (cmpfn == context->partsupfunc[keyno].fn_oid)
                            fmgr_info_copy(&context->stepcmpfuncs[stateidx],
                                           &context->partsupfunc[keyno],
                                           context->ppccontext);
                        else
                            fmgr_info_cxt(cmpfn, &context->stepcmpfuncs[stateidx],
                                          context->ppccontext);
                    }
    
                    values[keyno] = datum;
                    nvalues++;
                }
    
                lc1 = lnext(lc1);
                lc2 = lnext(lc2);
            }
        }
    
        /*
         * Point partsupfunc to the entry for the 0th key of this step; the
         * additional support functions, if any, follow consecutively.
         * 将partsupfunc指向此步骤第0个键的条目;
         *   附加的支持功能(如果有的话)是连续的。
         */
        stateidx = PruneCxtStateIdx(context->partnatts, opstep->step.step_id, 0);
        partsupfunc = &context->stepcmpfuncs[stateidx];
    
        switch (context->strategy)
        {
            case PARTITION_STRATEGY_HASH:
                return get_matching_hash_bounds(context,
                                                opstep->opstrategy,
                                                values, nvalues,
                                                partsupfunc,
                                                opstep->nullkeys);
    
            case PARTITION_STRATEGY_LIST:
                return get_matching_list_bounds(context,
                                                opstep->opstrategy,
                                                values[0], nvalues,
                                                &partsupfunc[0],
                                                opstep->nullkeys);
    
            case PARTITION_STRATEGY_RANGE:
                return get_matching_range_bounds(context,
                                                 opstep->opstrategy,
                                                 values, nvalues,
                                                 partsupfunc,
                                                 opstep->nullkeys);
    
            default:
                elog(ERROR, "unexpected partition strategy: %d",
                     (int) context->strategy);
                break;
        }
    
        return NULL;
    }
    
    /*
     * perform_pruning_combine_step
     *      Determines the indexes of datums obtained by combining those given
     *      by the steps identified by cstep->source_stepids using the specified
     *      combination method
     *      使用指定的组合方法将cstep->source_stepids标识的步骤组合在一起得到数据索引
     *
     * Since cstep may refer to the result of earlier steps, we also receive
     * step_results here.
     * 因为cstep可能引用前面步骤的结果，所以在这里也会收到step_results。
     */
    static PruneStepResult *
    perform_pruning_combine_step(PartitionPruneContext *context,
                                 PartitionPruneStepCombine *cstep,
                                 PruneStepResult **step_results)
    {
        ListCell   *lc1;
        PruneStepResult *result = NULL;
        bool        firststep;
    
        /*
         * A combine step without any source steps is an indication to not perform
         * any partition pruning, we just return all partitions.
         * 如无源步骤,则不执行分区pruning,返回所有分区
         */
        result = (PruneStepResult *) palloc0(sizeof(PruneStepResult));
        if (list_length(cstep->source_stepids) == 0)
        {
            PartitionBoundInfo boundinfo = context->boundinfo;
    
            result->bound_offsets = bms_add_range(NULL, 0, boundinfo->ndatums - 1);
            result->scan_default = partition_bound_has_default(boundinfo);
            result->scan_null = partition_bound_accepts_nulls(boundinfo);
            return result;
        }
    
        switch (cstep->combineOp)//根据组合操作类型确定相应逻辑
        {
            case PARTPRUNE_COMBINE_UNION://PARTPRUNE_COMBINE_UNION
                foreach(lc1, cstep->source_stepids)
                {
                    int         step_id = lfirst_int(lc1);
                    PruneStepResult *step_result;
    
                    /*
                     * step_results[step_id] must contain a valid result, which is
                     * confirmed by the fact that cstep's step_id is greater than
                     * step_id and the fact that results of the individual steps
                     * are evaluated in sequence of their step_ids.
                     * step_results[step_id]必须包含一个有效的结果，
                     *   cstep的step_id大于step_id，并且各个步骤的结果按其step_id的顺序计算，
                     *   这一点是可以确认的。
                     */
                    if (step_id >= cstep->step.step_id)
                        elog(ERROR, "invalid pruning combine step argument");
                    step_result = step_results[step_id];
                    Assert(step_result != NULL);
    
                    /* Record any additional datum indexes from this step */
                    //记录从该步骤产生的偏移索引
                    result->bound_offsets = bms_add_members(result->bound_offsets,
                                                            step_result->bound_offsets);
    
                    /* Update whether to scan null and default partitions. */
                    //更新扫描null/default分区的标记
                    if (!result->scan_null)
                        result->scan_null = step_result->scan_null;
                    if (!result->scan_default)
                        result->scan_default = step_result->scan_default;
                }
                break;
    
            case PARTPRUNE_COMBINE_INTERSECT://PARTPRUNE_COMBINE_INTERSECT
                firststep = true;
                foreach(lc1, cstep->source_stepids)
                {
                    int         step_id = lfirst_int(lc1);
                    PruneStepResult *step_result;
    
                    if (step_id >= cstep->step.step_id)
                        elog(ERROR, "invalid pruning combine step argument");
                    step_result = step_results[step_id];
                    Assert(step_result != NULL);
    
                    if (firststep)//第一个步骤
                    {
                        /* Copy step's result the first time. */
                      //第一次,拷贝步骤的结果
                        result->bound_offsets =
                            bms_copy(step_result->bound_offsets);
                        result->scan_null = step_result->scan_null;
                        result->scan_default = step_result->scan_default;
                        firststep = false;
                    }
                    else
                    {
                        /* Record datum indexes common to both steps */
                        //记录其他步骤产生的索引
                        result->bound_offsets =
                            bms_int_members(result->bound_offsets,
                                            step_result->bound_offsets);
    
                        /* Update whether to scan null and default partitions. */
                        //更新扫描null/default分区的标记
                        if (result->scan_null)
                            result->scan_null = step_result->scan_null;
                        if (result->scan_default)
                            result->scan_default = step_result->scan_default;
                    }
                }
                break;
        }
    
        return result;
    }
    
    
    /*
     * get_matching_hash_bounds
     *      Determine offset of the hash bound matching the specified values,
     *      considering that all the non-null values come from clauses containing
     *      a compatible hash equality operator and any keys that are null come
     *      from an IS NULL clause.
     *      考虑到所有非空值都来自包含兼容的哈希相等操作符的子句，
     *        而所有空键都来自IS NULL子句，因此确定匹配指定值的哈希边界的偏移量。
     *
     * Generally this function will return a single matching bound offset,
     * although if a partition has not been setup for a given modulus then we may
     * return no matches.  If the number of clauses found don't cover the entire
     * partition key, then we'll need to return all offsets.
     * 通常，这个函数会返回一个匹配的边界偏移量，
     *   但是如果没有为给定的模数设置分区，则可能不返回匹配值。
     * 如果找到的子句数量不包含整个分区键，那么我们需要返回所有偏移量。
     *
     * 'opstrategy' if non-zero must be HTEqualStrategyNumber.
     *  opstrategy - 如非0,则为HTEqualStrategyNumber
     *
     * 'values' contains Datums indexed by the partition key to use for pruning.
     *  values 包含用于分区键执行pruning的数据值
     *
     * 'nvalues', the number of Datums in the 'values' array.
     *  nvalues values数组大小
     *
     * 'partsupfunc' contains partition hashing functions that can produce correct
     * hash for the type of the values contained in 'values'.
     *  partsupfunc 存储分区hash函数,可以为values产生hash值
     * 
     * 'nullkeys' is the set of partition keys that are null.
     *  nullkeys 为null的分区键值集合
     */
    static PruneStepResult *
    get_matching_hash_bounds(PartitionPruneContext *context,
                             StrategyNumber opstrategy, Datum *values, int nvalues,
                             FmgrInfo *partsupfunc, Bitmapset *nullkeys)
    {
        PruneStepResult *result = (PruneStepResult *) palloc0(sizeof(PruneStepResult));
        PartitionBoundInfo boundinfo = context->boundinfo;
        int        *partindices = boundinfo->indexes;
        int         partnatts = context->partnatts;
        bool        isnull[PARTITION_MAX_KEYS];
        int         i;
        uint64      rowHash;
        int         greatest_modulus;
    
        Assert(context->strategy == PARTITION_STRATEGY_HASH);
    
        /*
         * For hash partitioning we can only perform pruning based on equality
         * clauses to the partition key or IS NULL clauses.  We also can only
         * prune if we got values for all keys.
         * 对于Hash分区,只能基于等值条件语句或者是IS NULL条件进行pruning.
         * 当然,如果可以拿到所有键的值,也可以执行prune
         */
        if (nvalues + bms_num_members(nullkeys) == partnatts)
        {
            /*
             * If there are any values, they must have come from clauses
             * containing an equality operator compatible with hash partitioning.
             * 如存在values,那么这些值必须从包含一个与hash分区兼容的等值操作符的条件语句而来
             */
            Assert(opstrategy == HTEqualStrategyNumber || nvalues == 0);
    
            for (i = 0; i < partnatts; i++)
                isnull[i] = bms_is_member(i, nullkeys);
    
            greatest_modulus = get_hash_partition_greatest_modulus(boundinfo);
            rowHash = compute_partition_hash_value(partnatts, partsupfunc,
                                                   values, isnull);
    
            if (partindices[rowHash % greatest_modulus] >= 0)
                result->bound_offsets =
                    bms_make_singleton(rowHash % greatest_modulus);
        }
        else
        {
            /* Getting here means at least one hash partition exists. */
            //程序执行到这里,意味着至少存在一个hash分区
            Assert(boundinfo->ndatums > 0);
            result->bound_offsets = bms_add_range(NULL, 0,
                                                  boundinfo->ndatums - 1);
        }
    
        /*
         * There is neither a special hash null partition or the default hash
         * partition.
         * 要么存在一个特别的hash null分区,要么是默认的hash分区
         */
        result->scan_null = result->scan_default = false;
    
        return result;
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

    
    
    (gdb) b get_matching_partitions
    Breakpoint 1 at 0x804d3b: file partprune.c, line 619.
    (gdb) c
    Continuing.
    
    Breakpoint 1, get_matching_partitions (context=0x7fff4a5e3930, pruning_steps=0x14d5300) at partprune.c:619
    619     int         num_steps = list_length(pruning_steps),
    (gdb) n
    626     if (num_steps == 0)
    

查看输入参数,pruning_steps是有3个ITEM的链表

    
    
    (gdb) p *pruning_steps
    $1 = {type = T_List, length = 3, head = 0x14d52d8, tail = 0x14d58d0}
    

pruning_steps的3个ITEM类型分别是PartitionPruneStepOp/PartitionPruneStepOp/PartitionPruneStepCombine  
第1和2个ITEM的expr是Const,即常量1和2

    
    
    (gdb) p *(Node *)pruning_steps->head->data.ptr_value
    $2 = {type = T_PartitionPruneStepOp} -->Node类型
    (gdb) p *(PartitionPruneStepOp *)pruning_steps->head->data.ptr_value
    $3 = {step = {type = T_PartitionPruneStepOp, step_id = 0}, opstrategy = 1, exprs = 0x14d52a0, cmpfns = 0x14d5240, 
      nullkeys = 0x0}
    (gdb) set $ppso=(PartitionPruneStepOp *)pruning_steps->head->data.ptr_value
    (gdb) p *$ppso->exprs
    $4 = {type = T_List, length = 1, head = 0x14d5278, tail = 0x14d5278}
    (gdb) p *(Node *)$ppso->exprs->head->data.ptr_value
    $5 = {type = T_Const}
    (gdb) p *(Const *)$ppso->exprs->head->data.ptr_value
    $6 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 1, 
      constisnull = false, constbyval = true, location = 42} -->第1个步骤的表达式,常量1
    (gdb) set $ppso=(PartitionPruneStepOp *)pruning_steps->head->next->data.ptr_value
    (gdb) p *(Const *)$ppso->exprs->head->data.ptr_value
    $7 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 2, 
      constisnull = false, constbyval = true, location = 52}  -->第2个步骤的表达式,常量2
    (gdb) p *(Node *)pruning_steps->head->next->next->data.ptr_value
    $8 = {type = T_PartitionPruneStepCombine}
    (gdb) set $ppsc=(PartitionPruneStepCombine *)pruning_steps->head->next->next->data.ptr_value
    (gdb) p *$ppsc
    $9 = {step = {type = T_PartitionPruneStepCombine, step_id = 2}, combineOp = PARTPRUNE_COMBINE_UNION, 
      source_stepids = 0x14d5480}  -->第3个步骤,组合操作是PARTPRUNE_COMBINE_UNION
    

为步骤结果分配内存

    
    
    (gdb) n
    640         palloc0(num_steps * sizeof(PruneStepResult *));
    (gdb) p num_steps
    $10 = 3
    (gdb) n
    639     results = (PruneStepResult **)
    (gdb) 
    641     foreach(lc, pruning_steps)
    

开始遍历pruning步骤,进入perform_pruning_base_step函数

    
    
    (gdb) 
    643         PartitionPruneStep *step = lfirst(lc);
    (gdb) 
    645         switch (nodeTag(step))
    (gdb) 
    648                 results[step->step_id] =
    (gdb) step
    649                     perform_pruning_base_step(context,
    (gdb) 
    perform_pruning_base_step (context=0x7fff4a5e3930, opstep=0x14d4e98) at partprune.c:2995
    2995        Assert(list_length(opstep->exprs) == list_length(opstep->cmpfns));
    

perform_pruning_base_step->查看输入参数,比较函数是OID=425的函数

    
    
    (gdb) p *opstep->cmpfns
    $12 = {type = T_OidList, length = 1, head = 0x14d5218, tail = 0x14d5218}
    (gdb) p opstep->cmpfns->head->data.oid_value
    $16 = 425
    (gdb) n
    2998        lc1 = list_head(opstep->exprs);
    (gdb) 
    2999        lc2 = list_head(opstep->cmpfns);
    (gdb) 
    

perform_pruning_base_step->遍历分区键

    
    
    (gdb) 
    3005        for (keyno = 0; keyno < context->partnatts; keyno++)
    (gdb) 
    3012            if (bms_is_member(keyno, opstep->nullkeys)) -->没有null键
    (gdb) 
    3019            if (keyno > nvalues && context->strategy == PARTITION_STRATEGY_RANGE) -->nvalues为0
    (gdb) p nvalues
    $17 = 0
    (gdb) n
    3022            if (lc1 != NULL)
    (gdb) 
    3028                expr = lfirst(lc1); -->获取步骤中的表达式,其实是常量1
    (gdb) 
    3029                stateidx = PruneCxtStateIdx(context->partnatts, -->stateidx为0
    (gdb) p *expr
    $18 = {type = T_Const}
    (gdb) p *(Const *)expr
    $19 = {xpr = {type = T_Const}, consttype = 23, consttypmod = -1, constcollid = 0, constlen = 4, constvalue = 1, 
      constisnull = false, constbyval = true, location = 42}
    (gdb) n
    3031                if (partkey_datum_from_expr(context, expr, stateidx,
    (gdb) p stateidx
    $20 = 0
    (gdb) n
    3041                    if (isnull) -->非NULL
    (gdb) p isnull
    $21 = false
    

perform_pruning_base_step->获取比较函数进行处理

    
    
    (gdb) n
    3054                    cmpfn = lfirst_oid(lc2); --> OID=425
    (gdb) 
    3055                    Assert(OidIsValid(cmpfn));
    (gdb) 
    3056                    if (cmpfn != context->stepcmpfuncs[stateidx].fn_oid) -->fn_oid为0
    (gdb) 
    3064                        if (cmpfn == context->partsupfunc[keyno].fn_oid) -->fn_oid为425
    (gdb) p context->stepcmpfuncs[stateidx].fn_oid
    $22 = 0
    (gdb) p context->partsupfunc[keyno].fn_oid
    $23 = 425
    (gdb) n
    3065                            fmgr_info_copy(&context->stepcmpfuncs[stateidx],
    (gdb) 
    3066                                           &context->partsupfunc[keyno],
    (gdb) 
    3065                            fmgr_info_copy(&context->stepcmpfuncs[stateidx],
    (gdb) 
    3066                                           &context->partsupfunc[keyno],
    (gdb) 
    3065                            fmgr_info_copy(&context->stepcmpfuncs[stateidx], --> 拷贝函数
    (gdb) 
    3073                    values[keyno] = datum;-->设置值
    (gdb) p datum
    $24 = 1
    (gdb) n
    3074                    nvalues++;
    (gdb) 
    3077                lc1 = lnext(lc1);
    (gdb) 
    3078                lc2 = lnext(lc2);
    (gdb) 
    

perform_pruning_base_step->完成分区键遍历

    
    
    (gdb) n
    3005        for (keyno = 0; keyno < context->partnatts; keyno++)
    (gdb) 
    3086        stateidx = PruneCxtStateIdx(context->partnatts, opstep->step.step_id, 0);
    (gdb) 
    3087        partsupfunc = &context->stepcmpfuncs[stateidx];
    (gdb) 
    3089        switch (context->strategy)
    (gdb) 
    3092                return get_matching_hash_bounds(context,
    (gdb) 
    

perform_pruning_base_step->进入get_matching_hash_bounds函数

    
    
    (gdb) 
    3092                return get_matching_hash_bounds(context,
    (gdb) step
    3093                                                opstep->opstrategy,
    (gdb) 
    3092                return get_matching_hash_bounds(context,
    (gdb) 
    get_matching_hash_bounds (context=0x7fff4a5e3930, opstrategy=1, values=0x7fff4a5e3750, nvalues=1, partsupfunc=0x14d3068, 
        nullkeys=0x0) at partprune.c:2156
    2156        PruneStepResult *result = (PruneStepResult *) palloc0(sizeof(PruneStepResult));
    (gdb) 
    

get_matching_hash_bounds->变量赋值

    
    
    (gdb) n
    2157        PartitionBoundInfo boundinfo = context->boundinfo;
    (gdb) 
    2158        int        *partindices = boundinfo->indexes;
    (gdb) 
    2159        int         partnatts = context->partnatts;
    (gdb) 
    2165        Assert(context->strategy == PARTITION_STRATEGY_HASH);
    (gdb) 
    2172        if (nvalues + bms_num_members(nullkeys) == partnatts)
    (gdb) 
    

get_matching_hash_bounds->分区边界信息,共有6个分区,Index分别是0-5

    
    
    (gdb) p boundinfo
    $25 = (PartitionBoundInfo) 0x14d32a0
    (gdb) p *boundinfo
    $26 = {strategy = 104 'h', ndatums = 6, datums = 0x14d32f8, kind = 0x0, indexes = 0x14d2e00, null_index = -1, 
      default_index = -1}
    (gdb) p *boundinfo->datums
    $27 = (Datum *) 0x14d3350
    (gdb) p **boundinfo->datums
    $28 = 6
    (gdb) p **boundinfo->indexes
    Cannot access memory at address 0x0
    (gdb) p *boundinfo->indexes
    $29 = 0
    (gdb) p boundinfo->indexes[0]
    $30 = 0
    (gdb) p boundinfo->indexes[1]
    $31 = 1
    (gdb) p boundinfo->indexes[5]
    $32 = 5
    (gdb) 
    

get_matching_hash_bounds->分区索引和分区键数

    
    
    (gdb) p *partindices
    $33 = 0
    (gdb) p partnatts
    $34 = 1
    (gdb) 
    (gdb) p nvalues
    $35 = 1
    (gdb) p bms_num_members(nullkeys)
    $36 = 0
    

get_matching_hash_bounds->遍历分区键,判断值是否落在分区中

    
    
    (gdb) n
    2178            Assert(opstrategy == HTEqualStrategyNumber || nvalues == 0);
    (gdb) 
    2180            for (i = 0; i < partnatts; i++)
    (gdb) 
    2181                isnull[i] = bms_is_member(i, nullkeys);
    (gdb) 
    2180            for (i = 0; i < partnatts; i++)
    (gdb) 
    2183            greatest_modulus = get_hash_partition_greatest_modulus(boundinfo);
    (gdb) 
    2184            rowHash = compute_partition_hash_value(partnatts, partsupfunc,
    (gdb) 
    2187            if (partindices[rowHash % greatest_modulus] >= 0)
    
    

get_matching_hash_bounds->

    
    
    (gdb) p values
    $43 = (Datum *) 0x7fff4a5e3750
    (gdb) p *values
    $44 = 1 --> 约束条件
    (gdb) p isnull[0]
    $38 = false -->不为NULL
    (gdb) p greatest_modulus
    $39 = 6 -->6个分区
    (gdb) p rowHash
    $40 = 11274504255086170040 -->values算出的Hash值
    (gdb) p rowHash % greatest_modulus
    $41 = 2 --> 所在的分区
    (gdb) p partindices[2]
    $42 = 2 -->存在该分区
    (gdb) 
    

get_matching_hash_bounds->返回结果(bound_offsets->words=4,即编号2)

    
    
    (gdb) n
    2189                    bms_make_singleton(rowHash % greatest_modulus);
    (gdb) 
    2188                result->bound_offsets =
    (gdb) 
    2203        result->scan_null = result->scan_default = false;
    (gdb) 
    2205        return result;
    (gdb) 
    2206    }
    (gdb) p *result
    $45 = {bound_offsets = 0x14d59b8, scan_default = false, scan_null = false}
    (gdb) p *result->bound_offsets
    $46 = {nwords = 1, words = 0x14d59bc}
    (gdb) p *result->bound_offsets->words
    $47 = 4
    (gdb) 
    

回到get_matching_partitions

    
    
    (gdb) n
    perform_pruning_base_step (context=0x7fff4a5e3930, opstep=0x14d4e98) at partprune.c:3119
    3119    }
    (gdb) 
    get_matching_partitions (context=0x7fff4a5e3930, pruning_steps=0x14d5300) at partprune.c:648
    648                 results[step->step_id] =
    (gdb) 
    651                 break;
    (gdb) 
    641     foreach(lc, pruning_steps)
    (gdb) 
    643         PartitionPruneStep *step = lfirst(lc);
    (gdb) 
    645         switch (nodeTag(step))
    (gdb) p 
    

继续执行步骤,进入perform_pruning_combine_step

    
    
    654                 results[step->step_id] =
    (gdb) 
    655                     perform_pruning_combine_step(context,
    (gdb) step
    perform_pruning_combine_step (context=0x7fff4a5e3930, cstep=0x14d5898, step_results=0x14d5958) at partprune.c:3136
    3136        PruneStepResult *result = NULL;
    (gdb) 
    

perform_pruning_combine_step->进入PARTPRUNE_COMBINE_UNION处理逻辑

    
    
    (gdb) n
    3143        result = (PruneStepResult *) palloc0(sizeof(PruneStepResult));
    (gdb) 
    3144        if (list_length(cstep->source_stepids) == 0)
    (gdb) 
    3154        switch (cstep->combineOp)
    (gdb) 
    3157                foreach(lc1, cstep->source_stepids)
    (gdb) 
    (gdb) p *cstep
    $49 = {step = {type = T_PartitionPruneStepCombine, step_id = 2}, combineOp = PARTPRUNE_COMBINE_UNION, 
      source_stepids = 0x14d5480}
    

perform_pruning_combine_step->遍历组合步骤的源步骤 cstep->source_stepids,合并这些步骤的结果

    
    
    (gdb) n
    3159                    int         step_id = lfirst_int(lc1);
    ...
    (gdb) 
    3174                    result->bound_offsets = bms_add_members(result->bound_offsets,
    (gdb) 
    3178                    if (!result->scan_null)
    (gdb) 
    3179                        result->scan_null = step_result->scan_null;
    (gdb) 
    3180                    if (!result->scan_default)
    (gdb) 
    3181                        result->scan_default = step_result->scan_default;
    (gdb) 
    3157                foreach(lc1, cstep->source_stepids)
    (gdb) 
    3183                break;
    (gdb) 
    3223        return result;
    (gdb) 
    

perform_pruning_combine_step->最终结果

    
    
    (gdb) p *result
    $54 = {bound_offsets = 0x14d5a48, scan_default = false, scan_null = false}
    (gdb) p *result->bound_offsets
    $55 = {nwords = 1, words = 0x14d5a4c}
    (gdb) p *result->bound_offsets->words
    $56 = 5
    

perform_pruning_combine_step->回到get_matching_partitions

    
    
    (gdb) n
    3224    }
    (gdb) 
    get_matching_partitions (context=0x7fff4a5e3930, pruning_steps=0x14d5300) at partprune.c:654
    654                 results[step->step_id] =
    

完成所有步骤的处理

    
    
    (gdb) n
    658                 break;
    (gdb) 
    641     foreach(lc, pruning_steps)
    (gdb) n
    671     final_result = results[num_steps - 1];
    (gdb) 
    672     Assert(final_result != NULL);
    (gdb) 
    

构造结果位图集

    
    
    ...
    675     while ((i = bms_next_member(final_result->bound_offsets, i)) >= 0)
    (gdb) n
    677         int         partindex = context->boundinfo->indexes[i];
    (gdb) 
    687         if (partindex >= 0)
    (gdb) 
    688             result = bms_add_member(result, partindex);
    (gdb) 
    

完成调用

    
    
    gdb) 
    675     while ((i = bms_next_member(final_result->bound_offsets, i)) >= 0)
    (gdb) 
    692     if (final_result->scan_null)
    (gdb) 
    698     if (final_result->scan_default)
    (gdb) 
    706     return result;
    (gdb) 
    707 }
    (gdb) 
    

DONE!

### 四、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

