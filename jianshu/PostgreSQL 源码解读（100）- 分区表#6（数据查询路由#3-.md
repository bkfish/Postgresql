本节介绍了PG在查询分区表的时候如何确定查询的是哪个分区。在规划阶段,函数set_rel_size中，如RTE为分区表（rte->inh=T），则调用set_append_rel_size函数，在set_append_rel_size中通过prune_append_rel_partitions函数获取“仍存活”的分区，下面介绍了prune_append_rel_partitions函数的主逻辑和依赖的函数gen_partprune_steps。

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
    
    
    

### 二、源码解读

prune_append_rel_partitions函数返回必须扫描以满足rel约束条件baserestrictinfo
quals的最小子分区集的RT索引。

    
    
     
    /*
     * prune_append_rel_partitions
     *      Returns RT indexes of the minimum set of child partitions which must
     *      be scanned to satisfy rel's baserestrictinfo quals.
     *      返回必须扫描以满足rel约束条件baserestrictinfo quals的最小子分区集的RT索引。
     *
     * Callers must ensure that 'rel' is a partitioned table.
     * 调用者必须确保rel是分区表
     */
    Relids
    prune_append_rel_partitions(RelOptInfo *rel)
    {
        Relids      result;
        List       *clauses = rel->baserestrictinfo;
        List       *pruning_steps;
        bool        contradictory;
        PartitionPruneContext context;
        Bitmapset  *partindexes;
        int         i;
    
        Assert(clauses != NIL);
        Assert(rel->part_scheme != NULL);
    
        /* If there are no partitions, return the empty set */
        //如无分区,则返回NULL
        if (rel->nparts == 0)
            return NULL;
    
        /*
         * Process clauses.  If the clauses are found to be contradictory, we can
         * return the empty set.
         * 处理条件子句.
         * 如果发现约束条件相互矛盾，返回NULL。
         */
        pruning_steps = gen_partprune_steps(rel, clauses, &contradictory);
        if (contradictory)
            return NULL;
    
        /* Set up PartitionPruneContext */
        //配置PartitionPruneContext上下文
        context.strategy = rel->part_scheme->strategy;
        context.partnatts = rel->part_scheme->partnatts;
        context.nparts = rel->nparts;
        context.boundinfo = rel->boundinfo;
        context.partcollation = rel->part_scheme->partcollation;
        context.partsupfunc = rel->part_scheme->partsupfunc;
        context.stepcmpfuncs = (FmgrInfo *) palloc0(sizeof(FmgrInfo) *
                                                    context.partnatts *
                                                    list_length(pruning_steps));
        context.ppccontext = CurrentMemoryContext;
    
        /* These are not valid when being called from the planner */
        //如从规划器调用,这些状态变量为NULL
        context.planstate = NULL;
        context.exprstates = NULL;
        context.exprhasexecparam = NULL;
        context.evalexecparams = false;
    
        /* Actual pruning happens here. */
        //这是实现逻辑
        partindexes = get_matching_partitions(&context, pruning_steps);
    
        /* Add selected partitions' RT indexes to result. */
        //把选中的分区RT索引放到结果中
        i = -1;
        result = NULL;
        while ((i = bms_next_member(partindexes, i)) >= 0)
            result = bms_add_member(result, rel->part_rels[i]->relid);
    
        return result;
    }
    
    
    
    /*
     * gen_partprune_steps
     *      Process 'clauses' (a rel's baserestrictinfo list of clauses) and return
     *      a list of "partition pruning steps"
     *      处理“子句”(rel的baserestrictinfo子句链表)并返回“分区pruning步骤”链表
     * 
     * If the clauses in the input list are contradictory or there is a
     * pseudo-constant "false", *contradictory is set to true upon return.
     * 如果输入链表中的条件子句是互斥矛盾的，或者有一个伪常量“false”，返回时将*contradictory设置为true。
     */
    static List *
    gen_partprune_steps(RelOptInfo *rel, List *clauses, bool *contradictory)
    {
        GeneratePruningStepsContext context;
    
        context.next_step_id = 0;
        context.steps = NIL;
    
        /* The clauses list may be modified below, so better make a copy. */
        //为确保安全,拷贝一份副本
        clauses = list_copy(clauses);
    
        /*
         * For sub-partitioned tables there's a corner case where if the
         * sub-partitioned table shares any partition keys with its parent, then
         * it's possible that the partitioning hierarchy allows the parent
         * partition to only contain a narrower range of values than the
         * sub-partitioned table does.  In this case it is possible that we'd
         * include partitions that could not possibly have any tuples matching
         * 'clauses'.  The possibility of such a partition arrangement is perhaps
         * unlikely for non-default partitions, but it may be more likely in the
         * case of default partitions, so we'll add the parent partition table's
         * partition qual to the clause list in this case only.  This may result
         * in the default partition being eliminated.
         * 对于子分区表，如果子分区表与其父分区表共享分区键，
         *     那么分区层次结构可能只允许父分区包含比分区表更窄的值范围。
         * 在这种情况下，可能会包含不可能有任何匹配“子句”的元组的分区。
         * 对于非默认分区，这种分区设计的可能性不大，但是对于默认分区，这种可能性更大，
         *     所以只在这种情况下将父分区表的分区条件qual添加到子句链表中。
         * 这可能会导致默认分区被消除。
         */
        if (partition_bound_has_default(rel->boundinfo) &&
            rel->partition_qual != NIL)
        {
            List       *partqual = rel->partition_qual;//分区条件链表
    
            partqual = (List *) expression_planner((Expr *) partqual);
    
            /* Fix Vars to have the desired varno */
            //修正Vars,以使其具备合适的编号varno
            if (rel->relid != 1)
                ChangeVarNodes((Node *) partqual, 1, rel->relid, 0);
    
            clauses = list_concat(clauses, partqual);//添加到条件链表中
        }
    
        /* Down into the rabbit-hole. */
        //进入到"兔子洞"中(实际生成步骤)
        gen_partprune_steps_internal(&context, rel, clauses, contradictory);
    
        return context.steps;
    }
     
    
    
    /*
     * gen_partprune_steps_internal
     *      Processes 'clauses' to generate partition pruning steps.
     * 处理条件“子句”以生成分区修剪(pruning)步骤。 
     *
     * From OpExpr clauses that are mutually AND'd, we find combinations of those
     * that match to the partition key columns and for every such combination,
     * we emit a PartitionPruneStepOp containing a vector of expressions whose
     * values are used as a look up key to search partitions by comparing the
     * values with partition bounds.  Relevant details of the operator and a
     * vector of (possibly cross-type) comparison functions is also included with
     * each step.
     * From中的OpExpr条件是AND,
     *   我们发现那些匹配的组合分区键列,对于每一个这样的组合,
     *   构造PartitionPruneStepOp结构体,
     *   以包含一个向量表达式的值用作查找关键搜索分区比较值和分区范围。
     * 每个步骤还包括操作符的相关细节和(可能是交叉类型的)比较函数的向量。
     *
     * For BoolExpr clauses, we recursively generate steps for each argument, and
     * return a PartitionPruneStepCombine of their results.
     * 对于BoolExpr子句，我们递归地为每个参数生成步骤，
     *   并组合他们的结果返回PartitionPruneStepCombine。
     *
     * The return value is a list of the steps generated, which are also added to
     * the context's steps list.  Each step is assigned a step identifier, unique
     * even across recursive calls.
     * 返回值是生成的步骤链表，这些步骤也被添加到上下文的步骤链表中。
     * 每个步骤都被分配一个步骤标识符，即使在递归调用中也是惟一的。
     *
     * If we find clauses that are mutually contradictory, or a pseudoconstant
     * clause that contains false, we set *contradictory to true and return NIL
     * (that is, no pruning steps).  Caller should consider all partitions as
     * pruned in that case.  Otherwise, *contradictory is set to false.
     * 如果我们发现相互矛盾的子句，或包含false的伪常量子句，
     *   我们将*contradictory设置为true并返回NIL(即没有修剪步骤).
     * 在这种情况下，调用方应将所有分区视为pruned的。
     * 否则，*contradictory设置为false。
     *
     * Note: the 'clauses' List may be modified inside this function. Callers may
     * like to make a copy of it before passing them to this function.
     * 注:“子句”链表可以在此函数中修改。
     * 调用者可能希望在将其传递给此函数之前复制它。
     */
    static List *
    gen_partprune_steps_internal(GeneratePruningStepsContext *context,
                                 RelOptInfo *rel, List *clauses,
                                 bool *contradictory)
    {
        PartitionScheme part_scheme = rel->part_scheme;
        List       *keyclauses[PARTITION_MAX_KEYS];
        Bitmapset  *nullkeys = NULL,
                   *notnullkeys = NULL;
        bool        generate_opsteps = false;
        List       *result = NIL;
        ListCell   *lc;
    
        *contradictory = false;
    
        memset(keyclauses, 0, sizeof(keyclauses));
        foreach(lc, clauses)
        {
            Expr       *clause = (Expr *) lfirst(lc);
            int         i;
    
            /* Look through RestrictInfo, if any */
            //RestrictInfo类型
            if (IsA(clause, RestrictInfo))
                clause = ((RestrictInfo *) clause)->clause;
    
            /* Constant-false-or-null is contradictory */
            //False或者是NULL,设置为互斥,返回NIL
            if (IsA(clause, Const) &&
                (((Const *) clause)->constisnull ||
                 !DatumGetBool(((Const *) clause)->constvalue)))
            {
                *contradictory = true;
                return NIL;
            }
    
            /* Get the BoolExpr's out of the way. */
            //Bool表达式
            if (IsA(clause, BoolExpr))
            {
                /*
                 * Generate steps for arguments.
                 * 生成步骤
                 *
                 * While steps generated for the arguments themselves will be
                 * added to context->steps during recursion and will be evaluated
                 * independently, collect their step IDs to be stored in the
                 * combine step we'll be creating.
                 * 在递归过程中，为参数本身生成的步骤将被添加到context->steps中，
                 *   并将独立地进行解析，同时收集它们的步骤id，以便存储在我们即将创建的组合步骤中。
                 */
                if (or_clause((Node *) clause))
                {
                    //OR
                    List       *arg_stepids = NIL;
                    bool        all_args_contradictory = true;
                    ListCell   *lc1;
    
                    /*
                     * Get pruning step for each arg.  If we get contradictory for
                     * all args, it means the OR expression is false as a whole.
                     * 对每个参数进行修剪(pruning)。
                     * 如果对条件中的所有arg都是互斥的，这意味着OR表达式作为一个整体是错误的。
                     */
                    foreach(lc1, ((BoolExpr *) clause)->args)//遍历条件参数
                    {
                        Expr       *arg = lfirst(lc1);
                        bool        arg_contradictory;
                        List       *argsteps;
    
                        argsteps =
                            gen_partprune_steps_internal(context, rel,
                                                         list_make1(arg),
                                                         &arg_contradictory);
                        if (!arg_contradictory)
                            all_args_contradictory = false;
    
                        if (argsteps != NIL)
                        {
                            PartitionPruneStep *step;
    
                            Assert(list_length(argsteps) == 1);
                            step = (PartitionPruneStep *) linitial(argsteps);
                            arg_stepids = lappend_int(arg_stepids, step->step_id);
                        }
                        else
                        {
                            /*
                             * No steps either means that arg_contradictory is
                             * true or the arg didn't contain a clause matching
                             * this partition key.
                             * 如无steps,则意味着要么arg_contradictory为T要么参数没有
                             *   包含匹配分区键的条件子句
                             *
                             * In case of the latter, we cannot prune using such
                             * an arg.  To indicate that to the pruning code, we
                             * must construct a dummy PartitionPruneStepCombine
                             * whose source_stepids is set to an empty List.
                             * However, if we can prove using constraint exclusion
                             * that the clause refutes the table's partition
                             * constraint (if it's sub-partitioned), we need not
                             * bother with that.  That is, we effectively ignore
                             * this OR arm.
                             * 如果是后者，我们不能使用这样的参数进行修剪prune。
                             * 为了向修剪prune代码表明这一点，我们必须构造一个虚构的PartitionPruneStepCombine，
                             *   它的source_stepids设置为一个空列表。
                             * 但是，如果我们可以使用约束排除证明条件子句与表的分区约束(如果它是子分区的)互斥，
                             *   那么就不需要为此费心了。也就是说，实际上忽略了这个OR。
                             * 
                             */
                            List       *partconstr = rel->partition_qual;
                            PartitionPruneStep *orstep;
    
                            /* Just ignore this argument. */
                            //忽略该参数
                            if (arg_contradictory)
                                continue;
    
                            if (partconstr)
                            {
                                partconstr = (List *)
                                    expression_planner((Expr *) partconstr);
                                if (rel->relid != 1)
                                    ChangeVarNodes((Node *) partconstr, 1,
                                                   rel->relid, 0);
                                if (predicate_refuted_by(partconstr,
                                                         list_make1(arg),
                                                         false))//没有匹配分区键
                                    continue;
                            }
                            //构造PARTPRUNE_COMBINE_UNION步骤
                            orstep = gen_prune_step_combine(context, NIL,
                                                            PARTPRUNE_COMBINE_UNION);
                            //ID
                            arg_stepids = lappend_int(arg_stepids, orstep->step_id);
                        }
                    }
                    //输出参数赋值
                    *contradictory = all_args_contradictory;
    
                    /* Check if any contradicting clauses were found */
                    //检查是否互斥,如是则返回NIL
                    if (*contradictory)
                        return NIL;
    
                    if (arg_stepids != NIL)
                    {
                        PartitionPruneStep *step;
                        //构造step
                        step = gen_prune_step_combine(context, arg_stepids,
                                                      PARTPRUNE_COMBINE_UNION);
                        result = lappend(result, step);
                    }
                    continue;
                }
                else if (and_clause((Node *) clause))
                {
                    //AND
                    List       *args = ((BoolExpr *) clause)->args;//参数链表
                    List       *argsteps,
                               *arg_stepids = NIL;
                    ListCell   *lc1;
    
                    /*
                     * args may itself contain clauses of arbitrary type, so just
                     * recurse and later combine the component partitions sets
                     * using a combine step.
                     * args本身可能包含任意类型的子句，
                     *   因此只需递归，然后使用组合步骤来对组件分区集合进行组合。
                     */
                    argsteps = gen_partprune_steps_internal(context, rel, args,
                                                            contradictory);
                    if (*contradictory)
                        return NIL;//互斥,返回NIL
    
                    foreach(lc1, argsteps)//遍历步骤
                    {
                        PartitionPruneStep *step = lfirst(lc1);
    
                        arg_stepids = lappend_int(arg_stepids, step->step_id);
                    }
    
                    if (arg_stepids != NIL)//组合步骤
                    {
                        PartitionPruneStep *step;
    
                        step = gen_prune_step_combine(context, arg_stepids,
                                                      PARTPRUNE_COMBINE_INTERSECT);
                        result = lappend(result, step);
                    }
                    continue;
                }
    
                /*
                 * Fall-through for a NOT clause, which if it's a Boolean clause,
                 * will be handled in match_clause_to_partition_key(). We
                 * currently don't perform any pruning for more complex NOT
                 * clauses.
                 * NOT子句的Fall-through(如果是Boolean子句)将在match_clause_to_partition_key()中处理。
                 * 目前不为更复杂的NOT子句执行任何修剪pruning。
                 */
            }
    
            /*
             * Must be a clause for which we can check if one of its args matches
             * the partition key.
             * 必须是一个条件子句，我们才可以检查它的一个参数是否与分区键匹配。
             */
            for (i = 0; i < part_scheme->partnatts; i++)
            {
                Expr       *partkey = linitial(rel->partexprs[i]);//分区键
                bool        clause_is_not_null = false;
                PartClauseInfo *pc = NULL;//分区条件信息
                List       *clause_steps = NIL;
                //尝试将给定的“条件子句”与指定的分区键匹配。
                switch (match_clause_to_partition_key(rel, context,
                                                      clause, partkey, i,
                                                      &clause_is_not_null,
                                                      &pc, &clause_steps))
                {
                    //存在匹配项,输出参数为条件
                    case PARTCLAUSE_MATCH_CLAUSE:
                        Assert(pc != NULL);
    
                        /*
                         * Since we only allow strict operators, check for any
                         * contradicting IS NULL.
                         * 因为我们只允许严格的操作符，所以检查任何互斥条件时都是NULL。
                         */
                        if (bms_is_member(i, nullkeys))
                        {
                            *contradictory = true;
                            return NIL;
                        }
                        generate_opsteps = true;
                        keyclauses[i] = lappend(keyclauses[i], pc);
                        break;
                    //存在匹配项，匹配的子句是“a is NULL”或“a is NOT NULL”子句    
                    case PARTCLAUSE_MATCH_NULLNESS:
                        if (!clause_is_not_null)
                        {
                            /* check for conflicting IS NOT NULL */
                            if (bms_is_member(i, notnullkeys))
                            {
                                *contradictory = true;
                                return NIL;
                            }
                            nullkeys = bms_add_member(nullkeys, i);
                        }
                        else
                        {
                            /* check for conflicting IS NULL */
                            if (bms_is_member(i, nullkeys))
                            {
                                *contradictory = true;
                                return NIL;
                            }
                            notnullkeys = bms_add_member(notnullkeys, i);
                        }
                        break;
                    //存在匹配项,输出参数是步骤    
                    case PARTCLAUSE_MATCH_STEPS:
                        Assert(clause_steps != NIL);
                        result = list_concat(result, clause_steps);
                        break;
    
                    case PARTCLAUSE_MATCH_CONTRADICT:
                        /* We've nothing more to do if a contradiction was found. */
                        *contradictory = true;
                        return NIL;
                    //不存在匹配项
                    case PARTCLAUSE_NOMATCH:
    
                        /*
                         * Clause didn't match this key, but it might match the
                         * next one.
                         * 子句与这个键不匹配，但它可能与下一个键匹配。
                         */
                        continue;
                    //该子句不能用于pruning
                    case PARTCLAUSE_UNSUPPORTED:
                        /* This clause cannot be used for pruning. */
                        break;
                }
    
                /* done; go check the next clause. */
                //完成一个子句的处理,继续下一个
                break;
            }
        }
    
        /*-----------         * Now generate some (more) pruning steps.  We have three strategies:
         * 现在生成一些(更多)修剪pruning步骤。有三个策略:
         *
         * 1) Generate pruning steps based on IS NULL clauses:
         *   a) For list partitioning, null partition keys can only be found in
         *      the designated null-accepting partition, so if there are IS NULL
         *      clauses containing partition keys we should generate a pruning
         *      step that gets rid of all partitions but that one.  We can
         *      disregard any OpExpr we may have found.
         *   b) For range partitioning, only the default partition can contain
         *      NULL values, so the same rationale applies.
         *   c) For hash partitioning, we only apply this strategy if we have
         *      IS NULL clauses for all the keys.  Strategy 2 below will take
         *      care of the case where some keys have OpExprs and others have
         *      IS NULL clauses.
         * 1) 基于IS NULL子句生成修剪步骤:
         *   a)对于列表分区，空分区键只能在指定的接受null的分区中找到，
         *     因此，如果存在包含分区键的null子句，我们应该生成一个删除步骤，
         *     删除除该分区之外的所有分区。
         *     可以忽略我们可能发现的任何OpExpr。
         *   b)对于范围分区，只有默认分区可以包含NULL值，所以应用相同的原理。
         *   c)对于哈希分区，只有在所有键都有IS NULL子句时才应用这种策略。
         *     下面的策略2将处理一些键具有OpExprs而另一些键具有IS NULL子句的情况。
         *   
         * 2) If not, generate steps based on OpExprs we have (if any).
         * 2) 如果没有，根据我们拥有的OpExprs生成步骤(如果有)。
         *
         * 3) If this doesn't work either, we may be able to generate steps to
         *    prune just the null-accepting partition (if one exists), if we have
         *    IS NOT NULL clauses for all partition keys.
         * 3) 如果这两种方法都不起作用，那么如果我们对所有分区键都有IS NOT NULL子句，
         *    我们就可以生成步骤来只删除接受NULL的分区(如果存在的话)。
         */
        if (!bms_is_empty(nullkeys) &&
            (part_scheme->strategy == PARTITION_STRATEGY_LIST ||
             part_scheme->strategy == PARTITION_STRATEGY_RANGE ||
             (part_scheme->strategy == PARTITION_STRATEGY_HASH &&
              bms_num_members(nullkeys) == part_scheme->partnatts)))
        {
            PartitionPruneStep *step;
    
            /* Strategy 1 */
            //策略1
            step = gen_prune_step_op(context, InvalidStrategy,
                                     false, NIL, NIL, nullkeys);
            result = lappend(result, step);
        }
        else if (generate_opsteps)
        {
            PartitionPruneStep *step;
    
            /* Strategy 2 */
            //策略2
            step = gen_prune_steps_from_opexps(part_scheme, context,
                                               keyclauses, nullkeys);
            if (step != NULL)
                result = lappend(result, step);
        }
        else if (bms_num_members(notnullkeys) == part_scheme->partnatts)
        {
            PartitionPruneStep *step;
    
            /* Strategy 3 */
            //策略3
            step = gen_prune_step_op(context, InvalidStrategy,
                                     false, NIL, NIL, NULL);
            result = lappend(result, step);
        }
    
        /*
         * Finally, results from all entries appearing in result should be
         * combined using an INTERSECT combine step, if more than one.
         * 最后，如果结果中出现的所有条目的结果多于一个，则应该使用相交组合步骤组合。
         */
        if (list_length(result) > 1)
        {
            List       *step_ids = NIL;
    
            foreach(lc, result)
            {
                PartitionPruneStep *step = lfirst(lc);
    
                step_ids = lappend_int(step_ids, step->step_id);
            }
    
            if (step_ids != NIL)
            {
                PartitionPruneStep *step;
    
                step = gen_prune_step_combine(context, step_ids,
                                              PARTPRUNE_COMBINE_INTERSECT);
                result = lappend(result, step);
            }
        }
    
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

    
    
    (gdb) b prune_append_rel_partitions 
    Breakpoint 1 at 0x804b07: file partprune.c, line 555.
    (gdb) c
    Continuing.
    
    Breakpoint 1, prune_append_rel_partitions (rel=0x20faba0) at partprune.c:555
    555     List       *clauses = rel->baserestrictinfo;
    

获取约束条件

    
    
    (gdb) n
    562     Assert(clauses != NIL);
    (gdb) 
    563     Assert(rel->part_scheme != NULL);
    (gdb) 
    566     if (rel->nparts == 0)
    (gdb) 
    573     pruning_steps = gen_partprune_steps(rel, clauses, &contradictory);
    

进入gen_partprune_steps

    
    
    (gdb) step
    gen_partprune_steps (rel=0x20faba0, clauses=0x21c4d20, contradictory=0x7ffe1953a8d7) at partprune.c:505
    505     context.next_step_id = 0;
    

gen_partprune_steps->判断是否有默认分区(无)

    
    
    (gdb) n
    506     context.steps = NIL;
    (gdb) n
    509     clauses = list_copy(clauses);
    (gdb) 
    524     if (partition_bound_has_default(rel->boundinfo) &&
    (gdb) 
    

gen_partprune_steps_internal->进入gen_partprune_steps_internal

    
    
    (gdb) step
    gen_partprune_steps_internal (context=0x7ffe1953a830, rel=0x20faba0, clauses=0x21c4e00, contradictory=0x7ffe1953a8d7)
        at partprune.c:741
    741     PartitionScheme part_scheme = rel->part_scheme;
    

gen_partprune_steps_internal->查看分区方案(PartitionScheme)

    
    
    (gdb) n
    743     Bitmapset  *nullkeys = NULL,
    (gdb) p *part_scheme
    $1 = {strategy = 104 'h', partnatts = 1, partopfamily = 0x21c3180, partopcintype = 0x21c31a0, partcollation = 0x21c31c0, 
      parttyplen = 0x21c31e0, parttypbyval = 0x21c3200, partsupfunc = 0x21c3220}
    (gdb) p *part_scheme->partopfamily
    $2 = 1977
    (gdb) p *part_scheme->partopcintype
    $3 = 23
    (gdb) p *part_scheme->partcollation
    $4 = 0
    (gdb) p *part_scheme->parttyplen
    $5 = 4
    (gdb) p *part_scheme->parttypbyval
    $6 = true
    (gdb) p *part_scheme->partsupfunc
    $7 = {fn_addr = 0x4c85e7 <hashint4extended>, fn_oid = 425, fn_nargs = 2, fn_strict = true, fn_retset = false, 
      fn_stats = 2 '\002', fn_extra = 0x0, fn_mcxt = 0x20f8db0, fn_expr = 0x0}  
    

gen_partprune_steps_internal->SQL查询结果  
opfamily->integer_ops,整型操作

    
    
    testdb=# select * from pg_opfamily where oid=1977;
     opfmethod |   opfname   | opfnamespace | opfowner 
    -----------+-------------+--------------+----------           405 | integer_ops |           11 |       10
    (1 row)
    
    

gen_partprune_steps_internal->初始化变量

    
    
    (gdb) n
    744                *notnullkeys = NULL;
    (gdb) 
    745     bool        generate_opsteps = false;
    (gdb) 
    746     List       *result = NIL;
    (gdb) 
    749     *contradictory = false;
    (gdb) 
    751     memset(keyclauses, 0, sizeof(keyclauses));
    (gdb) 
    

gen_partprune_steps_internal->循环处理条件子句

    
    
    752     foreach(lc, clauses)
    (gdb) n
    754         Expr       *clause = (Expr *) lfirst(lc);
    (gdb) p *clauses
    $8 = {type = T_List, length = 1, head = 0x21c4dd8, tail = 0x21c4dd8}
    (gdb) p *clause
    $9 = {type = T_RestrictInfo}
    (gdb) n
    759             clause = ((RestrictInfo *) clause)->clause;
    (gdb) 
    762         if (IsA(clause, Const) &&
    (gdb) p *clause
    $10 = {type = T_BoolExpr}
    

gen_partprune_steps_internal->布尔表达式,进入相应的处理逻辑

    
    
    (gdb) n
    771         if (IsA(clause, BoolExpr))
    (gdb) 
    781             if (or_clause((Node *) clause))
    (gdb) 
    

gen_partprune_steps_internal->OR子句,进入相应的实现逻辑

    
    
    (gdb) 
    783                 List       *arg_stepids = NIL;
    (gdb) 
    (gdb) 
    784                 bool        all_args_contradictory = true;
    (gdb) 
    791                 foreach(lc1, ((BoolExpr *) clause)->args)
    (gdb) 
    793                     Expr       *arg = lfirst(lc1);
    (gdb) 
    798                         gen_partprune_steps_internal(context, rel,
    (gdb) 
    

gen_partprune_steps_internal->OR子句的相关信息

    
    
    (gdb) p *((BoolExpr *) clause)->args
    $3 = {type = T_List, length = 2, head = 0x21bf138, tail = 0x21bf198}
    (gdb) p *(OpExpr *)arg
    $4 = {xpr = {type = T_OpExpr}, opno = 96, opfuncid = 65, opresulttyp
    

gen_partprune_steps_internal->递归调用gen_partprune_steps_internal,返回argsteps链表

    
    
    797                     argsteps =
    (gdb) n
    801                     if (!arg_contradictory)
    (gdb) 
    802                         all_args_contradictory = false;
    (gdb) 
    804                     if (argsteps != NIL)
    (gdb) 
    808                         Assert(list_length(argsteps) == 1);
    (gdb) p argsteps
    $6 = (List *) 0x21c29b0
    (gdb) p *argsteps
    $7 = {type = T_List, length = 1, head = 0x21c2988, tail = 0x21c2988}
    (gdb) p *(Node *)argsteps->head->data.ptr_value
    $8 = {type = T_PartitionPruneStepOp}
    (gdb) p *(PartitionPruneStepOp *)argsteps->head->data.ptr_value
    $9 = {step = {type = T_PartitionPruneStepOp, step_id = 0}, opstrategy = 1, exprs = 0x21c2830, cmpfns = 0x21c27d0, 
      nullkeys = 0x0}
    

gen_partprune_steps_internal->构造step,继续OR子句的下一个条件

    
    
    (gdb) n
    809                         step = (PartitionPruneStep *) linitial(argsteps);
    (gdb) 
    810                         arg_stepids = lappend_int(arg_stepids, step->step_id);
    (gdb) 
    (gdb) 
    791                 foreach(lc1, ((BoolExpr *) clause)->args)
    

gen_partprune_steps_internal->递归调用gen_partprune_steps_internal,进入递归调用gen_partprune_steps_internal函数

    
    
    (gdb) step
    gen_partprune_steps_internal (context=0x7ffe1953a830, rel=0x20fab08, clauses=0x21c2a70, contradictory=0x7ffe1953a60f)
        at partprune.c:741
    741     PartitionScheme part_scheme = rel->part_scheme;
    (gdb) 
    ...
    

递归调用gen_partprune_steps_internal->遍历条件

    
    
    752     foreach(lc, clauses)
    (gdb) 
    754         Expr       *clause = (Expr *) lfirst(lc);
    (gdb) 
    758         if (IsA(clause, RestrictInfo))
    (gdb) p *(Expr *)clause
    $14 = {type = T_OpExpr}
    (gdb) p *(OpExpr *)clause
    $15 = {xpr = {type = T_OpExpr}, opno = 96, opfuncid = 65, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 0, args = 0x21becf8, location = 50}
    (gdb) n
    762         if (IsA(clause, Const) &&
    (gdb) 
    771         if (IsA(clause, BoolExpr))
    (gdb) 
    918         for (i = 0; i < part_scheme->partnatts; i++)
    

递归调用gen_partprune_steps_internal->遍历分区方案

    
    
    (gdb) 
    920             Expr       *partkey = linitial(rel->partexprs[i]);
    (gdb) 
    921             bool        clause_is_not_null = false;
    (gdb) p *(Expr *)partkey
    $16 = {type = T_Var}
    (gdb) p *(Var *)partkey
    $17 = {xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 1, varoattno = 1, location = -1}
    

递归调用gen_partprune_steps_internal->尝试将给定的“条件子句”与指定的分区键匹配,match_clause_to_partition_key函数输出结果为PARTCLAUSE_MATCH_CLAUSE(存在匹配项,输出参数为条件)

    
    
    (gdb) n
    922             PartClauseInfo *pc = NULL;
    (gdb) 
    923             List       *clause_steps = NIL;
    (gdb) 
    925             switch (match_clause_to_partition_key(rel, context,
    (gdb) 
    931                     Assert(pc != NULL);
    (gdb) 
    937                     if (bms_is_member(i, nullkeys))
    942                     generate_opsteps = true;
    (gdb) 
    943                     keyclauses[i] = lappend(keyclauses[i], pc);
    (gdb) 
    944                     break;
    (gdb) p keyclauses[i]
    $18 = (List *) 0x21c2b08
    (gdb) p *keyclauses[i]
    $19 = {type = T_List, length = 1, head = 0x21c2ae0, tail = 0x21c2ae0}
    (gdb) p *(Node *)keyclauses[i]->head->data.ptr_value
    $20 = {type = T_Invalid}
    

递归调用gen_partprune_steps_internal->完成条件遍历,开始生产pruning步骤,使用第2种策略(根据拥有的OpExprs生成步骤)生成

    
    
    (gdb) n
    752     foreach(lc, clauses)
    (gdb) n
    1019        if (!bms_is_empty(nullkeys) &&
    (gdb) 
    1032        else if (generate_opsteps)
    (gdb) 
    1037            step = gen_prune_steps_from_opexps(part_scheme, context,
    (gdb) n
    1039            if (step != NULL)
    (gdb) p *step
    $21 = {type = T_PartitionPruneStepOp, step_id = 1}
    (gdb) n
    1040                result = lappend(result, step);
    (gdb) 
    1056        if (list_length(result) > 1)
    (gdb) p *result
    $22 = {type = T_List, length = 1, head = 0x21c2da0, tail = 0x21c2da0}
    (gdb) n
    1077        return result;
    (gdb) 
    1078    }
    (gdb)     
    

gen_partprune_steps_internal->递归调用返回,完成OR子句的处理

    
    
    (gdb) 
    801                     if (!arg_contradictory)
    (gdb) 
    802                         all_args_contradictory = false;
    (gdb) 
    804                     if (argsteps != NIL)
    (gdb) 
    808                         Assert(list_length(argsteps) == 1);
    (gdb) 
    809                         step = (PartitionPruneStep *) linitial(argsteps);
    (gdb) 
    810                         arg_stepids = lappend_int(arg_stepids, step->step_id);
    (gdb) 
    791                 foreach(lc1, ((BoolExpr *) clause)->args)
    (gdb) 
    (gdb) 
    855                 *contradictory = all_args_contradictory;
    (gdb) 
    858                 if (*contradictory)
    (gdb) p all_args_contradictory
    $23 = false
    (gdb) n
    861                 if (arg_stepids != NIL)
    (gdb) 
    865                     step = gen_prune_step_combine(context, arg_stepids,
    (gdb) 
    867                     result = lappend(result, step);
    (gdb) 
    869                 continue;
    (gdb) p *step
    $24 = {<text variable, no debug info>} 0x7f4522678be0 <__step>
    (gdb) p *result
    $25 = {type = T_List, length = 1, head = 0x21c2e88, tail = 0x21c2e88}
    

gen_partprune_steps_internal->完成所有条件子句的遍历,返回result

    
    
    (gdb) n
    752     foreach(lc, clauses)
    (gdb) 
    1019        if (!bms_is_empty(nullkeys) &&
    (gdb) 
    1032        else if (generate_opsteps)
    (gdb) 
    1042        else if (bms_num_members(notnullkeys) == part_scheme->partnatts)
    (gdb) 
    1056        if (list_length(result) > 1)
    (gdb) 
    1077        return result;
    (gdb) 
    1078    }
    (gdb) 
    

gen_partprune_steps->回到gen_partprune_steps,返回steps链表

    
    
    (gdb) 
    gen_partprune_steps (rel=0x20fab08, clauses=0x21c2390, contradictory=0x7ffe1953a8d7) at partprune.c:541
    541     return context.steps;
    (gdb) p *result
    $26 = 0 '\000'
    (gdb) p context.steps
    $27 = (List *) 0x21c2890
    (gdb) p *context.steps
    $28 = {type = T_List, length = 3, head = 0x21c2868, tail = 0x21c2e60}
    $29 = {type = T_PartitionPruneStepOp}
    (gdb) p *(PartitionPruneStepOp *)context.steps->head->data.ptr_value
    $30 = {step = {type = T_PartitionPruneStepOp, step_id = 0}, opstrategy = 1, exprs = 0x21c2830, cmpfns = 0x21c27d0, 
      nullkeys = 0x0}
    (gdb) p *(PartitionPruneStepOp *)context.steps->head->next->data.ptr_value
    $31 = {step = {type = T_PartitionPruneStepOp, step_id = 1}, opstrategy = 1, exprs = 0x21c2c28, cmpfns = 0x21c2bc8, 
      nullkeys = 0x0}
    (gdb) p *(PartitionPruneStepOp *)context.steps->head->next->next->data.ptr_value
    $32 = {step = {type = T_PartitionPruneStepCombine, step_id = 2}, opstrategy = 0, exprs = 0x21c2a10, cmpfns = 0x7e, 
      nullkeys = 0x10}
    (gdb) 
    

gen_partprune_steps->回到prune_append_rel_partitions

    
    
    (gdb) n
    542 }
    (gdb) 
    prune_append_rel_partitions (rel=0x20fab08) at partprune.c:574
    574     if (contradictory)
    (gdb) 
    

prune_append_rel_partitions->设置上下文环境

    
    
    (gdb) 
    578     context.strategy = rel->part_scheme->strategy;
    (gdb) 
    579     context.partnatts = rel->part_scheme->partnatts;
    ...
    

prune_append_rel_partitions->调用get_matching_partitions,获取匹配的分区编号(Indexes)  
结果为5,即数组下标为0和2的Rel(part_rels数组)

    
    
    597     partindexes = get_matching_partitions(&context, pruning_steps);
    (gdb) 
    600     i = -1;
    (gdb) p partindexes
    $33 = (Bitmapset *) 0x21c2ff8
    (gdb) p *partindexes
    $34 = {nwords = 1, words = 0x21c2ffc}
    (gdb) p *partindexes->words
    $35 = 5
    

prune_append_rel_partitions->生成Relids  
结果为40,即8+32,即3号和5号Rel

    
    
    (gdb) n
    601     result = NULL;
    (gdb) 
    602     while ((i = bms_next_member(partindexes, i)) >= 0)
    (gdb) 
    603         result = bms_add_member(result, rel->part_rels[i]->relid);
    (gdb) p i
    $39 = 0
    (gdb) n
    602     while ((i = bms_next_member(partindexes, i)) >= 0)
    (gdb) 
    603         result = bms_add_member(result, rel->part_rels[i]->relid);
    (gdb) p i
    $40 = 2
    (gdb) n
    602     while ((i = bms_next_member(partindexes, i)) >= 0)
    (gdb) 
    605     return result;
    (gdb) p result
    $41 = (Relids) 0x21c3018
    (gdb) p *result
    $42 = {nwords = 1, words = 0x21c301c}
    (gdb) p result->words[0]
    $43 = 40
    

prune_append_rel_partitions->完成调用

    
    
    606 }
    (gdb) 
    set_append_rel_size (root=0x2120378, rel=0x20fab08, rti=1, rte=0x20fa3d0) at allpaths.c:922
    922         did_pruning = true;
    (gdb) 
    

DONE!

### 四、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

