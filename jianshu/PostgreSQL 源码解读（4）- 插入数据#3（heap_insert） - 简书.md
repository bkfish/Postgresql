本文简单介绍了PG插入数据部分的源码，这是第三部分，主要内容包括heap_insert函数的实现逻辑，该函数在源文件heapam.c中。

### 一、基础信息

heap_insert使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**

    
    
    1、CommandId
    32bit无符号整型
    typedef uint32 CommandId;
    
    2、options
    整型，标记bits
     /* "options" flag bits for heap_insert */
     #define HEAP_INSERT_SKIP_WAL    0x0001
     #define HEAP_INSERT_SKIP_FSM    0x0002
     #define HEAP_INSERT_FROZEN      0x0004
     #define HEAP_INSERT_SPECULATIVE 0x0008
    
    3、BulkInsertState
    批量插入状态指针
      /*
      * state for bulk inserts --- private to heapam.c and hio.c
      *
      * If current_buf isn't InvalidBuffer, then we are holding an extra pin
      * on that buffer.
      *
      * "typedef struct BulkInsertStateData *BulkInsertState" is in heapam.h
      */
     typedef struct BulkInsertStateData
     {
         BufferAccessStrategy strategy;  /* our BULKWRITE strategy object */
         Buffer      current_buf;    /* current insertion target page */
     }     BulkInsertStateData;
     
    typedef struct BulkInsertStateData *BulkInsertState;
    
    4、TransactionId
    32bit无符号整型
     typedef uint32 TransactionId;
     typedef uint32 LocalTransactionId;
     typedef uint32 SubTransactionId;
    
    5、xl_heap_insert
     typedef struct xl_heap_insert
     {
         OffsetNumber offnum;        /* inserted tuple's offset */
         uint8       flags;
         /* xl_heap_header & TUPLE DATA in backup block 0 */
     } xl_heap_insert;
     
     #define SizeOfHeapInsert    (offsetof(xl_heap_insert, flags) + sizeof(uint8))
    
    6、xl_heap_header
     typedef struct xl_heap_header
     {
         uint16      t_infomask2;
         uint16      t_infomask;
         uint8       t_hoff;
     } xl_heap_header;
     
     #define SizeOfHeapHeader    (offsetof(xl_heap_header, t_hoff) + sizeof(uint8))
    
    7、XLogRecPtr
    64bit无符号长整型
     typedef uint64 XLogRecPtr;
    
    

**依赖的函数**  
_1、heap_prepare_insert_

    
    
    /*
     * Subroutine for heap_insert(). Prepares a tuple for insertion. This sets the
     * tuple header fields, assigns an OID, and toasts the tuple if necessary.
     * Returns a toasted version of the tuple if it was toasted, or the original
     * tuple if not. Note that in any case, the header fields are also set in
     * the original tuple.
     */
    static HeapTuple
    heap_prepare_insert(Relation relation, HeapTuple tup, TransactionId xid,
                        CommandId cid, int options)
    {
        /*
         * Parallel operations are required to be strictly read-only in a parallel
         * worker.  Parallel inserts are not safe even in the leader in the
         * general case, because group locking means that heavyweight locks for
         * relation extension or GIN page locks will not conflict between members
         * of a lock group, but we don't prohibit that case here because there are
         * useful special cases that we can safely allow, such as CREATE TABLE AS.
         */
        //暂不支持并行操作
        if (IsParallelWorker())
            ereport(ERROR,
                    (errcode(ERRCODE_INVALID_TRANSACTION_STATE),
                     errmsg("cannot insert tuples in a parallel worker")));
    
        //设置Oid
        if (relation->rd_rel->relhasoids)
        {
    #ifdef NOT_USED
            /* this is redundant with an Assert in HeapTupleSetOid */
            Assert(tup->t_data->t_infomask & HEAP_HASOID);
    #endif
    
            /*
             * If the object id of this tuple has already been assigned, trust the
             * caller.  There are a couple of ways this can happen.  At initial db
             * creation, the backend program sets oids for tuples. When we define
             * an index, we set the oid.  Finally, in the future, we may allow
             * users to set their own object ids in order to support a persistent
             * object store (objects need to contain pointers to one another).
             */
            if (!OidIsValid(HeapTupleGetOid(tup)))
                HeapTupleSetOid(tup, GetNewOid(relation));
        }
        else
        {
            /* check there is not space for an OID */
            Assert(!(tup->t_data->t_infomask & HEAP_HASOID));
        }
    
        //设置标记位t_infomask/t_infomask2
        tup->t_data->t_infomask &= ~(HEAP_XACT_MASK);//HEAP_XACT_MASK=0xFFF0，取反
        tup->t_data->t_infomask2 &= ~(HEAP2_XACT_MASK);//HEAP2_XACT_MASK=0xE000，取反
        tup->t_data->t_infomask |= HEAP_XMAX_INVALID;//插入数据，XMAX设置为invalid
        HeapTupleHeaderSetXmin(tup->t_data, xid);//设置xmin为当前事务id
        if (options & HEAP_INSERT_FROZEN)//冻结型插入（在事务id回卷时发生）
            HeapTupleHeaderSetXminFrozen(tup->t_data);
        //设置cid
        HeapTupleHeaderSetCmin(tup->t_data, cid);
        //设置xmax=0
        HeapTupleHeaderSetXmax(tup->t_data, 0); /* for cleanliness */
        //设置Oid
        tup->t_tableOid = RelationGetRelid(relation);
    
        /*
         * If the new tuple is too big for storage or contains already toasted
         * out-of-line attributes from some other relation, invoke the toaster.
         */
        if (relation->rd_rel->relkind != RELKIND_RELATION &&
            relation->rd_rel->relkind != RELKIND_MATVIEW)
        {
            /* toast table entries should never be recursively toasted */
            Assert(!HeapTupleHasExternal(tup));
            return tup;
        }
        else if (HeapTupleHasExternal(tup) || tup->t_len > TOAST_TUPLE_THRESHOLD)
            return toast_insert_or_update(relation, tup, NULL, options);
        else
            return tup;
    }
    

_2、RelationGetBufferForTuple_  
_稍长，请耐心阅读，如能读懂，必有收获_

    
    
    /*
    输入：
        relation-数据表
        len-需要的空间大小
        otherBuffer-用于update场景，上一次pinned的buffer
        options-处理选项
        bistate-BulkInsert标记
        vmbuffer-第1个vm(visibilitymap)
        vmbuffer_other-用于update场景，上一次pinned的buffer对应的vm(visibilitymap)
        注意:
        otherBuffer这个参数让人觉得困惑，原因是PG的机制使然
        Update时，不是原地更新，而是原数据保留（更新xmax），新数据插入
        原数据&新数据如果在不同Block中，锁定Block的时候可能会出现Deadlock
        举个例子：Session A更新表T的第一行，第一行在Block 0中，新数据存储在Block 2中
                  Session B更新表T的第二行，第二行在Block 0中，新数据存储在Block 2中
                  Block 0/2均要锁定才能完整实现Update操作：
                  如果Session A先锁定了Block 2，Session B先锁定了Block 0，
                  然后Session A尝试锁定Block 0，Session B尝试锁定Block 2，这时候就会出现死锁
                  为了避免这种情况，PG规定锁定时，同一个Relation，按Block的编号顺序锁定，
                  如需要锁定0和2，那必须先锁定Block 0，再锁定2
    输出：
        为Tuple分配的Buffer
    附：
    Pinned buffers：means buffers are currently being used,it should not be flushed out.
    */
    Buffer
    RelationGetBufferForTuple(Relation relation, Size len,
                              Buffer otherBuffer, int options,
                              BulkInsertState bistate,
                              Buffer *vmbuffer, Buffer *vmbuffer_other)
    {
        bool        use_fsm = !(options & HEAP_INSERT_SKIP_FSM);//是否使用FSM寻找空闲空间
        Buffer      buffer = InvalidBuffer;//
        Page        page;//
        Size        pageFreeSpace = 0,//page空闲空间
                    saveFreeSpace = 0;//page需要预留的空间
        BlockNumber targetBlock,//目标Block
                    otherBlock;//上一次pinned的buffer对应的Block
        bool        needLock;//是否需要上锁
    
        len = MAXALIGN(len);        /* be conservative *///大小对齐
    
        /* Bulk insert is not supported for updates, only inserts. */
        Assert(otherBuffer == InvalidBuffer || !bistate);//otherBuffer有效，说明是update操作，不支持bi(BulkInsert)
    
        /*
         * If we're gonna fail for oversize tuple, do it right away
         */
        if (len > MaxHeapTupleSize)
            ereport(ERROR,
                    (errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
                     errmsg("row is too big: size %zu, maximum size %zu",
                            len, MaxHeapTupleSize)));
    
        /* Compute desired extra freespace due to fillfactor option */
        //获取预留空间
        saveFreeSpace = RelationGetTargetPageFreeSpace(relation,
                                                       HEAP_DEFAULT_FILLFACTOR);
        //update操作,获取上次pinned buffer对应的Block
        if (otherBuffer != InvalidBuffer)
            otherBlock = BufferGetBlockNumber(otherBuffer);
        else
            otherBlock = InvalidBlockNumber;    /* just to keep compiler quiet */
    
        /*
         * We first try to put the tuple on the same page we last inserted a tuple
         * on, as cached in the BulkInsertState or relcache entry.  If that
         * doesn't work, we ask the Free Space Map to locate a suitable page.
         * Since the FSM's info might be out of date, we have to be prepared to
         * loop around and retry multiple times. (To insure this isn't an infinite
         * loop, we must update the FSM with the correct amount of free space on
         * each page that proves not to be suitable.)  If the FSM has no record of
         * a page with enough free space, we give up and extend the relation.
         *
         * When use_fsm is false, we either put the tuple onto the existing target
         * page or extend the relation.
         */
        if (len + saveFreeSpace > MaxHeapTupleSize)
        {
            //如果需要的大小+预留空间大于可容纳的最大Tuple大小，不使用FSM，扩展后再尝试
            /* can't fit, don't bother asking FSM */
            targetBlock = InvalidBlockNumber;
            use_fsm = false;
        }
        else if (bistate && bistate->current_buf != InvalidBuffer)//BulkInsert模式
            targetBlock = BufferGetBlockNumber(bistate->current_buf);
        else
            targetBlock = RelationGetTargetBlock(relation);//普通Insert模式
    
        if (targetBlock == InvalidBlockNumber && use_fsm)//还没有找到合适的BlockNumber，需要使用FSM
        {
            /*
             * We have no cached target page, so ask the FSM for an initial
             * target.
             */
            //使用FSM申请空闲空间=len + saveFreeSpace的块
            targetBlock = GetPageWithFreeSpace(relation, len + saveFreeSpace);
    
            /*
             * If the FSM knows nothing of the rel, try the last page before we
             * give up and extend.  This avoids one-tuple-per-page syndrome during
             * bootstrapping or in a recently-started system.
             */
            //申请不到，使用最后一个块，否则扩展或者放弃
            if (targetBlock == InvalidBlockNumber)
            {
                BlockNumber nblocks = RelationGetNumberOfBlocks(relation);
    
                if (nblocks > 0)
                    targetBlock = nblocks - 1;
            }
        }
    
    loop:
        while (targetBlock != InvalidBlockNumber)//已成功获取插入数据的块号
        {
            /*
             * Read and exclusive-lock the target block, as well as the other
             * block if one was given, taking suitable care with lock ordering and
             * the possibility they are the same block.
             *
             * If the page-level all-visible flag is set, caller will need to
             * clear both that and the corresponding visibility map bit.  However,
             * by the time we return, we'll have x-locked the buffer, and we don't
             * want to do any I/O while in that state.  So we check the bit here
             * before taking the lock, and pin the page if it appears necessary.
             * Checking without the lock creates a risk of getting the wrong
             * answer, so we'll have to recheck after acquiring the lock.
             */
            if (otherBuffer == InvalidBuffer)//非Update操作
            {
                /* easy case */
                buffer = ReadBufferBI(relation, targetBlock, bistate);//获取Buffer
                if (PageIsAllVisible(BufferGetPage(buffer)))
                    //如果Page全局可见，那么把Page Pin在内存中（Pin的意思是固定/保留）
                    visibilitymap_pin(relation, targetBlock, vmbuffer);
                LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);//锁定buffer
            }
            else if (otherBlock == targetBlock)//Update操作，新记录跟原记录在同一个Block中
            {
                /* also easy case */
                buffer = otherBuffer;
                if (PageIsAllVisible(BufferGetPage(buffer)))
                    visibilitymap_pin(relation, targetBlock, vmbuffer);
                LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
            }
            else if (otherBlock < targetBlock)//Update操作，原记录所在的Block < 新记录的Block
            {
                /* lock other buffer first */
                buffer = ReadBuffer(relation, targetBlock);
                if (PageIsAllVisible(BufferGetPage(buffer)))
                    visibilitymap_pin(relation, targetBlock, vmbuffer);
                LockBuffer(otherBuffer, BUFFER_LOCK_EXCLUSIVE);//优先锁定BlockNumber小的那个
                LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
            }
            else//Update操作，原记录所在的Block > 新记录的Block
            {
                /* lock target buffer first */
                buffer = ReadBuffer(relation, targetBlock);
                if (PageIsAllVisible(BufferGetPage(buffer)))
                    visibilitymap_pin(relation, targetBlock, vmbuffer);
                LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);//优先锁定BlockNumber小的那个
                LockBuffer(otherBuffer, BUFFER_LOCK_EXCLUSIVE);
            }
    
            /*
             * We now have the target page (and the other buffer, if any) pinned
             * and locked.  However, since our initial PageIsAllVisible checks
             * were performed before acquiring the lock, the results might now be
             * out of date, either for the selected victim buffer, or for the
             * other buffer passed by the caller.  In that case, we'll need to
             * give up our locks, go get the pin(s) we failed to get earlier, and
             * re-lock.  That's pretty painful, but hopefully shouldn't happen
             * often.
             *
             * Note that there's a small possibility that we didn't pin the page
             * above but still have the correct page pinned anyway, either because
             * we've already made a previous pass through this loop, or because
             * caller passed us the right page anyway.
             *
             * Note also that it's possible that by the time we get the pin and
             * retake the buffer locks, the visibility map bit will have been
             * cleared by some other backend anyway.  In that case, we'll have
             * done a bit of extra work for no gain, but there's no real harm
             * done.
             */
            if (otherBuffer == InvalidBuffer || buffer <= otherBuffer)
                GetVisibilityMapPins(relation, buffer, otherBuffer,
                                     targetBlock, otherBlock, vmbuffer,
                                     vmbuffer_other);//Pin VM在内存中
            else
                GetVisibilityMapPins(relation, otherBuffer, buffer,
                                     otherBlock, targetBlock, vmbuffer_other,
                                     vmbuffer);//Pin VM在内存中
    
            /*
             * Now we can check to see if there's enough free space here. If so,
             * we're done.
             */
            page = BufferGetPage(buffer);
            pageFreeSpace = PageGetHeapFreeSpace(page);
            if (len + saveFreeSpace <= pageFreeSpace)//有足够的空间存储数据，返回此Buffer
            {
                /* use this page as future insert target, too */
                RelationSetTargetBlock(relation, targetBlock);
                return buffer;
            }
    
            /*
             * Not enough space, so we must give up our page locks and pin (if
             * any) and prepare to look elsewhere.  We don't care which order we
             * unlock the two buffers in, so this can be slightly simpler than the
             * code above.
             */
            LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
            if (otherBuffer == InvalidBuffer)
                ReleaseBuffer(buffer);
            else if (otherBlock != targetBlock)
            {
                LockBuffer(otherBuffer, BUFFER_LOCK_UNLOCK);
                ReleaseBuffer(buffer);
            }
    
            /* Without FSM, always fall out of the loop and extend */
            if (!use_fsm)//不使用FSM定位空闲空间，跳出循环，执行扩展
                break;
    
            /*
             * Update FSM as to condition of this page, and ask for another page
             * to try.
             */
            //使用FSM获取下一个备选的Block
            //注意：如果全部扫描后发现没有满足条件的Block，targetBlock = InvalidBlockNumber，跳出循环
            targetBlock = RecordAndGetPageWithFreeSpace(relation,
                                                        targetBlock,
                                                        pageFreeSpace,
                                                        len + saveFreeSpace);
        }
        
        //没有获取满足条件的Block，扩展表
        /*
         * Have to extend the relation.
         *
         * We have to use a lock to ensure no one else is extending the rel at the
         * same time, else we will both try to initialize the same new page.  We
         * can skip locking for new or temp relations, however, since no one else
         * could be accessing them.
         */
        needLock = !RELATION_IS_LOCAL(relation);//新创建的数据表或者临时表，无需Lock
    
        /*
         * If we need the lock but are not able to acquire it immediately, we'll
         * consider extending the relation by multiple blocks at a time to manage
         * contention on the relation extension lock.  However, this only makes
         * sense if we're using the FSM; otherwise, there's no point.
         */
        if (needLock)//需要锁定
        {
            if (!use_fsm)
                LockRelationForExtension(relation, ExclusiveLock);
            else if (!ConditionalLockRelationForExtension(relation, ExclusiveLock))
            {
                /* Couldn't get the lock immediately; wait for it. */
                LockRelationForExtension(relation, ExclusiveLock);
    
                /*
                 * Check if some other backend has extended a block for us while
                 * we were waiting on the lock.
                 */
                //如有其它进程扩展了数据表，那么可以成功获取满足条件的targetBlock
                targetBlock = GetPageWithFreeSpace(relation, len + saveFreeSpace);
    
                /*
                 * If some other waiter has already extended the relation, we
                 * don't need to do so; just use the existing freespace.
                 */
                if (targetBlock != InvalidBlockNumber)
                {
                    UnlockRelationForExtension(relation, ExclusiveLock);
                    goto loop;
                }
    
                /* Time to bulk-extend. */
                //其它进程没有扩展
                //Just extend it!
                RelationAddExtraBlocks(relation, bistate);
            }
        }
    
        /*
         * In addition to whatever extension we performed above, we always add at
         * least one block to satisfy our own request.
         *
         * XXX This does an lseek - rather expensive - but at the moment it is the
         * only way to accurately determine how many blocks are in a relation.  Is
         * it worth keeping an accurate file length in shared memory someplace,
         * rather than relying on the kernel to do it for us?
         */
        //扩展表后，New Page！
        buffer = ReadBufferBI(relation, P_NEW, bistate);
    
        /*
         * We can be certain that locking the otherBuffer first is OK, since it
         * must have a lower page number.
         */
        if (otherBuffer != InvalidBuffer)
            LockBuffer(otherBuffer, BUFFER_LOCK_EXCLUSIVE);//otherBuffer的顺序一定在扩展的Block之后，Lock it！
    
        /*
         * Now acquire lock on the new page.
         */
        LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);//锁定New Page
    
        /*
         * Release the file-extension lock; it's now OK for someone else to extend
         * the relation some more.  Note that we cannot release this lock before
         * we have buffer lock on the new page, or we risk a race condition
         * against vacuumlazy.c --- see comments therein.
         */
        if (needLock)
            UnlockRelationForExtension(relation, ExclusiveLock);//释放扩展锁
    
        /*
         * We need to initialize the empty new page.  Double-check that it really
         * is empty (this should never happen, but if it does we don't want to
         * risk wiping out valid data).
         */
        page = BufferGetPage(buffer);//获取相应的Page
    
        if (!PageIsNew(page))//不是New Page，那一定某个地方搞错了！
            elog(ERROR, "page %u of relation \"%s\" should be empty but is not",
                 BufferGetBlockNumber(buffer),
                 RelationGetRelationName(relation));
        //初始化New Page
        PageInit(page, BufferGetPageSize(buffer), 0);
        //New Page也满足不了要求的大小，报错
        if (len > PageGetHeapFreeSpace(page))
        {
            /* We should not get here given the test at the top */
            elog(PANIC, "tuple is too big: size %zu", len);
        }
    
        /*
         * Remember the new page as our target for future insertions.
         *
         * XXX should we enter the new page into the free space map immediately,
         * or just keep it for this backend's exclusive use in the short run
         * (until VACUUM sees it)?  Seems to depend on whether you expect the
         * current backend to make more insertions or not, which is probably a
         * good bet most of the time.  So for now, don't add it to FSM yet.
         */
        //终于找到了可用于存储数据的Block
        RelationSetTargetBlock(relation, BufferGetBlockNumber(buffer));
        //返回
        return buffer;
    }
    
    //-------------------------------------------------------------------------------    /*
     * Read in a buffer, using bulk-insert strategy if bistate isn't NULL.
     */
    static Buffer
    ReadBufferBI(Relation relation, BlockNumber targetBlock,
                 BulkInsertState bistate)
    {
        Buffer      buffer;
    
        /* If not bulk-insert, exactly like ReadBuffer */
        if (!bistate)
            return ReadBuffer(relation, targetBlock);//非BulkInsert模式，使用常规方法获取
        
        //TODO 以下为BI模式
        /* If we have the desired block already pinned, re-pin and return it */
        if (bistate->current_buf != InvalidBuffer)
        {
            if (BufferGetBlockNumber(bistate->current_buf) == targetBlock)
            {
                IncrBufferRefCount(bistate->current_buf);
                return bistate->current_buf;
            }
            /* ... else drop the old buffer */
            ReleaseBuffer(bistate->current_buf);
            bistate->current_buf = InvalidBuffer;
        }
    
        /* Perform a read using the buffer strategy */
        buffer = ReadBufferExtended(relation, MAIN_FORKNUM, targetBlock,
                                    RBM_NORMAL, bistate->strategy);
    
        /* Save the selected block as target for future inserts */
        IncrBufferRefCount(buffer);
        bistate->current_buf = buffer;
    
        return buffer;
    }
    
     /*
      * ReadBuffer -- a shorthand for ReadBufferExtended, for reading from main
      *      fork with RBM_NORMAL mode and default strategy.
      */
     Buffer
     ReadBuffer(Relation reln, BlockNumber blockNum)
     {
         return ReadBufferExtended(reln, MAIN_FORKNUM, blockNum, RBM_NORMAL, NULL);
     }
    
     typedef enum ForkNumber
     {
         InvalidForkNumber = -1,
         MAIN_FORKNUM = 0,
         FSM_FORKNUM,
         VISIBILITYMAP_FORKNUM,
         INIT_FORKNUM
     
         /*
          * NOTE: if you add a new fork, change MAX_FORKNUM and possibly
          * FORKNAMECHARS below, and update the forkNames array in
          * src/common/relpath.c
          */
     } ForkNumber; 
    //参考url : https://www.postgresql.org/docs/11/static/storage-file-layout.html
    
     /*
      * ReadBufferExtended -- returns a buffer containing the requested
      *      block of the requested relation.  If the blknum
      *      requested is P_NEW, extend the relation file and
      *      allocate a new block.  (Caller is responsible for
      *      ensuring that only one backend tries to extend a
      *      relation at the same time!)
      *
      * Returns: the buffer number for the buffer containing
      *      the block read.  The returned buffer has been pinned.
      *      Does not return on error --- elog's instead.
      *
      * Assume when this function is called, that reln has been opened already.
      *
      * In RBM_NORMAL mode, the page is read from disk, and the page header is
      * validated.  An error is thrown if the page header is not valid.  (But
      * note that an all-zero page is considered "valid"; see PageIsVerified().)
      *
      * RBM_ZERO_ON_ERROR is like the normal mode, but if the page header is not
      * valid, the page is zeroed instead of throwing an error. This is intended
      * for non-critical data, where the caller is prepared to repair errors.
      *
      * In RBM_ZERO_AND_LOCK mode, if the page isn't in buffer cache already, it's
      * filled with zeros instead of reading it from disk.  Useful when the caller
      * is going to fill the page from scratch, since this saves I/O and avoids
      * unnecessary failure if the page-on-disk has corrupt page headers.
      * The page is returned locked to ensure that the caller has a chance to
      * initialize the page before it's made visible to others.
      * Caution: do not use this mode to read a page that is beyond the relation's
      * current physical EOF; that is likely to cause problems in md.c when
      * the page is modified and written out. P_NEW is OK, though.
      *
      * RBM_ZERO_AND_CLEANUP_LOCK is the same as RBM_ZERO_AND_LOCK, but acquires
      * a cleanup-strength lock on the page.
      *
      * RBM_NORMAL_NO_LOG mode is treated the same as RBM_NORMAL here.
      *
      * If strategy is not NULL, a nondefault buffer access strategy is used.
      * See buffer/README for details.
      */
     Buffer
     ReadBufferExtended(Relation reln, ForkNumber forkNum, BlockNumber blockNum,
                        ReadBufferMode mode, BufferAccessStrategy strategy)
     {
         bool        hit;
         Buffer      buf;
     
         /* Open it at the smgr level if not already done */
         RelationOpenSmgr(reln);//Smgr=Storage Manager，数据表存储管理封装
     
         /*
          * Reject attempts to read non-local temporary relations; we would be
          * likely to get wrong data since we have no visibility into the owning
          * session's local buffers.
          */
         if (RELATION_IS_OTHER_TEMP(reln))
             ereport(ERROR,
                     (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                      errmsg("cannot access temporary tables of other sessions")));
     
         /*
          * Read the buffer, and update pgstat counters to reflect a cache hit or
          * miss.
          */
         pgstat_count_buffer_read(reln);//统计信息
         //TODO Buffer管理后续再行解读
         buf = ReadBuffer_common(reln->rd_smgr, reln->rd_rel->relpersistence,
                                 forkNum, blockNum, mode, strategy, &hit);
         if (hit)
             pgstat_count_buffer_hit(reln);//统计信息
         return buf;
     }
     
    

_3、CheckForSerializableConflictIn_  
检查序列化操作是否会出现冲突。比如并发执行delete  & update操作的时候。

    
    
    /*
      * CheckForSerializableConflictIn
      *      We are writing the given tuple.  If that indicates a rw-conflict
      *      in from another serializable transaction, take appropriate action.
      *
      * Skip checking for any granularity for which a parameter is missing.
      *
      * A tuple update or delete is in conflict if we have a predicate lock
      * against the relation or page in which the tuple exists, or against the
      * tuple itself.
      */
     void
     CheckForSerializableConflictIn(Relation relation, HeapTuple tuple,
                                    Buffer buffer)
     {
         PREDICATELOCKTARGETTAG targettag;
     
         if (!SerializationNeededForWrite(relation))
             return;
     
         /* Check if someone else has already decided that we need to die */
         if (SxactIsDoomed(MySerializableXact))
             ereport(ERROR,
                     (errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),
                      errmsg("could not serialize access due to read/write dependencies among transactions"),
                      errdetail_internal("Reason code: Canceled on identification as a pivot, during conflict in checking."),
                      errhint("The transaction might succeed if retried.")));
     
         /*
          * We're doing a write which might cause rw-conflicts now or later.
          * Memorize that fact.
          */
         MyXactDidWrite = true;
     
         /*
          * It is important that we check for locks from the finest granularity to
          * the coarsest granularity, so that granularity promotion doesn't cause
          * us to miss a lock.  The new (coarser) lock will be acquired before the
          * old (finer) locks are released.
          *
          * It is not possible to take and hold a lock across the checks for all
          * granularities because each target could be in a separate partition.
          */
         if (tuple != NULL)
         {
             SET_PREDICATELOCKTARGETTAG_TUPLE(targettag,
                                              relation->rd_node.dbNode,
                                              relation->rd_id,
                                              ItemPointerGetBlockNumber(&(tuple->t_self)),
                                              ItemPointerGetOffsetNumber(&(tuple->t_self)));
             CheckTargetForConflictsIn(&targettag);
         }
     
         if (BufferIsValid(buffer))
         {
             SET_PREDICATELOCKTARGETTAG_PAGE(targettag,
                                             relation->rd_node.dbNode,
                                             relation->rd_id,
                                             BufferGetBlockNumber(buffer));
             CheckTargetForConflictsIn(&targettag);
         }
     
         SET_PREDICATELOCKTARGETTAG_RELATION(targettag,
                                             relation->rd_node.dbNode,
                                             relation->rd_id);
         CheckTargetForConflictsIn(&targettag);
     }
    

4、START_CRIT_SECTION

    
    
    extern PGDLLIMPORT volatile uint32 CritSectionCount; 
    #define START_CRIT_SECTION()  (CritSectionCount++)
    

5、PageIsAllVisible

    
    
    通过位操作判断Page是否All Visible
    #define PageIsAllVisible(page) \
         (((PageHeader) (page))->pd_flags & PD_ALL_VISIBLE)
    

6、PageClearAllVisible

    
    
    通过位操作清除All Visible标记
     #define PageClearAllVisible(page) \
      (((PageHeader) (page))->pd_flags &= ~PD_ALL_VISIBLE)
    

7、visibilitymap_clear

    
    
    //TODO 缓冲区管理相关的设置，待进一步理解
     /*
      *  visibilitymap_clear - clear specified bits for one page in visibility map
      *
      * You must pass a buffer containing the correct map page to this function.
      * Call visibilitymap_pin first to pin the right one. This function doesn't do
      * any I/O.  Returns true if any bits have been cleared and false otherwise.
      */
     bool
     visibilitymap_clear(Relation rel, BlockNumber heapBlk, Buffer buf, uint8 flags)
     {
         BlockNumber mapBlock = HEAPBLK_TO_MAPBLOCK(heapBlk);
         int         mapByte = HEAPBLK_TO_MAPBYTE(heapBlk);
         int         mapOffset = HEAPBLK_TO_OFFSET(heapBlk);
         uint8       mask = flags << mapOffset;
         char       *map;
         bool        cleared = false;
     
         Assert(flags & VISIBILITYMAP_VALID_BITS);
     
     #ifdef TRACE_VISIBILITYMAP
         elog(DEBUG1, "vm_clear %s %d", RelationGetRelationName(rel), heapBlk);
     #endif
          if (!BufferIsValid(buf) || BufferGetBlockNumber(buf) != mapBlock)
             elog(ERROR, "wrong buffer passed to visibilitymap_clear");
     
         LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
         map = PageGetContents(BufferGetPage(buf));
     
         if (map[mapByte] & mask)
         {
             map[mapByte] &= ~mask;
              MarkBufferDirty(buf);
             cleared = true;
         }
          LockBuffer(buf, BUFFER_LOCK_UNLOCK);
          return cleared;
     }
    

8、MarkBufferDirty

    
    
    //设置缓冲块为Dirty（待Flush到数据文件）
    //TODO 缓冲区相关管理
     /*
      * MarkBufferDirty
      *
      *      Marks buffer contents as dirty (actual write happens later).
      *
      * Buffer must be pinned and exclusive-locked.  (If caller does not hold
      * exclusive lock, then somebody could be in process of writing the buffer,
      * leading to risk of bad data written to disk.)
      */
     void
     MarkBufferDirty(Buffer buffer)
     {
         BufferDesc *bufHdr;
         uint32      buf_state;
         uint32      old_buf_state;
     
         if (!BufferIsValid(buffer))
             elog(ERROR, "bad buffer ID: %d", buffer);
     
         if (BufferIsLocal(buffer))
         {
             MarkLocalBufferDirty(buffer);
             return;
         }
     
         bufHdr = GetBufferDescriptor(buffer - 1);
     
         Assert(BufferIsPinned(buffer));
         Assert(LWLockHeldByMeInMode(BufferDescriptorGetContentLock(bufHdr),
                                     LW_EXCLUSIVE));
     
         old_buf_state = pg_atomic_read_u32(&bufHdr->state);
         for (;;)
         {
             if (old_buf_state & BM_LOCKED)
                 old_buf_state = WaitBufHdrUnlocked(bufHdr);
     
             buf_state = old_buf_state;
     
             Assert(BUF_STATE_GET_REFCOUNT(buf_state) > 0);
             buf_state |= BM_DIRTY | BM_JUST_DIRTIED;
     
             if (pg_atomic_compare_exchange_u32(&bufHdr->state, &old_buf_state,
                                                buf_state))
                 break;
         }
     
         /*
          * If the buffer was not dirty already, do vacuum accounting.
          */
         if (!(old_buf_state & BM_DIRTY))
         {
             VacuumPageDirty++;
             pgBufferUsage.shared_blks_dirtied++;
             if (VacuumCostActive)
                 VacuumCostBalance += VacuumCostPageDirty;
         }
     }
    

9、RelationNeedsWAL

    
    
    非临时表，需持久化的数据表
     /*
      * RelationNeedsWAL
      *      True if relation needs WAL.
      */
     #define RelationNeedsWAL(relation) \
         ((relation)->rd_rel->relpersistence == RELPERSISTENCE_PERMANENT)
    

10、RelationIsAccessibleInLogicalDecoding

    
    
     /*
      * RelationIsAccessibleInLogicalDecoding
      *      True if we need to log enough information to have access via
      *      decoding snapshot.
      */
     #define RelationIsAccessibleInLogicalDecoding(relation) \
         (XLogLogicalInfoActive() && \ //处于逻辑复制活动状态
          RelationNeedsWAL(relation) && \ //需要写WAL日志
          (IsCatalogRelation(relation) || RelationIsUsedAsCatalogTable(relation)))//Catalog类型表
    

11、log_heap_new_cid

    
    
     /*
      * Perform XLogInsert of an XLOG_HEAP2_NEW_CID record
      *
      * This is only used in wal_level >= WAL_LEVEL_LOGICAL, and only for catalog
      * tuples.
      */
     static XLogRecPtr
     log_heap_new_cid(Relation relation, HeapTuple tup)
     {
         xl_heap_new_cid xlrec;
     
         XLogRecPtr  recptr;
         HeapTupleHeader hdr = tup->t_data;
     
         Assert(ItemPointerIsValid(&tup->t_self));
         Assert(tup->t_tableOid != InvalidOid);
     
         xlrec.top_xid = GetTopTransactionId();
         xlrec.target_node = relation->rd_node;
         xlrec.target_tid = tup->t_self;
     
         /*
          * If the tuple got inserted & deleted in the same TX we definitely have a
          * combocid, set cmin and cmax.
          */
         if (hdr->t_infomask & HEAP_COMBOCID)
         {
             Assert(!(hdr->t_infomask & HEAP_XMAX_INVALID));
             Assert(!HeapTupleHeaderXminInvalid(hdr));
             xlrec.cmin = HeapTupleHeaderGetCmin(hdr);
             xlrec.cmax = HeapTupleHeaderGetCmax(hdr);
             xlrec.combocid = HeapTupleHeaderGetRawCommandId(hdr);
         }
         /* No combocid, so only cmin or cmax can be set by this TX */
         else
         {
             /*
              * Tuple inserted.
              *
              * We need to check for LOCK ONLY because multixacts might be
              * transferred to the new tuple in case of FOR KEY SHARE updates in
              * which case there will be an xmax, although the tuple just got
              * inserted.
              */
             if (hdr->t_infomask & HEAP_XMAX_INVALID ||
                 HEAP_XMAX_IS_LOCKED_ONLY(hdr->t_infomask))
             {
                 xlrec.cmin = HeapTupleHeaderGetRawCommandId(hdr);
                 xlrec.cmax = InvalidCommandId;
             }
             /* Tuple from a different tx updated or deleted. */
             else
             {
                 xlrec.cmin = InvalidCommandId;
                 xlrec.cmax = HeapTupleHeaderGetRawCommandId(hdr);
     
             }
             xlrec.combocid = InvalidCommandId;
         }
     
         /*
          * Note that we don't need to register the buffer here, because this
          * operation does not modify the page. The insert/update/delete that
          * called us certainly did, but that's WAL-logged separately.
          */
         XLogBeginInsert();
         XLogRegisterData((char *) &xlrec, SizeOfHeapNewCid);
     
         /* will be looked at irrespective of origin */
     
         recptr = XLogInsert(RM_HEAP2_ID, XLOG_HEAP2_NEW_CID);
     
         return recptr;
     }
    

12、RelationIsLogicallyLogged  
判断数据表是否正在可用于逻辑复制，如需要，则需要记录足够信息用于后续的日志解析

    
    
     /*
      * RelationIsLogicallyLogged
      *      True if we need to log enough information to extract the data from the
      *      WAL stream.
      *
      * We don't log information for unlogged tables (since they don't WAL log
      * anyway) and for system tables (their content is hard to make sense of, and
      * it would complicate decoding slightly for little gain). Note that we *do*
      * log information for user defined catalog tables since they presumably are
      * interesting to the user...
      */
     #define RelationIsLogicallyLogged(relation) \
         (XLogLogicalInfoActive() && \
          RelationNeedsWAL(relation) && \
          !IsCatalogRelation(relation))
     
    

13、XLog*

    
    
    XLogBeginInsert
    XLogRegisterData
    XLogRegisterBuffer
    XLogRegisterBufData
    XLogSetRecordFlags
    XLogInsert
    

14、PageSetLSN  
设置PageHeader的LSN（先前已解析）

    
    
     #define PageSetLSN(page, lsn) \
      PageXLogRecPtrSet(((PageHeader) (page))->pd_lsn, lsn)
    

15、END_CRIT_SECTION

    
    
     #define END_CRIT_SECTION() \
     do { \
         Assert(CritSectionCount > 0); \
         CritSectionCount--; \
     } while(0)
    

16、UnlockReleaseBuffer  
释放Buffer锁

    
    
     /*
      * UnlockReleaseBuffer -- release the content lock and pin on a buffer
      *
      * This is just a shorthand for a common combination.
      */
     void
     UnlockReleaseBuffer(Buffer buffer)
     {
         LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
         ReleaseBuffer(buffer);
     }
    

17、ReleaseBuffer  
Unpin Buffer，意味着Buffer可Flush用于其他地方

    
    
     /*
      * ReleaseBuffer -- release the pin on a buffer
      */
     void
     ReleaseBuffer(Buffer buffer)
     {
         if (!BufferIsValid(buffer))
             elog(ERROR, "bad buffer ID: %d", buffer);
     
         if (BufferIsLocal(buffer))
         {
             ResourceOwnerForgetBuffer(CurrentResourceOwner, buffer);
              Assert(LocalRefCount[-buffer - 1] > 0);
             LocalRefCount[-buffer - 1]--;
             return;
         }
          UnpinBuffer(GetBufferDescriptor(buffer - 1), true);
     }
    

18、CacheInvalidateHeapTuple  
缓存那些已“无用”的Tuple，比如Update操作的原记录，Delete操作的原记录等。

    
    
     /*
      * CacheInvalidateHeapTuple
      *      Register the given tuple for invalidation at end of command
      *      (ie, current command is creating or outdating this tuple).
      *      Also, detect whether a relcache invalidation is implied.
      *
      * For an insert or delete, tuple is the target tuple and newtuple is NULL.
      * For an update, we are called just once, with tuple being the old tuple
      * version and newtuple the new version.  This allows avoidance of duplicate
      * effort during an update.
      */
     void
     CacheInvalidateHeapTuple(Relation relation,
                              HeapTuple tuple,
                              HeapTuple newtuple)
     {
         Oid         tupleRelId;
         Oid         databaseId;
         Oid         relationId;
     
         /* Do nothing during bootstrap */
         if (IsBootstrapProcessingMode())
             return;
     
         /*
          * We only need to worry about invalidation for tuples that are in system
          * catalogs; user-relation tuples are never in catcaches and can't affect
          * the relcache either.
          */
         if (!IsCatalogRelation(relation))
             return;
     
         /*
          * IsCatalogRelation() will return true for TOAST tables of system
          * catalogs, but we don't care about those, either.
          */
         if (IsToastRelation(relation))
             return;
     
         /*
          * If we're not prepared to queue invalidation messages for this
          * subtransaction level, get ready now.
          */
         PrepareInvalidationState();
     
         /*
          * First let the catcache do its thing
          */
         tupleRelId = RelationGetRelid(relation);
         if (RelationInvalidatesSnapshotsOnly(tupleRelId))
         {
             databaseId = IsSharedRelation(tupleRelId) ? InvalidOid : MyDatabaseId;
             RegisterSnapshotInvalidation(databaseId, tupleRelId);
         }
         else
             PrepareToInvalidateCacheTuple(relation, tuple, newtuple,
                                           RegisterCatcacheInvalidation);
     
         /*
          * Now, is this tuple one of the primary definers of a relcache entry? See
          * comments in file header for deeper explanation.
          *
          * Note we ignore newtuple here; we assume an update cannot move a tuple
          * from being part of one relcache entry to being part of another.
          */
         if (tupleRelId == RelationRelationId)
         {
             Form_pg_class classtup = (Form_pg_class) GETSTRUCT(tuple);
     
             relationId = HeapTupleGetOid(tuple);
             if (classtup->relisshared)
                 databaseId = InvalidOid;
             else
                 databaseId = MyDatabaseId;
         }
         else if (tupleRelId == AttributeRelationId)
         {
             Form_pg_attribute atttup = (Form_pg_attribute) GETSTRUCT(tuple);
     
             relationId = atttup->attrelid;
     
             /*
              * KLUGE ALERT: we always send the relcache event with MyDatabaseId,
              * even if the rel in question is shared (which we can't easily tell).
              * This essentially means that only backends in this same database
              * will react to the relcache flush request.  This is in fact
              * appropriate, since only those backends could see our pg_attribute
              * change anyway.  It looks a bit ugly though.  (In practice, shared
              * relations can't have schema changes after bootstrap, so we should
              * never come here for a shared rel anyway.)
              */
             databaseId = MyDatabaseId;
         }
         else if (tupleRelId == IndexRelationId)
         {
             Form_pg_index indextup = (Form_pg_index) GETSTRUCT(tuple);
     
             /*
              * When a pg_index row is updated, we should send out a relcache inval
              * for the index relation.  As above, we don't know the shared status
              * of the index, but in practice it doesn't matter since indexes of
              * shared catalogs can't have such updates.
              */
             relationId = indextup->indexrelid;
             databaseId = MyDatabaseId;
         }
         else
             return;
     
         /*
          * Yes.  We need to register a relcache invalidation event.
          */
         RegisterRelcacheInvalidation(databaseId, relationId);
     }
    

19、heap_freetuple  
释放内存

    
    
     /*
      * heap_freetuple
      */
     void
     heap_freetuple(HeapTuple htup)
     {
         pfree(htup);
     }
    

### 二、源码解读

_heap_insert函数本身不复杂，最为复杂的子函数RelationGetBufferForTuple已在上一小节解析_

    
    
    /*
    输入：
        relation-数据表结构体
        tup-Heap Tuple数据（包括头部数据等），亦即数据行
        cid-命令ID（顺序）
        options-选项
        bistate-BulkInsert状态
    输出：
        Oid-数据表Oid
    */
    Oid
    heap_insert(Relation relation, HeapTuple tup, CommandId cid,
                int options, BulkInsertState bistate)
    {
        TransactionId xid = GetCurrentTransactionId();//事务id
        HeapTuple   heaptup;//Heap Tuple数据，亦即数据行
        Buffer      buffer;//数据缓存块
        Buffer      vmbuffer = InvalidBuffer;//vm缓冲块
        bool        all_visible_cleared = false;//标记
    
        /*
         * Fill in tuple header fields, assign an OID, and toast the tuple if
         * necessary.
         *
         * Note: below this point, heaptup is the data we actually intend to store
         * into the relation; tup is the caller's original untoasted data.
         */
        //插入前准备工作，比如设置t_infomask标记等
        heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
    
        /*
         * Find buffer to insert this tuple into.  If the page is all visible,
         * this will also pin the requisite visibility map page.
         */
        //获取相应的buffer，详见上面的子函数解析
        buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
                                           InvalidBuffer, options, bistate,
                                           &vmbuffer, NULL);
    
        /*
         * We're about to do the actual insert -- but check for conflict first, to
         * avoid possibly having to roll back work we've just done.
         *
         * This is safe without a recheck as long as there is no possibility of
         * another process scanning the page between this check and the insert
         * being visible to the scan (i.e., an exclusive buffer content lock is
         * continuously held from this point until the tuple insert is visible).
         *
         * For a heap insert, we only need to check for table-level SSI locks. Our
         * new tuple can't possibly conflict with existing tuple locks, and heap
         * page locks are only consolidated versions of tuple locks; they do not
         * lock "gaps" as index page locks do.  So we don't need to specify a
         * buffer when making the call, which makes for a faster check.
         */
        //检查序列化是否冲突
        CheckForSerializableConflictIn(relation, NULL, InvalidBuffer);
    
        /* NO EREPORT(ERROR) from here till changes are logged */
        //开始，变量+1
        START_CRIT_SECTION();
        //插入数据（详见上一节对该函数的解析）
        RelationPutHeapTuple(relation, buffer, heaptup,
                             (options & HEAP_INSERT_SPECULATIVE) != 0);
        //如Page is All Visible
        if (PageIsAllVisible(BufferGetPage(buffer)))
        {
            //复位
            all_visible_cleared = true;
            PageClearAllVisible(BufferGetPage(buffer));
            visibilitymap_clear(relation,
                                ItemPointerGetBlockNumber(&(heaptup->t_self)),
                                vmbuffer, VISIBILITYMAP_VALID_BITS);
        }
    
        /*
         * XXX Should we set PageSetPrunable on this page ?
         *
         * The inserting transaction may eventually abort thus making this tuple
         * DEAD and hence available for pruning. Though we don't want to optimize
         * for aborts, if no other tuple in this page is UPDATEd/DELETEd, the
         * aborted tuple will never be pruned until next vacuum is triggered.
         *
         * If you do add PageSetPrunable here, add it in heap_xlog_insert too.
         */
        //设置缓冲块为脏块
        MarkBufferDirty(buffer);
    
        /* XLOG stuff */
        //记录日志
        if (!(options & HEAP_INSERT_SKIP_WAL) && RelationNeedsWAL(relation))
        {
            xl_heap_insert xlrec;
            xl_heap_header xlhdr;
            XLogRecPtr  recptr;
            Page        page = BufferGetPage(buffer);
            uint8       info = XLOG_HEAP_INSERT;
            int         bufflags = 0;
    
            /*
             * If this is a catalog, we need to transmit combocids to properly
             * decode, so log that as well.
             */
            if (RelationIsAccessibleInLogicalDecoding(relation))
                log_heap_new_cid(relation, heaptup);
    
            /*
             * If this is the single and first tuple on page, we can reinit the
             * page instead of restoring the whole thing.  Set flag, and hide
             * buffer references from XLogInsert.
             */
            if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
                PageGetMaxOffsetNumber(page) == FirstOffsetNumber)
            {
                info |= XLOG_HEAP_INIT_PAGE;
                bufflags |= REGBUF_WILL_INIT;
            }
    
            xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
            xlrec.flags = 0;
            if (all_visible_cleared)
                xlrec.flags |= XLH_INSERT_ALL_VISIBLE_CLEARED;
            if (options & HEAP_INSERT_SPECULATIVE)
                xlrec.flags |= XLH_INSERT_IS_SPECULATIVE;
            Assert(ItemPointerGetBlockNumber(&heaptup->t_self) == BufferGetBlockNumber(buffer));
    
            /*
             * For logical decoding, we need the tuple even if we're doing a full
             * page write, so make sure it's included even if we take a full-page
             * image. (XXX We could alternatively store a pointer into the FPW).
             */
            if (RelationIsLogicallyLogged(relation))
            {
                xlrec.flags |= XLH_INSERT_CONTAINS_NEW_TUPLE;
                bufflags |= REGBUF_KEEP_DATA;
            }
    
            XLogBeginInsert();
            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
    
            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
            xlhdr.t_infomask = heaptup->t_data->t_infomask;
            xlhdr.t_hoff = heaptup->t_data->t_hoff;
    
            /*
             * note we mark xlhdr as belonging to buffer; if XLogInsert decides to
             * write the whole page to the xlog, we don't need to store
             * xl_heap_header in the xlog.
             */
            XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
            /* PG73FORMAT: write bitmap [+ padding] [+ oid] + data */
            XLogRegisterBufData(0,
                                (char *) heaptup->t_data + SizeofHeapTupleHeader,
                                heaptup->t_len - SizeofHeapTupleHeader);
    
            /* filtering by origin on a row level is much more efficient */
            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
    
            recptr = XLogInsert(RM_HEAP_ID, info);
    
            PageSetLSN(page, recptr);
        }
        //完成！
        END_CRIT_SECTION();
        //解锁Buffer，包括vm buffer
        UnlockReleaseBuffer(buffer);
        if (vmbuffer != InvalidBuffer)
            ReleaseBuffer(vmbuffer);
    
        /*
         * If tuple is cachable, mark it for invalidation from the caches in case
         * we abort.  Note it is OK to do this after releasing the buffer, because
         * the heaptup data structure is all in local memory, not in the shared
         * buffer.
         */
        //缓存操作后变“无效”的Tuple
        CacheInvalidateHeapTuple(relation, heaptup, NULL);
    
        /* Note: speculative insertions are counted too, even if aborted later */
        //更新统计信息
        pgstat_count_heap_insert(relation, 1);
    
        /*
         * If heaptup is a private copy, release it.  Don't forget to copy t_self
         * back to the caller's image, too.
         */
        if (heaptup != tup)
        {
            tup->t_self = heaptup->t_self;
            heap_freetuple(heaptup);
        }
    
        return HeapTupleGetOid(tup);
    }
    
    

### 三、跟踪分析

插入一条记录，使用gdb进行跟踪分析：

    
    
    -- 这次启动事务
    testdb=# begin;
    BEGIN
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1556
    (1 row)
    testdb=# insert into t_insert values(11,'11','11','11');
    （挂起）
    
    

启动gdb：

    
    
    [root@localhost ~]# gdb -p 1556
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b heap_insert
    Breakpoint 1 at 0x4c343c: file heapam.c, line 2444.
    #输入参数：
    (gdb) p *relation
    $1 = {rd_node = {spcNode = 1663, dbNode = 16477, relNode = 26731}, rd_smgr = 0x0, rd_refcnt = 1, rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, 
      rd_indexvalid = 0 '\000', rd_statvalid = false, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f5fdd1771f0, rd_att = 0x7f5fdd177300, rd_id = 26731, rd_lockInfo = {lockRelId = {
          relId = 26731, dbId = 16477}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, rd_replidindex = 0, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, 
      rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, 
      rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, rd_exclstrats = 0x0, 
      rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x146f9b8}
    (gdb) p *tup
    $2 = {t_len = 61, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 26731, t_data = 0x14b19f8}
    (gdb) p *(tup->t_data)
    $3 = {t_choice = {t_heap = {t_xmin = 244, t_xmax = 4294967295, t_field3 = {t_cid = 2249, t_xvac = 2249}}, t_datum = {datum_len_ = 244, datum_typmod = -1, datum_typeid = 2249}}, t_ctid = {ip_blkid = {
          bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_infomask2 = 4, t_infomask = 2, t_hoff = 24 '\030', t_bits = 0x14b1a0f ""}
    (gdb) p *(tup->t_data->t_bits)
    $4 = 0 '\000'
    (gdb) p cid
    $5 = 0
    (gdb) p options
    $6 = 0
    (gdb) p bistate
    $7 = (BulkInsertState) 0x0
    (gdb) next
    2447        Buffer      vmbuffer = InvalidBuffer;
    (gdb) p xid
    $8 = 1612859
    (gdb) next
    2448        bool        all_visible_cleared = false;
    (gdb) 
    2457        heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
    (gdb) 
    2463        buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
    (gdb) p *heaptup
    $9 = {t_len = 61, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 26731, t_data = 0x14b19f8}
    (gdb) next
    2482        CheckForSerializableConflictIn(relation, NULL, InvalidBuffer);
    (gdb) p buffer
    $10 = 185
    (gdb) next
    2485        START_CRIT_SECTION();
    (gdb) 
    2488                             (options & HEAP_INSERT_SPECULATIVE) != 0);
    (gdb) 
    2487        RelationPutHeapTuple(relation, buffer, heaptup,
    (gdb) 
    2490        if (PageIsAllVisible(BufferGetPage(buffer)))
    (gdb) 
    2510        MarkBufferDirty(buffer);
    (gdb) p buffer
    $11 = 185
    (gdb) next
    2513        if (!(options & HEAP_INSERT_SKIP_WAL) && RelationNeedsWAL(relation))
    (gdb) 
    2518            Page        page = BufferGetPage(buffer);
    (gdb) 
    2519            uint8       info = XLOG_HEAP_INSERT;
    (gdb) p *page
    $12 = 1 '\001'
    (gdb) p *(PageHeader)page
    $13 = {pd_lsn = {xlogid = 1, xrecoff = 3677481952}, pd_checksum = 0, pd_flags = 0, pd_lower = 64, pd_upper = 7552, pd_special = 8192, pd_pagesize_version = 8196, pd_prune_xid = 0, 
      pd_linp = 0x7f5fc5409318}
    (gdb) next
    2520            int         bufflags = 0;
    (gdb) 
    2526            if (RelationIsAccessibleInLogicalDecoding(relation))
    (gdb) 
    2534            if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
    (gdb) 
    2541            xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
    (gdb) 
    2542            xlrec.flags = 0;
    (gdb) 
    2543            if (all_visible_cleared)
    (gdb) 
    2545            if (options & HEAP_INSERT_SPECULATIVE)
    (gdb) 
    2554            if (RelationIsLogicallyLogged(relation))
    (gdb) 
    2560            XLogBeginInsert();
    (gdb) 
    2561            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
    (gdb) p xlrec
    $14 = {offnum = 10, flags = 0 '\000'}
    (gdb) next
    2563            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
    (gdb) 
    2564            xlhdr.t_infomask = heaptup->t_data->t_infomask;
    (gdb) 
    2565            xlhdr.t_hoff = heaptup->t_data->t_hoff;
    (gdb) 
    2572            XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
    (gdb) 
    2573            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
    (gdb) 
    2577                                heaptup->t_len - SizeofHeapTupleHeader);
    (gdb) 
    2575            XLogRegisterBufData(0,
    (gdb) 
    2576                                (char *) heaptup->t_data + SizeofHeapTupleHeader,
    (gdb) 
    2575            XLogRegisterBufData(0,
    (gdb) 
    2580            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
    (gdb) 
    2582            recptr = XLogInsert(RM_HEAP_ID, info);
    (gdb) 
    2584            PageSetLSN(page, recptr);
    (gdb) 
    2587        END_CRIT_SECTION();
    (gdb) 
    2589        UnlockReleaseBuffer(buffer);
    (gdb) 
    2590        if (vmbuffer != InvalidBuffer)
    (gdb) 
    2599        CacheInvalidateHeapTuple(relation, heaptup, NULL);
    (gdb) 
    2602        pgstat_count_heap_insert(relation, 1);
    (gdb) 
    2608        if (heaptup != tup)
    (gdb) 
    2614        return HeapTupleGetOid(tup);
    (gdb) p *tup
    $15 = {t_len = 61, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 10}, t_tableOid = 26731, t_data = 0x14b19f8}
    (gdb) p *(tup->t_data)
    $16 = {t_choice = {t_heap = {t_xmin = 1612859, t_xmax = 0, t_field3 = {t_cid = 0, t_xvac = 0}}, t_datum = {datum_len_ = 1612859, datum_typmod = 0, datum_typeid = 0}}, t_ctid = {ip_blkid = {
          bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_infomask2 = 4, t_infomask = 2050, t_hoff = 24 '\030', t_bits = 0x14b1a0f ""}
    (gdb)
    (gdb) n
    2615    }
    (gdb) n
    #done!
    ExecInsert (mtstate=0x14b0c10, slot=0x14b1250, planSlot=0x14b1250, estate=0x14b08c0, canSetTag=true) at nodeModifyTable.c:534
    534             if (resultRelInfo->ri_NumIndices > 0) 
    

### 四、小结

1、简单的反面是复杂：插入一行数据，涉及缓冲区管理（在PG中还需要考虑死锁）、日志处理等一系列的细节，原理/理论是简单的，但要在工程上实现得漂亮，不容易！程序猿们，加油吧！  
2、NoSQL是“简单”的，RDBMS是“复杂”的：NoSQL不需要考虑事务，简化了日志处理，实现逻辑相对简单；RDBMS需要考虑A/B/C/D...，权衡了各种利弊，值得深入学习。

