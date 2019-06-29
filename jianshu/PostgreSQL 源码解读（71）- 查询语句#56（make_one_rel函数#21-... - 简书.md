本节大体介绍了动态规划算法实现(standard_join_search)中的join_search_one_level->make_join_rel->populate_joinrel_with_paths->add_paths_to_joinrel->hash_inner_and_outer中的initial_cost_hashjoin和final_cost_nestloop函数，这些函数用于计算hash
join的Cost。

### 一、数据结构

**Cost相关**  
注意:实际使用的参数值通过系统配置文件定义,而不是这里的常量定义!

    
    
     typedef double Cost; /* execution cost (in page-access units) */
    
     /* defaults for costsize.c's Cost parameters */
     /* NB: cost-estimation code should use the variables, not these constants! */
     /* 注意:实际值通过系统配置文件定义,而不是这里的常量定义! */
     /* If you change these, update backend/utils/misc/postgresql.sample.conf */
     #define DEFAULT_SEQ_PAGE_COST  1.0       //顺序扫描page的成本
     #define DEFAULT_RANDOM_PAGE_COST  4.0      //随机扫描page的成本
     #define DEFAULT_CPU_TUPLE_COST  0.01     //处理一个元组的CPU成本
     #define DEFAULT_CPU_INDEX_TUPLE_COST 0.005   //处理一个索引元组的CPU成本
     #define DEFAULT_CPU_OPERATOR_COST  0.0025    //执行一次操作或函数的CPU成本
     #define DEFAULT_PARALLEL_TUPLE_COST 0.1    //并行执行,从一个worker传输一个元组到另一个worker的成本
     #define DEFAULT_PARALLEL_SETUP_COST  1000.0  //构建并行执行环境的成本
     
     #define DEFAULT_EFFECTIVE_CACHE_SIZE  524288    /*先前已有介绍, measured in pages */
    
     double      seq_page_cost = DEFAULT_SEQ_PAGE_COST;
     double      random_page_cost = DEFAULT_RANDOM_PAGE_COST;
     double      cpu_tuple_cost = DEFAULT_CPU_TUPLE_COST;
     double      cpu_index_tuple_cost = DEFAULT_CPU_INDEX_TUPLE_COST;
     double      cpu_operator_cost = DEFAULT_CPU_OPERATOR_COST;
     double      parallel_tuple_cost = DEFAULT_PARALLEL_TUPLE_COST;
     double      parallel_setup_cost = DEFAULT_PARALLEL_SETUP_COST;
     
     int         effective_cache_size = DEFAULT_EFFECTIVE_CACHE_SIZE;
     Cost        disable_cost = 1.0e10;//1后面10个0,通过设置一个巨大的成本,让优化器自动放弃此路径
     
     int         max_parallel_workers_per_gather = 2;//每次gather使用的worker数
    

### 二、源码解读

hash join的算法实现伪代码如下:  
_Step 1_  
FOR small_table_row IN (SELECT * FROM small_table)  
LOOP  
slot := HASH(small_table_row.join_key);  
INSERT_HASH_TABLE(slot,small_table_row);  
END LOOP;

_Step 2_  
FOR large_table_row IN (SELECT * FROM large_table) LOOP  
slot := HASH(large_table_row.join_key);  
small_table_row = LOOKUP_HASH_TABLE(slot,large_table_row.join_key);  
IF small_table_row FOUND THEN  
output small_table_row + large_table_row;  
END IF;  
END LOOP;

initial_cost_hashjoin和final_cost_nestloop函数初步预估hash join访问路径的成本和计算最终hash
join的Cost。

**initial_cost_hashjoin**

    
    
    //----------------------------------------------------------------------------- initial_cost_hashjoin
    
    /*
     * initial_cost_hashjoin
     *    Preliminary estimate of the cost of a hashjoin path.
     *    初步预估hash join访问路径的成本
     *    注释请参照merge join和nestloop join
     * This must quickly produce lower-bound estimates of the path's startup and
     * total costs.  If we are unable to eliminate the proposed path from
     * consideration using the lower bounds, final_cost_hashjoin will be called
     * to obtain the final estimates.
     *
     * The exact division of labor between this function and final_cost_hashjoin
     * is private to them, and represents a tradeoff between speed of the initial
     * estimate and getting a tight lower bound.  We choose to not examine the
     * join quals here (other than by counting the number of hash clauses),
     * so we can't do much with CPU costs.  We do assume that
     * ExecChooseHashTableSize is cheap enough to use here.
     * 
     * 'workspace' is to be filled with startup_cost, total_cost, and perhaps
     *      other data to be used by final_cost_hashjoin
     * 'jointype' is the type of join to be performed
     * 'hashclauses' is the list of joinclauses to be used as hash clauses
     * 'outer_path' is the outer input to the join
     * 'inner_path' is the inner input to the join
     * 'extra' contains miscellaneous information about the join
     * 'parallel_hash' indicates that inner_path is partial and that a shared
     *      hash table will be built in parallel
     */
    void
    initial_cost_hashjoin(PlannerInfo *root, JoinCostWorkspace *workspace,
                          JoinType jointype,
                          List *hashclauses,
                          Path *outer_path, Path *inner_path,
                          JoinPathExtraData *extra,
                          bool parallel_hash)
    {
        Cost        startup_cost = 0;
        Cost        run_cost = 0;
        double      outer_path_rows = outer_path->rows;
        double      inner_path_rows = inner_path->rows;
        double      inner_path_rows_total = inner_path_rows;
        int         num_hashclauses = list_length(hashclauses);
        int         numbuckets;
        int         numbatches;
        int         num_skew_mcvs;
        size_t      space_allowed;  /* unused */
    
        /* cost of source data */
        startup_cost += outer_path->startup_cost;
        run_cost += outer_path->total_cost - outer_path->startup_cost;
        startup_cost += inner_path->total_cost;
    
        /*
         * Cost of computing hash function: must do it once per input tuple. We
         * charge one cpu_operator_cost for each column's hash function.  Also,
         * tack on one cpu_tuple_cost per inner row, to model the costs of
         * inserting the row into the hashtable.
         * 计算哈希函数的成本:每个输入元组必须执行一次。
         * 我们对每个列的哈希函数的计算成本设置为cpu_operator_cost。
         * 另外，在每个内表行的处理添加以cpu_tuple_cost为单位的成本，以模拟将行插入到散列表中的成本。
         *
         * XXX when a hashclause is more complex than a single operator, we really
         * should charge the extra eval costs of the left or right side, as
         * appropriate, here.  This seems more work than it's worth at the moment.
         * 当一个hash条件子句比单个操作符更复杂时，确实应该在这里计算连接左边或右边的额外的表达式的解析(eval)成本。
         * 这看起来会比现在的工作要多。
         */
        startup_cost += (cpu_operator_cost * num_hashclauses + cpu_tuple_cost)
            * inner_path_rows;
        run_cost += cpu_operator_cost * num_hashclauses * outer_path_rows;
    
        /*
         * If this is a parallel hash build, then the value we have for
         * inner_rows_total currently refers only to the rows returned by each
         * participant.  For shared hash table size estimation, we need the total
         * number, so we need to undo the division.
         * 如果这是一个并行散列构建，那么当前inner_rows_total的值仅引用每个参与者返回的行。
         * 对于共享哈希表大小估计，需要总数，因此需要修正。
         */
        if (parallel_hash)
            inner_path_rows_total *= get_parallel_divisor(inner_path);
    
        /*
         * Get hash table size that executor would use for inner relation.
         * 执行器根据内表信息,获取hash表的大小.
         * 
         * XXX for the moment, always assume that skew optimization will be
         * performed.  As long as SKEW_WORK_MEM_PERCENT is small, it's not worth
         * trying to determine that for sure.
         * 目前，总是假设会执行倾斜优化(skew optimization)。
         * 但是只要SKEW_WORK_MEM_PERCENT很小，就不需要考虑。
         *
         * XXX at some point it might be interesting to try to account for skew
         * optimization in the cost estimate, but for now, we don't.
         * 在某种程度上，尝试在成本估算中考虑歪斜优化可能是很有趣的，但现在，不使用这样的做法。
         */
        ExecChooseHashTableSize(inner_path_rows_total,
                                inner_path->pathtarget->width,
                                true,   /* useskew */
                                parallel_hash,  /* try_combined_work_mem */
                                outer_path->parallel_workers,
                                &space_allowed,
                                &numbuckets,
                                &numbatches,
                                &num_skew_mcvs);
    
        /*
         * If inner relation is too big then we will need to "batch" the join,
         * which implies writing and reading most of the tuples to disk an extra
         * time.  Charge seq_page_cost per page, since the I/O should be nice and
         * sequential.  Writing the inner rel counts as startup cost, all the rest
         * as run cost.
         * 如果内表太大，那么将需要“批处理”连接，这意味着要花额外的时间将大多数元组写入和读取到磁盘。
         * 因为是连续顺序I/O,所以每一页page的成本为eq_page_cost。
         * 写内表的成本统计作为启动成本，其余的都作为运行成本。
         */
        if (numbatches > 1)
        {
            double      outerpages = page_size(outer_path_rows,
                                               outer_path->pathtarget->width);
            double      innerpages = page_size(inner_path_rows,
                                               inner_path->pathtarget->width);
    
            startup_cost += seq_page_cost * innerpages;
            run_cost += seq_page_cost * (innerpages + 2 * outerpages);
        }
    
        /* 后续再计算CPU成本.CPU costs left for later */
    
        /* Public result fields */
        workspace->startup_cost = startup_cost;
        workspace->total_cost = startup_cost + run_cost;
        /* Save private data for final_cost_hashjoin */
        workspace->run_cost = run_cost;
        workspace->numbuckets = numbuckets;
        workspace->numbatches = numbatches;
        workspace->inner_rows_total = inner_path_rows_total;
    }
    
    //----------------------------------------------------------- ExecChooseHashTableSize
     /*
     * Compute appropriate size for hashtable given the estimated size of the
     * relation to be hashed (number of rows and average row width).
     * 给定需要散列的关系的估计大小(行数和平均行宽度),计算散列表的合适大小。
     * 
     * This is exported so that the planner's costsize.c can use it.
     * 该结果会导出到其他地方以便优化器的costsize.c可以使用此值.
     */
    
    /* Target bucket loading (tuples per bucket) */
    #define NTUP_PER_BUCKET         1
    
    void
    ExecChooseHashTableSize(double ntuples, int tupwidth, bool useskew,
                            bool try_combined_work_mem,
                            int parallel_workers,
                            size_t *space_allowed,
                            int *numbuckets,
                            int *numbatches,
                            int *num_skew_mcvs)
    {
        int         tupsize;
        double      inner_rel_bytes;
        long        bucket_bytes;
        long        hash_table_bytes;
        long        skew_table_bytes;
        long        max_pointers;
        long        mppow2;
        int         nbatch = 1;
        int         nbuckets;
        double      dbuckets;
    
        /* Force a plausible relation size if no info */
         //如元组数目没有统计,则默认值为1000
        if (ntuples <= 0.0)
            ntuples = 1000.0;
    
        /*
         * Estimate tupsize based on footprint of tuple in hashtable... note this
         * does not allow for any palloc overhead.  The manipulations of spaceUsed
         * don't count palloc overhead either.
         * 基于散列表中元组的足迹(footprint)估计元组大小.
         * 注意，这不允许任何palloc开销。空间使用的操作也不计入palloc的开销。
         */
        tupsize = HJTUPLE_OVERHEAD +
            MAXALIGN(SizeofMinimalTupleHeader) +
            MAXALIGN(tupwidth);
        inner_rel_bytes = ntuples * tupsize;//元组数 * 元组大小
    
        /*
         * Target in-memory hashtable size is work_mem kilobytes.
         * 目标hash表大小为work_mem
         */
        hash_table_bytes = work_mem * 1024L;
    
        /*
         * Parallel Hash tries to use the combined work_mem of all workers to
         * avoid the need to batch.  If that won't work, it falls back to work_mem
         * per worker and tries to process batches in parallel.
         * 并行散列尝试使用所有worker的work_mem总数来避免批处理。
         * 如果无法如此操作，将使用每个worker一个work_mem的方式，并尝试并行处理批处理。
         */
        if (try_combined_work_mem)
            hash_table_bytes += hash_table_bytes * parallel_workers;
    
        *space_allowed = hash_table_bytes;
    
        /*
         * If skew optimization is possible, estimate the number of skew buckets
         * that will fit in the memory allowed, and decrement the assumed space
         * available for the main hash table accordingly.
         * 如果倾斜优化是可能的，那么估计能够在内存中容纳的倾斜桶数，并相应地减小主哈希表的假设可用空间。
         *
         * We make the optimistic assumption that each skew bucket will contain
         * one inner-relation tuple.  If that turns out to be low, we will recover
         * at runtime by reducing the number of skew buckets.
         * 我们乐观地假设每一个斜桶包含一个内表元组。
         * 如果结果不容乐观，那么将通过减少倾斜桶的数量在运行时恢复。
         *
         * hashtable->skewBucket will have up to 8 times as many HashSkewBucket
         * pointers as the number of MCVs we allow, since ExecHashBuildSkewHash
         * will round up to the next power of 2 and then multiply by 4 to reduce
         * collisions.
         * hashtable->skewBucket的HashSkewBucket指针数量将是我们允许的MCVs数量的8倍，
         * 因为ExecHashBuildSkewHash将舍入到下一个2的乘方，然后乘以4以减少冲突。
         */
        if (useskew)//使用倾斜优化(数据列值非均匀分布)
        {
            skew_table_bytes = hash_table_bytes * SKEW_WORK_MEM_PERCENT / 100;
    
            /*----------             * Divisor is:
             * size of a hash tuple +
             * worst-case size of skewBucket[] per MCV +
             * size of skewBucketNums[] entry +
             * size of skew bucket struct itself
             *----------             */
            *num_skew_mcvs = skew_table_bytes / (tupsize +
                                                 (8 * sizeof(HashSkewBucket *)) +
                                                 sizeof(int) +
                                                 SKEW_BUCKET_OVERHEAD);
            if (*num_skew_mcvs > 0)
                hash_table_bytes -= skew_table_bytes;
        }
        else
            *num_skew_mcvs = 0;
    
        /*
         * Set nbuckets to achieve an average bucket load of NTUP_PER_BUCKET when
         * memory is filled, assuming a single batch; but limit the value so that
         * the pointer arrays we'll try to allocate do not exceed work_mem nor
         * MaxAllocSize.
         * 设置nbucket，使其在内存填充时达到平均的桶负载NTUP_PER_BUCKET，
         * 假设是一个批处理;但是需要限制这个值，试图分配的指针数组不超过work_mem或MaxAllocSize。
         * 
         * Note that both nbuckets and nbatch must be powers of 2 to make
         * ExecHashGetBucketAndBatch fast.
         * 注意,不管是nbuckets还是nbatch必须是2的倍数,以使ExecHashGetBucketAndBatch函数更快
         */
        max_pointers = *space_allowed / sizeof(HashJoinTuple);
        max_pointers = Min(max_pointers, MaxAllocSize / sizeof(HashJoinTuple));
        /* If max_pointers isn't a power of 2, must round it down to one */
        //设置为2的倍数
        mppow2 = 1L << my_log2(max_pointers);
        if (max_pointers != mppow2)
            max_pointers = mppow2 / 2;
    
        /* Also ensure we avoid integer overflow in nbatch and nbuckets */
        //确保没有整数溢出
        /* (this step is redundant given the current value of MaxAllocSize) */
        max_pointers = Min(max_pointers, INT_MAX / 2);
    
        dbuckets = ceil(ntuples / NTUP_PER_BUCKET);
        dbuckets = Min(dbuckets, max_pointers);
        nbuckets = (int) dbuckets;
        /* don't let nbuckets be really small, though ... */
        //nbuckets最大为1024
        nbuckets = Max(nbuckets, 1024);
        /* ... and force it to be a power of 2. */
        //左移一位,即结果*2
        nbuckets = 1 << my_log2(nbuckets);
    
        /*
         * If there's not enough space to store the projected number of tuples and
         * the required bucket headers, we will need multiple batches.
         * 如果没有足够的空间来存储元组的投影数量和所需的桶header，将需要多个批处理。
         */
        bucket_bytes = sizeof(HashJoinTuple) * nbuckets;
        if (inner_rel_bytes + bucket_bytes > hash_table_bytes)
        {
            //内表大小 + 桶大小 > hash表大小,需要拆分为多个批次
            /* We'll need multiple batches */
            long        lbuckets;
            double      dbatch;
            int         minbatch;
            long        bucket_size;
    
            /*
             * If Parallel Hash with combined work_mem would still need multiple
             * batches, we'll have to fall back to regular work_mem budget.
             */
            if (try_combined_work_mem)
            {
                ExecChooseHashTableSize(ntuples, tupwidth, useskew,
                                        false, parallel_workers,
                                        space_allowed,
                                        numbuckets,
                                        numbatches,
                                        num_skew_mcvs);
                return;
            }
    
            /*
             * Estimate the number of buckets we'll want to have when work_mem is
             * entirely full.  Each bucket will contain a bucket pointer plus
             * NTUP_PER_BUCKET tuples, whose projected size already includes
             * overhead for the hash code, pointer to the next tuple, etc.
             * 估计当work_mem完全满时，可以得到多少个桶。
             * 每个桶bucket将包含一个bucket指针加上NTUP_PER_BUCKET元组，
             * 它的投影大小已经包括哈希值的开销、指向下一个元组的指针等等。
             */
            bucket_size = (tupsize * NTUP_PER_BUCKET + sizeof(HashJoinTuple));
            lbuckets = 1L << my_log2(hash_table_bytes / bucket_size);
            lbuckets = Min(lbuckets, max_pointers);
            nbuckets = (int) lbuckets;
            nbuckets = 1 << my_log2(nbuckets);
            bucket_bytes = nbuckets * sizeof(HashJoinTuple);
    
            /*
             * Buckets are simple pointers to hashjoin tuples, while tupsize
             * includes the pointer, hash code, and MinimalTupleData.  So buckets
             * should never really exceed 25% of work_mem (even for
             * NTUP_PER_BUCKET=1); except maybe for work_mem values that are not
             * 2^N bytes, where we might get more because of doubling. So let's
             * look for 50% here.
             * 桶bucket是指向hash join元组的简单指针，而tupsize包括指针/hash值和MinimalTupleData。
             * 所以bucket不应该超过work_mem的25%(即使NTUP_PER_BUCKET=1);
             */
            Assert(bucket_bytes <= hash_table_bytes / 2);
    
            /* Calculate required number of batches. */
            //计算批次
            dbatch = ceil(inner_rel_bytes / (hash_table_bytes - bucket_bytes));
            dbatch = Min(dbatch, max_pointers);
            minbatch = (int) dbatch;
            nbatch = 2;
            while (nbatch < minbatch)
                nbatch <<= 1;
        }
    
        Assert(nbuckets > 0);
        Assert(nbatch > 0);
    
        *numbuckets = nbuckets;
        *numbatches = nbatch;
    }
     
    

**final_cost_hashjoin**

    
    
    //----------------------------------------------------------------------------- final_cost_hashjoin
    /*
     * final_cost_hashjoin
     *    Final estimate of the cost and result size of a hashjoin path.
     *    最后的成本估算和hashjoin访问路径的结果大小
     *
     * Note: the numbatches estimate is also saved into 'path' for use later
     * 注意:numbatches估算也被保存到path中以备以后使用
     * 
     * 'path' is already filled in except for the rows and cost fields and
     *      num_batches
     * 'workspace' is the result from initial_cost_hashjoin
     * 'extra' contains miscellaneous information about the join
     */
    void
    
    final_cost_hashjoin(PlannerInfo *root, HashPath *path,
                        JoinCostWorkspace *workspace,
                        JoinPathExtraData *extra)
    {
        Path       *outer_path = path->jpath.outerjoinpath;
        Path       *inner_path = path->jpath.innerjoinpath;
        double      outer_path_rows = outer_path->rows;
        double      inner_path_rows = inner_path->rows;
        double      inner_path_rows_total = workspace->inner_rows_total;
        List       *hashclauses = path->path_hashclauses;
        Cost        startup_cost = workspace->startup_cost;
        Cost        run_cost = workspace->run_cost;
        int         numbuckets = workspace->numbuckets;
        int         numbatches = workspace->numbatches;
        Cost        cpu_per_tuple;
        QualCost    hash_qual_cost;
        QualCost    qp_qual_cost;
        double      hashjointuples;
        double      virtualbuckets;
        Selectivity innerbucketsize;
        Selectivity innermcvfreq;
        ListCell   *hcl;
    
        /* Mark the path with the correct row estimate */
        if (path->jpath.path.param_info)
            path->jpath.path.rows = path->jpath.path.param_info->ppi_rows;
        else
            path->jpath.path.rows = path->jpath.path.parent->rows;
    
        /* For partial paths, scale row estimate. */
        if (path->jpath.path.parallel_workers > 0)
        {
            double      parallel_divisor = get_parallel_divisor(&path->jpath.path);
    
            path->jpath.path.rows =
                clamp_row_est(path->jpath.path.rows / parallel_divisor);
        }
    
        /*
         * We could include disable_cost in the preliminary estimate, but that
         * would amount to optimizing for the case where the join method is
         * disabled, which doesn't seem like the way to bet.
         */
        if (!enable_hashjoin)
            startup_cost += disable_cost;//禁用hash join
    
        /* mark the path with estimated # of batches */
        path->num_batches = numbatches;//处理批数
    
        /* store the total number of tuples (sum of partial row estimates) */
        //存储元组总数.
        path->inner_rows_total = inner_path_rows_total;
    
        /* and compute the number of "virtual" buckets in the whole join */
        //计算虚拟的桶数(buckets)
        virtualbuckets = (double) numbuckets * (double) numbatches;
    
        /*
         * Determine bucketsize fraction and MCV frequency for the inner relation.
         * We use the smallest bucketsize or MCV frequency estimated for any
         * individual hashclause; this is undoubtedly conservative.
         * 确定桶大小分数和MCV频率之间的内在联系。
         * 使用每个散列子句估计的最小桶(bucket)大小或MCV频率;但这无疑是保守的处理方法。
         * 
         * BUT: if inner relation has been unique-ified, we can assume it's good
         * for hashing.  This is important both because it's the right answer, and
         * because we avoid contaminating the cache with a value that's wrong for
         * non-unique-ified paths.
         * 但是:如果内表元组是唯一的，可以假设它对哈希处理有好处。
         * 这很重要，因为它是正确的答案，而且避免了脏缓存，而这个值对于非唯一化路径是错误的。
         */
        if (IsA(inner_path, UniquePath))
        {
            //内表唯一
            innerbucketsize = 1.0 / virtualbuckets;
            innermcvfreq = 0.0;
        }
        else//内表不唯一
        {
            innerbucketsize = 1.0;
            innermcvfreq = 1.0;
            foreach(hcl, hashclauses)//循环遍历hash条件
            {
                RestrictInfo *restrictinfo = lfirst_node(RestrictInfo, hcl);
                Selectivity thisbucketsize;
                Selectivity thismcvfreq;
    
                /*
                 * First we have to figure out which side of the hashjoin clause
                 * is the inner side.
                 * 首先，必须找出hashjoin子句的哪一边是内表边。
                 *
                 * Since we tend to visit the same clauses over and over when
                 * planning a large query, we cache the bucket stats estimates in
                 * the RestrictInfo node to avoid repeated lookups of statistics.
                 * 因为在计划大型查询时，往往会反复访问相同的子句，
                 * 所以将bucket stats估计缓存在限制条件信息节点中，以避免重复查找统计信息。
                 */
                if (bms_is_subset(restrictinfo->right_relids,
                                  inner_path->parent->relids))
                {
                    /* righthand side is inner */
                    thisbucketsize = restrictinfo->right_bucketsize;
                    if (thisbucketsize < 0)
                    {
                        /* not cached yet */
                        estimate_hash_bucket_stats(root,
                                                   get_rightop(restrictinfo->clause),
                                                   virtualbuckets,
                                                   &restrictinfo->right_mcvfreq,
                                                   &restrictinfo->right_bucketsize);
                        thisbucketsize = restrictinfo->right_bucketsize;
                    }
                    thismcvfreq = restrictinfo->right_mcvfreq;
                }
                else
                {
                    Assert(bms_is_subset(restrictinfo->left_relids,
                                         inner_path->parent->relids));
                    /* lefthand side is inner */
                    thisbucketsize = restrictinfo->left_bucketsize;
                    if (thisbucketsize < 0)
                    {
                        /* not cached yet */
                        estimate_hash_bucket_stats(root,
                                                   get_leftop(restrictinfo->clause),
                                                   virtualbuckets,
                                                   &restrictinfo->left_mcvfreq,
                                                   &restrictinfo->left_bucketsize);
                        thisbucketsize = restrictinfo->left_bucketsize;
                    }
                    thismcvfreq = restrictinfo->left_mcvfreq;
                }
    
                if (innerbucketsize > thisbucketsize)
                    innerbucketsize = thisbucketsize;
                if (innermcvfreq > thismcvfreq)
                    innermcvfreq = thismcvfreq;
            }
        }
    
        /*
         * If the bucket holding the inner MCV would exceed work_mem, we don't
         * want to hash unless there is really no other alternative, so apply
         * disable_cost.  (The executor normally copes with excessive memory usage
         * by splitting batches, but obviously it cannot separate equal values
         * that way, so it will be unable to drive the batch size below work_mem
         * when this is true.)
         * 如果包含内部MCV的bucket会超过work_mem，那么除非真的没有其他选择，否则我们不希望执行散列，因此禁用hash连接。
         * (执行程序通常通过分割批处理来处理过多的内存使用，但显然它不能以这种方式分离相等的值，
         * 因此如果这是真的，它将无法驱动work_mem以下的批处理大小。)
         */
        if (relation_byte_size(clamp_row_est(inner_path_rows * innermcvfreq),
                               inner_path->pathtarget->width) >
            (work_mem * 1024L))
            startup_cost += disable_cost;//MCV的值如果大于work_mem,则禁用hash连接
    
        /*
         * Compute cost of the hashquals and qpquals (other restriction clauses)
         * separately.
         * 分别计算hash quals连接条件和qpquals(其他限制条件)的成本。
         */
        cost_qual_eval(&hash_qual_cost, hashclauses, root);
        cost_qual_eval(&qp_qual_cost, path->jpath.joinrestrictinfo, root);
        qp_qual_cost.startup -= hash_qual_cost.startup;
        qp_qual_cost.per_tuple -= hash_qual_cost.per_tuple;
    
        /* CPU costs */
    
        if (path->jpath.jointype == JOIN_SEMI ||
            path->jpath.jointype == JOIN_ANTI ||
            extra->inner_unique)
        {
            double      outer_matched_rows;
            Selectivity inner_scan_frac;
    
            /*
             * With a SEMI or ANTI join, or if the innerrel is known unique, the
             * executor will stop after the first match.
             * 对于半连接或反连接，或者如果内表是唯一的，执行器将在第一次匹配后停止。
             *
             * For an outer-rel row that has at least one match, we can expect the
             * bucket scan to stop after a fraction 1/(match_count+1) of the
             * bucket's rows, if the matches are evenly distributed.  Since they
             * probably aren't quite evenly distributed, we apply a fuzz factor of
             * 2.0 to that fraction.  (If we used a larger fuzz factor, we'd have
             * to clamp inner_scan_frac to at most 1.0; but since match_count is
             * at least 1, no such clamp is needed now.)
             * 对于至少有一个匹配的外表outer-rel行，如果匹配是均匀分布的，
             * 那么桶扫描在桶行数*1/(match_count+1)之后就会停止。
             * 因为它们可能不是均匀分布的，所以对这个比例应用了2.0的模糊系数。
             * (如果我们使用更大的系数，将不得不将inner_scan_frac限制为1.0;
             * 但是因为match_count至少为1，所以现在不需要这样处理了。)
             */
            outer_matched_rows = rint(outer_path_rows * extra->semifactors.outer_match_frac);
            inner_scan_frac = 2.0 / (extra->semifactors.match_count + 1.0);
    
            startup_cost += hash_qual_cost.startup;
            run_cost += hash_qual_cost.per_tuple * outer_matched_rows *
                clamp_row_est(inner_path_rows * innerbucketsize * inner_scan_frac) * 0.5;
    
            /*
             * For unmatched outer-rel rows, the picture is quite a lot different.
             * In the first place, there is no reason to assume that these rows
             * preferentially hit heavily-populated buckets; instead assume they
             * are uncorrelated with the inner distribution and so they see an
             * average bucket size of inner_path_rows / virtualbuckets.  In the
             * second place, it seems likely that they will have few if any exact
             * hash-code matches and so very few of the tuples in the bucket will
             * actually require eval of the hash quals.  We don't have any good
             * way to estimate how many will, but for the moment assume that the
             * effective cost per bucket entry is one-tenth what it is for
             * matchable tuples.
             * 对于没有匹配的外表行来说，情况就大不相同了。
             * 首先，没有理由假设这些行优先命中了被大量填充的桶(bucket);
             * 相反，假设它们与内表分布不相关，因此它们看到的是inner_path_rows / virtualbuckets的平均桶大小。
             * 第二，如果有任何确切的哈希值匹配，它们可能会很少，因此桶中的元组实际上很少需要对散列quals条件进行eval解析。
             * 目前没有很好的方法来估计会有多少个，但是目前假设访问每个桶的实际耗费的成本是匹配元组的十分之一。
             */
            run_cost += hash_qual_cost.per_tuple *
                (outer_path_rows - outer_matched_rows) *
                clamp_row_est(inner_path_rows / virtualbuckets) * 0.05;
    
            /* Get # of tuples that will pass the basic join */
            //参与基础连接的元组数
            if (path->jpath.jointype == JOIN_ANTI)
                hashjointuples = outer_path_rows - outer_matched_rows;
            else
                hashjointuples = outer_matched_rows;
        }
        else
        {
            /*
             * The number of tuple comparisons needed is the number of outer
             * tuples times the typical number of tuples in a hash bucket, which
             * is the inner relation size times its bucketsize fraction.  At each
             * one, we need to evaluate the hashjoin quals.  But actually,
             * charging the full qual eval cost at each tuple is pessimistic,
             * since we don't evaluate the quals unless the hash values match
             * exactly.  For lack of a better idea, halve the cost estimate to
             * allow for that.
             * 需要的元组比较数量是外部元组的数量乘以散列桶中的典型元组数量，这是内部关系大小乘以它的桶大小比例。
             * 在每个函数中，都需要计算hash连接条件(hashjoin quals)。
             * 但实际上，认为在每个元组上耗费解析表达式(qual eval)的成本是较为悲观的，
             * 因为我们不计算条件quals，除非散列值完全匹配。
             * 由于缺乏更好的想法，将成本估算减半。
             */
            startup_cost += hash_qual_cost.startup;
            run_cost += hash_qual_cost.per_tuple * outer_path_rows *
                clamp_row_est(inner_path_rows * innerbucketsize) * 0.5;
    
            /*
             * Get approx # tuples passing the hashquals.  We use
             * approx_tuple_count here because we need an estimate done with
             * JOIN_INNER semantics.
             * 通过hash条件获得大约的元组数。
             * 这里使用approx_tuple_count，因为我们需要使用JOIN_INNER语义进行评估。
             */
            hashjointuples = approx_tuple_count(root, &path->jpath, hashclauses);
        }
    
        /*
         * For each tuple that gets through the hashjoin proper, we charge
         * cpu_tuple_cost plus the cost of evaluating additional restriction
         * clauses that are to be applied at the join.  (This is pessimistic since
         * not all of the quals may get evaluated at each tuple.)
         * 对于通过哈希连接的每个元组，每个元组的成本为cpu_tuple_cost，
         * 加上计算将在连接上应用的附加限制条款的成本。
         * (这是悲观的，因为并不是所有的条件表达式quals都能在每个元组上得到计算。)
         */
        startup_cost += qp_qual_cost.startup;
        cpu_per_tuple = cpu_tuple_cost + qp_qual_cost.per_tuple;
        run_cost += cpu_per_tuple * hashjointuples;
    
        /* tlist eval costs are paid per output row, not per tuple scanned */
        startup_cost += path->jpath.path.pathtarget->cost.startup;
        run_cost += path->jpath.path.pathtarget->cost.per_tuple * path->jpath.path.rows;
    
        path->jpath.path.startup_cost = startup_cost;
        path->jpath.path.total_cost = startup_cost + run_cost;
    }
    

### 三、跟踪分析

测试脚本如下

    
    
    select a.*,b.grbh,b.je 
    from t_dwxx a,
        lateral (select t1.dwbh,t1.grbh,t2.je 
         from t_grxx t1 
              inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    order by b.dwbh;
    
    

启动gdb,设置断点

    
    
    (gdb) b try_hashjoin_path
    Breakpoint 1 at 0x7af2a4: file joinpath.c, line 737.
    

进入函数try_hashjoin_path,连接类型为JOIN_INNER

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, try_hashjoin_path (root=0x1b73980, joinrel=0x1b91d30, outer_path=0x1b866c0, inner_path=0x1b8e780, 
        hashclauses=0x1b93200, jointype=JOIN_INNER, extra=0x7ffc09955e80) at joinpath.c:737
    737     required_outer = calc_non_nestloop_required_outer(outer_path,
    

进入initial_cost_hashjoin函数

    
    
    (gdb) n
    739     if (required_outer &&
    (gdb) 
    751     initial_cost_hashjoin(root, &workspace, jointype, hashclauses,
    (gdb) step
    initial_cost_hashjoin (root=0x1b73980, workspace=0x7ffc09955d00, jointype=JOIN_INNER, hashclauses=0x1b93200, 
        outer_path=0x1b866c0, inner_path=0x1b8e780, extra=0x7ffc09955e80, parallel_hash=false) at costsize.c:3160
    3160        Cost        startup_cost = 0;
    

初始赋值,hash连接条件只有1个,即t_dwxx.dwbh = t_grxx.dwbh(连接信息参照上一节)

    
    
    3160        Cost        startup_cost = 0;
    (gdb) n
    3161        Cost        run_cost = 0;
    (gdb) 
    3162        double      outer_path_rows = outer_path->rows;
    (gdb) 
    3163        double      inner_path_rows = inner_path->rows;
    (gdb) 
    3164        double      inner_path_rows_total = inner_path_rows;
    (gdb) 
    3165        int         num_hashclauses = list_length(hashclauses);
    (gdb) 
    3172        startup_cost += outer_path->startup_cost;
    (gdb) p num_hashclauses
    $1 = 1
    

进入ExecChooseHashTableSize函数,该函数根据给定需要散列的关系的估计大小(行数和平均行宽度),计算散列表的合适大小。

    
    
    ...
    (gdb) step
    ExecChooseHashTableSize (ntuples=100000, tupwidth=9, useskew=true, try_combined_work_mem=false, parallel_workers=0, 
        space_allowed=0x7ffc09955c38, numbuckets=0x7ffc09955c4c, numbatches=0x7ffc09955c48, num_skew_mcvs=0x7ffc09955c44)
        at nodeHash.c:677
    

获取元组大小->48Bytes,内部大小4800000Bytes(4.58M)和hash表大小->4194304(4M)

    
    
    (gdb) n
    690     tupsize = HJTUPLE_OVERHEAD +
    (gdb) 
    693     inner_rel_bytes = ntuples * tupsize;
    (gdb) 
    698     hash_table_bytes = work_mem * 1024L;
    (gdb) 
    705     if (try_combined_work_mem)
    (gdb) p tupsize
    $2 = 48
    (gdb) p inner_rel_bytes
    $3 = 4800000
    (gdb) p hash_table_bytes
    $4 = 4194304
    

使用行倾斜技术

    
    
    (gdb) n
    708     *space_allowed = hash_table_bytes;
    (gdb) 
    724     if (useskew)
    (gdb) 
    726         skew_table_bytes = hash_table_bytes * SKEW_WORK_MEM_PERCENT / 100;
    (gdb) p useskew
    $5 = true
    

行倾斜桶数为635,调整后的hash表大小4110418,最大的指针数524288

    
    
    ...
    (gdb) p *num_skew_mcvs
    $7 = 635
    (gdb) p hash_table_bytes
    $8 = 4110418
    ...
    (gdb) n
    756     max_pointers = Min(max_pointers, MaxAllocSize / sizeof(HashJoinTuple));
    (gdb) 
    758     mppow2 = 1L << my_log2(max_pointers);
    (gdb) 
    759     if (max_pointers != mppow2)
    (gdb) 
    764     max_pointers = Min(max_pointers, INT_MAX / 2);
    (gdb) 
    766     dbuckets = ceil(ntuples / NTUP_PER_BUCKET);
    (gdb) p max_pointers
    $10 = 524288
    

计算桶数=131072

    
    
    ...
    (gdb) n
    767     dbuckets = Min(dbuckets, max_pointers);
    (gdb) 
    768     nbuckets = (int) dbuckets;
    (gdb) 
    770     nbuckets = Max(nbuckets, 1024);
    (gdb) 
    772     nbuckets = 1 << my_log2(nbuckets);
    (gdb) 
    778     bucket_bytes = sizeof(HashJoinTuple) * nbuckets;
    (gdb) 
    779     if (inner_rel_bytes + bucket_bytes > hash_table_bytes)
    (gdb) p nbuckets
    $11 = 131072
    

如果没有足够的空间，将需要多个批处理。

    
    
    779     if (inner_rel_bytes + bucket_bytes > hash_table_bytes)
    (gdb) n
    791         if (try_combined_work_mem)
    

计算桶数->131072和批次->2等信息

    
    
    ...
    (gdb) 
    837     *numbuckets = nbuckets;
    (gdb) 
    838     *numbatches = nbatch;
    (gdb) p nbuckets
    $12 = 131072
    (gdb) p nbatch
    $13 = 2
    

回到initial_cost_hashjoin

    
    
    (gdb) 
    initial_cost_hashjoin (root=0x1b73980, workspace=0x7ffc09955d00, jointype=JOIN_INNER, hashclauses=0x1b93200, 
        outer_path=0x1b866c0, inner_path=0x1b8e780, extra=0x7ffc09955e80, parallel_hash=false) at costsize.c:3226
    3226        if (numbatches > 1)
    

initial_cost_hashjoin函数执行完毕,查看计算结果

    
    
    (gdb) 
    3246        workspace->inner_rows_total = inner_path_rows_total;
    (gdb) 
    3247    }
    (gdb) p workspace
    $14 = (JoinCostWorkspace *) 0x7ffc09955d00
    (gdb) p *workspace
    $15 = {startup_cost = 3465, total_cost = 4261, run_cost = 796, inner_run_cost = 0, 
      inner_rescan_run_cost = 6.9525149533036983e-310, outer_rows = 3.7882102964330281e-317, 
      inner_rows = 1.4285501039407471e-316, outer_skip_rows = 1.4285501039407471e-316, 
      inner_skip_rows = 6.9525149532894692e-310, numbuckets = 131072, numbatches = 2, inner_rows_total = 100000}
    (gdb) 
    

设置断点,进入final_cost_hashjoin

    
    
    (gdb) b final_cost_hashjoin
    Breakpoint 2 at 0x7a0b49: file costsize.c, line 3265.
    (gdb) c
    Continuing.
    
    Breakpoint 2, final_cost_hashjoin (root=0x1b73980, path=0x1b93238, workspace=0x7ffc09955d00, extra=0x7ffc09955e80)
        at costsize.c:3265
    3265        Path       *outer_path = path->jpath.outerjoinpath;
    

赋值,并计算虚拟桶数

    
    
    3265        Path       *outer_path = path->jpath.outerjoinpath;
    (gdb) n
    3266        Path       *inner_path = path->jpath.innerjoinpath;
    ...
    (gdb) 
    3314        virtualbuckets = (double) numbuckets * (double) numbatches;
    (gdb) 
    3326        if (IsA(inner_path, UniquePath))
    (gdb) p virtualbuckets
    $16 = 262144
    (gdb) 
    

开始遍历hash连接条件,确定桶大小分数和MCV频率之间的内在联系,更新连接条件中的相关信息。

    
    
    3335            foreach(hcl, hashclauses)
    (gdb) 
    3337                RestrictInfo *restrictinfo = lfirst_node(RestrictInfo, hcl);
    

内表在右端RTE=3,即t_grxx),更新相关信息

    
    
    3349                if (bms_is_subset(restrictinfo->right_relids,
    (gdb) 
    3353                    thisbucketsize = restrictinfo->right_bucketsize;
    (gdb) 
    3354                    if (thisbucketsize < 0)
    (gdb) 
    3357                        estimate_hash_bucket_stats(root,
    (gdb) 
    3358                                                   get_rightop(restrictinfo->clause),
    (gdb) 
    3357                        estimate_hash_bucket_stats(root,
    (gdb) 
    3362                        thisbucketsize = restrictinfo->right_bucketsize;
    (gdb) p *restrictinfo
    $17 = {type = T_RestrictInfo, clause = 0x1b87260, is_pushed_down = true, outerjoin_delayed = false, can_join = true, 
      pseudoconstant = false, leakproof = false, security_level = 0, clause_relids = 0x1b87380, required_relids = 0x1b86c68, 
      outer_relids = 0x0, nullable_relids = 0x0, left_relids = 0x1b87340, right_relids = 0x1b87360, orclause = 0x0, 
      parent_ec = 0x1b85b88, eval_cost = {startup = 0, per_tuple = 0.0025000000000000001}, norm_selec = 0.0001, 
      outer_selec = -1, mergeopfamilies = 0x1b873c8, left_ec = 0x1b85b88, right_ec = 0x1b85b88, left_em = 0x1b85d58, 
      right_em = 0x1b85c80, scansel_cache = 0x1b92e50, outer_is_left = true, hashjoinoperator = 98, left_bucketsize = -1, 
      right_bucketsize = 0.00010013016921998598, left_mcvfreq = -1, right_mcvfreq = 0}
    

计算过程与先前介绍的类似,计算表达式的成本等等,具体可自行debug.  
最终的计算结果

    
    
    3505    }
    (gdb) p *path
    $19 = {jpath = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x1b91d30, pathtarget = 0x1b91f68, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 3465, total_cost = 5386, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
        outerjoinpath = 0x1b866c0, innerjoinpath = 0x1b8e780, joinrestrictinfo = 0x1b92208}, path_hashclauses = 0x1b93200, 
      num_batches = 2, inner_rows_total = 100000}
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

