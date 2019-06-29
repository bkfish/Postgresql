本节介绍了插入数据时与WAL相关的处理逻辑，主要是heap_insert->XLogInsert中依赖的函数，包括WALInsertLockAcquireExclusive、WALInsertLockAcquire和WALInsertLockRelease等。

### 一、数据结构

**静态变量**  
进程中全局共享

    
    
    static int  num_rdatas;         /* entries currently used */
    //已分配的空间大小
    static int  max_rdatas;         /* allocated size */
    //是否调用XLogBeginInsert函数
    static bool begininsert_called = false;
    
    static XLogCtlData *XLogCtl = NULL;
    
    /* flags for the in-progress insertion */
    static uint8 curinsert_flags = 0;
    
    /*
     * ProcLastRecPtr points to the start of the last XLOG record inserted by the
     * current backend.  It is updated for all inserts.  XactLastRecEnd points to
     * end+1 of the last record, and is reset when we end a top-level transaction,
     * or start a new one; so it can be used to tell if the current transaction has
     * created any XLOG records.
     * ProcLastRecPtr指向当前后端插入的最后一条XLOG记录的开头。
     * 它针对所有插入进行更新。
     * XactLastRecEnd指向最后一条记录的末尾位置 + 1，
     *   并在结束顶级事务或启动新事务时重置;
     *   因此，它可以用来判断当前事务是否创建了任何XLOG记录。
     *
     * While in parallel mode, this may not be fully up to date.  When committing,
     * a transaction can assume this covers all xlog records written either by the
     * user backend or by any parallel worker which was present at any point during
     * the transaction.  But when aborting, or when still in parallel mode, other
     * parallel backends may have written WAL records at later LSNs than the value
     * stored here.  The parallel leader advances its own copy, when necessary,
     * in WaitForParallelWorkersToFinish.
     * 在并行模式下，这可能不是完全是最新的。
     * 在提交时，事务可以假定覆盖了用户后台进程或在事务期间出现的并行worker进程的所有xlog记录。
     * 但是，当中止时，或者仍然处于并行模式时，其他并行后台进程可能在较晚的LSNs中写入了WAL记录，
     *   而不是存储在这里的值。
     * 当需要时，并行处理进程的leader在WaitForParallelWorkersToFinish中会推进自己的副本。
     */
    XLogRecPtr  ProcLastRecPtr = InvalidXLogRecPtr;
    XLogRecPtr  XactLastRecEnd = InvalidXLogRecPtr;
    XLogRecPtr XactLastCommitEnd = InvalidXLogRecPtr;
    
    /* For WALInsertLockAcquire/Release functions */
    //用于WALInsertLockAcquire/Release函数
    static int  MyLockNo = 0;
    static bool holdingAllLocks = false;
    
    
    

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
    
    //-------------------------------------------------- 锁相关
    /*
     * Fake spinlock implementation using semaphores --- slow and prone
     * to fall foul of kernel limits on number of semaphores, so don't use this
     * unless you must!  The subroutines appear in spin.c.
     * 使用信号量的伪自旋锁实现——很慢而且容易与内核对信号量的限制相冲突，
     *   所以除非必须，否则不要使用它!子例程出现在spin.c中。
     */
    typedef int slock_t;
    
    typedef uint32 pg_crc32c;
    
    #define SpinLockInit(lock)  S_INIT_LOCK(lock)
    
    #define SpinLockAcquire(lock) S_LOCK(lock)
    
    #define SpinLockRelease(lock) S_UNLOCK(lock)
    
    #define SpinLockFree(lock)  S_LOCK_FREE(lock)
    
    #define XLogSegmentOffset(xlogptr, wal_segsz_bytes) \
        ((xlogptr) & ((wal_segsz_bytes) - 1))
    
    #define LW_FLAG_HAS_WAITERS         ((uint32) 1 << 30)
    #define LW_FLAG_RELEASE_OK          ((uint32) 1 << 29)
    #define LW_FLAG_LOCKED              ((uint32) 1 << 28)
    
    #define LW_VAL_EXCLUSIVE            ((uint32) 1 << 24)
    #define LW_VAL_SHARED               1
    
    #define LW_LOCK_MASK                ((uint32) ((1 << 25)-1))
    /* Must be greater than MAX_BACKENDS - which is 2^23-1, so we're fine. */
    #define LW_SHARED_MASK              ((uint32) ((1 << 24)-1))
    

**LWLock**  
lwlock.c外的代码不应直接操作这个结构的内容,但我们必须声明该结构体以便将LWLocks合并到其他数据结构中。

    
    
    /*
     * Code outside of lwlock.c should not manipulate the contents of this
     * structure directly, but we have to declare it here to allow LWLocks to be
     * incorporated into other data structures.
     * lwlock.c外的代码不应直接操作这个结构的内容,
     *   但我们必须声明该结构体以便将LWLocks合并到其他数据结构中。
     */
    typedef struct LWLock
    {
        uint16      tranche;        /* tranche ID */
        //独占/非独占locker的状态
        pg_atomic_uint32 state;     /* state of exclusive/nonexclusive lockers */
        //正在等待的PGPROCs链表
        proclist_head waiters;      /* list of waiting PGPROCs */
    #ifdef LOCK_DEBUG//用于DEBUG
        //waiters的数量
        pg_atomic_uint32 nwaiters;  /* number of waiters */
        //锁的最后独占者
        struct PGPROC *owner;       /* last exclusive owner of the lock */
    #endif
    } LWLock;
    
    

### 二、源码解读

**heap_insert**  
主要实现逻辑是插入元组到堆中,其中存在对WAL(XLog)进行处理的部分.  
参见[PostgreSQL 源码解读（104）- WAL#1（Insert & WAL-heap_insert函数#1）](https://www.jianshu.com/p/70f019643e76)

**XLogInsert/XLogInsertRecord**  
插入一个具有指定的RMID和info字节的XLOG记录，该记录的主体是先前通过XLogRegister*调用注册的数据和缓冲区引用。  
参见[PostgreSQL 源码解读（106）- WAL#3（Insert & WAL-heap_insert函数#3）](https://www.jianshu.com/p/349ddcb0b1c2)

**WALInsertLockXXX**  
包括WALInsertLockAcquireExclusive、WALInsertLockAcquire和WALInsertLockRelease等

    
    
    //----------------------------------------------------------- WALInsertLockAcquireExclusive
    /*
     * Acquire all WAL insertion locks, to prevent other backends from inserting
     * to WAL.
     * 请求所有的WAL insertion锁,以避免其他后台进程插入数据到WAL中
     */
    static void
    WALInsertLockAcquireExclusive(void)
    {
        int         i;
    
        /*
         * When holding all the locks, all but the last lock's insertingAt
         * indicator is set to 0xFFFFFFFFFFFFFFFF, which is higher than any real
         * XLogRecPtr value, to make sure that no-one blocks waiting on those.
         * 在持有所有的locks时,除了最后一个锁的insertingAt指示器外,
         *   其余均设置为0xFFFFFFFFFFFFFFFF,
         *   该值比所有实际的XLogRecPtr都要大,以确保没有阻塞这些锁。.
         */
        for (i = 0; i < NUM_XLOGINSERT_LOCKS - 1; i++)//NUM_XLOGINSERT_LOCKS
        {
            LWLockAcquire(&WALInsertLocks[i].l.lock, LW_EXCLUSIVE);
            LWLockUpdateVar(&WALInsertLocks[i].l.lock,
                            &WALInsertLocks[i].l.insertingAt,
                            PG_UINT64_MAX);
        }
        /* Variable value reset to 0 at release */
        //在释放时,变量值重置为0
        LWLockAcquire(&WALInsertLocks[i].l.lock, LW_EXCLUSIVE);
        //设置标记
        holdingAllLocks = true;
    }
    
    
    /*
     * LWLockAcquire - acquire a lightweight lock in the specified mode
     * LWLockAcquire - 申请指定模式的轻量级锁
     *
     * If the lock is not available, sleep until it is.  Returns true if the lock
     * was available immediately, false if we had to sleep.
     * 如果锁不可用,休眠直至可用.
     * 如果锁马上可用则返回T,需要休眠则返回F
     * 
     * Side effect: cancel/die interrupts are held off until lock release.
     * 副作用:在锁释放的时候才能允许中断/终止
     */
    bool
    LWLockAcquire(LWLock *lock, LWLockMode mode)
    {
        PGPROC     *proc = MyProc;//PGPROC数据结构
        bool        result = true;
        int         extraWaits = 0;
    #ifdef LWLOCK_STATS
        lwlock_stats *lwstats;
    
        lwstats = get_lwlock_stats_entry(lock);//获得锁的统计入口
    #endif
        //模式验证
        AssertArg(mode == LW_SHARED || mode == LW_EXCLUSIVE);
    
        PRINT_LWDEBUG("LWLockAcquire", lock, mode);
    
    #ifdef LWLOCK_STATS
        /* Count lock acquisition attempts */
        if (mode == LW_EXCLUSIVE)
            lwstats->ex_acquire_count++;
        else
            lwstats->sh_acquire_count++;
    #endif                          /* LWLOCK_STATS */
    
        /*
         * We can't wait if we haven't got a PGPROC.  This should only occur
         * during bootstrap or shared memory initialization.  Put an Assert here
         * to catch unsafe coding practices.
         * 如果还没有得到PGPROC则不能等待.
         * 这种情况可能出现在bootstrap或者共享内存初始化时.
         * 在这里加入Assert代码以确保捕获不安全的编码实践.
         */
        Assert(!(proc == NULL && IsUnderPostmaster));
    
        /* Ensure we will have room to remember the lock */
        //确保我们有足够的地方存储锁
        if (num_held_lwlocks >= MAX_SIMUL_LWLOCKS)
            elog(ERROR, "too many LWLocks taken");
    
        /*
         * Lock out cancel/die interrupts until we exit the code section protected
         * by the LWLock.  This ensures that interrupts will not interfere with
         * manipulations of data structures in shared memory.
         * 退出使用LWLock锁保护的实现逻辑时才能允许取消或者中断.
         * 这样可以确保中断不会与共享内存中的数据结构管理逻辑发现关系.
         */
        HOLD_INTERRUPTS();
    
        /*
         * Loop here to try to acquire lock after each time we are signaled by
         * LWLockRelease.
         * 循环,在每次LWLockRelease信号产生时获取锁.
         *
         * NOTE: it might seem better to have LWLockRelease actually grant us the
         * lock, rather than retrying and possibly having to go back to sleep. But
         * in practice that is no good because it means a process swap for every
         * lock acquisition when two or more processes are contending for the same
         * lock.  Since LWLocks are normally used to protect not-very-long
         * sections of computation, a process needs to be able to acquire and
         * release the same lock many times during a single CPU time slice, even
         * in the presence of contention.  The efficiency of being able to do that
         * outweighs the inefficiency of sometimes wasting a process dispatch
         * cycle because the lock is not free when a released waiter finally gets
         * to run.  See pgsql-hackers archives for 29-Dec-01.
         * 注意:看起来相对于不断的重入和休眠而言LWLockRelease的实际持有者授予我们锁会更好,
         *   但在工程实践上来看,
         *   这样的做法并不好因为这意味着当两个或多个进程争用同一个锁时对每个锁都会出现进程交换.
         * 由于LWLocks通常来说用于保护并不是太长时间的计算逻辑,
         *   甚至在出现争用的时候,一个进程需要能够在一个CPU时间片期间获取和释放同样的锁很多次.
         * 那样子做的收获会导致有时候进程调度的低效,
         *   因为当一个已释放的进程终于可以运行时，锁却没有获取.
         */
        for (;;)
        {
            bool        mustwait;
    
            /*
             * Try to grab the lock the first time, we're not in the waitqueue
             * yet/anymore.
             * 第一次试着获取锁，我们已经不在等待队列中了。
             */
            mustwait = LWLockAttemptLock(lock, mode);
    
            if (!mustwait)
            {
                LOG_LWDEBUG("LWLockAcquire", lock, "immediately acquired lock");
                break;              /* 成功!got the lock */
            }
    
            /*
             * Ok, at this point we couldn't grab the lock on the first try. We
             * cannot simply queue ourselves to the end of the list and wait to be
             * woken up because by now the lock could long have been released.
             * Instead add us to the queue and try to grab the lock again. If we
             * succeed we need to revert the queuing and be happy, otherwise we
             * recheck the lock. If we still couldn't grab it, we know that the
             * other locker will see our queue entries when releasing since they
             * existed before we checked for the lock.
             * 在这个点,我们不能在第一次就获取锁.
             * 我们不能在链表的末尾进行简单的排队然后等待唤醒,因为锁可能已经释放很长时间了.
             * 相反,我们需要重新加入到队列中再次尝试获取锁.
             * 如果成功了,我们需要翻转队列,否则的话需要重新检查锁.
             * 如果还是不能获取锁,我们知道其他locker在释放时可以看到我们的队列入口,
             *   因为在我们检查锁时它们已经存在了.
             */
    
            /* add to the queue */
            //添加到队列中
            LWLockQueueSelf(lock, mode);
    
            /* we're now guaranteed to be woken up if necessary */
            //在需要的时候,确保可以被唤醒
            mustwait = LWLockAttemptLock(lock, mode);
    
            /* ok, grabbed the lock the second time round, need to undo queueing */
            //第二次尝试获取锁,需要取消排队
            if (!mustwait)
            {
                LOG_LWDEBUG("LWLockAcquire", lock, "acquired, undoing queue");
    
                LWLockDequeueSelf(lock);//出列
                break;
            }
    
            /*
             * Wait until awakened.
             * 等待直至被唤醒
             * 
             * Since we share the process wait semaphore with the regular lock
             * manager and ProcWaitForSignal, and we may need to acquire an LWLock
             * while one of those is pending, it is possible that we get awakened
             * for a reason other than being signaled by LWLockRelease. If so,
             * loop back and wait again.  Once we've gotten the LWLock,
             * re-increment the sema by the number of additional signals received,
             * so that the lock manager or signal manager will see the received
             * signal when it next waits.
             * 由于我们使用常规的锁管理和ProcWaitForSignal信号共享进程等待信号量,
             *   我们可能需要在其中一个挂起时获取LWLock,
             *   原因是有可能是由于其他的原因而不是通过LWLockRelease信号被唤醒.
             * 如果是这种情况,则继续循环等待.
             * 一旦我们获得LWLock,根据接收到的额外信号数目，再次增加信号量，
             *   以便锁管理器或者信号管理器在下次等待时可以看到已接收的信号.
             */
            LOG_LWDEBUG("LWLockAcquire", lock, "waiting");
    
    #ifdef LWLOCK_STATS
            lwstats->block_count++;//统计
    #endif
    
            LWLockReportWaitStart(lock);//报告等待
            TRACE_POSTGRESQL_LWLOCK_WAIT_START(T_NAME(lock), mode);
    
            for (;;)
            {
                PGSemaphoreLock(proc->sem);
                if (!proc->lwWaiting)//如果不是LWLock等待,跳出循环
                    break;
                extraWaits++;//额外的等待
            }
    
            /* Retrying, allow LWLockRelease to release waiters again. */
            //重试,允许LWLockRelease再次释放waiters
            pg_atomic_fetch_or_u32(&lock->state, LW_FLAG_RELEASE_OK);
    
    #ifdef LOCK_DEBUG
            {
                /* not waiting anymore */
                //无需等待
                uint32      nwaiters PG_USED_FOR_ASSERTS_ONLY = pg_atomic_fetch_sub_u32(&lock->nwaiters, 1);
    
                Assert(nwaiters < MAX_BACKENDS);
            }
    #endif
    
            TRACE_POSTGRESQL_LWLOCK_WAIT_DONE(T_NAME(lock), mode);
            LWLockReportWaitEnd();
    
            LOG_LWDEBUG("LWLockAcquire", lock, "awakened");
            //再次循环以再次请求锁
            /* Now loop back and try to acquire lock again. */
            result = false;
        }
    
        TRACE_POSTGRESQL_LWLOCK_ACQUIRE(T_NAME(lock), mode);
        //获取成功!
        /* Add lock to list of locks held by this backend */
        //在该后台进程持有的锁链表中添加锁
        held_lwlocks[num_held_lwlocks].lock = lock;
        held_lwlocks[num_held_lwlocks++].mode = mode;
    
        /*
         * Fix the process wait semaphore's count for any absorbed wakeups.
         * 修正进程咋等待信号量计数的其他absorbed唤醒。
         */
        while (extraWaits-- > 0)
            PGSemaphoreUnlock(proc->sem);
    
        return result;
    }
     
    
    /*
     * Internal function that tries to atomically acquire the lwlock in the passed
     * in mode.
     * 尝试使用指定模式原子性获取LWLock锁的内部函数.
     *
     * This function will not block waiting for a lock to become free - that's the
     * callers job.
     * 该函数不会阻塞等待锁释放的进程 -- 这是调用者的工作.
     *
     * Returns true if the lock isn't free and we need to wait.
     * 如果锁仍未释放,仍需要等待,则返回T
     */
    static bool
    LWLockAttemptLock(LWLock *lock, LWLockMode mode)
    {
        uint32      old_state;
    
        AssertArg(mode == LW_EXCLUSIVE || mode == LW_SHARED);
    
        /*
         * Read once outside the loop, later iterations will get the newer value
         * via compare & exchange.
         * 在循环外先读取一次,后续可以通过比较和交换获得较新的值
         */
        old_state = pg_atomic_read_u32(&lock->state);
    
        /* loop until we've determined whether we could acquire the lock or not */
        //循环指针我们确定是否可以获得锁位置
        while (true)
        {
            uint32      desired_state;
            bool        lock_free;
    
            desired_state = old_state;
    
            if (mode == LW_EXCLUSIVE)//独占
            {
                lock_free = (old_state & LW_LOCK_MASK) == 0;
                if (lock_free)
                    desired_state += LW_VAL_EXCLUSIVE;
            }
            else
            {
                //非独占
                lock_free = (old_state & LW_VAL_EXCLUSIVE) == 0;
                if (lock_free)
                    desired_state += LW_VAL_SHARED;
            }
    
            /*
             * Attempt to swap in the state we are expecting. If we didn't see
             * lock to be free, that's just the old value. If we saw it as free,
             * we'll attempt to mark it acquired. The reason that we always swap
             * in the value is that this doubles as a memory barrier. We could try
             * to be smarter and only swap in values if we saw the lock as free,
             * but benchmark haven't shown it as beneficial so far.
             * 尝试在我们期望的状态下进行交换。
             * 如果没有看到锁被释放,那么这回是旧的值.
             * 如果锁已释放,尝试标记锁已被获取.
             * 我们通常交换值的理由是会使用双倍的内存barrier.
             * 我们尝试变得更好:只交换我们看到已释放的锁,但压力测试显示并没有什么性能改善.
             *
             * Retry if the value changed since we last looked at it.
             * 在最后一次查找后如果值改变,则重试
             */
            if (pg_atomic_compare_exchange_u32(&lock->state,
                                               &old_state, desired_state))
            {
                if (lock_free)
                {
                    /* Great! Got the lock. */
                    //很好,获取锁!
    #ifdef LOCK_DEBUG
                    if (mode == LW_EXCLUSIVE)
                        lock->owner = MyProc;
    #endif
                    return false;
                }
                else
                    return true;    /* 某人还持有锁.somebody else has the lock */
            }
        }
        pg_unreachable();//正常来说,程序逻辑不应到这里
    }
     
    
    //----------------------------------------------------------- WALInsertLockAcquire
    /*
     * Acquire a WAL insertion lock, for inserting to WAL.
     * 在写入WAL前获取wAL insertion锁
     */
    static void
    WALInsertLockAcquire(void)
    {
        bool        immed;
    
        /*
         * It doesn't matter which of the WAL insertion locks we acquire, so try
         * the one we used last time.  If the system isn't particularly busy, it's
         * a good bet that it's still available, and it's good to have some
         * affinity to a particular lock so that you don't unnecessarily bounce
         * cache lines between processes when there's no contention.
         * 我们请求获取哪个WAL insertion锁无关紧要,因此获取最后使用的那个.
         * 如果系统并不繁忙,那么运气好的话,仍然可用,
         * 与特定的锁保持一定的亲缘关系是很好的，这样在没有争用的情况下，
         *   就可用避免不必要地在进程之间切换缓存line。
         *
         * If this is the first time through in this backend, pick a lock
         * (semi-)randomly.  This allows the locks to be used evenly if you have a
         * lot of very short connections.
         * 如果这是该进程的第一次获取,随机获取一个锁.
         * 如果有很多非常短的连接的情况下,这样可以均匀地使用锁。
         */
        static int  lockToTry = -1;
    
        if (lockToTry == -1)
            lockToTry = MyProc->pgprocno % NUM_XLOGINSERT_LOCKS;
        MyLockNo = lockToTry;
    
        /*
         * The insertingAt value is initially set to 0, as we don't know our
         * insert location yet.
         * insertingAt值初始化为0,因为我们还不知道我们插入的位置.
         */
        immed = LWLockAcquire(&WALInsertLocks[MyLockNo].l.lock, LW_EXCLUSIVE);
        if (!immed)
        {
            /*
             * If we couldn't get the lock immediately, try another lock next
             * time.  On a system with more insertion locks than concurrent
             * inserters, this causes all the inserters to eventually migrate to a
             * lock that no-one else is using.  On a system with more inserters
             * than locks, it still helps to distribute the inserters evenly
             * across the locks.
             * 如果不能马上获得锁,下回尝试另外一个锁.
             * 在一个insertion锁比并发插入者更多的系统中,
             *   这会导致所有的inserters周期性的迁移到没有使用的锁上面
             * 相反,仍然可以有助于周期性的分发插入者到不同的锁上.
             */
            lockToTry = (lockToTry + 1) % NUM_XLOGINSERT_LOCKS;
        }
    }
    
    //----------------------------------------------------------- WALInsertLockRelease
    /*
     * Release our insertion lock (or locks, if we're holding them all).
     * 释放insertion锁
     * 
     * NB: Reset all variables to 0, so they cause LWLockWaitForVar to block the
     * next time the lock is acquired.
     * 注意:重置所有的变量为0,这样它们可以使LWLockWaitForVar在下一次获取锁时阻塞.
     */
    static void
    WALInsertLockRelease(void)
    {
        if (holdingAllLocks)//如持有所有锁
        {
            int         i;
    
            for (i = 0; i < NUM_XLOGINSERT_LOCKS; i++)
                LWLockReleaseClearVar(&WALInsertLocks[i].l.lock,
                                      &WALInsertLocks[i].l.insertingAt,
                                      0);
    
            holdingAllLocks = false;
        }
        else
        {
            LWLockReleaseClearVar(&WALInsertLocks[MyLockNo].l.lock,
                                  &WALInsertLocks[MyLockNo].l.insertingAt,
                                  0);
        }
    }
     
    
    /*
     * LWLockReleaseClearVar - release a previously acquired lock, reset variable
     * LWLockReleaseClearVar - 释放先前获取的锁并重置变量
     */
    void
    LWLockReleaseClearVar(LWLock *lock, uint64 *valptr, uint64 val)
    {
        LWLockWaitListLock(lock);
    
        /*
         * Set the variable's value before releasing the lock, that prevents race
         * a race condition wherein a new locker acquires the lock, but hasn't yet
         * set the variables value.
         * 在释放锁之前设置变量的值，这可以防止一个新的locker在没有设置变量值的情况下获取锁时的争用.
         */
        *valptr = val;
        LWLockWaitListUnlock(lock);
    
        LWLockRelease(lock);
    }
    
    
    /*
    * Lock the LWLock's wait list against concurrent activity.
    * 锁定针对并发活动的LWLock等待链表
    *
    * NB: even though the wait list is locked, non-conflicting lock operations
    * may still happen concurrently.
    * 注意:虽然等待链表被锁定,非冲突锁操作仍然可能会并发出现
    *
    * Time spent holding mutex should be short!
    * 耗费在持有mutex的时间应该尽可能的短
    */
    static void
    LWLockWaitListLock(LWLock *lock)
    {
        uint32      old_state;
    #ifdef LWLOCK_STATS
        lwlock_stats *lwstats;
        uint32      delays = 0;
    
        lwstats = get_lwlock_stats_entry(lock);
    #endif
    
        while (true)
        {
            /* always try once to acquire lock directly */
            //首次尝试直接获取锁
            old_state = pg_atomic_fetch_or_u32(&lock->state, LW_FLAG_LOCKED);
            if (!(old_state & LW_FLAG_LOCKED))
                break;              /* 成功获取;got lock */
    
            /* and then spin without atomic operations until lock is released */
            //然后在没有原子操作的情况下spin，直到锁释放
            {
                SpinDelayStatus delayStatus;//SpinDelay状态
    
                init_local_spin_delay(&delayStatus);//初始化
    
                while (old_state & LW_FLAG_LOCKED)//获取Lock
                {
                    perform_spin_delay(&delayStatus);
                    old_state = pg_atomic_read_u32(&lock->state);
                }
    #ifdef LWLOCK_STATS
                delays += delayStatus.delays;
    #endif
                finish_spin_delay(&delayStatus);
            }
    
            /*
             * Retry. The lock might obviously already be re-acquired by the time
             * we're attempting to get it again.
             * 重试,锁有可能在尝试在此获取时已通过重新请求而获得.
             */
        }
    
    #ifdef LWLOCK_STATS
        lwstats->spin_delay_count += delays;//延迟计数
    #endif
    }
    
    
    
     /*
     * Unlock the LWLock's wait list.
     * 解锁LWLock的等待链表
     *
     * Note that it can be more efficient to manipulate flags and release the
     * locks in a single atomic operation.
     * 注意，在单个原子操作中操作标志和释放锁可能更有效。
     */
    static void
    LWLockWaitListUnlock(LWLock *lock)
    {
        uint32      old_state PG_USED_FOR_ASSERTS_ONLY;
    
        old_state = pg_atomic_fetch_and_u32(&lock->state, ~LW_FLAG_LOCKED);
    
        Assert(old_state & LW_FLAG_LOCKED);
    }
    
    
    /*
    * LWLockRelease - release a previously acquired lock
    * LWLockRelease - 释放先前获取的锁
    */
    void
    LWLockRelease(LWLock *lock)
    {
        LWLockMode  mode;
        uint32      oldstate;
        bool        check_waiters;
        int         i;
    
        /*
         * Remove lock from list of locks held.  Usually, but not always, it will
         * be the latest-acquired lock; so search array backwards.
         * 从持有的锁链表中清除锁.
         * 通常来说(但不是总是如此),清除的是最后请求的锁,因此从后往前搜索数组.
         */
        for (i = num_held_lwlocks; --i >= 0;)
            if (lock == held_lwlocks[i].lock)
                break;
    
        if (i < 0)
            elog(ERROR, "lock %s is not held", T_NAME(lock));
    
        mode = held_lwlocks[i].mode;//模式
    
        num_held_lwlocks--;//减一
        for (; i < num_held_lwlocks; i++)
            held_lwlocks[i] = held_lwlocks[i + 1];
    
        PRINT_LWDEBUG("LWLockRelease", lock, mode);
    
        /*
         * Release my hold on lock, after that it can immediately be acquired by
         * others, even if we still have to wakeup other waiters.
         * 释放"我"持有的锁,
         */
        if (mode == LW_EXCLUSIVE)
            oldstate = pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_EXCLUSIVE);
        else
            oldstate = pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_SHARED);
    
        /* nobody else can have that kind of lock */
        //舍我其谁!
        Assert(!(oldstate & LW_VAL_EXCLUSIVE));
    
    
        /*
         * We're still waiting for backends to get scheduled, don't wake them up
         * again.
         * 仍然在等待后台进程获得调度,暂时不需要再次唤醒它们
         */
        if ((oldstate & (LW_FLAG_HAS_WAITERS | LW_FLAG_RELEASE_OK)) ==
            (LW_FLAG_HAS_WAITERS | LW_FLAG_RELEASE_OK) &&
            (oldstate & LW_LOCK_MASK) == 0)
            check_waiters = true;
        else
            check_waiters = false;
    
        /*
         * As waking up waiters requires the spinlock to be acquired, only do so
         * if necessary.
         * 因为唤醒等待器需要获取spinlock，所以只有在必要时才这样做。
         */
        if (check_waiters)
        {
            /* XXX: remove before commit? */
            //XXX: 在commit前清除?
            LOG_LWDEBUG("LWLockRelease", lock, "releasing waiters");
            LWLockWakeup(lock);
        }
    
        TRACE_POSTGRESQL_LWLOCK_RELEASE(T_NAME(lock));
    
        /*
         * Now okay to allow cancel/die interrupts.
         * 现在可以允许中断操作了.
         */
        RESUME_INTERRUPTS();
    }
    
    

### 三、跟踪分析

N/A

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PgSQL · 特性分析 · 数据库崩溃恢复（上）](http://mysql.taobao.org/monthly/2017/05/03/)  
[PgSQL · 特性分析 · 数据库崩溃恢复（下）](http://mysql.taobao.org/monthly/2017/06/04/)  
[PgSQL · 特性分析 · Write-Ahead
Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02/)  
[PostgreSQL WAL Buffers, Clog Buffers Deep
Dive](https://www.slideshare.net/pgday_seoul/pgdayseoul-2017-3-postgresql-wal-buffers-clog-buffers-deep-dive?from_action=save)

