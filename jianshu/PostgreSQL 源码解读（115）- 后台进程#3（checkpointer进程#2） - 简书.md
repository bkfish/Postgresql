[Return to ITPUB blog](http://blog.itpub.net/6906/)

本节简单介绍了PostgreSQL的后台进程:checkpointer,主要分析CreateCheckPoint函数的实现逻辑。

### 一、数据结构

**CheckPoint**  
CheckPoint XLOG record结构体.

    
    
    /*
     * Body of CheckPoint XLOG records.  This is declared here because we keep
     * a copy of the latest one in pg_control for possible disaster recovery.
     * Changing this struct requires a PG_CONTROL_VERSION bump.
     * CheckPoint XLOG record结构体.
     * 在这里声明是因为我们在pg_control中保存了最新的副本，
     *   以便进行可能的灾难恢复。
     * 改变这个结构体需要一个PG_CONTROL_VERSION bump。
     */
    typedef struct CheckPoint
    {
        //在开始创建CheckPoint时下一个可用的RecPtr(比如REDO的开始点)
        XLogRecPtr  redo;           /* next RecPtr available when we began to
                                     * create CheckPoint (i.e. REDO start point) */
        //当前的时间线
        TimeLineID  ThisTimeLineID; /* current TLI */
        //上一个时间线(如该记录正在开启一条新的时间线,否则等于当前时间线)
        TimeLineID  PrevTimeLineID; /* previous TLI, if this record begins a new
                                     * timeline (equals ThisTimeLineID otherwise) */
        //是否full-page-write
        bool        fullPageWrites; /* current full_page_writes */
        //nextXid的高阶位
        uint32      nextXidEpoch;   /* higher-order bits of nextXid */
        //下一个free的XID
        TransactionId nextXid;      /* next free XID */
        //下一个free的OID
        Oid         nextOid;        /* next free OID */
        //下一个fredd的MultiXactId
        MultiXactId nextMulti;      /* next free MultiXactId */
        //下一个空闲的MultiXact偏移
        MultiXactOffset nextMultiOffset;    /* next free MultiXact offset */
        //集群范围内的最小datfrozenxid
        TransactionId oldestXid;    /* cluster-wide minimum datfrozenxid */
        //最小datfrozenxid所在的database
        Oid         oldestXidDB;    /* database with minimum datfrozenxid */
        //集群范围内的最小datminmxid
        MultiXactId oldestMulti;    /* cluster-wide minimum datminmxid */
        //最小datminmxid所在的database
        Oid         oldestMultiDB;  /* database with minimum datminmxid */
        //checkpoint的时间戳
        pg_time_t   time;           /* time stamp of checkpoint */
        //带有有效提交时间戳的最老Xid
        TransactionId oldestCommitTsXid;    /* oldest Xid with valid commit
                                             * timestamp */
        //带有有效提交时间戳的最新Xid
        TransactionId newestCommitTsXid;    /* newest Xid with valid commit
                                             * timestamp */
    
        /*
         * Oldest XID still running. This is only needed to initialize hot standby
         * mode from an online checkpoint, so we only bother calculating this for
         * online checkpoints and only when wal_level is replica. Otherwise it's
         * set to InvalidTransactionId.
         * 最老的XID还在运行。
         * 这只需要从online checkpoint初始化热备模式，因此我们只需要为在线检查点计算此值，
         *   并且只在wal_level是replica时才计算此值。
         * 否则它被设置为InvalidTransactionId。
         */
        TransactionId oldestActiveXid;
    } CheckPoint;
    
    /* XLOG info values for XLOG rmgr */
    #define XLOG_CHECKPOINT_SHUTDOWN        0x00
    #define XLOG_CHECKPOINT_ONLINE          0x10
    #define XLOG_NOOP                       0x20
    #define XLOG_NEXTOID                    0x30
    #define XLOG_SWITCH                     0x40
    #define XLOG_BACKUP_END                 0x50
    #define XLOG_PARAMETER_CHANGE           0x60
    #define XLOG_RESTORE_POINT              0x70
    #define XLOG_FPW_CHANGE                 0x80
    #define XLOG_END_OF_RECOVERY            0x90
    #define XLOG_FPI_FOR_HINT               0xA0
    #define XLOG_FPI                        0xB0
    
    

**CheckpointerShmem**  
checkpointer进程和其他后台进程之间通讯的共享内存结构.

    
    
    /*----------     * Shared memory area for communication between checkpointer and backends
     * checkpointer进程和其他后台进程之间通讯的共享内存结构.
     *
     * The ckpt counters allow backends to watch for completion of a checkpoint
     * request they send.  Here's how it works:
     *  * At start of a checkpoint, checkpointer reads (and clears) the request
     *    flags and increments ckpt_started, while holding ckpt_lck.
     *  * On completion of a checkpoint, checkpointer sets ckpt_done to
     *    equal ckpt_started.
     *  * On failure of a checkpoint, checkpointer increments ckpt_failed
     *    and sets ckpt_done to equal ckpt_started.
     * ckpt计数器可以让后台进程监控它们发出来的checkpoint请求是否已完成.其工作原理如下:
     *  * 在checkpoint启动阶段,checkpointer进程获取并持有ckpt_lck锁后,
     *    读取(并清除)请求标志并增加ckpt_started计数.
     *  * checkpoint成功完成时,checkpointer设置ckpt_done值等于ckpt_started.
     *  * checkpoint如执行失败,checkpointer增加ckpt_failed计数,并设置ckpt_done值等于ckpt_started.
     *
     * The algorithm for backends is:
     *  1. Record current values of ckpt_failed and ckpt_started, and
     *     set request flags, while holding ckpt_lck.
     *  2. Send signal to request checkpoint.
     *  3. Sleep until ckpt_started changes.  Now you know a checkpoint has
     *     begun since you started this algorithm (although *not* that it was
     *     specifically initiated by your signal), and that it is using your flags.
     *  4. Record new value of ckpt_started.
     *  5. Sleep until ckpt_done >= saved value of ckpt_started.  (Use modulo
     *     arithmetic here in case counters wrap around.)  Now you know a
     *     checkpoint has started and completed, but not whether it was
     *     successful.
     *  6. If ckpt_failed is different from the originally saved value,
     *     assume request failed; otherwise it was definitely successful.
     * 算法如下:
     *  1.获取并持有ckpt_lck锁后,记录ckpt_failed和ckpt_started的当前值,并设置请求标志.
     *  2.发送信号,请求checkpoint.
     *  3.休眠直至ckpt_started发生变化.
     *    现在您知道自您启动此算法以来检查点已经开始(尽管*不是*它是由您的信号具体发起的)，并且它正在使用您的标志。
     *  4.记录ckpt_started的新值.
     *  5.休眠,直至ckpt_done >= 已保存的ckpt_started值(取模).现在已知checkpoint已启动&已完成,但checkpoint不一定成功.
     *  6.如果ckpt_failed与原来保存的值不同,则可以认为请求失败,否则它肯定是成功的.
     *
     * ckpt_flags holds the OR of the checkpoint request flags sent by all
     * requesting backends since the last checkpoint start.  The flags are
     * chosen so that OR'ing is the correct way to combine multiple requests.
     * ckpt_flags保存自上次检查点启动以来所有后台进程发送的检查点请求标志的OR或标记。
     * 选择标志，以便OR'ing是组合多个请求的正确方法。
     * 
     * num_backend_writes is used to count the number of buffer writes performed
     * by user backend processes.  This counter should be wide enough that it
     * can't overflow during a single processing cycle.  num_backend_fsync
     * counts the subset of those writes that also had to do their own fsync,
     * because the checkpointer failed to absorb their request.
     * num_backend_writes用于计算用户后台进程写入的缓冲区个数.
     * 在一个单独的处理过程中,该计数器必须足够大以防溢出.
     * num_backend_fsync计数那些必须执行fsync写操作的子集，
     *   因为checkpointer进程未能接受它们的请求。
     *
     * The requests array holds fsync requests sent by backends and not yet
     * absorbed by the checkpointer.
     * 请求数组存储后台进程发出的未被checkpointer进程拒绝的fsync请求.
     *
     * Unlike the checkpoint fields, num_backend_writes, num_backend_fsync, and
     * the requests fields are protected by CheckpointerCommLock.
     * 不同于checkpoint域,num_backend_writes/num_backend_fsync通过CheckpointerCommLock保护.
     * 
     *----------     */
    typedef struct
    {
        RelFileNode rnode;//表空间/数据库/Relation信息
        ForkNumber  forknum;//fork编号
        BlockNumber segno;          /* see md.c for special values */
        /* might add a real request-type field later; not needed yet */
    } CheckpointerRequest;
    
    typedef struct
    {
        //checkpoint进程的pid(为0则进程未启动)
        pid_t       checkpointer_pid;   /* PID (0 if not started) */
        //用于保护所有的ckpt_*域
        slock_t     ckpt_lck;       /* protects all the ckpt_* fields */
        //在checkpoint启动时计数
        int         ckpt_started;   /* advances when checkpoint starts */
        //在checkpoint完成时计数
        int         ckpt_done;      /* advances when checkpoint done */
        //在checkpoint失败时计数
        int         ckpt_failed;    /* advances when checkpoint fails */
        //检查点标记,在xlog.h中定义
        int         ckpt_flags;     /* checkpoint flags, as defined in xlog.h */
        //计数后台进程缓存写的次数
        uint32      num_backend_writes; /* counts user backend buffer writes */
        //计数后台进程fsync调用次数
        uint32      num_backend_fsync;  /* counts user backend fsync calls */
        //当前的请求编号
        int         num_requests;   /* current # of requests */
        //最大的请求编号
        int         max_requests;   /* allocated array size */
        //请求数组
        CheckpointerRequest requests[FLEXIBLE_ARRAY_MEMBER];
    } CheckpointerShmemStruct;
    //静态变量(CheckpointerShmemStruct结构体指针)
    static CheckpointerShmemStruct *CheckpointerShmem;
    
    

**VirtualTransactionId**  
最顶层的事务通过VirtualTransactionIDs定义.

    
    
    /*
     * Top-level transactions are identified by VirtualTransactionIDs comprising
     * the BackendId of the backend running the xact, plus a locally-assigned
     * LocalTransactionId.  These are guaranteed unique over the short term,
     * but will be reused after a database restart; hence they should never
     * be stored on disk.
     * 最高层的事务通过VirtualTransactionIDs定义.
     * VirtualTransactionIDs由执行事务的后台进程BackendId和逻辑分配的LocalTransactionId组成.
     *
     * Note that struct VirtualTransactionId can not be assumed to be atomically
     * assignable as a whole.  However, type LocalTransactionId is assumed to
     * be atomically assignable, and the backend ID doesn't change often enough
     * to be a problem, so we can fetch or assign the two fields separately.
     * We deliberately refrain from using the struct within PGPROC, to prevent
     * coding errors from trying to use struct assignment with it; instead use
     * GET_VXID_FROM_PGPROC().
     * 请注意，不能假设struct VirtualTransactionId作为一个整体是原子可分配的。
     * 但是,类型LocalTransactionId是假定原子可分配的,同时后台进程ID不会经常变换,因此这不是一个问题,
     *   因此我们可以单独提取或者分配这两个域字段.
     * 
     */
    typedef struct
    {
        BackendId   backendId;      /* determined at backend startup */
        LocalTransactionId localTransactionId;  /* backend-local transaction id */
    } VirtualTransactionId;
    
    

### 二、源码解读

CreateCheckPoint函数,执行checkpoint,不管是在shutdown过程还是在运行中.

    
    
    /*
     * Perform a checkpoint --- either during shutdown, or on-the-fly
     * 执行checkpoint,不管是在shutdown过程还是在运行中 
     *
     * flags is a bitwise OR of the following:
     *  CHECKPOINT_IS_SHUTDOWN: checkpoint is for database shutdown.
     *  CHECKPOINT_END_OF_RECOVERY: checkpoint is for end of WAL recovery.
     *  CHECKPOINT_IMMEDIATE: finish the checkpoint ASAP,
     *      ignoring checkpoint_completion_target parameter.
     *  CHECKPOINT_FORCE: force a checkpoint even if no XLOG activity has occurred
     *      since the last one (implied by CHECKPOINT_IS_SHUTDOWN or
     *      CHECKPOINT_END_OF_RECOVERY).
     *  CHECKPOINT_FLUSH_ALL: also flush buffers of unlogged tables.
     * flags标记说明:
     *  CHECKPOINT_IS_SHUTDOWN: 数据库关闭过程中的checkpoint
     *  CHECKPOINT_END_OF_RECOVERY: 通过WAL恢复后的checkpoint
     *  CHECKPOINT_IMMEDIATE: 尽可能快的完成checkpoint,忽略checkpoint_completion_target参数
     *  CHECKPOINT_FORCE: 在最后一次checkpoint后就算没有任何的XLOG活动发生,也强制执行checkpoint
     *                    (意味着CHECKPOINT_IS_SHUTDOWN或CHECKPOINT_END_OF_RECOVERY)
     *  CHECKPOINT_FLUSH_ALL: 包含unlogged tables一并刷盘
     *
     * Note: flags contains other bits, of interest here only for logging purposes.
     * In particular note that this routine is synchronous and does not pay
     * attention to CHECKPOINT_WAIT.
     * 注意:标志还包含其他位，此处仅用于日志记录。
     * 特别注意的是该过程同步执行,并不会理会CHECKPOINT_WAIT.
     *
     * If !shutdown then we are writing an online checkpoint. This is a very special
     * kind of operation and WAL record because the checkpoint action occurs over
     * a period of time yet logically occurs at just a single LSN. The logical
     * position of the WAL record (redo ptr) is the same or earlier than the
     * physical position. When we replay WAL we locate the checkpoint via its
     * physical position then read the redo ptr and actually start replay at the
     * earlier logical position. Note that we don't write *anything* to WAL at
     * the logical position, so that location could be any other kind of WAL record.
     * All of this mechanism allows us to continue working while we checkpoint.
     * As a result, timing of actions is critical here and be careful to note that
     * this function will likely take minutes to execute on a busy system.
     * 如果并不处在shutdown过程中,那么我们会等待一个在线checkpoint.
     * 这是一种非常特殊的操作和WAL记录，因为检查点操作发生在一段时间内，而逻辑上只发生在一个LSN上。
     * WAL Record(redo ptr)的逻辑位置与物理位置相同或者小于物理位置.
     * 在回放WAL的时候我们通过checkpoint的物理位置定位位置,然后读取redo ptr,
     *   实际上在更早的逻辑位置开始回放,这样该位置可以是任意类型的WAL Record.
     * 这种机制的目的是允许我们在checkpoint的时候不需要暂停.
     * 这种机制的结果是操作的时间会比较长,要小心的是在繁忙的系统中,该操作可能会持续数分钟.
     */
    void
    CreateCheckPoint(int flags)
    {
        bool        shutdown;//是否处于shutdown?
        CheckPoint  checkPoint;//checkpoint
        XLogRecPtr  recptr;//XLOG Record位置
        XLogSegNo   _logSegNo;//LSN(uint64)
        XLogCtlInsert *Insert = &XLogCtl->Insert;//控制器
        uint32      freespace;//空闲空间
        XLogRecPtr  PriorRedoPtr;//上一个Redo point
        XLogRecPtr  curInsert;//当前插入的位置
        XLogRecPtr  last_important_lsn;//上一个重要的LSN
        VirtualTransactionId *vxids;//虚拟事务ID
        int         nvxids;
    
        /*
         * An end-of-recovery checkpoint is really a shutdown checkpoint, just
         * issued at a different time.
         * end-of-recovery checkpoint事实上是shutdown checkpoint,只不过是在一个不同的时间发生的.
         */
        if (flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY))
            shutdown = true;
        else
            shutdown = false;
    
        /* sanity check */
        //验证
        if (RecoveryInProgress() && (flags & CHECKPOINT_END_OF_RECOVERY) == 0)
            elog(ERROR, "can't create a checkpoint during recovery");
    
        /*
         * Initialize InitXLogInsert working areas before entering the critical
         * section.  Normally, this is done by the first call to
         * RecoveryInProgress() or LocalSetXLogInsertAllowed(), but when creating
         * an end-of-recovery checkpoint, the LocalSetXLogInsertAllowed call is
         * done below in a critical section, and InitXLogInsert cannot be called
         * in a critical section.
         * 在进入critical section前,初始化InitXLogInsert工作空间.
         * 通常来说,第一次调用RecoveryInProgress() or LocalSetXLogInsertAllowed()时已完成,
         *   但在创建end-of-recovery checkpoint时,在下面的逻辑中LocalSetXLogInsertAllowed调用完成时,
         *   InitXLogInsert不能在critical section中调用.
         */
        InitXLogInsert();
    
        /*
         * Acquire CheckpointLock to ensure only one checkpoint happens at a time.
         * (This is just pro forma, since in the present system structure there is
         * only one process that is allowed to issue checkpoints at any given
         * time.)
         * 请求CheckpointLock确保在同一时刻只能存在一个checkpoint.
         * (这只是形式上的，因为在目前的系统架构中，在任何给定的时间只允许一个进程发出检查点。)
         */
        LWLockAcquire(CheckpointLock, LW_EXCLUSIVE);
    
        /*
         * Prepare to accumulate statistics.
         * 为统计做准备.
         *
         * Note: because it is possible for log_checkpoints to change while a
         * checkpoint proceeds, we always accumulate stats, even if
         * log_checkpoints is currently off.
         * 注意:在checkpoint执行过程总,log_checkpoints可能会出现变化,
         *   因此我们通常会累计stats,即使log_checkpoints为off
         */
        MemSet(&CheckpointStats, 0, sizeof(CheckpointStats));
        CheckpointStats.ckpt_start_t = GetCurrentTimestamp();
    
        /*
         * Use a critical section to force system panic if we have trouble.
         * 使用critical section,强制系统在出现问题时进行应对.
         */
        START_CRIT_SECTION();
    
        if (shutdown)
        {
            //shutdown = T
            //更新control file(pg_control文件)
            LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
            ControlFile->state = DB_SHUTDOWNING;
            ControlFile->time = (pg_time_t) time(NULL);
            UpdateControlFile();
            LWLockRelease(ControlFileLock);
        }
    
        /*
         * Let smgr prepare for checkpoint; this has to happen before we determine
         * the REDO pointer.  Note that smgr must not do anything that'd have to
         * be undone if we decide no checkpoint is needed.
         * 让smgr(资源管理器)为checkpoint作准备.
         * 在确定REDO pointer时必须执行.
         * 请注意，如果我们决定不执行checkpoint，那么smgr不能执行任何必须撤消的操作。
         */
        smgrpreckpt();
    
        /* Begin filling in the checkpoint WAL record */
        //填充Checkpoint XLOG Record
        MemSet(&checkPoint, 0, sizeof(checkPoint));
        checkPoint.time = (pg_time_t) time(NULL);//时间
    
        /*
         * For Hot Standby, derive the oldestActiveXid before we fix the redo
         * pointer. This allows us to begin accumulating changes to assemble our
         * starting snapshot of locks and transactions.
         * 对于Hot Standby,在修改redo pointer前,推导出oldestActiveXid.
         * 这可以让我们可以累计变化以组装开始的snapshot的locks和transactions.
         */
        if (!shutdown && XLogStandbyInfoActive())
            checkPoint.oldestActiveXid = GetOldestActiveTransactionId();
        else
            checkPoint.oldestActiveXid = InvalidTransactionId;
    
        /*
         * Get location of last important record before acquiring insert locks (as
         * GetLastImportantRecPtr() also locks WAL locks).
         * 在请求插入locks前,获取最后一个重要的XLOG Record的位置.
         * (GetLastImportantRecPtr()函数会获取WAL locks)
         */
        last_important_lsn = GetLastImportantRecPtr();
    
        /*
         * We must block concurrent insertions while examining insert state to
         * determine the checkpoint REDO pointer.
         * 在检查插入状态确定checkpoint的REDO pointer时,必须阻塞同步插入操作.
         */
        WALInsertLockAcquireExclusive();
        curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);
    
        /*
         * If this isn't a shutdown or forced checkpoint, and if there has been no
         * WAL activity requiring a checkpoint, skip it.  The idea here is to
         * avoid inserting duplicate checkpoints when the system is idle.
         * 不是shutdow或强制checkpoint,而且在请求时如果没有WAL活动,则跳过.
         * 这里的思想是避免在系统空闲时插入重复的checkpoints
         */
        if ((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY |
                      CHECKPOINT_FORCE)) == 0)
        {
            if (last_important_lsn == ControlFile->checkPoint)
            {
                WALInsertLockRelease();
                LWLockRelease(CheckpointLock);
                END_CRIT_SECTION();
                ereport(DEBUG1,
                        (errmsg("checkpoint skipped because system is idle")));
                return;
            }
        }
    
        /*
         * An end-of-recovery checkpoint is created before anyone is allowed to
         * write WAL. To allow us to write the checkpoint record, temporarily
         * enable XLogInsertAllowed.  (This also ensures ThisTimeLineID is
         * initialized, which we need here and in AdvanceXLInsertBuffer.)
         * 在允许写入WAL后才会创建end-of-recovery checkpoint.
         * 这可以让我们写Checkpoint Record,临时启用XLogInsertAllowed.
         * (这同样可以确保已初始化在这里和AdvanceXLInsertBuffer中需要的变量ThisTimeLineID)
         */
        if (flags & CHECKPOINT_END_OF_RECOVERY)
            LocalSetXLogInsertAllowed();
    
        checkPoint.ThisTimeLineID = ThisTimeLineID;
        if (flags & CHECKPOINT_END_OF_RECOVERY)
            checkPoint.PrevTimeLineID = XLogCtl->PrevTimeLineID;
        else
            checkPoint.PrevTimeLineID = ThisTimeLineID;
    
        checkPoint.fullPageWrites = Insert->fullPageWrites;
    
        /*
         * Compute new REDO record ptr = location of next XLOG record.
         * 计算新的REDO record ptr = 下一个XLOG Record的位置.
         * 
         * NB: this is NOT necessarily where the checkpoint record itself will be,
         * since other backends may insert more XLOG records while we're off doing
         * the buffer flush work.  Those XLOG records are logically after the
         * checkpoint, even though physically before it.  Got that?
         * 注意:这并不一定是检查点记录本身所在的位置，因为当我们停止缓冲区刷新工作时，
         *   其他后台进程可能会插入更多的XLOG Record。
         * 这些XLOG Records逻辑上会在checkpoint之后,虽然物理上可能在checkpoint之前.
         */
        freespace = INSERT_FREESPACE(curInsert);//获取空闲空间
        if (freespace == 0)
        {
            //没有空闲空间了
            if (XLogSegmentOffset(curInsert, wal_segment_size) == 0)
                curInsert += SizeOfXLogLongPHD;//新的WAL segment file,偏移为LONG header
            else
                curInsert += SizeOfXLogShortPHD;//原WAL segment file,偏移为常规的header
        }
        checkPoint.redo = curInsert;
    
        /*
         * Here we update the shared RedoRecPtr for future XLogInsert calls; this
         * must be done while holding all the insertion locks.
         * 在这里，我们更新共享的RedoRecPtr以备将来的XLogInsert调用;
         *   这必须在持有所有插入锁才能完成。
         *
         * Note: if we fail to complete the checkpoint, RedoRecPtr will be left
         * pointing past where it really needs to point.  This is okay; the only
         * consequence is that XLogInsert might back up whole buffers that it
         * didn't really need to.  We can't postpone advancing RedoRecPtr because
         * XLogInserts that happen while we are dumping buffers must assume that
         * their buffer changes are not included in the checkpoint.
         * 注意:如果checkpoint失败,RedoRecPtr仍会指向实际上它应指向的位置.
         * 这种做法没有问题,唯一需要处理的XLogInsert可能会备份它并不真正需要的整个缓冲区.
         * 我们不能推迟推进RedoRecPtr，因为在转储缓冲区时发生的XLogInserts,
         *   必须假设它们的缓冲区更改不包含在该检查点中。
         */
        RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;
    
        /*
         * Now we can release the WAL insertion locks, allowing other xacts to
         * proceed while we are flushing disk buffers.
         * 现在可以释放WAL插入锁,允许其他事务在刷新磁盘缓冲区时可以执行.
         */
        WALInsertLockRelease();
    
        /* Update the info_lck-protected copy of RedoRecPtr as well */
        //同时,更新RedoRecPtr的info_lck-protected拷贝锁.
        SpinLockAcquire(&XLogCtl->info_lck);
        XLogCtl->RedoRecPtr = checkPoint.redo;
        SpinLockRelease(&XLogCtl->info_lck);
    
        /*
         * If enabled, log checkpoint start.  We postpone this until now so as not
         * to log anything if we decided to skip the checkpoint.
         * 如启用log_checkpoints,则记录checkpoint日志启动.
         * 我们将此推迟到现在，以便在决定跳过检查点时不记录任何东西。
         */
        if (log_checkpoints)
            LogCheckpointStart(flags, false);
    
        TRACE_POSTGRESQL_CHECKPOINT_START(flags);
    
        /*
         * Get the other info we need for the checkpoint record.
         * 获取其他组装checkpoint记录的信息.
         *
         * We don't need to save oldestClogXid in the checkpoint, it only matters
         * for the short period in which clog is being truncated, and if we crash
         * during that we'll redo the clog truncation and fix up oldestClogXid
         * there.
         * 我们不需要在检查点中保存oldestClogXid，它只在截断clog的短时间内起作用，
         *   如果在此期间崩溃，我们将重新截断clog并在修复oldestClogXid。
         */
        LWLockAcquire(XidGenLock, LW_SHARED);
        checkPoint.nextXid = ShmemVariableCache->nextXid;
        checkPoint.oldestXid = ShmemVariableCache->oldestXid;
        checkPoint.oldestXidDB = ShmemVariableCache->oldestXidDB;
        LWLockRelease(XidGenLock);
    
        LWLockAcquire(CommitTsLock, LW_SHARED);
        checkPoint.oldestCommitTsXid = ShmemVariableCache->oldestCommitTsXid;
        checkPoint.newestCommitTsXid = ShmemVariableCache->newestCommitTsXid;
        LWLockRelease(CommitTsLock);
    
        /* Increase XID epoch if we've wrapped around since last checkpoint */
        //如果我们从上一个checkpoint开始wrapped around，则增加XID epoch
        checkPoint.nextXidEpoch = ControlFile->checkPointCopy.nextXidEpoch;
        if (checkPoint.nextXid < ControlFile->checkPointCopy.nextXid)
            checkPoint.nextXidEpoch++;
    
        LWLockAcquire(OidGenLock, LW_SHARED);
        checkPoint.nextOid = ShmemVariableCache->nextOid;
        if (!shutdown)
            checkPoint.nextOid += ShmemVariableCache->oidCount;
        LWLockRelease(OidGenLock);
    
        MultiXactGetCheckptMulti(shutdown,
                                 &checkPoint.nextMulti,
                                 &checkPoint.nextMultiOffset,
                                 &checkPoint.oldestMulti,
                                 &checkPoint.oldestMultiDB);
    
        /*
         * Having constructed the checkpoint record, ensure all shmem disk buffers
         * and commit-log buffers are flushed to disk.
         * 在构造checkpoint XLOG Record之后，确保所有shmem disk buffers和clog缓冲区都被刷到磁盘中。
         *
         * This I/O could fail for various reasons.  If so, we will fail to
         * complete the checkpoint, but there is no reason to force a system
         * panic. Accordingly, exit critical section while doing it.
         * 刷盘I/O可能会因为很多原因失败.
         * 如果出现问题,那么checkpoint会失败,但没有理由强制要求系统panic.
         * 相反,在做这些工作时退出critical section.
         */
        END_CRIT_SECTION();
    
        /*
         * In some cases there are groups of actions that must all occur on one
         * side or the other of a checkpoint record. Before flushing the
         * checkpoint record we must explicitly wait for any backend currently
         * performing those groups of actions.
         * 在某些情况下，必须在checkpoint XLOG Record的一边或另一边执行一组操作。
         * 在刷新checkpoint XLOG Record之前，我们必须显式地等待当前执行这些操作组的所有后台进程。
         *
         * One example is end of transaction, so we must wait for any transactions
         * that are currently in commit critical sections.  If an xact inserted
         * its commit record into XLOG just before the REDO point, then a crash
         * restart from the REDO point would not replay that record, which means
         * that our flushing had better include the xact's update of pg_xact.  So
         * we wait till he's out of his commit critical section before proceeding.
         * See notes in RecordTransactionCommit().
         * 其中一个例子是事务结束,我们必须等待当前正处于commit critical sections的事务结束.
         * 如果某个事务正好在REDO point前插入commit record到XLOG中,
         *   如果系统crash,则重启后,从REDO point起读取时不会回放该commit记录,
         *   这意味着我们的刷盘最好包含xact对pg_xact的更新.
         * 所以我们要等到该进程离开commit critical section后再继续。
         * 参见RecordTransactionCommit()中的注释。
         *
         * Because we've already released the insertion locks, this test is a bit
         * fuzzy: it is possible that we will wait for xacts we didn't really need
         * to wait for.  But the delay should be short and it seems better to make
         * checkpoint take a bit longer than to hold off insertions longer than
         * necessary. (In fact, the whole reason we have this issue is that xact.c
         * does commit record XLOG insertion and clog update as two separate steps
         * protected by different locks, but again that seems best on grounds of
         * minimizing lock contention.)
         * 因为我们已经释放了插入锁,这个测试有点模糊:有可能我们将等待我们实际上不需要等待的xacts。
         * 但是延迟应该很短，让检查点花费的时间比延迟插入所需的时间长一些似乎更好。
         * (实际上,我们遇到这个问题的原因是xact.c将commit record XLOG插入和clog更新作为两个单独的步骤提交，
         *  这两个操作由不同的锁进行保护，但基于最小化锁争用的理由这看起来是最好的。)
         *
         * A transaction that has not yet set delayChkpt when we look cannot be at
         * risk, since he's not inserted his commit record yet; and one that's
         * already cleared it is not at risk either, since he's done fixing clog
         * and we will correctly flush the update below.  So we cannot miss any
         * xacts we need to wait for.
         * 在我们搜索时,尚未设置delayChkpt的事务不会存在风险，因为该事务还没有插入它的提交记录;
         * 同样的已清除了delayChkpt的事务也不会有风险,因为该事务已修改了clog,
         *   我们可以正确的在下面的处理逻辑中刷新更新.
         * 因此我们不能错失我们需要等待的所有xacts.
         */
        vxids = GetVirtualXIDsDelayingChkpt(&nvxids);//获取虚拟事务XID
        if (nvxids > 0)
        {
            do
            {
                //等待10ms
                pg_usleep(10000L);  /* wait for 10 msec */
            } while (HaveVirtualXIDsDelayingChkpt(vxids, nvxids));
        }
        pfree(vxids);
        //把共享内存中的数据刷到磁盘上,并执行fsync
        CheckPointGuts(checkPoint.redo, flags);
    
        /*
         * Take a snapshot of running transactions and write this to WAL. This
         * allows us to reconstruct the state of running transactions during
         * archive recovery, if required. Skip, if this info disabled.
         * 获取正在运行的事务的快照，并将其写入WAL。
         * 如果需要，这允许我们在归档恢复期间重建正在运行的事务的状态。
         * 如果禁用此消息,则禁用。
         * 
         * If we are shutting down, or Startup process is completing crash
         * recovery we don't need to write running xact data.
         * 如果正在关闭数据库,或者启动进程已完成crash recovery,
         *   则不需要写正在运行的事务数据.
         */
        if (!shutdown && XLogStandbyInfoActive())
            LogStandbySnapshot();
    
        START_CRIT_SECTION();//进入critical section.
    
        /*
         * Now insert the checkpoint record into XLOG.
         * 现在可以插入checkpoint record到XLOG中了.
         */
        XLogBeginInsert();//开始插入
        XLogRegisterData((char *) (&checkPoint), sizeof(checkPoint));//注册数据
        recptr = XLogInsert(RM_XLOG_ID,
                            shutdown ? XLOG_CHECKPOINT_SHUTDOWN :
                            XLOG_CHECKPOINT_ONLINE);//执行插入
    
        XLogFlush(recptr);//刷盘
    
        /*
         * We mustn't write any new WAL after a shutdown checkpoint, or it will be
         * overwritten at next startup.  No-one should even try, this just allows
         * sanity-checking.  In the case of an end-of-recovery checkpoint, we want
         * to just temporarily disable writing until the system has exited
         * recovery.
         * 我们不能在关闭检查点之后写入任何新的WAL，否则它将在下一次启动时被覆盖。
         * 而且不应该进行这样的尝试，只允许健康检查。
         * 在end-of-recovery checkpoint情况下，我们只想暂时禁用写入，直到系统退出恢复。
         */
        if (shutdown)
        {
            //关闭过程中
            if (flags & CHECKPOINT_END_OF_RECOVERY)
                LocalXLogInsertAllowed = -1;    /* return to "check" state */
            else
                LocalXLogInsertAllowed = 0; /* never again write WAL */
        }
    
        /*
         * We now have ProcLastRecPtr = start of actual checkpoint record, recptr
         * = end of actual checkpoint record.
         * 现在我们有:
         *   ProcLastRecPtr = 实际的checkpoint XLOG record的起始位置,
         *   recptr = 实际checkpoint XLOG record的结束位置.
         */
        if (shutdown && checkPoint.redo != ProcLastRecPtr)
            ereport(PANIC,
                    (errmsg("concurrent write-ahead log activity while database system is shutting down")));
    
        /*
         * Remember the prior checkpoint's redo ptr for
         * UpdateCheckPointDistanceEstimate()
         * 为UpdateCheckPointDistanceEstimate()记录上一个checkpoint的REDO ptr
         */
        PriorRedoPtr = ControlFile->checkPointCopy.redo;
    
        /*
         * Update the control file.
         * 更新控制文件(pg_control)
         */
        LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
        if (shutdown)
            ControlFile->state = DB_SHUTDOWNED;
        ControlFile->checkPoint = ProcLastRecPtr;
        ControlFile->checkPointCopy = checkPoint;
        ControlFile->time = (pg_time_t) time(NULL);
        /* crash recovery should always recover to the end of WAL */
        //crash recovery通常来说应恢复至WAL的末尾
        ControlFile->minRecoveryPoint = InvalidXLogRecPtr;
        ControlFile->minRecoveryPointTLI = 0;
    
        /*
         * Persist unloggedLSN value. It's reset on crash recovery, so this goes
         * unused on non-shutdown checkpoints, but seems useful to store it always
         * for debugging purposes.
         * 持久化unloggedLSN值.
         * 它是在崩溃恢复时重置的，因此在非关闭检查点上不使用，但是为了调试目的而总是存储它似乎很有用。
         */
        SpinLockAcquire(&XLogCtl->ulsn_lck);
        ControlFile->unloggedLSN = XLogCtl->unloggedLSN;
        SpinLockRelease(&XLogCtl->ulsn_lck);
    
        UpdateControlFile();
        LWLockRelease(ControlFileLock);
    
        /* Update shared-memory copy of checkpoint XID/epoch */
        //更新checkpoint XID/epoch的共享内存拷贝
        SpinLockAcquire(&XLogCtl->info_lck);
        XLogCtl->ckptXidEpoch = checkPoint.nextXidEpoch;
        XLogCtl->ckptXid = checkPoint.nextXid;
        SpinLockRelease(&XLogCtl->info_lck);
    
        /*
         * We are now done with critical updates; no need for system panic if we
         * have trouble while fooling with old log segments.
         * 已完成critical updates.
         */
        END_CRIT_SECTION();
    
        /*
         * Let smgr do post-checkpoint cleanup (eg, deleting old files).
         * 让smgr执行checkpoint收尾工作(比如删除旧文件等).
         */
        smgrpostckpt();
    
        /*
         * Update the average distance between checkpoints if the prior checkpoint
         * exists.
         * 如上一个checkpoint存在,则更新两者之间的平均距离.
         */
        if (PriorRedoPtr != InvalidXLogRecPtr)
            UpdateCheckPointDistanceEstimate(RedoRecPtr - PriorRedoPtr);
    
        /*
         * Delete old log files, those no longer needed for last checkpoint to
         * prevent the disk holding the xlog from growing full.
         * 删除旧的日志文件，这些文件自最后一个检查点后已不再需要，
         *   以防止保存xlog的磁盘撑满。
         */
        XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
        KeepLogSeg(recptr, &_logSegNo);
        _logSegNo--;
        RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr);
    
        /*
         * Make more log segments if needed.  (Do this after recycling old log
         * segments, since that may supply some of the needed files.)
         * 如需要,申请更多的log segments.
         * (在循环使用旧的log segments时才来做这个事情,因为那样会需要一些需要的文件)
         */
        if (!shutdown)
            PreallocXlogFiles(recptr);
    
        /*
         * Truncate pg_subtrans if possible.  We can throw away all data before
         * the oldest XMIN of any running transaction.  No future transaction will
         * attempt to reference any pg_subtrans entry older than that (see Asserts
         * in subtrans.c).  During recovery, though, we mustn't do this because
         * StartupSUBTRANS hasn't been called yet.
         * 如可能,截断pg_subtrans.
         * 我们可以在任何正在运行的事务的最老的XMIN之前丢弃所有数据。
         * 以后的事务都不会尝试引用任何比这更早的pg_subtrans条目(参见sub.c中的断言)。
         * 但是在恢复期间，我们不能这样做，因为StartupSUBTRANS还没有被调用。
         * 
         */
        if (!RecoveryInProgress())
            TruncateSUBTRANS(GetOldestXmin(NULL, PROCARRAY_FLAGS_DEFAULT));
    
        /* Real work is done, but log and update stats before releasing lock. */
        //实际的工作已完成,除了记录日志已经更新统计信息.
        LogCheckpointEnd(false);
    
        TRACE_POSTGRESQL_CHECKPOINT_DONE(CheckpointStats.ckpt_bufs_written,
                                         NBuffers,
                                         CheckpointStats.ckpt_segs_added,
                                         CheckpointStats.ckpt_segs_removed,
                                         CheckpointStats.ckpt_segs_recycled);
        //释放锁
        LWLockRelease(CheckpointLock);
    }
    
    /*
     * Flush all data in shared memory to disk, and fsync
     * 把共享内存中的数据刷到磁盘上,并执行fsync
     *
     * This is the common code shared between regular checkpoints and
     * recovery restartpoints.
     * 不管是普通的checkpoints还是recovery restartpoints,这些代码都是共享的.
     */
    static void
    CheckPointGuts(XLogRecPtr checkPointRedo, int flags)
    {
        CheckPointCLOG();
        CheckPointCommitTs();
        CheckPointSUBTRANS();
        CheckPointMultiXact();
        CheckPointPredicate();
        CheckPointRelationMap();
        CheckPointReplicationSlots();
        CheckPointSnapBuild();
        CheckPointLogicalRewriteHeap();
        CheckPointBuffers(flags);   /* performs all required fsyncs */
        CheckPointReplicationOrigin();
        /* We deliberately delay 2PC checkpointing as long as possible */
        CheckPointTwoPhase(checkPointRedo);
    }
    
    

### 三、跟踪分析

更新数据,执行checkpoint.

    
    
    testdb=# update t_wal_ckpt set c2 = 'C2_'||substr(c2,4,40);
    UPDATE 1
    testdb=# checkpoint;
    
    

启动gdb,设置信号控制,设置断点,进入CreateCheckPoint

    
    
    (gdb) handle SIGINT print nostop pass
    SIGINT is used by the debugger.
    Are you sure you want to change it? (y or n) y
    Signal        Stop  Print Pass to program Description
    SIGINT        No  Yes Yes   Interrupt
    (gdb) 
    (gdb) b CreateCheckPoint
    Breakpoint 1 at 0x55b4fb: file xlog.c, line 8668.
    (gdb) c
    Continuing.
    
    Program received signal SIGINT, Interrupt.
    
    Breakpoint 1, CreateCheckPoint (flags=44) at xlog.c:8668
    8668        XLogCtlInsert *Insert = &XLogCtl->Insert;
    (gdb) 
    

获取XLOG插入控制器

    
    
    8668        XLogCtlInsert *Insert = &XLogCtl->Insert;
    (gdb) n
    8680        if (flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY))
    (gdb) p XLogCtl
    $1 = (XLogCtlData *) 0x7fadf8f6fa80
    (gdb) p *XLogCtl
    $2 = {Insert = {insertpos_lck = 0 '\000', CurrBytePos = 5505269968, PrevBytePos = 5505269928, 
        pad = '\000' <repeats 127 times>, RedoRecPtr = 5521450856, forcePageWrites = false, fullPageWrites = true, 
        exclusiveBackupState = EXCLUSIVE_BACKUP_NONE, nonExclusiveBackups = 0, lastBackupStart = 0, 
        WALInsertLocks = 0x7fadf8f74100}, LogwrtRqst = {Write = 5521451392, Flush = 5521451392}, RedoRecPtr = 5521450856, 
      ckptXidEpoch = 0, ckptXid = 2307, asyncXactLSN = 5521363848, replicationSlotMinLSN = 0, lastRemovedSegNo = 0, 
      unloggedLSN = 1, ulsn_lck = 0 '\000', lastSegSwitchTime = 1546915130, lastSegSwitchLSN = 5521363360, LogwrtResult = {
        Write = 5521451392, Flush = 5521451392}, InitializedUpTo = 5538226176, pages = 0x7fadf8f76000 "\230\320\006", 
      xlblocks = 0x7fadf8f70088, XLogCacheBlck = 2047, ThisTimeLineID = 1, PrevTimeLineID = 1, 
      archiveCleanupCommand = '\000' <repeats 1023 times>, SharedRecoveryInProgress = false, SharedHotStandbyActive = false, 
      WalWriterSleeping = true, recoveryWakeupLatch = {is_set = 0, is_shared = true, owner_pid = 0}, lastCheckPointRecPtr = 0, 
      lastCheckPointEndPtr = 0, lastCheckPoint = {redo = 0, ThisTimeLineID = 0, PrevTimeLineID = 0, fullPageWrites = false, 
        nextXidEpoch = 0, nextXid = 0, nextOid = 0, nextMulti = 0, nextMultiOffset = 0, oldestXid = 0, oldestXidDB = 0, 
        oldestMulti = 0, oldestMultiDB = 0, time = 0, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}, 
      lastReplayedEndRecPtr = 0, lastReplayedTLI = 0, replayEndRecPtr = 0, replayEndTLI = 0, recoveryLastXTime = 0, 
      currentChunkStartTime = 0, recoveryPause = false, lastFpwDisableRecPtr = 0, info_lck = 0 '\000'}
    (gdb) p *Insert
    $4 = {insertpos_lck = 0 '\000', CurrBytePos = 5505269968, PrevBytePos = 5505269928, pad = '\000' <repeats 127 times>, 
      RedoRecPtr = 5521450856, forcePageWrites = false, fullPageWrites = true, exclusiveBackupState = EXCLUSIVE_BACKUP_NONE, 
      nonExclusiveBackups = 0, lastBackupStart = 0, WALInsertLocks = 0x7fadf8f74100}
    (gdb)   
    

RedoRecPtr = 5521450856,这是REDO point,与pg_control文件中的值一致

    
    
    [xdb@localhost ~]$ echo "obase=16;ibase=10;5521450856"|bc
    1491AA768
    [xdb@localhost ~]$ pg_controldata|grep REDO
    Latest checkpoint's REDO location:    1/491AA768
    Latest checkpoint's REDO WAL file:    000000010000000100000049
    [xdb@localhost ~]$ 
    

在进入critical section前,初始化InitXLogInsert工作空间.  
请求CheckpointLock确保在同一时刻只能存在一个checkpoint.

    
    
    (gdb) n
    8683            shutdown = false;
    (gdb) 
    8686        if (RecoveryInProgress() && (flags & CHECKPOINT_END_OF_RECOVERY) == 0)
    (gdb) 
    8697        InitXLogInsert();
    (gdb) 
    8705        LWLockAcquire(CheckpointLock, LW_EXCLUSIVE);
    (gdb) 
    8714        MemSet(&CheckpointStats, 0, sizeof(CheckpointStats));
    (gdb) 
    8715        CheckpointStats.ckpt_start_t = GetCurrentTimestamp();
    (gdb) 
    

进入critical section,让smgr(资源管理器)为checkpoint作准备.

    
    
    8720        START_CRIT_SECTION();
    (gdb) 
    (gdb) 
    8722        if (shutdown)
    (gdb) 
    8736        smgrpreckpt();
    (gdb) 
    8739        MemSet(&checkPoint, 0, sizeof(checkPoint));
    (gdb) 
    

开始填充Checkpoint XLOG Record

    
    
    (gdb) 
    8740        checkPoint.time = (pg_time_t) time(NULL);
    (gdb) p checkPoint
    $5 = {redo = 0, ThisTimeLineID = 0, PrevTimeLineID = 0, fullPageWrites = false, nextXidEpoch = 0, nextXid = 0, nextOid = 0, 
      nextMulti = 0, nextMultiOffset = 0, oldestXid = 0, oldestXidDB = 0, oldestMulti = 0, oldestMultiDB = 0, time = 0, 
      oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}
    (gdb) n
    8747        if (!shutdown && XLogStandbyInfoActive())
    (gdb) 
    8750            checkPoint.oldestActiveXid = InvalidTransactionId;
    
    

在请求插入locks前,获取最后一个重要的XLOG Record的位置.

    
    
    (gdb) 
    8756        last_important_lsn = GetLastImportantRecPtr();
    (gdb) 
    8762        WALInsertLockAcquireExclusive();
    (gdb) 
    (gdb) p last_important_lsn
    $6 = 5521451352 --> 0x1491AA958
    

在检查插入状态确定checkpoint的REDO pointer时,必须阻塞同步插入操作.

    
    
    (gdb) n
    8763        curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);
    (gdb) 
    8770        if ((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY |
    (gdb) p curInsert
    $7 = 5521451392 --> 0x1491AA980
    (gdb) 
    

继续填充Checkpoint XLOG Record

    
    
    (gdb) n
    8790        if (flags & CHECKPOINT_END_OF_RECOVERY)
    (gdb) 
    8793        checkPoint.ThisTimeLineID = ThisTimeLineID;
    (gdb) 
    8794        if (flags & CHECKPOINT_END_OF_RECOVERY)
    (gdb) 
    8797            checkPoint.PrevTimeLineID = ThisTimeLineID;
    (gdb) p ThisTimeLineID
    $8 = 1
    (gdb) n
    8799        checkPoint.fullPageWrites = Insert->fullPageWrites;
    (gdb) 
    8809        freespace = INSERT_FREESPACE(curInsert);
    (gdb) 
    8810        if (freespace == 0)
    (gdb) p freespace
    $9 = 5760
    (gdb) n
    8817        checkPoint.redo = curInsert;
    (gdb) 
    8830        RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;
    (gdb) 
    (gdb) p checkPoint
    $10 = {redo = 5521451392, ThisTimeLineID = 1, PrevTimeLineID = 1, fullPageWrites = true, nextXidEpoch = 0, nextXid = 0, 
      nextOid = 0, nextMulti = 0, nextMultiOffset = 0, oldestXid = 0, oldestXidDB = 0, oldestMulti = 0, oldestMultiDB = 0, 
      time = 1546933255, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}
    (gdb) 
    

更新共享的RedoRecPtr以备将来的XLogInsert调用,必须在持有所有插入锁才能完成。

    
    
    (gdb) n
    8836        WALInsertLockRelease();
    (gdb) 
    8839        SpinLockAcquire(&XLogCtl->info_lck);
    (gdb) 
    8840        XLogCtl->RedoRecPtr = checkPoint.redo;
    (gdb) 
    8841        SpinLockRelease(&XLogCtl->info_lck);
    (gdb) 
    8847        if (log_checkpoints)
    (gdb) 
    (gdb) p XLogCtl->RedoRecPtr
    $11 = 5521451392
    

获取其他组装checkpoint记录的信息.

    
    
    (gdb) n
    8850        TRACE_POSTGRESQL_CHECKPOINT_START(flags);
    (gdb) 
    8860        LWLockAcquire(XidGenLock, LW_SHARED);
    (gdb) 
    8861        checkPoint.nextXid = ShmemVariableCache->nextXid;
    (gdb) 
    8862        checkPoint.oldestXid = ShmemVariableCache->oldestXid;
    (gdb) 
    8863        checkPoint.oldestXidDB = ShmemVariableCache->oldestXidDB;
    (gdb) 
    8864        LWLockRelease(XidGenLock);
    (gdb) 
    8866        LWLockAcquire(CommitTsLock, LW_SHARED);
    (gdb) 
    8867        checkPoint.oldestCommitTsXid = ShmemVariableCache->oldestCommitTsXid;
    (gdb) 
    8868        checkPoint.newestCommitTsXid = ShmemVariableCache->newestCommitTsXid;
    (gdb) 
    8869        LWLockRelease(CommitTsLock);
    (gdb) 
    8872        checkPoint.nextXidEpoch = ControlFile->checkPointCopy.nextXidEpoch;
    (gdb) n
    8873        if (checkPoint.nextXid < ControlFile->checkPointCopy.nextXid)
    (gdb) 
    8876        LWLockAcquire(OidGenLock, LW_SHARED);
    (gdb) 
    8877        checkPoint.nextOid = ShmemVariableCache->nextOid;
    (gdb) p checkPoint
    $13 = {redo = 5521451392, ThisTimeLineID = 1, PrevTimeLineID = 1, fullPageWrites = true, nextXidEpoch = 0, nextXid = 2308, 
      nextOid = 0, nextMulti = 0, nextMultiOffset = 0, oldestXid = 561, oldestXidDB = 16400, oldestMulti = 0, 
      oldestMultiDB = 0, time = 1546933255, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}
    (gdb) n
    8878        if (!shutdown)
    (gdb) 
    8879            checkPoint.nextOid += ShmemVariableCache->oidCount;
    (gdb) 
    8880        LWLockRelease(OidGenLock);
    (gdb) p *ShmemVariableCache
    $14 = {nextOid = 42575, oidCount = 8189, nextXid = 2308, oldestXid = 561, xidVacLimit = 200000561, 
      xidWarnLimit = 2136484208, xidStopLimit = 2146484208, xidWrapLimit = 2147484208, oldestXidDB = 16400, 
      oldestCommitTsXid = 0, newestCommitTsXid = 0, latestCompletedXid = 2307, oldestClogXid = 561}
    (gdb) n
    8882        MultiXactGetCheckptMulti(shutdown,
    (gdb) 
    

再次查看checkpoint结构体

    
    
    (gdb) p checkPoint
    $15 = {redo = 5521451392, ThisTimeLineID = 1, PrevTimeLineID = 1, fullPageWrites = true, nextXidEpoch = 0, nextXid = 2308, 
      nextOid = 50764, nextMulti = 1, nextMultiOffset = 0, oldestXid = 561, oldestXidDB = 16400, oldestMulti = 1, 
      oldestMultiDB = 16402, time = 1546933255, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}
    (gdb) 
    

结束CRIT_SECTION

    
    
    (gdb) 
    8896        END_CRIT_SECTION();
    

获取虚拟事务ID(无效的信息)

    
    
    (gdb) n
    8927        vxids = GetVirtualXIDsDelayingChkpt(&nvxids);
    (gdb) 
    8928        if (nvxids > 0)
    (gdb) p vxids
    $16 = (VirtualTransactionId *) 0x2f4eb20
    (gdb) p *vxids
    $17 = {backendId = 2139062143, localTransactionId = 2139062143}
    (gdb) p nvxids
    $18 = 0
    (gdb) 
    (gdb) n
    8935        pfree(vxids);
    (gdb) 
    

把共享内存中的数据刷到磁盘上,并执行fsync

    
    
    (gdb) 
    8937        CheckPointGuts(checkPoint.redo, flags);
    (gdb) p flags
    $19 = 44
    (gdb) n
    8947        if (!shutdown && XLogStandbyInfoActive())
    (gdb) 
    

进入critical section.

    
    
    (gdb) n
    8950        START_CRIT_SECTION();
    (gdb) 
    

现在可以插入checkpoint record到XLOG中了.

    
    
    (gdb) 
    8955        XLogBeginInsert();
    (gdb) n
    8956        XLogRegisterData((char *) (&checkPoint), sizeof(checkPoint));
    (gdb) 
    8957        recptr = XLogInsert(RM_XLOG_ID,
    (gdb) 
    8961        XLogFlush(recptr);
    (gdb) 
    8970        if (shutdown)
    (gdb) 
    

更新控制文件(pg_control),首先为UpdateCheckPointDistanceEstimate()记录上一个checkpoint的REDO
ptr

    
    
    (gdb) 
    8982        if (shutdown && checkPoint.redo != ProcLastRecPtr)
    (gdb) 
    8990        PriorRedoPtr = ControlFile->checkPointCopy.redo;
    (gdb) 
    8995        LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
    (gdb) p ControlFile->checkPointCopy.redo
    $20 = 5521450856
    (gdb) n
    8996        if (shutdown)
    (gdb) 
    8998        ControlFile->checkPoint = ProcLastRecPtr;
    (gdb) 
    8999        ControlFile->checkPointCopy = checkPoint;
    (gdb) 
    9000        ControlFile->time = (pg_time_t) time(NULL);
    (gdb) 
    9002        ControlFile->minRecoveryPoint = InvalidXLogRecPtr;
    (gdb) 
    9003        ControlFile->minRecoveryPointTLI = 0;
    (gdb) 
    9010        SpinLockAcquire(&XLogCtl->ulsn_lck);
    (gdb) 
    9011        ControlFile->unloggedLSN = XLogCtl->unloggedLSN;
    (gdb) 
    9012        SpinLockRelease(&XLogCtl->ulsn_lck);
    (gdb) 
    9014        UpdateControlFile();
    (gdb) 
    9015        LWLockRelease(ControlFileLock);
    (gdb) 
    9018        SpinLockAcquire(&XLogCtl->info_lck);
    (gdb) p *ControlFile
    $21 = {system_identifier = 6624362124887945794, pg_control_version = 1100, catalog_version_no = 201809051, 
      state = DB_IN_PRODUCTION, time = 1546934255, checkPoint = 5521451392, checkPointCopy = {redo = 5521451392, 
        ThisTimeLineID = 1, PrevTimeLineID = 1, fullPageWrites = true, nextXidEpoch = 0, nextXid = 2308, nextOid = 50764, 
        nextMulti = 1, nextMultiOffset = 0, oldestXid = 561, oldestXidDB = 16400, oldestMulti = 1, oldestMultiDB = 16402, 
        time = 1546933255, oldestCommitTsXid = 0, newestCommitTsXid = 0, oldestActiveXid = 0}, unloggedLSN = 1, 
      minRecoveryPoint = 0, minRecoveryPointTLI = 0, backupStartPoint = 0, backupEndPoint = 0, backupEndRequired = false, 
      wal_level = 0, wal_log_hints = false, MaxConnections = 100, max_worker_processes = 8, max_prepared_xacts = 0, 
      max_locks_per_xact = 64, track_commit_timestamp = false, maxAlign = 8, floatFormat = 1234567, blcksz = 8192, 
      relseg_size = 131072, xlog_blcksz = 8192, xlog_seg_size = 16777216, nameDataLen = 64, indexMaxKeys = 32, 
      toast_max_chunk_size = 1996, loblksize = 2048, float4ByVal = true, float8ByVal = true, data_checksum_version = 0, 
      mock_authentication_nonce = "\220\277\067Vg\003\205\232U{\177 h\216\271D\266\063[\\=6\365S\tA\353\361ߧw\301", 
      crc = 930305687}
    (gdb) 
    

更新checkpoint XID/epoch的共享内存拷贝,退出critical
section,并让smgr执行checkpoint收尾工作(比如删除旧文件等).

    
    
    (gdb) n
    9019        XLogCtl->ckptXidEpoch = checkPoint.nextXidEpoch;
    (gdb) 
    9020        XLogCtl->ckptXid = checkPoint.nextXid;
    (gdb) 
    9021        SpinLockRelease(&XLogCtl->info_lck);
    (gdb) 
    9027        END_CRIT_SECTION();
    (gdb) 
    9032        smgrpostckpt();
    (gdb) 
    

删除旧的日志文件，这些文件自最后一个检查点后已不再需要，以防止保存xlog的磁盘撑满。

    
    
    (gdb) n
    9038        if (PriorRedoPtr != InvalidXLogRecPtr)
    (gdb) p PriorRedoPtr
    $23 = 5521450856
    (gdb) n
    9039            UpdateCheckPointDistanceEstimate(RedoRecPtr - PriorRedoPtr);
    (gdb) 
    9045        XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
    (gdb) 
    9046        KeepLogSeg(recptr, &_logSegNo);
    (gdb) p RedoRecPtr
    $24 = 5521451392
    (gdb) p _logSegNo
    $25 = 329
    (gdb) p wal_segment_size
    $26 = 16777216
    (gdb) n
    9047        _logSegNo--;
    (gdb) 
    9048        RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr);
    (gdb) 
    9054        if (!shutdown)
    (gdb) p recptr
    $27 = 5521451504
    (gdb) 
    

执行其他相关收尾工作

    
    
    (gdb) n
    9055            PreallocXlogFiles(recptr);
    (gdb) 
    9064        if (!RecoveryInProgress())
    (gdb) 
    9065            TruncateSUBTRANS(GetOldestXmin(NULL, PROCARRAY_FLAGS_DEFAULT));
    (gdb) 
    9068        LogCheckpointEnd(false);
    (gdb) 
    9070        TRACE_POSTGRESQL_CHECKPOINT_DONE(CheckpointStats.ckpt_bufs_written,
    (gdb) 
    9076        LWLockRelease(CheckpointLock);
    (gdb) 
    9077    }
    (gdb) 
    

完成调用

    
    
    (gdb) 
    CheckpointerMain () at checkpointer.c:488
    488                 ckpt_performed = true;
    (gdb) 
    

DONE!

### 四、参考资料

[checkpointer.c](https://doxygen.postgresql.org/checkpointer_8c_source.html)

