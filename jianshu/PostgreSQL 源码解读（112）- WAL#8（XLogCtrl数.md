本节简单介绍了XLOG全局(所有进程之间)共享的数据结构:XLogCtlData和XLogCtlInsert。在这两个结构体中,存储了REDO
point/Lock等相关重要的信息.

### 一、数据结构

**XLogCtlInsert**  
WAL插入记录时使用的共享数据结构

    
    
    /*
     * Shared state data for WAL insertion.
     * WAL插入记录时使用的共享数据结构
     */
    typedef struct XLogCtlInsert
    {
        //包含CurrBytePos和PrevBytePos的lock
        slock_t     insertpos_lck;  /* protects CurrBytePos and PrevBytePos */
    
        /*
         * CurrBytePos is the end of reserved WAL. The next record will be
         * inserted at that position. PrevBytePos is the start position of the
         * previously inserted (or rather, reserved) record - it is copied to the
         * prev-link of the next record. These are stored as "usable byte
         * positions" rather than XLogRecPtrs (see XLogBytePosToRecPtr()).
         * CurrBytePos是保留WAL的结束位置。
         *   下一条记录将插入到那个位置。
         * PrevBytePos是先前插入(或者保留)记录的起始位置——它被复制到下一条记录的prev-link中。
         * 这些存储为“可用字节位置”，而不是XLogRecPtrs(参见XLogBytePosToRecPtr())。
         */
        uint64      CurrBytePos;
        uint64      PrevBytePos;
    
        /*
         * Make sure the above heavily-contended spinlock and byte positions are
         * on their own cache line. In particular, the RedoRecPtr and full page
         * write variables below should be on a different cache line. They are
         * read on every WAL insertion, but updated rarely, and we don't want
         * those reads to steal the cache line containing Curr/PrevBytePos.
         * 确保以上激烈竞争的自旋锁和字节位置在它们自己的缓存line上。
         * 特别是，RedoRecPtr和下面的全页写变量应该位于不同的缓存line上。
         * 它们在每次插入WAL时都被读取，但很少更新，
         *   我们不希望这些读取窃取包含Curr/PrevBytePos的缓存line。
         */
        char        pad[PG_CACHE_LINE_SIZE];
    
        /*
         * fullPageWrites is the master copy used by all backends to determine
         * whether to write full-page to WAL, instead of using process-local one.
         * This is required because, when full_page_writes is changed by SIGHUP,
         * we must WAL-log it before it actually affects WAL-logging by backends.
         * Checkpointer sets at startup or after SIGHUP.
         * fullpagewrite是所有后台进程使用的主副本，
         *   用于确定是否将整个页面写入WAL，而不是使用process-local副本。
         * 这是必需的，因为当SIGHUP更改full_page_write时，
         *   我们必须在它通过后台进程实际影响WAL-logging之前对其进行WAL-log记录。
         * Checkpointer检查点设置在启动或SIGHUP之后。
         *
         * To read these fields, you must hold an insertion lock. To modify them,
         * you must hold ALL the locks.
         * 为了读取这些域,必须持有insertion lock.
         * 如需更新,则需要持有所有这些lock. 
         */
        //插入时的当前redo point
        XLogRecPtr  RedoRecPtr;     /* current redo point for insertions */
        //为PITR强制执行full-page写?
        bool        forcePageWrites;    /* forcing full-page writes for PITR? */
        //是否全页写?
        bool        fullPageWrites;
    
        /*
         * exclusiveBackupState indicates the state of an exclusive backup (see
         * comments of ExclusiveBackupState for more details). nonExclusiveBackups
         * is a counter indicating the number of streaming base backups currently
         * in progress. forcePageWrites is set to true when either of these is
         * non-zero. lastBackupStart is the latest checkpoint redo location used
         * as a starting point for an online backup.
         * exclusive sivebackupstate表示排他备份的状态
         * (有关详细信息，请参阅exclusive sivebackupstate的注释)。
         * 非排他性备份是一个计数器，指示当前正在进行的流基础备份的数量。
         * forcePageWrites在这两个值都不为零时被设置为true。
         * lastBackupStart用作在线备份起点的最新检查点的重做位置。
         */
        ExclusiveBackupState exclusiveBackupState;
        int         nonExclusiveBackups;
        XLogRecPtr  lastBackupStart;
    
        /*
         * WAL insertion locks.
         * WAL写入锁
         */
        WALInsertLockPadded *WALInsertLocks;
    } XLogCtlInsert;
    
    

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
    
    

### 二、跟踪分析

跟踪任意一个后台进程,打印全局变量XLogCtl.

    
    
    (gdb) p XLogCtl
    $6 = (XLogCtlData *) 0x7f391e00ea80
    (gdb) p *XLogCtl
    $7 = {Insert = {insertpos_lck = 0 '\000', CurrBytePos = 5494680728, PrevBytePos = 5494680616, 
        pad = '\000' <repeats 127 times>, RedoRecPtr = 5510830896, forcePageWrites = false, fullPageWrites = true, 
        exclusiveBackupState = EXCLUSIVE_BACKUP_NONE, nonExclusiveBackups = 0, lastBackupStart = 0, 
        WALInsertLocks = 0x7f391e013100}, LogwrtRqst = {Write = 5510831008, Flush = 5510831008}, RedoRecPtr = 5510830896, 
      ckptXidEpoch = 0, ckptXid = 2036, asyncXactLSN = 5510830896, replicationSlotMinLSN = 0, lastRemovedSegNo = 0, 
      unloggedLSN = 1, ulsn_lck = 0 '\000', lastSegSwitchTime = 1545962218, lastSegSwitchLSN = 5507670464, LogwrtResult = {
        Write = 5510831008, Flush = 5510831008}, InitializedUpTo = 5527601152, pages = 0x7f391e014000 "\230\320\006", 
      xlblocks = 0x7f391e00f088, XLogCacheBlck = 2047, ThisTimeLineID = 1, PrevTimeLineID = 1, 
      archiveCleanupCommand = '\000' <repeats 1023 times>, SharedRecoveryInProgress = false, SharedHotStandbyActive = false, 
      WalWriterSleeping = true, recoveryWakeupLatch = {is_set = 0, is_shared = true, owner_pid = 0}, lastCheckPointRecPtr = 0, 
      lastCheckPointEndPtr = 0, lastCheckPoint = {redo = 0, ThisTimeLineID = 0, PrevTimeLineID = 0, fullPageWrites = false, 
        nextXidEpoch = 0, nextXid = 0, nextOid = 0, nextMulti = 0, nextMultiOffset = 0, oldestXid = 0, oldestXidDB = 0, 
        oldestMulti = 0, oldestMultiDB = 0, time = 0, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}, 
      lastReplayedEndRecPtr = 0, lastReplayedTLI = 0, replayEndRecPtr = 0, replayEndTLI = 0, recoveryLastXTime = 0, 
      currentChunkStartTime = 0, recoveryPause = false, lastFpwDisableRecPtr = 0, info_lck = 0 '\000'}
    (gdb) 
    

其中:  
1.XLogCtl->Insert是XLogCtlInsert结构体变量.  
2.RedoRecPtr为5510830896 -> 1/48789B30,该值与pg_control文件中的REDO location相对应.

    
    
    [xdb@localhost ~]$ pg_controldata|grep REDO
    Latest checkpoint's REDO location:    1/48789B30
    Latest checkpoint's REDO WAL file:    000000010000000100000048
    

3.ThisTimeLineID&PrevTimeLineID,时间线ID,值为1.  
其他相关信息可对照结构体定义阅读.

### 三、参考资料

[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PG Source Code](https://doxygen.postgresql.org)

