本节简单介绍了PostgreSQL的后台进程:checkpointer,包括相关的数据结构和CheckpointerMain函数。

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
    
    

### 二、源码解读

CheckpointerMain函数是checkpointer进程的入口.  
该函数首先为信号设置控制器(如熟悉Java
OO开发,对这样的写法应不陌生),然后创建进程的内存上下文,接着进入循环(forever),在"合适"的时候执行checkpoint.

    
    
    /*
     * Main entry point for checkpointer process
     * checkpointer进程的入口.
     * 
     * This is invoked from AuxiliaryProcessMain, which has already created the
     * basic execution environment, but not enabled signals yet.
     * 在AuxiliaryProcessMain中调用,已创建了基本的运行环境,但尚未启用信号.
     */
    void
    CheckpointerMain(void)
    {
        sigjmp_buf  local_sigjmp_buf;
        MemoryContext checkpointer_context;
    
        CheckpointerShmem->checkpointer_pid = MyProcPid;
    
        //为信号设置控制器(如熟悉Java OO开发,对这样的写法应不陌生)
        /*
         * Properly accept or ignore signals the postmaster might send us
         * 接收或忽略postmaster进程可能发给我们的信息.
         *
         * Note: we deliberately ignore SIGTERM, because during a standard Unix
         * system shutdown cycle, init will SIGTERM all processes at once.  We
         * want to wait for the backends to exit, whereupon the postmaster will
         * tell us it's okay to shut down (via SIGUSR2).
         * 注意:我们有意忽略SIGTERM，因为在标准的Unix系统关闭周期中，
         *   init将同时SIGTERM所有进程。
         * 我们希望等待后台进程退出，然后postmaster会通知checkpointer进程可以关闭(通过SIGUSR2)。
         */
        //设置标志,读取配置文件
        pqsignal(SIGHUP, ChkptSigHupHandler);   /* set flag to read config file */
        //请求checkpoint
        pqsignal(SIGINT, ReqCheckpointHandler); /* request checkpoint */
        //忽略SIGTERM
        pqsignal(SIGTERM, SIG_IGN); /* ignore SIGTERM */
        //宕机
        pqsignal(SIGQUIT, chkpt_quickdie);  /* hard crash time */
        //忽略SIGALRM & SIGPIPE
        pqsignal(SIGALRM, SIG_IGN);
        pqsignal(SIGPIPE, SIG_IGN);
        pqsignal(SIGUSR1, chkpt_sigusr1_handler);
        //请求关闭
        pqsignal(SIGUSR2, ReqShutdownHandler);  /* request shutdown */
    
        /*
         * Reset some signals that are accepted by postmaster but not here
         * 重置某些postmaster接收而不是在这里的信号
         */
        pqsignal(SIGCHLD, SIG_DFL);
    
        /* We allow SIGQUIT (quickdie) at all times */
        //运行SIGQUIT信号
        sigdelset(&BlockSig, SIGQUIT);
    
        /*
         * Initialize so that first time-driven event happens at the correct time.
         * 初始化以便时间驱动的事件在正确的时间发生.
         */
        last_checkpoint_time = last_xlog_switch_time = (pg_time_t) time(NULL);
    
        /*
         * Create a memory context that we will do all our work in.  We do this so
         * that we can reset the context during error recovery and thereby avoid
         * possible memory leaks.  Formerly this code just ran in
         * TopMemoryContext, but resetting that would be a really bad idea.
         * 创建进程的内存上下文.
         * 之所以这样做是我们可以在出现异常执行恢复期间重置上下文以确保不会出现内存泄漏.
         * 
         */
        checkpointer_context = AllocSetContextCreate(TopMemoryContext,
                                                     "Checkpointer",
                                                     ALLOCSET_DEFAULT_SIZES);
        MemoryContextSwitchTo(checkpointer_context);
    
        /*
         * If an exception is encountered, processing resumes here.
         * 如出现异常,在这里处理恢复.
         * 
         * See notes in postgres.c about the design of this coding.
         * 这部分的设计可参照postgres.c中的注释
         */
        if (sigsetjmp(local_sigjmp_buf, 1) != 0)
        {
            /* Since not using PG_TRY, must reset error stack by hand */
            //没有使用PG_TRY,必须重置错误栈
            error_context_stack = NULL;
    
            /* Prevent interrupts while cleaning up */
            //在清除期间必须避免中断
            HOLD_INTERRUPTS();
    
            /* Report the error to the server log */
            //在日志中报告错误信息
            EmitErrorReport();
    
            /*
             * These operations are really just a minimal subset of
             * AbortTransaction().  We don't have very many resources to worry
             * about in checkpointer, but we do have LWLocks, buffers, and temp
             * files.
             * 这些操作实际上只是AbortTransaction()的最小集合.
             * 我们不需要耗费太多的资源在checkpointer进程上,但需要持有LWLocks/缓存和临时文件
             */
            LWLockReleaseAll();
            ConditionVariableCancelSleep();
            pgstat_report_wait_end();
            AbortBufferIO();
            UnlockBuffers();
            ReleaseAuxProcessResources(false);
            AtEOXact_Buffers(false);
            AtEOXact_SMgr();
            AtEOXact_Files(false);
            AtEOXact_HashTables(false);
    
            /* Warn any waiting backends that the checkpoint failed. */
            //通知正在等待的后台进程:checkpoint执行失败
            if (ckpt_active)
            {
                SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
                CheckpointerShmem->ckpt_failed++;
                CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
                SpinLockRelease(&CheckpointerShmem->ckpt_lck);
    
                ckpt_active = false;
            }
    
            /*
             * Now return to normal top-level context and clear ErrorContext for
             * next time.
             * 回到常规的顶层上下文,为下一次checkpoint清空ErrorContext
             */
            MemoryContextSwitchTo(checkpointer_context);
            FlushErrorState();
    
            /* Flush any leaked data in the top-level context */
            //在顶层上下文刷新泄漏的数据
            MemoryContextResetAndDeleteChildren(checkpointer_context);
    
            /* Now we can allow interrupts again */
            //现在我们可以允许中断了
            RESUME_INTERRUPTS();
    
            /*
             * Sleep at least 1 second after any error.  A write error is likely
             * to be repeated, and we don't want to be filling the error logs as
             * fast as we can.
             * 出现错误后,至少休眠1s.
             * 写入错误可能会重复出现,但我们不希望频繁出现错误日志,因此需要休眠1s.
             */
            pg_usleep(1000000L);
    
            /*
             * Close all open files after any error.  This is helpful on Windows,
             * where holding deleted files open causes various strange errors.
             * It's not clear we need it elsewhere, but shouldn't hurt.
             * 出现错误后,关闭所有打开的文件句柄.
             * 尤其在Windows平台,仍持有已删除的文件句柄会导致莫名其妙的错误.
             * 目前还不清楚我们是否需要在其他地方使用它，但这不会导致其他额外的问题。
             */
            smgrcloseall();
        }
    
        /* We can now handle ereport(ERROR) */
        //现在可以处理ereport(ERROR)调用了.
        PG_exception_stack = &local_sigjmp_buf;
    
        /*
         * Unblock signals (they were blocked when the postmaster forked us)
         * 解锁信号(在postmaster fork进程的时候,会阻塞信号)
         */
        PG_SETMASK(&UnBlockSig);
    
        /*
         * Ensure all shared memory values are set correctly for the config. Doing
         * this here ensures no race conditions from other concurrent updaters.
         * 确保所有的共享内存变量已正确配置.
         * 在这里执行这样的检查确保不存在来自其他并发更新进程的竞争条件.
         */
        UpdateSharedMemoryConfig();
    
        /*
         * Advertise our latch that backends can use to wake us up while we're
         * sleeping.
         * 广播本进程的latch,在进程休眠时其他进程可以使用此latch唤醒.
         */
        ProcGlobal->checkpointerLatch = &MyProc->procLatch;
    
        /*
         * Loop forever
         * 循环,循环,循环...
         */
        for (;;)
        {
            bool        do_checkpoint = false;//是否执行checkpoint
            int         flags = 0;//标记
            pg_time_t   now;//时间
            int         elapsed_secs;//已消逝的时间
            int         cur_timeout;//timeout时间
    
            /* Clear any already-pending wakeups */
            ResetLatch(MyLatch);
    
            /*
             * Process any requests or signals received recently.
             * 处理最近接收到的请求或信号
             */
            AbsorbFsyncRequests();
    
            if (got_SIGHUP)//
            {
                got_SIGHUP = false;
                ProcessConfigFile(PGC_SIGHUP);
    
                /*
                 * Checkpointer is the last process to shut down, so we ask it to
                 * hold the keys for a range of other tasks required most of which
                 * have nothing to do with checkpointing at all.
                 * Checkpointer是最后一个关闭的进程,因此我们要求它保存一些一系列其他任务需要的键值,
                 *   虽然其中大部分任务与检查点完全无关.
                 *
                 * For various reasons, some config values can change dynamically
                 * so the primary copy of them is held in shared memory to make
                 * sure all backends see the same value.  We make Checkpointer
                 * responsible for updating the shared memory copy if the
                 * parameter setting changes because of SIGHUP.
                 * 由于各种原因,某些配置项可以动态修改,
                 *   因此这些配置项的拷贝在共享内存中存储以确保所有的后台进程看到的值是一样的.
                 * 如果参数设置是因为SIGHUP引起的,那么我们让Checkpointer进程负责更新共享内存中的配置项拷贝.
                 */
                UpdateSharedMemoryConfig();
            }
            if (checkpoint_requested)
            {
                //接收到checkpoint请求
                checkpoint_requested = false;//重置标志
                do_checkpoint = true;//需要执行checkpoint
                BgWriterStats.m_requested_checkpoints++;//计数
            }
            if (shutdown_requested)
            {
                //接收到关闭请求
                /*
                 * From here on, elog(ERROR) should end with exit(1), not send
                 * control back to the sigsetjmp block above
                 * 从这里开始，日志(错误)应该以exit(1)结束，而不是将控制发送回上面的sigsetjmp块
                 */
                ExitOnAnyError = true;
                /* Close down the database */
                //关闭数据库
                ShutdownXLOG(0, 0);
                /* Normal exit from the checkpointer is here */
                //checkpointer在这里正常退出
                proc_exit(0);       /* done */
            }
    
            /*
             * Force a checkpoint if too much time has elapsed since the last one.
             * Note that we count a timed checkpoint in stats only when this
             * occurs without an external request, but we set the CAUSE_TIME flag
             * bit even if there is also an external request.
             * 在上次checkpoint后,已超时,则执行checkpoint.
             * 注意，只有在没有外部请求的情况下，我们才会在统计数据中计算定时检查点,
             *   但计算出现;了外部请求,我们也会设置CAUSE_TIME标志位.
             */
            now = (pg_time_t) time(NULL);//当前时间
            elapsed_secs = now - last_checkpoint_time;//已消逝的时间
            if (elapsed_secs >= CheckPointTimeout)
            {
                //超时
                if (!do_checkpoint)
                    BgWriterStats.m_timed_checkpoints++;//没有接收到checkpoint请求,进行统计
                do_checkpoint = true;//设置标记
                flags |= CHECKPOINT_CAUSE_TIME;//设置标记
            }
    
            /*
             * Do a checkpoint if requested.
             * 执行checkpoint
             */
            if (do_checkpoint)
            {
                bool        ckpt_performed = false;//设置标记
                bool        do_restartpoint;
    
                /*
                 * Check if we should perform a checkpoint or a restartpoint. As a
                 * side-effect, RecoveryInProgress() initializes TimeLineID if
                 * it's not set yet.
                 * 检查我们是否需要执行checkpoint或restartpoint.
                 * 可能的其他影响是,如仍未设置TimeLineID,那么RecoveryInProgress()会初始化TimeLineID
                 */
                do_restartpoint = RecoveryInProgress();
    
                /*
                 * Atomically fetch the request flags to figure out what kind of a
                 * checkpoint we should perform, and increase the started-counter
                 * to acknowledge that we've started a new checkpoint.
                 * 自动提取请求标志,以决定那种checkpoint需要执行,同时增加开始计数已确认我们已启动了新的checkpoint.
                 */
                SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
                flags |= CheckpointerShmem->ckpt_flags;
                CheckpointerShmem->ckpt_flags = 0;
                CheckpointerShmem->ckpt_started++;
                SpinLockRelease(&CheckpointerShmem->ckpt_lck);
    
                /*
                 * The end-of-recovery checkpoint is a real checkpoint that's
                 * performed while we're still in recovery.
                 * end-of-recovery checkpoint是在数据库恢复过程中执行的checkpoint.
                 */
                if (flags & CHECKPOINT_END_OF_RECOVERY)
                    do_restartpoint = false;
    
                /*
                 * We will warn if (a) too soon since last checkpoint (whatever
                 * caused it) and (b) somebody set the CHECKPOINT_CAUSE_XLOG flag
                 * since the last checkpoint start.  Note in particular that this
                 * implementation will not generate warnings caused by
                 * CheckPointTimeout < CheckPointWarning.
                 * 如果checkpoint发生的太频繁(不管是什么原因)
                 *   或者在上次checkpoint启动后某个进程设置了CHECKPOINT_CAUSE_XLOG标志,
                 * 我们都会发出警告.
                 * 请特别注意，此实现不会生成由CheckPointTimeout < CheckPointWarning引起的警告。
                 */
                if (!do_restartpoint &&
                    (flags & CHECKPOINT_CAUSE_XLOG) &&
                    elapsed_secs < CheckPointWarning)
                    ereport(LOG,
                            (errmsg_plural("checkpoints are occurring too frequently (%d second apart)",
                                           "checkpoints are occurring too frequently (%d seconds apart)",
                                           elapsed_secs,
                                           elapsed_secs),
                             errhint("Consider increasing the configuration parameter \"max_wal_size\".")));
    
                /*
                 * Initialize checkpointer-private variables used during
                 * checkpoint.
                 * 初始化checkpointer进程在checkpoint过程中需使用的私有变量
                 */
                ckpt_active = true;
                if (do_restartpoint)
                    //执行restartpoint
                    ckpt_start_recptr = GetXLogReplayRecPtr(NULL);//获取Redo pint
                else
                    //执行checkpoint
                    ckpt_start_recptr = GetInsertRecPtr();//获取checkpoint XLOG Record插入的位置
                ckpt_start_time = now;//开始时间
                ckpt_cached_elapsed = 0;//消逝时间
    
                /*
                 * Do the checkpoint.
                 * 执行checkpoint.
                 */
                if (!do_restartpoint)
                {
                    //执行checkpoint
                    CreateCheckPoint(flags);//创建checkpoint
                    ckpt_performed = true;//DONE!
                }
                else
                    //恢复过程的restartpoint
                    ckpt_performed = CreateRestartPoint(flags);
    
                /*
                 * After any checkpoint, close all smgr files.  This is so we
                 * won't hang onto smgr references to deleted files indefinitely.
                 * 执行checkpoint完成后,关闭所有的smgr文件.
                 * 这样我们就不需要无限期的持有已删除文件的smgr引用.
                 */
                smgrcloseall();
    
                /*
                 * Indicate checkpoint completion to any waiting backends.
                 * 通知等待的进程,checkpoint完成.
                 */
                SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
                CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
                SpinLockRelease(&CheckpointerShmem->ckpt_lck);
    
                if (ckpt_performed)
                {
                    //已完成checkpoint
                    /*
                     * Note we record the checkpoint start time not end time as
                     * last_checkpoint_time.  This is so that time-driven
                     * checkpoints happen at a predictable spacing.
                     * 注意我们记录了checkpoint的开始时间而不是结束时间作为last_checkpoint_time.
                     * 这样，时间驱动的检查点就会以可预测的间隔出现。
                     */
                    last_checkpoint_time = now;
                }
                else
                {
                    //
                    /*
                     * We were not able to perform the restartpoint (checkpoints
                     * throw an ERROR in case of error).  Most likely because we
                     * have not received any new checkpoint WAL records since the
                     * last restartpoint. Try again in 15 s.
                     * 没有成功执行restartpoint(如果是checkpoint出现问题会直接报错,不会进入到这里).
                     * 最有可能的原因是因为在上次restartpoint后没有接收到新的checkpoint WAL记录.
                     * 15s后尝试.
                     */
                    last_checkpoint_time = now - CheckPointTimeout + 15;
                }
    
                ckpt_active = false;
            }
    
            /* Check for archive_timeout and switch xlog files if necessary. */
            //在需要的时候,检查archive_timeout并切换xlog文件.
            CheckArchiveTimeout();
    
            /*
             * Send off activity statistics to the stats collector.  (The reason
             * why we re-use bgwriter-related code for this is that the bgwriter
             * and checkpointer used to be just one process.  It's probably not
             * worth the trouble to split the stats support into two independent
             * stats message types.)
             * 发送活动统计到统计收集器.
             */
            pgstat_send_bgwriter();
    
            /*
             * Sleep until we are signaled or it's time for another checkpoint or
             * xlog file switch.
             * 休眠,直至接收到信号或者需要启动新的checkpoint或xlog文件切换.
             */
            //重置相关变量
            now = (pg_time_t) time(NULL);
            elapsed_secs = now - last_checkpoint_time;
            if (elapsed_secs >= CheckPointTimeout)
                continue;           /* no sleep for us ... */
            cur_timeout = CheckPointTimeout - elapsed_secs;
            if (XLogArchiveTimeout > 0 && !RecoveryInProgress())
            {
                elapsed_secs = now - last_xlog_switch_time;
                if (elapsed_secs >= XLogArchiveTimeout)
                    continue;       /* no sleep for us ... */
                cur_timeout = Min(cur_timeout, XLogArchiveTimeout - elapsed_secs);//获得最小休眠时间
            }
    
            (void) WaitLatch(MyLatch,
                             WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
                             cur_timeout * 1000L /* convert to ms */ ,
                             WAIT_EVENT_CHECKPOINTER_MAIN);//休眠
        }
    }
     
    
    /*
     * Unix-like signal handler installation
     * Unix风格的信号处理器
     * Only called on main thread, no sync required
     * 只需要在主线程执行,不需要sync同步.
     */
    pqsigfunc
    pqsignal(int signum, pqsigfunc handler)
    {
        pqsigfunc   prevfunc;//函数
    
        if (signum >= PG_SIGNAL_COUNT || signum < 0)
            return SIG_ERR;//验证不通过,返回错误
        prevfunc = pg_signal_array[signum];//获取先前的处理函数
        pg_signal_array[signum] = handler;//注册函数
        return prevfunc;//返回先前注册的函数
    }
    
    
    /*
     * GetInsertRecPtr -- Returns the current insert position.
     * 返回当前插入位置
     *
     * NOTE: The value *actually* returned is the position of the last full
     * xlog page. It lags behind the real insert position by at most 1 page.
     * For that, we don't need to scan through WAL insertion locks, and an
     * approximation is enough for the current usage of this function.
     * 注意:返回的值*实际上*是最后一个完整xlog页面的位置.
     * 它比实际插入位置最多落后1页。
     * 为此，我们不需要遍历WAL插入锁，满足该函数的当前使用目的，近似值已足够。
     */
    XLogRecPtr
    GetInsertRecPtr(void)
    {
        XLogRecPtr  recptr;
    
        SpinLockAcquire(&XLogCtl->info_lck);
        recptr = XLogCtl->LogwrtRqst.Write;//获取插入位置
        SpinLockRelease(&XLogCtl->info_lck);
    
        return recptr;
    }
    
    

### 三、跟踪分析

创建数据表,插入数据,执行checkpoint

    
    
    testdb=# drop table t_wal_ckpt;
    DROP TABLE
    testdb=# create table t_wal_ckpt(c1 int not null,c2  varchar(40),c3 varchar(40));
    CREATE TABLE
    testdb=# insert into t_wal_ckpt(c1,c2,c3) values(1,'C2-1','C3-1');
    INSERT 0 1
    testdb=# 
    testdb=# checkpoint; --> 第一次checkpoint
    

更新数据,执行checkpoint.

    
    
    testdb=# update t_wal_ckpt set c2 = 'C2#'||substr(c2,4,40);
    UPDATE 1
    testdb=# checkpoint;
    
    

启动gdb,设置信号控制

    
    
    (gdb) handle SIGINT print nostop pass
    SIGINT is used by the debugger.
    Are you sure you want to change it? (y or n) y
    Signal        Stop  Print Pass to program Description
    SIGINT        No  Yes Yes   Interrupt
    (gdb) 
    (gdb) b checkpointer.c:441
    Breakpoint 1 at 0x815197: file checkpointer.c, line 441.
    (gdb) c
    Continuing.
    
    Program received signal SIGINT, Interrupt.
    
    Breakpoint 1, CheckpointerMain () at checkpointer.c:441
    441       flags |= CheckpointerShmem->ckpt_flags;
    (gdb) 
    

查看共享内存信息CheckpointerShmem

    
    
    (gdb) p *CheckpointerShmem
    $1 = {checkpointer_pid = 1650, ckpt_lck = 1 '\001', ckpt_started = 2, ckpt_done = 2, ckpt_failed = 0, ckpt_flags = 44, 
      num_backend_writes = 0, num_backend_fsync = 0, num_requests = 0, max_requests = 65536, requests = 0x7f2cdda07b28}
    (gdb) 
    
    

设置相关信息CheckpointerShmem

    
    
    441       flags |= CheckpointerShmem->ckpt_flags;
    (gdb) n
    442       CheckpointerShmem->ckpt_flags = 0;
    (gdb) 
    443       CheckpointerShmem->ckpt_started++;
    (gdb) 
    444       SpinLockRelease(&CheckpointerShmem->ckpt_lck);
    (gdb) 
    450       if (flags & CHECKPOINT_END_OF_RECOVERY)
    (gdb) 
    460       if (!do_restartpoint &&
    (gdb) 
    461         (flags & CHECKPOINT_CAUSE_XLOG) &&
    (gdb) 
    460       if (!do_restartpoint &&
    
    

初始化checkpointer进程在checkpoint过程中需使用的私有变量.  
其中ckpt_start_recptr为插入点,即Redo point,5521180544转换为16进制为0x1 49168780

    
    
    (gdb) 
    474       ckpt_active = true;
    (gdb) 
    475       if (do_restartpoint)
    (gdb) 
    478         ckpt_start_recptr = GetInsertRecPtr();
    (gdb) p XLogCtl->LogwrtRqst
    $1 = {Write = 5521180544, Flush = 5521180544}
    (gdb) n
    479       ckpt_start_time = now;
    (gdb) p ckpt_start_recptr
    $2 = 5521180544
    (gdb) n
    480       ckpt_cached_elapsed = 0;
    (gdb) 
    485       if (!do_restartpoint)
    (gdb) 
    

执行checkpoint.OK!

    
    
    (gdb) 
    487         CreateCheckPoint(flags);
    (gdb) 
    488         ckpt_performed = true;
    (gdb) 
    

关闭资源,并设置共享内存中的信息

    
    
    497       smgrcloseall();
    (gdb) 
    502       SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    (gdb) 
    503       CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
    (gdb) 
    504       SpinLockRelease(&CheckpointerShmem->ckpt_lck);
    (gdb) 
    506       if (ckpt_performed)
    (gdb) p CheckpointerShmem
    $3 = (CheckpointerShmemStruct *) 0x7fcecc063b00
    (gdb) p *CheckpointerShmem
    $4 = {checkpointer_pid = 1697, ckpt_lck = 0 '\000', ckpt_started = 1, ckpt_done = 1, ckpt_failed = 0, ckpt_flags = 0, 
      num_backend_writes = 0, num_backend_fsync = 0, num_requests = 0, max_requests = 65536, requests = 0x7fcecc063b28}
    (gdb) 
    

checkpoint请求已清空

    
    
    (gdb) p CheckpointerShmem->requests[0]
    $5 = {rnode = {spcNode = 0, dbNode = 0, relNode = 0}, forknum = MAIN_FORKNUM, segno = 0}
    

在需要的时候,检查archive_timeout并切换xlog文件.  
休眠,直至接收到信号或者需要启动新的checkpoint或xlog文件切换.

    
    
    (gdb) n
    513         last_checkpoint_time = now;
    (gdb) 
    526       ckpt_active = false;
    (gdb) 
    530     CheckArchiveTimeout();
    (gdb) 
    539     pgstat_send_bgwriter();
    (gdb) 
    545     now = (pg_time_t) time(NULL);
    (gdb) 
    546     elapsed_secs = now - last_checkpoint_time;
    (gdb) 
    547     if (elapsed_secs >= CheckPointTimeout)
    (gdb) p elapsed_secs
    $7 = 1044
    (gdb) p CheckPointTimeout
    $8 = 900
    (gdb) n
    548       continue;     /* no sleep for us ... */
    

已超时,执行新的checkpoint

    
    
    (gdb) 
    569   }
    (gdb) 
    352     bool    do_checkpoint = false;
    (gdb) 
    353     int     flags = 0;
    (gdb) n
    360     ResetLatch(MyLatch);
    (gdb) 
    365     AbsorbFsyncRequests();
    (gdb) 
    367     if (got_SIGHUP)
    (gdb) 
    385     if (checkpoint_requested)
    (gdb) 
    391     if (shutdown_requested)
    (gdb) 
    410     now = (pg_time_t) time(NULL);
    (gdb) 
    411     elapsed_secs = now - last_checkpoint_time;
    (gdb) 
    412     if (elapsed_secs >= CheckPointTimeout)
    (gdb) p elapsed_secs
    $9 = 1131
    (gdb) n
    414       if (!do_checkpoint)
    (gdb) 
    415         BgWriterStats.m_timed_checkpoints++;
    (gdb) 
    416       do_checkpoint = true;
    (gdb) 
    417       flags |= CHECKPOINT_CAUSE_TIME;
    (gdb) 
    423     if (do_checkpoint)
    (gdb) 
    425       bool    ckpt_performed = false;
    (gdb) 
    433       do_restartpoint = RecoveryInProgress();
    (gdb) 
    440       SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    (gdb) 
    
    Breakpoint 1, CheckpointerMain () at checkpointer.c:441
    441       flags |= CheckpointerShmem->ckpt_flags;
    (gdb) 
    442       CheckpointerShmem->ckpt_flags = 0;
    (gdb) 
    443       CheckpointerShmem->ckpt_started++;
    (gdb) c
    Continuing.
    

DONE!

**CreateCheckPoint函数下一节介绍**

### 四、参考资料

[checkpointer.c](https://doxygen.postgresql.org/checkpointer_8c_source.html)

