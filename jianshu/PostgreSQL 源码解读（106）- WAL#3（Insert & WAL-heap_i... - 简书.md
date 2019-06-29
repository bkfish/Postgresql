本节介绍了插入数据时与WAL相关的处理逻辑，主要是heap_insert->XLogInsert函数。

### 一、数据结构

**静态变量**  
进程中全局共享

    
    
    /*
     * An array of XLogRecData structs, to hold registered data.
     * XLogRecData结构体数组,存储已注册的数据
     */
    static XLogRecData *rdatas;
    //已使用的入口
    static int  num_rdatas;         /* entries currently used */
    //已分配的空间大小
    static int  max_rdatas;         /* allocated size */
    //是否调用XLogBeginInsert函数
    static bool begininsert_called = false;
    
    

**宏定义**

    
    
    typedef char* Pointer;//指针
    typedef Pointer Page;//Page
    
    #define XLOG_HEAP_INSERT   0x00
    
    /*
     * Pointer to a location in the XLOG.  These pointers are 64 bits wide,
     * because we don't want them ever to overflow.
     * 指向XLOG中的位置.
     * 这些指针大小为64bit,以确保指针不会溢出.
     */
    typedef uint64 XLogRecPtr;
    
    
    /*
     * Additional macros for access to page headers. (Beware multiple evaluation
     * of the arguments!)
     */
    #define PageGetLSN(page) \
        PageXLogRecPtrGet(((PageHeader) (page))->pd_lsn)
    #define PageSetLSN(page, lsn) \
        PageXLogRecPtrSet(((PageHeader) (page))->pd_lsn, lsn)
    
    /* Buffer size required to store a compressed version of backup block image */
    //存储压缩会后的块镜像所需要的缓存空间大小
    #define PGLZ_MAX_BLCKSZ PGLZ_MAX_OUTPUT(BLCKSZ)
    
    /*
     * Fake spinlock implementation using semaphores --- slow and prone
     * to fall foul of kernel limits on number of semaphores, so don't use this
     * unless you must!  The subroutines appear in spin.c.
     * 使用信号量的伪自旋锁实现——很慢而且容易与内核对信号量的限制相冲突，
     *   所以除非必须，否则不要使用它!
     *   相关的子例程出现在spin.c中。
     */
    typedef int slock_t;
    
    

**XLogCtl**  
XLOG的所有共享内存状态信息

    
    
    /*
     * Total shared-memory state for XLOG.
     * XLOG的所有共享内存状态信息
     */
    typedef struct XLogCtlData
    {
        XLogCtlInsert Insert;//插入控制器
    
        /* Protected by info_lck: */
        //------ 通过info_lck锁保护
        XLogwrtRqst LogwrtRqst;
        //Insert->RedoRecPtr最近的拷贝
        XLogRecPtr  RedoRecPtr;     /* a recent copy of Insert->RedoRecPtr */
        //最后的checkpoint的nextXID & epoch
        uint32      ckptXidEpoch;   /* nextXID & epoch of latest checkpoint */
        TransactionId ckptXid;
        //最新异步提交/回滚的LSN
        XLogRecPtr  asyncXactLSN;   /* LSN of newest async commit/abort */
        //slot需要的最"老"的LSN
        XLogRecPtr  replicationSlotMinLSN;  /* oldest LSN needed by any slot */
        //最后移除/回收的XLOG段
        XLogSegNo   lastRemovedSegNo;   /* latest removed/recycled XLOG segment */
    
        /* Fake LSN counter, for unlogged relations. Protected by ulsn_lck. */
        //---- "伪装"的LSN计数器,用于不需要记录日志的关系.通过ulsn_lck锁保护
        XLogRecPtr  unloggedLSN;
        slock_t     ulsn_lck;
    
        /* Time and LSN of last xlog segment switch. Protected by WALWriteLock. */
        //---- 切换后最新的xlog段的时间线和LSN,通过WALWriteLock锁保护
        pg_time_t   lastSegSwitchTime;
        XLogRecPtr  lastSegSwitchLSN;
    
        /*
         * Protected by info_lck and WALWriteLock (you must hold either lock to
         * read it, but both to update)
         * 通过info_lck和WALWriteLock保护
         * (必须持有其中之一才能读取,必须全部持有才能更新)
         */
        XLogwrtResult LogwrtResult;
    
        /*
         * Latest initialized page in the cache (last byte position + 1).
         * 在缓存中最后初始化的page(最后一个字节位置 + 1)
         * 
         * To change the identity of a buffer (and InitializedUpTo), you need to
         * hold WALBufMappingLock.  To change the identity of a buffer that's
         * still dirty, the old page needs to be written out first, and for that
         * you need WALWriteLock, and you need to ensure that there are no
         * in-progress insertions to the page by calling
         * WaitXLogInsertionsToFinish().
         * 如需改变缓冲区的标识(以及InitializedUpTo),需要持有WALBufMappingLock锁.
         * 改变标记为dirty的缓冲区的标识符,旧的page需要先行写出,因此必须持有WALWriteLock锁,
         *   而且必须确保没有正在通过调用WaitXLogInsertionsToFinish()进行执行中的插入page操作
         */
        XLogRecPtr  InitializedUpTo;
    
        /*
         * These values do not change after startup, although the pointed-to pages
         * and xlblocks values certainly do.  xlblock values are protected by
         * WALBufMappingLock.
         * 在启动后这些值不会修改,虽然pointed-to pages和xlblocks值通常会更改.
         * xlblock的值通过WALBufMappingLock锁保护.
         */
        //未写入的XLOG pages的缓存
        char       *pages;          /* buffers for unwritten XLOG pages */
        //ptr-s的第一个字节 + XLOG_BLCKSZ
        XLogRecPtr *xlblocks;       /* 1st byte ptr-s + XLOG_BLCKSZ */
        //已分配的xlog缓冲的索引最高值
        int         XLogCacheBlck;  /* highest allocated xlog buffer index */
    
        /*
         * Shared copy of ThisTimeLineID. Does not change after end-of-recovery.
         * If we created a new timeline when the system was started up,
         * PrevTimeLineID is the old timeline's ID that we forked off from.
         * Otherwise it's equal to ThisTimeLineID.
         * ThisTimeLineID的共享拷贝.
         * 在完成恢复后不要修改.
         * 如果在系统启动后创建了一个新的时间线,PrevTimeLineID是从旧时间线分叉的ID.
         * 否则,PrevTimeLineID = ThisTimeLineID
         */
        TimeLineID  ThisTimeLineID;
        TimeLineID  PrevTimeLineID;
    
        /*
         * SharedRecoveryInProgress indicates if we're still in crash or archive
         * recovery.  Protected by info_lck.
         * SharedRecoveryInProgress标记是否处于宕机或者归档恢复中,通过info_lck锁保护.
         */
        bool        SharedRecoveryInProgress;
    
        /*
         * SharedHotStandbyActive indicates if we're still in crash or archive
         * recovery.  Protected by info_lck.
         * SharedHotStandbyActive标记是否处于宕机或者归档恢复中,通过info_lck锁保护.
         */
        bool        SharedHotStandbyActive;
    
        /*
         * WalWriterSleeping indicates whether the WAL writer is currently in
         * low-power mode (and hence should be nudged if an async commit occurs).
         * Protected by info_lck.
         * WalWriterSleeping标记WAL writer进程是否处于"节能"模式
         * (因此，如果发生异步提交，应该对其进行微操作).
         * 通过info_lck锁保护.
         */
        bool        WalWriterSleeping;
    
        /*
         * recoveryWakeupLatch is used to wake up the startup process to continue
         * WAL replay, if it is waiting for WAL to arrive or failover trigger file
         * to appear.
         * recoveryWakeupLatch等待WAL arrive或者failover触发文件出现,
         *   如出现则唤醒启动进程继续执行WAL回放.
         * 
         */
        Latch       recoveryWakeupLatch;
    
        /*
         * During recovery, we keep a copy of the latest checkpoint record here.
         * lastCheckPointRecPtr points to start of checkpoint record and
         * lastCheckPointEndPtr points to end+1 of checkpoint record.  Used by the
         * checkpointer when it wants to create a restartpoint.
         * 在恢复期间,我们保存最后检查点记录的一个拷贝在这里.
         * lastCheckPointRecPtr指向检查点的起始位置
         * lastCheckPointEndPtr指向执行检查点的结束点+1位置
         * 在checkpointer进程希望创建一个重新启动的点时使用.
         *
         * Protected by info_lck.
         * 使用info_lck锁保护.
         */
        XLogRecPtr  lastCheckPointRecPtr;
        XLogRecPtr  lastCheckPointEndPtr;
        CheckPoint  lastCheckPoint;
    
        /*
         * lastReplayedEndRecPtr points to end+1 of the last record successfully
         * replayed. When we're currently replaying a record, ie. in a redo
         * function, replayEndRecPtr points to the end+1 of the record being
         * replayed, otherwise it's equal to lastReplayedEndRecPtr.
         * lastReplayedEndRecPtr指向最后一个成功回放的记录的结束点 + 1的位置.
         * 如果正处于redo函数回放记录期间,那么replayEndRecPtr指向正在恢复的记录的结束点 + 1的位置,
         * 否则replayEndRecPtr = lastReplayedEndRecPtr
         */
        XLogRecPtr  lastReplayedEndRecPtr;
        TimeLineID  lastReplayedTLI;
        XLogRecPtr  replayEndRecPtr;
        TimeLineID  replayEndTLI;
        /* timestamp of last COMMIT/ABORT record replayed (or being replayed) */
        //最后的COMMIT/ABORT回放(或正在回放)记录的时间戳
        TimestampTz recoveryLastXTime;
    
        /*
         * timestamp of when we started replaying the current chunk of WAL data,
         * only relevant for replication or archive recovery
         * 我们开始回放当前的WAL chunk的时间戳(仅与复制或存档恢复相关)
         */
        TimestampTz currentChunkStartTime;
        /* Are we requested to pause recovery? */
        //是否请求暂停恢复
        bool        recoveryPause;
    
        /*
         * lastFpwDisableRecPtr points to the start of the last replayed
         * XLOG_FPW_CHANGE record that instructs full_page_writes is disabled.
         * lastFpwDisableRecPtr指向最后已回放的XLOG_FPW_CHANGE记录(禁用对整个页面的写指令)的起始点.
         */
        XLogRecPtr  lastFpwDisableRecPtr;
        //锁结构
        slock_t     info_lck;       /* locks shared variables shown above */
    } XLogCtlData;
    
    static XLogCtlData *XLogCtl = NULL;
    
    

### 二、源码解读

**heap_insert**  
主要实现逻辑是插入元组到堆中,其中存在对WAL(XLog)进行处理的部分.  
参见[PostgreSQL 源码解读（104）- WAL#1（Insert & WAL-heap_insert函数#1）](https://www.jianshu.com/p/70f019643e76)

**XLogInsert**  
插入一个具有指定的RMID和info字节的XLOG记录，该记录的主体是先前通过XLogRegister*调用注册的数据和缓冲区引用。

    
    
    /*
     * Insert an XLOG record having the specified RMID and info bytes, with the
     * body of the record being the data and buffer references registered earlier
     * with XLogRegister* calls.
     * 插入一个具有指定的RMID和info字节的XLOG记录，
     *   该记录的主体是先前通过XLogRegister*调用注册的数据和缓冲区引用。
     *
     * Returns XLOG pointer to end of record (beginning of next record).
     * This can be used as LSN for data pages affected by the logged action.
     * (LSN is the XLOG point up to which the XLOG must be flushed to disk
     * before the data page can be written out.  This implements the basic
     * WAL rule "write the log before the data".)
     * 返回XLOG指针到记录的结束点(下一条记录的开始)。
     * 这可以用作受日志操作影响的数据页的LSN。
     * (LSN是必须将XLOG刷新到磁盘才能写出数据页的XLOG点。
     *  这实现了基本的WAL规则:“在数据之前写日志”。)
     */
    XLogRecPtr
    XLogInsert(RmgrId rmid, uint8 info)
    {
        XLogRecPtr  EndPos;//uint64
    
        /* XLogBeginInsert() must have been called. */
        //在此前,XLogBeginInsert()必须已调用
        if (!begininsert_called)
            elog(ERROR, "XLogBeginInsert was not called");
    
        /*
         * The caller can set rmgr bits, XLR_SPECIAL_REL_UPDATE and
         * XLR_CHECK_CONSISTENCY; the rest are reserved for use by me.
         * 调用方必须设置rmgr位:XLR_SPECIAL_REL_UPDATE & XLR_CHECK_CONSISTENCY.
         * 其余在这里保留使用
         */
        if ((info & ~(XLR_RMGR_INFO_MASK |
                      XLR_SPECIAL_REL_UPDATE |
                      XLR_CHECK_CONSISTENCY)) != 0)
            elog(PANIC, "invalid xlog info mask %02X", info);
    
        TRACE_POSTGRESQL_WAL_INSERT(rmid, info);
    
        /*
         * In bootstrap mode, we don't actually log anything but XLOG resources;
         * return a phony record pointer.
         * 在bootstrap模式,除了XLOG资源外,不需要实际记录内容.
         * 返回一个伪记录指针.
         */
        if (IsBootstrapProcessingMode() && rmid != RM_XLOG_ID)
        {
            XLogResetInsertion();
            EndPos = SizeOfXLogLongPHD; /* 返回伪记录指针;start of 1st chkpt record */
            return EndPos;
        }
    
        do
        {
            //循环
            XLogRecPtr  RedoRecPtr;
            bool        doPageWrites;
            XLogRecPtr  fpw_lsn;
            XLogRecData *rdt;
    
            /*
             * Get values needed to decide whether to do full-page writes. Since
             * we don't yet have an insertion lock, these could change under us,
             * but XLogInsertRecord will recheck them once it has a lock.
             * 获取决定是否执行全页写入所需的值。
             * 由于我们还没有插入锁，所以这些可能会在我们的操作期间被更改，
             *   但是XLogInsertRecord一旦有了锁，就会重新检查它们。
             */
            GetFullPageWriteInfo(&RedoRecPtr, &doPageWrites);
    
            rdt = XLogRecordAssemble(rmid, info, RedoRecPtr, doPageWrites,
                                     &fpw_lsn);
            //curinsert_flags类型为uint8
            EndPos = XLogInsertRecord(rdt, fpw_lsn, curinsert_flags);
        } while (EndPos == InvalidXLogRecPtr);
    
        XLogResetInsertion();
    
        return EndPos;
    }
    
    

**XLogInsertRecord**  
插入一个由已经构造的数据chunks链表示的XLOG记录。

    
    
    /*
     * Insert an XLOG record represented by an already-constructed chain of data
     * chunks.  This is a low-level routine; to construct the WAL record header
     * and data, use the higher-level routines in xloginsert.c.
     * 插入一个由已经构造的数据chunks链表示的XLOG记录。
     * 这是一个比较底层的处理逻辑实现,
     *   使用xloginsert.c中高层的子程序构造WAL记录的头部和数据
     *
     * If 'fpw_lsn' is valid, it is the oldest LSN among the pages that this
     * WAL record applies to, that were not included in the record as full page
     * images.  If fpw_lsn <= RedoRecPtr, the function does not perform the
     * insertion and returns InvalidXLogRecPtr.  The caller can then recalculate
     * which pages need a full-page image, and retry.  If fpw_lsn is invalid, the
     * record is always inserted.
     * 如"fpw_lsn"是有效的,那么该值为在所有的WAL记录应用到pages中最小的LSN,
     *   但该值不包括全页镜像的记录.
     * 如fpw_lsn <= RedoRecPtr,该函数不会执行插入同时会返回InvalidXLogRecPtr.
     * 调用者可以重新计算哪些pages需要full-page image以及记录入口.
     * 如果fpw_lsn无效,那么记录已被插入.
     *
     * 'flags' gives more in-depth control on the record being inserted. See
     * XLogSetRecordFlags() for details.
     * "flags"在即将插入的记录上给定了更多的深层次的控制.
     * 查看函数XLogSetRecordFlags()获取更多的细节信息.
     *
     * The first XLogRecData in the chain must be for the record header, and its
     * data must be MAXALIGNed.  XLogInsertRecord fills in the xl_prev and
     * xl_crc fields in the header, the rest of the header must already be filled
     * by the caller.
     * 链中的第一个XLogRecData必须是吉林的头部,数据必须已被MAXALIGNed.
     * XLogInsertRecord填充在头部的xl_prev和xl_crc域中,
     *   头部的其他域已通过调用者提供.
     *
     * Returns XLOG pointer to end of record (beginning of next record).
     * This can be used as LSN for data pages affected by the logged action.
     * (LSN is the XLOG point up to which the XLOG must be flushed to disk
     * before the data page can be written out.  This implements the basic
     * WAL rule "write the log before the data".)
     * 返回XLOG指针,指向记录结束的位置(下一记录的起始点).
     * 这可以用作受日志操作影响的数据页的LSN。
     * (LSN是必须将XLOG刷新到磁盘上才能写出数据页的XLOG点。
     *  这实现了WAL的基本规则"在写数据前写日志")
     */
    XLogRecPtr
    XLogInsertRecord(XLogRecData *rdata,
                     XLogRecPtr fpw_lsn,
                     uint8 flags)
    {
        XLogCtlInsert *Insert = &XLogCtl->Insert;//XLOG写入控制器
        pg_crc32c   rdata_crc;//uint32
        bool        inserted;
        XLogRecord *rechdr = (XLogRecord *) rdata->data;
        uint8       info = rechdr->xl_info & ~XLR_INFO_MASK;
        bool        isLogSwitch = (rechdr->xl_rmid == RM_XLOG_ID &&
                                   info == XLOG_SWITCH);
        XLogRecPtr  StartPos;
        XLogRecPtr  EndPos;
        bool        prevDoPageWrites = doPageWrites;
    
        /* we assume that all of the record header is in the first chunk */
        //假定所有的记录头部数据都处于第一个chunk中
        Assert(rdata->len >= SizeOfXLogRecord);
    
        /* cross-check on whether we should be here or not */
        //交叉检查
        if (!XLogInsertAllowed())
            elog(ERROR, "cannot make new WAL entries during recovery");
    
        /*----------         *
         * We have now done all the preparatory work we can without holding a
         * lock or modifying shared state. From here on, inserting the new WAL
         * record to the shared WAL buffer cache is a two-step process:
         * 现在，我们已经完成了所有的准备工作，无需持有锁或修改共享状态。
         * 从这里开始，将新的WAL记录插入到共享的WAL缓冲区缓存需要两个步骤:
         * 
         * 1. Reserve the right amount of space from the WAL. The current head of
         *    reserved space is kept in Insert->CurrBytePos, and is protected by
         *    insertpos_lck.
         * 1. 从WAL中预留合适的空间.预留空间的头部保存在Insert->CurrBytePos中,
         *    通过insertpos_lck锁保护
         *
         * 2. Copy the record to the reserved WAL space. This involves finding the
         *    correct WAL buffer containing the reserved space, and copying the
         *    record in place. This can be done concurrently in multiple processes.
         * 2. 拷贝记录到保留的WAL空间中.这会涉及到寻找持有保留空间的正确的WAL缓冲区,
         *      以及拷贝记录到合适的位置上.
         *    在多进程间必须同步完成.
         *
         * To keep track of which insertions are still in-progress, each concurrent
         * inserter acquires an insertion lock. In addition to just indicating that
         * an insertion is in progress, the lock tells others how far the inserter
         * has progressed. There is a small fixed number of insertion locks,
         * determined by NUM_XLOGINSERT_LOCKS. When an inserter crosses a page
         * boundary, it updates the value stored in the lock to the how far it has
         * inserted, to allow the previous buffer to be flushed.
         * 为了跟踪那个插入操作仍处于进行当中,每一个当前的插入器需要insertion锁.
         * 除了用于标识那个insertion处于进行当中,锁同时会告知其他插入器可以处理的边界界限.
         * 系统有少数几个固定数量的insertion所,通过参数NUM_XLOGINSERT_LOCKS定义.
         * 如果某个插入器跨越了page的边界,该插入器会更新存储在锁中的值以表示它已插入的大小,
         *   这样方便刷新先前的缓存.
         *
         * Holding onto an insertion lock also protects RedoRecPtr and
         * fullPageWrites from changing until the insertion is finished.
         * 持有插入锁还可以保护RedoRecPtr和fullpagewrite在插入完成之前不受更改。
         * 
         * Step 2 can usually be done completely in parallel. If the required WAL
         * page is not initialized yet, you have to grab WALBufMappingLock to
         * initialize it, but the WAL writer tries to do that ahead of insertions
         * to avoid that from happening in the critical path.
         * 步骤2通常可以完全并行完成。
         * 如果所需的WAL页面还没有初始化，您必须获取WALBufMappingLock来初始化它，
         *   但是WAL writer进程会在插入之前尝试这样做，以避免在关键路径中发生这种情况。
         *
         *----------         */
        START_CRIT_SECTION();
        if (isLogSwitch)
            WALInsertLockAcquireExclusive();
        else
            WALInsertLockAcquire();
    
        /*
         * Check to see if my copy of RedoRecPtr is out of date. If so, may have
         * to go back and have the caller recompute everything. This can only
         * happen just after a checkpoint, so it's better to be slow in this case
         * and fast otherwise.
         * 看看进程的RedoRecPtr是不是过期了。
         * 如果是，可能需要返回并让调用方重新计算所有内容。
         * 这只会在检查点之后才会发生，所以在这种情况下最好慢一点，否则最好快一点。
         * 
         * Also check to see if fullPageWrites or forcePageWrites was just turned
         * on; if we weren't already doing full-page writes then go back and
         * recompute.
         * 还要检查是否打开了fullpagewrite或forcepagewrite;
         *   如果我们还没有完成整页的写操作，那么返回并重新计算。
         *
         * If we aren't doing full-page writes then RedoRecPtr doesn't actually
         * affect the contents of the XLOG record, so we'll update our local copy
         * but not force a recomputation.  (If doPageWrites was just turned off,
         * we could recompute the record without full pages, but we choose not to
         * bother.)
         * 如果我们并没有在执行全页写操作，那么RedoRecPtr实际上不会影响XLOG记录的内容，
         *   因此我们将更新本地副本，但不会强制进行重新计算。
         * (如果doPageWrites关闭，可以在没有完整页面的情况下重新计算记录，但我们没有这种麻烦的做法。)
         * 
         */
        if (RedoRecPtr != Insert->RedoRecPtr)
        {
            Assert(RedoRecPtr < Insert->RedoRecPtr);
            RedoRecPtr = Insert->RedoRecPtr;
        }
        doPageWrites = (Insert->fullPageWrites || Insert->forcePageWrites);
    
        if (doPageWrites &&
            (!prevDoPageWrites ||
             (fpw_lsn != InvalidXLogRecPtr && fpw_lsn <= RedoRecPtr)))
        {
            /*
             * Oops, some buffer now needs to be backed up that the caller didn't
             * back up.  Start over.
             * 糟糕，现在需要备份一些调用者没有备份的缓冲区。
             * 让我们重新开始吧。
             */
            WALInsertLockRelease();
            END_CRIT_SECTION();
            return InvalidXLogRecPtr;
        }
    
        /*
         * Reserve space for the record in the WAL. This also sets the xl_prev
         * pointer.
         * 在WAL预留记录空间.同时会设置xl_prev指针.
         * 
         */
        if (isLogSwitch)
            inserted = ReserveXLogSwitch(&StartPos, &EndPos, &rechdr->xl_prev);
        else
        {
            ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos,
                                      &rechdr->xl_prev);
            inserted = true;
        }
    
        if (inserted)
        {
            /*
             * Now that xl_prev has been filled in, calculate CRC of the record
             * header.
             * 现在xl_prev指针已填充,计算记录头部的CRC
             */
            rdata_crc = rechdr->xl_crc;
            COMP_CRC32C(rdata_crc, rechdr, offsetof(XLogRecord, xl_crc));
            FIN_CRC32C(rdata_crc);
            rechdr->xl_crc = rdata_crc;
    
            /*
             * All the record data, including the header, is now ready to be
             * inserted. Copy the record in the space reserved.
             * 所有的记录数据,包括头部数据,准备插入!
             * 拷贝记录到保留空间中.
             */
            CopyXLogRecordToWAL(rechdr->xl_tot_len, isLogSwitch, rdata,
                                StartPos, EndPos);
    
            /*
             * Unless record is flagged as not important, update LSN of last
             * important record in the current slot. When holding all locks, just
             * update the first one.
             * 除非记录被标记为不重要，否则更新当前slot中最后一条重要记录的LSN。
             * 如持有所有锁，只需更新第一个。
             */
            if ((flags & XLOG_MARK_UNIMPORTANT) == 0)
            {
                int         lockno = holdingAllLocks ? 0 : MyLockNo;
    
                WALInsertLocks[lockno].l.lastImportantAt = StartPos;
            }
        }
        else
        {
            /*
             * This was an xlog-switch record, but the current insert location was
             * already exactly at the beginning of a segment, so there was no need
             * to do anything.
             * 这是一个xlog-switch记录，但是当前插入位置已经确切地位于段的开头，所以不需要做任何事情。
             */
        }
    
        /*
         * Done! Let others know that we're finished.
         * 全部完成!让其他插入器知道我们已经完成了!
         */
        WALInsertLockRelease();
    
        MarkCurrentTransactionIdLoggedIfAny();
    
        END_CRIT_SECTION();
    
        /*
         * Update shared LogwrtRqst.Write, if we crossed page boundary.
         * 如跨越了page边界,更新共享的LogwrtRqst.Write变量
         */
        if (StartPos / XLOG_BLCKSZ != EndPos / XLOG_BLCKSZ)
        {
            SpinLockAcquire(&XLogCtl->info_lck);
            /* advance global request to include new block(s) */
            //预先请求包含新块(s)
            if (XLogCtl->LogwrtRqst.Write < EndPos)
                XLogCtl->LogwrtRqst.Write = EndPos;
            /* update local result copy while I have the chance */
            //如有机会,更新本地的结果拷贝
            LogwrtResult = XLogCtl->LogwrtResult;
            SpinLockRelease(&XLogCtl->info_lck);
        }
    
        /*
         * If this was an XLOG_SWITCH record, flush the record and the empty
         * padding space that fills the rest of the segment, and perform
         * end-of-segment actions (eg, notifying archiver).
         * 如果这是一条XLOG_SWITCH记录，
         *   刷新记录和填充该段其余部分的空白填充空间，
         *   并执行段结束操作(例如，通知归档器)。
         */
        if (isLogSwitch)
        {
            TRACE_POSTGRESQL_WAL_SWITCH();
            XLogFlush(EndPos);
    
            /*
             * Even though we reserved the rest of the segment for us, which is
             * reflected in EndPos, we return a pointer to just the end of the
             * xlog-switch record.
             * 即使我们为自己保留了段的其余部分(这反映在EndPos中)，
             *   我们也只返回一个指向xlog-switch记录末尾的指针。
             */
            if (inserted)
            {
                EndPos = StartPos + SizeOfXLogRecord;
                if (StartPos / XLOG_BLCKSZ != EndPos / XLOG_BLCKSZ)
                {
                    uint64      offset = XLogSegmentOffset(EndPos, wal_segment_size);
    
                    if (offset == EndPos % XLOG_BLCKSZ)
                        EndPos += SizeOfXLogLongPHD;
                    else
                        EndPos += SizeOfXLogShortPHD;
                }
            }
        }
    
    #ifdef WAL_DEBUG//DEBUG代码
        if (XLOG_DEBUG)
        {
            static XLogReaderState *debug_reader = NULL;
            StringInfoData buf;
            StringInfoData recordBuf;
            char       *errormsg = NULL;
            MemoryContext oldCxt;
    
            oldCxt = MemoryContextSwitchTo(walDebugCxt);
    
            initStringInfo(&buf);
            appendStringInfo(&buf, "INSERT @ %X/%X: ",
                             (uint32) (EndPos >> 32), (uint32) EndPos);
    
            /*
             * We have to piece together the WAL record data from the XLogRecData
             * entries, so that we can pass it to the rm_desc function as one
             * contiguous chunk.
             */
            initStringInfo(&recordBuf);
            for (; rdata != NULL; rdata = rdata->next)
                appendBinaryStringInfo(&recordBuf, rdata->data, rdata->len);
    
            if (!debug_reader)
                debug_reader = XLogReaderAllocate(wal_segment_size, NULL, NULL);
    
            if (!debug_reader)
            {
                appendStringInfoString(&buf, "error decoding record: out of memory");
            }
            else if (!DecodeXLogRecord(debug_reader, (XLogRecord *) recordBuf.data,
                                       &errormsg))
            {
                appendStringInfo(&buf, "error decoding record: %s",
                                 errormsg ? errormsg : "no error message");
            }
            else
            {
                appendStringInfoString(&buf, " - ");
                xlog_outdesc(&buf, debug_reader);
            }
            elog(LOG, "%s", buf.data);
    
            pfree(buf.data);
            pfree(recordBuf.data);
            MemoryContextSwitchTo(oldCxt);
        }
    #endif
    
        /*
         * Update our global variables
         * 更新全局变量
         */
        ProcLastRecPtr = StartPos;
        XactLastRecEnd = EndPos;
    
        return EndPos;
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    insert into t_wal_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    
    

启动gdb,设置断点,进入XLogInsert

    
    
    (gdb) b XLogInsert
    Breakpoint 1 at 0x5652d6: file xloginsert.c, line 420.
    (gdb) c
    Continuing.
    
    Breakpoint 1, XLogInsert (rmid=10 '\n', info=0 '\000') at xloginsert.c:420
    420     if (!begininsert_called)
    

在此前,XLogBeginInsert()必须已调用

    
    
    420     if (!begininsert_called)
    (gdb) n
    

调用方必须设置rmgr位:XLR_SPECIAL_REL_UPDATE & XLR_CHECK_CONSISTENCY.其余在这里保留使用

    
    
    427     if ((info & ~(XLR_RMGR_INFO_MASK |
    (gdb) n
    432     TRACE_POSTGRESQL_WAL_INSERT(rmid, info);
    

进入循环

    
    
    (gdb) n
    438     if (IsBootstrapProcessingMode() && rmid != RM_XLOG_ID)
    (gdb) 
    457         GetFullPageWriteInfo(&RedoRecPtr, &doPageWrites);
    

获取决定是否执行全页写入所需的值

    
    
    (gdb) p *RedoRecPtr
    $1 = 1166604425
    (gdb) p doPageWrites
    $2 = false
    (gdb) n
    459         rdt = XLogRecordAssemble(rmid, info, RedoRecPtr, doPageWrites,
    (gdb) p RedoRecPtr
    $3 = 5411227832
    (gdb) p doPageWrites
    $4 = true
    

获取rdt

    
    
    (gdb) n
    462         EndPos = XLogInsertRecord(rdt, fpw_lsn, curinsert_flags);
    (gdb) p *rdt
    $5 = {next = 0x2a911b8, data = 0x2a8f460 <incomplete sequence \322>, len = 51}
    

XLogInsertRecord->调用XLogInsertRecord,进入XLogInsertRecord函数  
fpw_lsn=0, flags=1 '\001'

    
    
    (gdb) step
    XLogInsertRecord (rdata=0xf9cc70 <hdr_rdt>, fpw_lsn=0, flags=1 '\001') at xlog.c:970
    970     XLogCtlInsert *Insert = &XLogCtl->Insert;
    

XLogInsertRecord->获取插入管理器

    
    
    (gdb) n
    973     XLogRecord *rechdr = (XLogRecord *) rdata->data;
    (gdb) p *Insert
    $6 = {insertpos_lck = 0 '\000', CurrBytePos = 5395369608, PrevBytePos = 5395369552, pad = '\000' <repeats 127 times>, 
      RedoRecPtr = 5411227832, forcePageWrites = false, fullPageWrites = true, exclusiveBackupState = EXCLUSIVE_BACKUP_NONE, 
      nonExclusiveBackups = 0, lastBackupStart = 0, WALInsertLocks = 0x7fa2523d4100}
    

XLogInsertRecord->变量赋值

    
    
    (gdb) n
    974     uint8       info = rechdr->xl_info & ~XLR_INFO_MASK;
    (gdb) 
    975     bool        isLogSwitch = (rechdr->xl_rmid == RM_XLOG_ID &&
    (gdb) 
    979     bool        prevDoPageWrites = doPageWrites;
    (gdb) 
    982     Assert(rdata->len >= SizeOfXLogRecord);
    (gdb) 
    (gdb) p *rechdr
    $7 = {xl_tot_len = 210, xl_xid = 1948, xl_prev = 0, xl_info = 0 '\000', xl_rmid = 10 '\n', xl_crc = 3212449170}
    (gdb) p info
    $8 = 0 '\000'
    (gdb) p isLogSwitch
    $9 = false
    (gdb) p prevDoPageWrites
    $10 = true
    

XLogInsertRecord->执行相关判断,开启CRIT_SECTION,并获取WAL插入锁

    
    
    (gdb) n
    985     if (!XLogInsertAllowed())
    (gdb) 
    1020        START_CRIT_SECTION();
    (gdb) 
    1021        if (isLogSwitch)
    (gdb) 
    1024            WALInsertLockAcquire();
    (gdb) 
    1042        if (RedoRecPtr != Insert->RedoRecPtr)
    (gdb) 
    

XLogInsertRecord->执行相关判断,更新doPageWrites

    
    
    (gdb) p RedoRecPtr
    $11 = 5411227832
    (gdb) p Insert->RedoRecPtr
    $12 = 5411227832
    (gdb) n
    1047        doPageWrites = (Insert->fullPageWrites || Insert->forcePageWrites);
    (gdb) 
    1049        if (doPageWrites &&
    (gdb) p doPageWrites
    $13 = true
    (gdb) n
    1050            (!prevDoPageWrites ||
    (gdb) 
    1049        if (doPageWrites &&
    

XLogInsertRecord->在WAL预留记录空间.同时会设置xl_prev指针.

    
    
    (gdb) 
    1050            (!prevDoPageWrites ||
    (gdb) 
    1066        if (isLogSwitch)
    (gdb) 
    1070            ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos,
    (gdb) 
    1072            inserted = true;
    (gdb) p rechdr->xl_tot_len
    $14 = 210
    (gdb) p StartPos
    $15 = 5411228000
    (gdb) p EndPos
    $16 = 5411228216
    (gdb) p *rechdr->xl_prev
    Cannot access memory at address 0x14288c928
    (gdb) p rechdr->xl_prev
    $17 = 5411227944
    (gdb) 
    

XLogInsertRecord->现在xl_prev指针已填充,计算记录头部的CRC

    
    
    (gdb) n
    1075        if (inserted)
    (gdb) 
    1081            rdata_crc = rechdr->xl_crc;
    (gdb) 
    1082            COMP_CRC32C(rdata_crc, rechdr, offsetof(XLogRecord, xl_crc));
    (gdb) 
    1083            FIN_CRC32C(rdata_crc);
    (gdb) 
    1084            rechdr->xl_crc = rdata_crc;
    (gdb) 
    1090            CopyXLogRecordToWAL(rechdr->xl_tot_len, isLogSwitch, rdata,
    (gdb) p rdata_crc
    $18 = 2310972234
    (gdb) p *rechdr
    $19 = {xl_tot_len = 210, xl_xid = 1948, xl_prev = 5411227944, xl_info = 0 '\000', xl_rmid = 10 '\n', xl_crc = 2310972234}
    (gdb) 
    

XLogInsertRecord->所有的记录数据,包括头部数据已OK,准备插入!拷贝记录到保留空间中.  
除非记录被标记为不重要，否则更新当前slot中最后一条重要记录的LSN.

    
    
    (gdb) n
    1098            if ((flags & XLOG_MARK_UNIMPORTANT) == 0)
    (gdb) 
    1100                int         lockno = holdingAllLocks ? 0 : MyLockNo;
    (gdb) 
    (gdb) n
    1102                WALInsertLocks[lockno].l.lastImportantAt = StartPos;
    (gdb) 
    1117        WALInsertLockRelease();
    

XLogInsertRecord->全部完成!让其他插入器知道我们已经完成了!  
如跨越了page边界,更新共享的LogwrtRqst.Write变量

    
    
    (gdb) 
    1117        WALInsertLockRelease();
    (gdb) n
    1119        MarkCurrentTransactionIdLoggedIfAny();
    (gdb) 
    1121        END_CRIT_SECTION();
    (gdb) 
    1126        if (StartPos / XLOG_BLCKSZ != EndPos / XLOG_BLCKSZ)
    (gdb) 
    1142        if (isLogSwitch)
    

XLogInsertRecord->更新全局变量,函数返回

    
    
    (gdb) 
    1220        ProcLastRecPtr = StartPos;
    (gdb) 
    1221        XactLastRecEnd = EndPos;
    (gdb) 
    1223        return EndPos;
    (gdb) 
    1224    }
    

返回XLogInsert,重置insertion,返回EndPos,结束

    
    
    (gdb) 
    XLogInsert (rmid=10 '\n', info=0 '\000') at xloginsert.c:463
    463     } while (EndPos == InvalidXLogRecPtr);
    (gdb) n
    465     XLogResetInsertion();
    (gdb) 
    467     return EndPos;
    (gdb) 
    468 }
    (gdb) p EndPos
    $20 = 5411228216
    (gdb) 
    $21 = 5411228216
    (gdb) n
    heap_insert (relation=0x7fa280616228, tup=0x2b15440, cid=0, options=0, bistate=0x0) at heapam.c:2590
    2590            PageSetLSN(page, recptr);
    (gdb) 
    

DONE!

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PgSQL · 特性分析 · 数据库崩溃恢复（上）](http://mysql.taobao.org/monthly/2017/05/03/)  
[PgSQL · 特性分析 · 数据库崩溃恢复（下）](http://mysql.taobao.org/monthly/2017/06/04/)  
[PgSQL · 特性分析 · Write-Ahead
Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02/)  
[PostgreSQL WAL Buffers, Clog Buffers Deep
Dive](https://www.slideshare.net/pgday_seoul/pgdayseoul-2017-3-postgresql-wal-buffers-clog-buffers-deep-dive?from_action=save)

