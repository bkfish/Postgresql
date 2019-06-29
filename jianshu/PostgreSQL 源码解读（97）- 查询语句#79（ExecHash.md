本节是ExecHashJoin函数介绍的第五部分，主要介绍了ExecHashJoin中依赖的其他函数的实现逻辑，这些函数在HJ_NEED_NEW_BATCH阶段中使用，主要的函数是ExecHashJoinNewBatch。

### 一、数据结构

**JoinState**  
Hash/NestLoop/Merge Join的基类

    
    
    /* ----------------     *   JoinState information
     *
     *      Superclass for state nodes of join plans.
     *      Hash/NestLoop/Merge Join的基类
     * ----------------     */
    typedef struct JoinState
    {
        PlanState   ps;//基类PlanState
        JoinType    jointype;//连接类型
        //在找到一个匹配inner tuple的时候,如需要跳转到下一个outer tuple,则该值为T
        bool        single_match;   /* True if we should skip to next outer tuple
                                     * after finding one inner match */
        //连接条件表达式(除了ps.qual)
        ExprState  *joinqual;       /* JOIN quals (in addition to ps.qual) */
    } JoinState;
    
    

**HashJoinState**  
Hash Join运行期状态结构体

    
    
    /* these structs are defined in executor/hashjoin.h: */
    typedef struct HashJoinTupleData *HashJoinTuple;
    typedef struct HashJoinTableData *HashJoinTable;
    
    typedef struct HashJoinState
    {
        JoinState   js;             /* 基类;its first field is NodeTag */
        ExprState  *hashclauses;//hash连接条件
        List       *hj_OuterHashKeys;   /* 外表条件链表;list of ExprState nodes */
        List       *hj_InnerHashKeys;   /* 内表连接条件;list of ExprState nodes */
        List       *hj_HashOperators;   /* 操作符OIDs链表;list of operator OIDs */
        HashJoinTable hj_HashTable;//Hash表
        uint32      hj_CurHashValue;//当前的Hash值
        int         hj_CurBucketNo;//当前的bucket编号
        int         hj_CurSkewBucketNo;//行倾斜bucket编号
        HashJoinTuple hj_CurTuple;//当前元组
        TupleTableSlot *hj_OuterTupleSlot;//outer relation slot
        TupleTableSlot *hj_HashTupleSlot;//Hash tuple slot
        TupleTableSlot *hj_NullOuterTupleSlot;//用于外连接的outer虚拟slot
        TupleTableSlot *hj_NullInnerTupleSlot;//用于外连接的inner虚拟slot
        TupleTableSlot *hj_FirstOuterTupleSlot;//
        int         hj_JoinState;//JoinState状态
        bool        hj_MatchedOuter;//是否匹配
        bool        hj_OuterNotEmpty;//outer relation是否为空
    } HashJoinState;
    
    

**HashJoinTable**  
Hash表数据结构

    
    
    typedef struct HashJoinTableData
    {
        int         nbuckets;       /* 内存中的hash桶数;# buckets in the in-memory hash table */
        int         log2_nbuckets;  /* 2的对数(nbuckets必须是2的幂);its log2 (nbuckets must be a power of 2) */
    
        int         nbuckets_original;  /* 首次hash时的桶数;# buckets when starting the first hash */
        int         nbuckets_optimal;   /* 优化后的桶数(每个批次);optimal # buckets (per batch) */
        int         log2_nbuckets_optimal;  /* 2的对数;log2(nbuckets_optimal) */
    
        /* buckets[i] is head of list of tuples in i'th in-memory bucket */
        //bucket [i]是内存中第i个桶中的元组链表的head item
        union
        {
            /* unshared array is per-batch storage, as are all the tuples */
            //未共享数组是按批处理存储的，所有元组均如此
            struct HashJoinTupleData **unshared;
            /* shared array is per-query DSA area, as are all the tuples */
            //共享数组是每个查询的DSA区域，所有元组均如此
            dsa_pointer_atomic *shared;
        }           buckets;
    
        bool        keepNulls;      /*如不匹配则存储NULL元组,该值为T;true to store unmatchable NULL tuples */
    
        bool        skewEnabled;    /*是否使用倾斜优化?;are we using skew optimization? */
        HashSkewBucket **skewBucket;    /* 倾斜的hash表桶数;hashtable of skew buckets */
        int         skewBucketLen;  /* skewBucket数组大小;size of skewBucket array (a power of 2!) */
        int         nSkewBuckets;   /* 活动的倾斜桶数;number of active skew buckets */
        int        *skewBucketNums; /* 活动倾斜桶数组索引;array indexes of active skew buckets */
    
        int         nbatch;         /* 批次数;number of batches */
        int         curbatch;       /* 当前批次,第一轮为0;current batch #; 0 during 1st pass */
    
        int         nbatch_original;    /* 在开始inner扫描时的批次;nbatch when we started inner scan */
        int         nbatch_outstart;    /* 在开始outer扫描时的批次;nbatch when we started outer scan */
    
        bool        growEnabled;    /* 关闭nbatch增加的标记;flag to shut off nbatch increases */
    
        double      totalTuples;    /* 从inner plan获得的元组数;# tuples obtained from inner plan */
        double      partialTuples;  /* 通过hashjoin获得的inner元组数;# tuples obtained from inner plan by me */
        double      skewTuples;     /* 倾斜元组数;# tuples inserted into skew tuples */
    
        /*
         * These arrays are allocated for the life of the hash join, but only if
         * nbatch > 1.  A file is opened only when we first write a tuple into it
         * (otherwise its pointer remains NULL).  Note that the zero'th array
         * elements never get used, since we will process rather than dump out any
         * tuples of batch zero.
         * 这些数组在散列连接的生命周期内分配，但仅当nbatch > 1时分配。
         * 只有当第一次将元组写入文件时，文件才会打开(否则它的指针将保持NULL)。
         * 注意，第0个数组元素永远不会被使用，因为批次0的元组永远不会转储.
         */
        BufFile   **innerBatchFile; /* 每个批次的inner虚拟临时文件缓存;buffered virtual temp file per batch */
        BufFile   **outerBatchFile; /* 每个批次的outer虚拟临时文件缓存;buffered virtual temp file per batch */
    
        /*
         * Info about the datatype-specific hash functions for the datatypes being
         * hashed. These are arrays of the same length as the number of hash join
         * clauses (hash keys).
         * 有关正在散列的数据类型的特定于数据类型的散列函数的信息。
         * 这些数组的长度与散列连接子句(散列键)的数量相同。
         */
        FmgrInfo   *outer_hashfunctions;    /* outer hash函数FmgrInfo结构体;lookup data for hash functions */
        FmgrInfo   *inner_hashfunctions;    /* inner hash函数FmgrInfo结构体;lookup data for hash functions */
        bool       *hashStrict;     /* 每个hash操作符是严格?is each hash join operator strict? */
    
        Size        spaceUsed;      /* 元组使用的当前内存空间大小;memory space currently used by tuples */
        Size        spaceAllowed;   /* 空间使用上限;upper limit for space used */
        Size        spacePeak;      /* 峰值的空间使用;peak space used */
        Size        spaceUsedSkew;  /* 倾斜哈希表的当前空间使用情况;skew hash table's current space usage */
        Size        spaceAllowedSkew;   /* 倾斜哈希表的使用上限;upper limit for skew hashtable */
    
        MemoryContext hashCxt;      /* 整个散列连接存储的上下文;context for whole-hash-join storage */
        MemoryContext batchCxt;     /* 该批次存储的上下文;context for this-batch-only storage */
    
        /* used for dense allocation of tuples (into linked chunks) */
        //用于密集分配元组(到链接块中)
        HashMemoryChunk chunks;     /* 整个批次使用一个链表;one list for the whole batch */
    
        /* Shared and private state for Parallel Hash. */
        //并行hash使用的共享和私有状态
        HashMemoryChunk current_chunk;  /* 后台进程的当前chunk;this backend's current chunk */
        dsa_area   *area;           /* 用于分配内存的DSA区域;DSA area to allocate memory from */
        ParallelHashJoinState *parallel_state;//并行执行状态
        ParallelHashJoinBatchAccessor *batches;//并行访问器
        dsa_pointer current_chunk_shared;//当前chunk的开始指针
    } HashJoinTableData;
    
    typedef struct HashJoinTableData *HashJoinTable;
    
    

**HashJoinTupleData**  
Hash连接元组数据

    
    
    /* ----------------------------------------------------------------     *              hash-join hash table structures
     *
     * Each active hashjoin has a HashJoinTable control block, which is
     * palloc'd in the executor's per-query context.  All other storage needed
     * for the hashjoin is kept in private memory contexts, two for each hashjoin.
     * This makes it easy and fast to release the storage when we don't need it
     * anymore.  (Exception: data associated with the temp files lives in the
     * per-query context too, since we always call buffile.c in that context.)
     * 每个活动的hashjoin都有一个可散列的控制块，它在执行程序的每个查询上下文中都是通过palloc分配的。
     * hashjoin所需的所有其他存储都保存在私有内存上下文中，每个hashjoin有两个。
     * 当不再需要它的时候，这使得释放它变得简单和快速。
     * (例外:与临时文件相关的数据也存在于每个查询上下文中，因为在这种情况下总是调用buffile.c。)
     *
     * The hashtable contexts are made children of the per-query context, ensuring
     * that they will be discarded at end of statement even if the join is
     * aborted early by an error.  (Likewise, any temporary files we make will
     * be cleaned up by the virtual file manager in event of an error.)
     * hashtable上下文是每个查询上下文的子上下文，确保在语句结束时丢弃它们，即使连接因错误而提前中止。
     *   (同样，如果出现错误，虚拟文件管理器将清理创建的任何临时文件。)
     *
     * Storage that should live through the entire join is allocated from the
     * "hashCxt", while storage that is only wanted for the current batch is
     * allocated in the "batchCxt".  By resetting the batchCxt at the end of
     * each batch, we free all the per-batch storage reliably and without tedium.
     * 通过整个连接的存储空间应从“hashCxt”分配，而只需要当前批处理的存储空间在“batchCxt”中分配。
     * 通过在每个批处理结束时重置batchCxt，可以可靠地释放每个批处理的所有存储，而不会感到单调乏味。
     * 
     * During first scan of inner relation, we get its tuples from executor.
     * If nbatch > 1 then tuples that don't belong in first batch get saved
     * into inner-batch temp files. The same statements apply for the
     * first scan of the outer relation, except we write tuples to outer-batch
     * temp files.  After finishing the first scan, we do the following for
     * each remaining batch:
     *  1. Read tuples from inner batch file, load into hash buckets.
     *  2. Read tuples from outer batch file, match to hash buckets and output.
     * 在内部关系的第一次扫描中，从执行者那里得到了它的元组。
     * 如果nbatch > 1，那么不属于第一批的元组将保存到批内临时文件中。
     * 相同的语句适用于外关系的第一次扫描，但是我们将元组写入外部批处理临时文件。
     * 完成第一次扫描后，我们对每批剩余的元组做如下处理: 
     * 1.从内部批处理文件读取元组，加载到散列桶中。
     * 2.从外部批处理文件读取元组，匹配哈希桶和输出。 
     *
     * It is possible to increase nbatch on the fly if the in-memory hash table
     * gets too big.  The hash-value-to-batch computation is arranged so that this
     * can only cause a tuple to go into a later batch than previously thought,
     * never into an earlier batch.  When we increase nbatch, we rescan the hash
     * table and dump out any tuples that are now of a later batch to the correct
     * inner batch file.  Subsequently, while reading either inner or outer batch
     * files, we might find tuples that no longer belong to the current batch;
     * if so, we just dump them out to the correct batch file.
     * 如果内存中的哈希表太大，可以动态增加nbatch。
     * 散列值到批处理的计算是这样安排的:
     *   这只会导致元组进入比以前认为的更晚的批处理，而不会进入更早的批处理。
     * 当增加nbatch时，重新扫描哈希表，并将现在属于后面批处理的任何元组转储到正确的内部批处理文件。
     * 随后，在读取内部或外部批处理文件时，可能会发现不再属于当前批处理的元组;
     *   如果是这样，只需将它们转储到正确的批处理文件即可。
     * ----------------------------------------------------------------     */
    
    /* these are in nodes/execnodes.h: */
    /* typedef struct HashJoinTupleData *HashJoinTuple; */
    /* typedef struct HashJoinTableData *HashJoinTable; */
    
    typedef struct HashJoinTupleData
    {
        /* link to next tuple in same bucket */
        //link同一个桶中的下一个元组
        union
        {
            struct HashJoinTupleData *unshared;
            dsa_pointer shared;
        }           next;
        uint32      hashvalue;      /* 元组的hash值;tuple's hash code */
        /* Tuple data, in MinimalTuple format, follows on a MAXALIGN boundary */
    }           HashJoinTupleData;
    
    #define HJTUPLE_OVERHEAD  MAXALIGN(sizeof(HashJoinTupleData))
    #define HJTUPLE_MINTUPLE(hjtup)  \
        ((MinimalTuple) ((char *) (hjtup) + HJTUPLE_OVERHEAD))
    
    

### 二、源码解读

**ExecHashJoinNewBatch**  
切换到新的hashjoin批次,如成功,则返回T;已完成,返回F

    
    
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_FILL_OUTER_TUPLE 阶段
    ----------------------------------------------------------------------------------------------------*/
    //参见ExecHashJoin
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_FILL_INNER_TUPLES 阶段
    ----------------------------------------------------------------------------------------------------*/
    //参见ExecHashJoin
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_NEED_NEW_BATCH 阶段
    ----------------------------------------------------------------------------------------------------*/
    /*
     * ExecHashJoinNewBatch
     *      switch to a new hashjoin batch
     *      切换到新的hashjoin批次
     *
     * Returns true if successful, false if there are no more batches.
     * 如成功,则返回T;已完成,返回F
     */
    static bool
    ExecHashJoinNewBatch(HashJoinState *hjstate)
    {
        HashJoinTable hashtable = hjstate->hj_HashTable;//Hash表
        int         nbatch;//批次数
        int         curbatch;//当前批次
        BufFile    *innerFile;//inner relation缓存文件
        TupleTableSlot *slot;//slot
        uint32      hashvalue;//hash值
    
        nbatch = hashtable->nbatch;
        curbatch = hashtable->curbatch;
    
        if (curbatch > 0)
        {
            /*
             * We no longer need the previous outer batch file; close it right
             * away to free disk space.
             * 不再需要以前的外批处理文件;关闭它以释放磁盘空间。
             */
            if (hashtable->outerBatchFile[curbatch])
                BufFileClose(hashtable->outerBatchFile[curbatch]);
            hashtable->outerBatchFile[curbatch] = NULL;
        }
        else                        /* curbatch ==0,刚刚完成了第一个批次;we just finished the first batch */
        {
            /*
             * Reset some of the skew optimization state variables, since we no
             * longer need to consider skew tuples after the first batch. The
             * memory context reset we are about to do will release the skew
             * hashtable itself.
             * 重置一些倾斜优化状态变量，因为在第一批之后我们不再需要考虑倾斜元组。
             * 我们将要进行的内存上下文重置将释放倾斜散链表本身。
             */
            hashtable->skewEnabled = false;
            hashtable->skewBucket = NULL;
            hashtable->skewBucketNums = NULL;
            hashtable->nSkewBuckets = 0;
            hashtable->spaceUsedSkew = 0;
        }
    
        /*
         * We can always skip over any batches that are completely empty on both
         * sides.  We can sometimes skip over batches that are empty on only one
         * side, but there are exceptions:
         * 可以跳过任何两边都是空的批次。有时我们可以跳过只在一侧为空的批处理，但也有例外:
         *
         * 1. In a left/full outer join, we have to process outer batches even if
         * the inner batch is empty.  Similarly, in a right/full outer join, we
         * have to process inner batches even if the outer batch is empty.
         * 1、在左/全外连接中，即使内部批是空的，我们也必须处理外批数据。
         *    类似地，在右/完整外部连接中，即使外批数据为空，也必须处理内批数据。
         *
         * 2. If we have increased nbatch since the initial estimate, we have to
         * scan inner batches since they might contain tuples that need to be
         * reassigned to later inner batches.
         * 2、如果在初始估算之后增加了nbatch，必须扫描内部批处理，
         *   因为它们可能包含需要重新分配到后面的内部批处理的元组。
         *
         * 3. Similarly, if we have increased nbatch since starting the outer
         * scan, we have to rescan outer batches in case they contain tuples that
         * need to be reassigned.
         * 3、类似地，如果在开始外部扫描之后增加了nbatch，必须重新扫描外部批处理，
         *   以防它们包含需要重新分配的元组。
         */
        curbatch++;
        while (curbatch < nbatch &&
               (hashtable->outerBatchFile[curbatch] == NULL ||
                hashtable->innerBatchFile[curbatch] == NULL))
        {
            if (hashtable->outerBatchFile[curbatch] &&
                HJ_FILL_OUTER(hjstate))
                break;              /* 符合规则1,需要处理;must process due to rule 1 */
            if (hashtable->innerBatchFile[curbatch] &&
                HJ_FILL_INNER(hjstate))
                break;              /* 符合规则1,需要处理;must process due to rule 1 */
            if (hashtable->innerBatchFile[curbatch] &&
                nbatch != hashtable->nbatch_original)
                break;              /* 符合规则2,需要处理;must process due to rule 2 */
            if (hashtable->outerBatchFile[curbatch] &&
                nbatch != hashtable->nbatch_outstart)
                break;              /* 符合规则3,需要处理;must process due to rule 3 */
            /* We can ignore this batch. */
            /* Release associated temp files right away. */
            //均不符合规则1-3,则可以忽略这个批次了
            //释放临时文件
            if (hashtable->innerBatchFile[curbatch])
                BufFileClose(hashtable->innerBatchFile[curbatch]);
            hashtable->innerBatchFile[curbatch] = NULL;
            if (hashtable->outerBatchFile[curbatch])
                BufFileClose(hashtable->outerBatchFile[curbatch]);
            hashtable->outerBatchFile[curbatch] = NULL;
            curbatch++;//下一个批次
        }
    
        if (curbatch >= nbatch)
            return false;           /* 已完成处理所有批次;no more batches */
    
        hashtable->curbatch = curbatch;//下一个批次
    
        /*
         * Reload the hash table with the new inner batch (which could be empty)
         * 使用新的内部批处理数据(有可能是空的)重新加载哈希表
         */
        ExecHashTableReset(hashtable);//重置Hash表
        //inner relation文件
        innerFile = hashtable->innerBatchFile[curbatch];
        //批次文件不为NULL
        if (innerFile != NULL)
        {
            if (BufFileSeek(innerFile, 0, 0L, SEEK_SET))//扫描innerFile,不成功,则报错
                ereport(ERROR,
                        (errcode_for_file_access(),
                         errmsg("could not rewind hash-join temporary file: %m")));
    
            while ((slot = ExecHashJoinGetSavedTuple(hjstate,
                                                     innerFile,
                                                     &hashvalue,
                                                     hjstate->hj_HashTupleSlot)))//
            {
                /*
                 * NOTE: some tuples may be sent to future batches.  Also, it is
                 * possible for hashtable->nbatch to be increased here!
                 * 注意:一些元组可能被发送到未来的批次中。
                 * 另外，这里也可以增加hashtable->nbatch !
                 */
                ExecHashTableInsert(hashtable, slot, hashvalue);
            }
    
            /*
             * after we build the hash table, the inner batch file is no longer
             * needed
             * 构建哈希表之后，内部批处理临时文件就不再需要了,关闭之
             */
            BufFileClose(innerFile);
            hashtable->innerBatchFile[curbatch] = NULL;
        }
    
        /*
         * Rewind outer batch file (if present), so that we can start reading it.
         */
        if (hashtable->outerBatchFile[curbatch] != NULL)
        {
            if (BufFileSeek(hashtable->outerBatchFile[curbatch], 0, 0L, SEEK_SET))
                ereport(ERROR,
                        (errcode_for_file_access(),
                         errmsg("could not rewind hash-join temporary file: %m")));
        }
    
        return true;
    }
    
    
    /*
     * ExecHashJoinGetSavedTuple
     *      read the next tuple from a batch file.  Return NULL if no more.
     *      从批次文件中读取下一个元组,如无则返回NULL
     *
     * On success, *hashvalue is set to the tuple's hash value, and the tuple
     * itself is stored in the given slot.
     * 如成功,*hashvalue参数设置为元组的Hash值,元组存储在给定的slot中
     */
    static TupleTableSlot *
    ExecHashJoinGetSavedTuple(HashJoinState *hjstate,
                              BufFile *file,
                              uint32 *hashvalue,
                              TupleTableSlot *tupleSlot)
    {
        uint32      header[2];
        size_t      nread;
        MinimalTuple tuple;
    
        /*
         * We check for interrupts here because this is typically taken as an
         * alternative code path to an ExecProcNode() call, which would include
         * such a check.
         * 在这里检查中断，因为这通常被作为ExecProcNode()调用的替代代码路径，通常包含这样的检查。
         */
        CHECK_FOR_INTERRUPTS();
    
        /*
         * Since both the hash value and the MinimalTuple length word are uint32,
         * we can read them both in one BufFileRead() call without any type
         * cheating.
         * 因为哈希值和最小长度字都是uint32，所以可以在一个BufFileRead()调用中读取它们，
         *   而不需要任何类型的cheating。
         */
        nread = BufFileRead(file, (void *) header, sizeof(header));//读取文件
        if (nread == 0)             /* end of file */
        {
            //已读取完毕,返回NULL
            ExecClearTuple(tupleSlot);
            return NULL;
        }
        if (nread != sizeof(header))//读取的大小不等于header的大小,报错
            ereport(ERROR,
                    (errcode_for_file_access(),
                     errmsg("could not read from hash-join temporary file: %m")));
        //hash值
        *hashvalue = header[0];
        //tuple,分配的内存大小为MinimalTuple结构体大小
        tuple = (MinimalTuple) palloc(header[1]);
        //元组大小
        tuple->t_len = header[1];
        //读取文件
        nread = BufFileRead(file,
                            (void *) ((char *) tuple + sizeof(uint32)),
                            header[1] - sizeof(uint32));
        //读取有误,报错
        if (nread != header[1] - sizeof(uint32))
            ereport(ERROR,
                    (errcode_for_file_access(),
                     errmsg("could not read from hash-join temporary file: %m")));
        //存储到slot中
        ExecForceStoreMinimalTuple(tuple, tupleSlot, true);
        return tupleSlot;//返回slot
    }
    
    
     /*
     * ExecHashTableInsert
     *      insert a tuple into the hash table depending on the hash value
     *      it may just go to a temp file for later batches
     *      根据哈希值向哈希表中插入一个tuple，它可能只是转到一个临时文件中以供以后的批处理
     *
     * Note: the passed TupleTableSlot may contain a regular, minimal, or virtual
     * tuple; the minimal case in particular is certain to happen while reloading
     * tuples from batch files.  We could save some cycles in the regular-tuple
     * case by not forcing the slot contents into minimal form; not clear if it's
     * worth the messiness required.
     * 注意:传递的TupleTableSlot可能包含一个常规、最小或虚拟元组;
     *   在从批处理文件中重新加载元组时，肯定会出现最小的情况。
     * 如为常规元组，可以通过不强制slot内容变成最小形式来节省一些处理时间;
     *   但不清楚这样的混乱是否值得。
     */
    void
    ExecHashTableInsert(HashJoinTable hashtable,
                        TupleTableSlot *slot,
                        uint32 hashvalue)
    {
        bool        shouldFree;//是否释放资源
        MinimalTuple tuple = ExecFetchSlotMinimalTuple(slot, &shouldFree);//获取一个MinimalTuple
        int         bucketno;//桶号
        int         batchno;//批次号
    
        ExecHashGetBucketAndBatch(hashtable, hashvalue,
                                  &bucketno, &batchno);//获取桶号和批次号
    
        /*
         * decide whether to put the tuple in the hash table or a temp file
         * 判断是否放到hash表中还是放到临时文件中
         */
        if (batchno == hashtable->curbatch)
        {
            //批次号==hash表的批次号,放到hash表中
            /*
             * put the tuple in hash table
             * 把元组放到hash表中
             */
            HashJoinTuple hashTuple;//hash tuple
            int         hashTupleSize;//大小
            double      ntuples = (hashtable->totalTuples - hashtable->skewTuples);//常规元组数量
    
            /* Create the HashJoinTuple */
            //创建HashJoinTuple
            hashTupleSize = HJTUPLE_OVERHEAD + tuple->t_len;//大小
            hashTuple = (HashJoinTuple) dense_alloc(hashtable, hashTupleSize);//分配存储空间
            //hash值
            hashTuple->hashvalue = hashvalue;
            //拷贝数据
            memcpy(HJTUPLE_MINTUPLE(hashTuple), tuple, tuple->t_len);
    
            /*
             * We always reset the tuple-matched flag on insertion.  This is okay
             * even when reloading a tuple from a batch file, since the tuple
             * could not possibly have been matched to an outer tuple before it
             * went into the batch file.
             * 我们总是在插入时重置元组匹配的标志。
             * 即使在从批处理文件中重新加载元组时，这也是可以的，
             *   因为在元组进入批处理文件之前，它不可能与外部元组匹配。
             */
            HeapTupleHeaderClearMatch(HJTUPLE_MINTUPLE(hashTuple));
    
            /* Push it onto the front of the bucket's list */
            //
            hashTuple->next.unshared = hashtable->buckets.unshared[bucketno];
            hashtable->buckets.unshared[bucketno] = hashTuple;
    
            /*
             * Increase the (optimal) number of buckets if we just exceeded the
             * NTUP_PER_BUCKET threshold, but only when there's still a single
             * batch.
             * 如果刚刚超过了NTUP_PER_BUCKET阈值，但是只有在仍然有单个批处理时，
             *  才增加桶的(优化后)数量。
             */
            if (hashtable->nbatch == 1 &&
                ntuples > (hashtable->nbuckets_optimal * NTUP_PER_BUCKET))
            {
                //只有1个批次而且元组数大于桶数*每桶的元组数
                /* Guard against integer overflow and alloc size overflow */
                //确保整数不要溢出
                if (hashtable->nbuckets_optimal <= INT_MAX / 2 &&
                    hashtable->nbuckets_optimal * 2 <= MaxAllocSize / sizeof(HashJoinTuple))
                {
                    hashtable->nbuckets_optimal *= 2;
                    hashtable->log2_nbuckets_optimal += 1;
                }
            }
    
            /* Account for space used, and back off if we've used too much */
            //声明使用的存储空间，如果使用太多，需要回退
            hashtable->spaceUsed += hashTupleSize;
            if (hashtable->spaceUsed > hashtable->spacePeak)
                hashtable->spacePeak = hashtable->spaceUsed;//超出峰值,则跳转
            if (hashtable->spaceUsed +
                hashtable->nbuckets_optimal * sizeof(HashJoinTuple)
                > hashtable->spaceAllowed)
                ExecHashIncreaseNumBatches(hashtable);//超出允许的空间,则增加批次
        }
        else
        {
            //不在这个批次中
            /*
             * put the tuple into a temp file for later batches
             * 放在临时文件中以便后续处理(减少重复扫描)
             */
            Assert(batchno > hashtable->curbatch);
            ExecHashJoinSaveTuple(tuple,
                                  hashvalue,
                                  &hashtable->innerBatchFile[batchno]);//存储tuple到临时文件中
        }
    
        if (shouldFree)//如需要释放空间,则处理之
            heap_free_minimal_tuple(tuple);
    }
    

### 三、跟踪分析

设置work_mem为较低的值(1MB),便于手工产生批次

    
    
    testdb=# set work_mem='1MB';
    SET
    testdb=# show work_mem;
     work_mem 
    ----------     1MB
    (1 row)
    

测试脚本如下

    
    
    testdb=# set enable_nestloop=false;
    SET
    testdb=# set enable_mergejoin=false;
    SET
    testdb=# explain verbose select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# order by dw.dwbh;
                                              QUERY PLAN                                           
    -----------------------------------------------------------------------------------------------     Sort  (cost=14828.83..15078.46 rows=99850 width=47)
       Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
       Sort Key: dw.dwbh
       ->  Hash Join  (cost=3176.00..6537.55 rows=99850 width=47)
             Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
             Hash Cond: ((gr.grbh)::text = (jf.grbh)::text)
             ->  Hash Join  (cost=289.00..2277.61 rows=99850 width=32)
                   Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm
                   Inner Unique: true
                   Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
                   ->  Seq Scan on public.t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
                         Output: gr.dwbh, gr.grbh, gr.xm, gr.xb, gr.nl
                   ->  Hash  (cost=164.00..164.00 rows=10000 width=20)
                         Output: dw.dwmc, dw.dwbh, dw.dwdz
                         ->  Seq Scan on public.t_dwxx dw  (cost=0.00..164.00 rows=10000 width=20)
                               Output: dw.dwmc, dw.dwbh, dw.dwdz
             ->  Hash  (cost=1637.00..1637.00 rows=100000 width=20)
                   Output: jf.ny, jf.je, jf.grbh
                   ->  Seq Scan on public.t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                         Output: jf.ny, jf.je, jf.grbh
    (20 rows)
    

启动gdb,设置断点,进入ExecHashJoinNewBatch

    
    
    (gdb) b ExecHashJoinNewBatch
    Breakpoint 1 at 0x7031f5: file nodeHashjoin.c, line 943.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecHashJoinNewBatch (hjstate=0x1c40738) at nodeHashjoin.c:943
    943     HashJoinTable hashtable = hjstate->hj_HashTable;
    

获取批次数(8个批次)和当前批次(0,第一个批次)

    
    
    (gdb) n
    950     nbatch = hashtable->nbatch;
    (gdb) 
    951     curbatch = hashtable->curbatch;
    (gdb) 
    953     if (curbatch > 0)
    (gdb) p nbatch
    $5 = 8
    (gdb) p curbatch
    $6 = 0
    

curbatch ==0,刚刚完成了第一个批次,重置倾斜优化的相关状态变量

    
    
    (gdb) n
    971         hashtable->skewEnabled = false;
    (gdb) 
    972         hashtable->skewBucket = NULL;
    (gdb) 
    973         hashtable->skewBucketNums = NULL;
    (gdb) 
    974         hashtable->nSkewBuckets = 0;
    (gdb) 
    975         hashtable->spaceUsedSkew = 0;
    (gdb) 
    995     curbatch++;
    

外表为空或内表为空时的优化,本次调用均不为空

    
    
    (gdb) n
    996     while (curbatch < nbatch &&
    (gdb) 
    997            (hashtable->outerBatchFile[curbatch] == NULL ||
    (gdb) p hashtable->outerBatchFile[curbatch]
    $7 = (BufFile *) 0x1d85290
    (gdb) p hashtable->outerBatchFile[curbatch]
    $8 = (BufFile *) 0x1d85290
    

设置当前批次,重建Hash表

    
    
    (gdb) 
    1023        if (curbatch >= nbatch)
    (gdb) 
    1026        hashtable->curbatch = curbatch;
    (gdb) 
    1031        ExecHashTableReset(hashtable);
    

获取inner relation批次临时文件

    
    
    (gdb) 
    1033        innerFile = hashtable->innerBatchFile[curbatch];
    (gdb) 
    1035        if (innerFile != NULL)
    (gdb) p innerFile
    $9 = (BufFile *) 0x1cc0540
    

临时文件不为NULL,读取文件

    
    
    (gdb) n
    1037            if (BufFileSeek(innerFile, 0, 0L, SEEK_SET))
    (gdb) 
    1042            while ((slot = ExecHashJoinGetSavedTuple(hjstate,
    

进入函数ExecHashJoinGetSavedTuple

    
    
    (gdb) step
    ExecHashJoinGetSavedTuple (hjstate=0x1c40fd8, file=0x1cc0540, hashvalue=0x7ffeace60824, tupleSlot=0x1c4cc20)
        at nodeHashjoin.c:1259
    1259        CHECK_FOR_INTERRUPTS();
    (gdb) 
    

ExecHashJoinGetSavedTuple->读取头部8个字节(header,类型为void *,在64 bit的机器上,大小8个字节)

    
    
    gdb) n
    1266        nread = BufFileRead(file, (void *) header, sizeof(header));
    (gdb) 
    1267        if (nread == 0)             /* end of file */
    (gdb) p nread
    $10 = 8
    (gdb) n
    1272        if (nread != sizeof(header))
    (gdb) 
    

ExecHashJoinGetSavedTuple->获取Hash值(1978434688)

    
    
    (gdb) 
    1276        *hashvalue = header[0];
    (gdb) n
    1277        tuple = (MinimalTuple) palloc(header[1]);
    (gdb) p *hashvalue
    $11 = 1978434688
    

ExecHashJoinGetSavedTuple->获取tuple&元组长度

    
    
    (gdb) n
    1278        tuple->t_len = header[1];
    (gdb) 
    1281                            header[1] - sizeof(uint32));
    (gdb) p tuple->t_len
    $16 = 24
    (gdb) p *tuple
    $17 = {t_len = 24, mt_padding = "\177\177\177\177\177\177", t_infomask2 = 32639, t_infomask = 32639, t_hoff = 127 '\177', 
      t_bits = 0x1c5202f "\177\177\177\177\177\177\177\177\177~\177\177\177\177\177\177\177"}
    (gdb) 
    

ExecHashJoinGetSavedTuple->根据大小读取文件获取元组

    
    
    (gdb) n
    1279        nread = BufFileRead(file,
    (gdb) 
    1282        if (nread != header[1] - sizeof(uint32))
    (gdb) p header[1]
    $18 = 24
    (gdb) p sizeof(uint32)
    $19 = 4
    (gdb) p *tuple
    $20 = {t_len = 24, mt_padding = "\000\000\000\000\000", t_infomask2 = 3, t_infomask = 2, t_hoff = 24 '\030', 
      t_bits = 0x1c5202f ""}
    

ExecHashJoinGetSavedTuple->存储到slot中,完成调用

    
    
    (gdb) n
    1286        return ExecStoreMinimalTuple(tuple, tupleSlot, true);
    (gdb) 
    1287    }
    (gdb) 
    ExecHashJoinNewBatch (hjstate=0x1c40fd8) at nodeHashjoin.c:1051
    1051                ExecHashTableInsert(hashtable, slot, hashvalue);
    

插入到Hash表中

    
    
    (gdb) 
    1051                ExecHashTableInsert(hashtable, slot, hashvalue);
    

进入ExecHashTableInsert

    
    
    (gdb) step
    ExecHashTableInsert (hashtable=0x1c6e1c0, slot=0x1c4cc20, hashvalue=3757101760) at nodeHash.c:1593
    1593        MinimalTuple tuple = ExecFetchSlotMinimalTuple(slot);
    (gdb) 
    

ExecHashTableInsert->获取批次号和hash桶号

    
    
    (gdb) n
    1597        ExecHashGetBucketAndBatch(hashtable, hashvalue,
    (gdb) 
    1603        if (batchno == hashtable->curbatch)
    (gdb) p batchno
    $21 = 1
    (gdb) p bucketno
    $22 = 21184
    (gdb) 
    (gdb) p hashtable->curbatch
    $23 = 1
    

ExecHashTableInsert->批次号与Hash表中的批次号一致,把元组放到Hash表中  
常规元组数量=100000

    
    
    (gdb) n
    1610            double      ntuples = (hashtable->totalTuples - hashtable->skewTuples);
    (gdb) n
    1613            hashTupleSize = HJTUPLE_OVERHEAD + tuple->t_len;
    (gdb) p ntuples
    $24 = 100000
    

ExecHashTableInsert->创建HashJoinTuple,重置元组匹配标记

    
    
    (gdb) n
    1614            hashTuple = (HashJoinTuple) dense_alloc(hashtable, hashTupleSize);
    (gdb) 
    1616            hashTuple->hashvalue = hashvalue;
    (gdb) 
    1617            memcpy(HJTUPLE_MINTUPLE(hashTuple), tuple, tuple->t_len);
    (gdb) 
    1625            HeapTupleHeaderClearMatch(HJTUPLE_MINTUPLE(hashTuple));
    (gdb) 
    

ExecHashTableInsert->元组放在Hash表桶链表的前面

    
    
    (gdb) n
    1628            hashTuple->next.unshared = hashtable->buckets.unshared[bucketno];
    (gdb) 
    1629            hashtable->buckets.unshared[bucketno] = hashTuple;
    (gdb) 
    1636            if (hashtable->nbatch == 1 &&
    (gdb) 
    

ExecHashTableInsert->调整或记录Hash表内存使用的峰值并返回,回到ExecHashJoinNewBatch

    
    
    (gdb) 
    1649            hashtable->spaceUsed += hashTupleSize;
    (gdb) 
    ...
    (gdb) 
    1667    }
    (gdb) n
    ExecHashJoinNewBatch (hjstate=0x1c40fd8) at nodeHashjoin.c:1042
    1042            while ((slot = ExecHashJoinGetSavedTuple(hjstate,
    

循环插入到Hash表中

    
    
    1042            while ((slot = ExecHashJoinGetSavedTuple(hjstate,
    (gdb) n
    1051                ExecHashTableInsert(hashtable, slot, hashvalue);
    ...
    

DONE!

### 四、参考资料

[Hash Joins: Past, Present and Future/PGCon
2017](https://www.pgcon.org/2017/schedule/events/1053.en.html)  
[A Look at How Postgres Executes a Tiny Join - Part
1](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-1)  
[A Look at How Postgres Executes a Tiny Join - Part
2](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-2)  
[Assignment 2 Symmetric Hash
Join](https://cs.uwaterloo.ca/~david/cs448/A2.pdf)

