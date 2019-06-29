PostgreSQL使用进程模式，对于每一个客户端会Fork一个后台进程响应客户端的请求。本节介绍了每个后台进程在共享内存中都存在一个的数据结构：PGPROC。

### 一、数据结构

**宏定义**

    
    
    /*
     * Note: MAX_BACKENDS is limited to 2^18-1 because that's the width reserved
     * for buffer references in buf_internals.h.  This limitation could be lifted
     * by using a 64bit state; but it's unlikely to be worthwhile as 2^18-1
     * backends exceed currently realistic configurations. Even if that limitation
     * were removed, we still could not a) exceed 2^23-1 because inval.c stores
     * the backend ID as a 3-byte signed integer, b) INT_MAX/4 because some places
     * compute 4*MaxBackends without any overflow check.  This is rechecked in the
     * relevant GUC check hooks and in RegisterBackgroundWorker().
     * 注意:MAX_BACKENDS限制为2^18-1,
     *   这是因为该值为buf_internals.h中定义的缓存依赖的最大宽度.
     * 该限制可以通过使用64bit的状态来提升,但它看起来并不值当.
     * 如果去掉该限制,我们仍然不能够超过:
     *   a) 2^23-1,因为inval.c使用3个字节的有符号整数存储后台进程ID
     *   b) INT_MAX/4 ,因为某些地方没有任何的溢出检查,直接计算4*MaxBackends的值.
     * 该值会在相关的GUC检查钩子和RegisterBackgroundWorker()函数中检查.
     */
    #define MAX_BACKENDS    0x3FFFF
    
    /* shmqueue.c */
    typedef struct SHM_QUEUE
    {
        struct SHM_QUEUE *prev;
        struct SHM_QUEUE *next;
    } SHM_QUEUE;
    
    /*
     * An invalid pgprocno.  Must be larger than the maximum number of PGPROC
     * structures we could possibly have.  See comments for MAX_BACKENDS.
     * 无效的pg进程号.
     * 必须大于我们可能拥有的最大的PGPROC数目.
     * 详细解释见MAX_BACKENDS
     */
    #define INVALID_PGPROCNO        PG_INT32_MAX
    

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
    
    

**PGPROC**  
每个后台进程在共享内存中都有一个PGPROC结构体.  
全局上也存在未使用的PGPROC结构体链表,用于重用以便为新的后台进程进行分配.  
该数据结构的作用是:

> PostgreSQL backend processes can't see each other's memory directly, nor can
the postmaster see into PostgreSQL backend process memory. Yet they need some
way to communicate and co-ordinate, and the postmaster needs a way to keep
track of them.

简单来说作用是为了进程间协同和通讯以及postmaster的跟踪.

    
    
    /*
     * Each backend has a PGPROC struct in shared memory.  There is also a list of
     * currently-unused PGPROC structs that will be reallocated to new backends.
     * 每个后台进程在共享内存中都有一个PGPROC结构体.
     * 存在未使用的PGPROC结构体链表,用于为新的后台进程重新进行分配.
     * 
     * links: list link for any list the PGPROC is in.  When waiting for a lock,
     * the PGPROC is linked into that lock's waitProcs queue.  A recycled PGPROC
     * is linked into ProcGlobal's freeProcs list.
     * links: PGPROC所在的链表的链接.
     *   在等待锁时,PGPROC链接到该锁的waiProc队列中.
     * 回收的PGPROC链接到ProcGlobal的freeProcs链表中.
     *
     * Note: twophase.c also sets up a dummy PGPROC struct for each currently
     * prepared transaction.  These PGPROCs appear in the ProcArray data structure
     * so that the prepared transactions appear to be still running and are
     * correctly shown as holding locks.  A prepared transaction PGPROC can be
     * distinguished from a real one at need by the fact that it has pid == 0.
     * The semaphore and lock-activity fields in a prepared-xact PGPROC are unused,
     * but its myProcLocks[] lists are valid.
     * 注意:twophase.c也会为每一个当前已准备妥当的事务配置一个虚拟的PGPROC结构.
     * 这些PGPROCs在数组ProcArray数据结构中出现,以便已准备的事务看起来仍在运行,
     *   并正确的显示为持有锁.
     * 已准备妥当的事务PGPROC与一个真正的PGPROC事实上的区别是pid == 0.
     * 在prepared-xact PGPROC中的信号量和活动锁域字段没有使用,但myProcLocks[]链表是有效的.
     */
    struct PGPROC
    {
        /* proc->links MUST BE FIRST IN STRUCT (see ProcSleep,ProcWakeup,etc) */
        //proc->links必须是结构体的第一个域(参考ProcSleep,ProcWakeup...等)
        //如进程在链表中,这是链表的链接
        SHM_QUEUE   links;          /* list link if process is in a list */
        //持有该PGPROC的procglobal链表数组
        PGPROC    **procgloballist; /* procglobal list that owns this PGPROC */
        //可以休眠的信号量
        PGSemaphore sem;            /* ONE semaphore to sleep on */
        //状态为:STATUS_WAITING, STATUS_OK or STATUS_ERROR
        int         waitStatus;     /* STATUS_WAITING, STATUS_OK or STATUS_ERROR */
        //进程通用的latch
        Latch       procLatch;      /* generic latch for process */
        //运行中的进程正在执行的最高层的事务本地ID,如无运行则为InvalidLocalTransactionId
        LocalTransactionId lxid;    /* local id of top-level transaction currently
                                     * being executed by this proc, if running;
                                     * else InvalidLocalTransactionId */
        //后台进程的ID,如为虚拟事务则为0
        int         pid;            /* Backend's process ID; 0 if prepared xact */
        int         pgprocno;
    
        /* These fields are zero while a backend is still starting up: */
        //------------ 这些域在进程正在启动时为0
        //已分配的后台进程的backend ID
        BackendId   backendId;      /* This backend's backend ID (if assigned) */
        //该进程使用的数据库ID
        Oid         databaseId;     /* OID of database this backend is using */
        //使用该进程的角色ID
        Oid         roleId;         /* OID of role using this backend */
        //该进程使用的临时schema OID
        Oid         tempNamespaceId;    /* OID of temp schema this backend is
                                         * using */
        //如后台进程,则为T
        bool        isBackgroundWorker; /* true if background worker. */
    
        /*
         * While in hot standby mode, shows that a conflict signal has been sent
         * for the current transaction. Set/cleared while holding ProcArrayLock,
         * though not required. Accessed without lock, if needed.
         * 如在hot standby模式,显示已为当前事务发送冲突信号.
         * 尽管不需要,设置/清除持有的ProcArrayLock.
         * 如需要,则在没有持有锁的情况下访问.
         */
        bool        recoveryConflictPending;
    
        /* Info about LWLock the process is currently waiting for, if any. */
        //-------------- 进程正在等待的LWLock相关信息
        //等待LW lock,为T
        bool        lwWaiting;      /* true if waiting for an LW lock */
        //正在等的LWLock锁模式
        uint8       lwWaitMode;     /* lwlock mode being waited for */
        //等待链表中的位置
        proclist_node lwWaitLink;   /* position in LW lock wait list */
    
        /* Support for condition variables. */
        //-------------- 支持条件变量
        //CV等待链表中的位置
        proclist_node cvWaitLink;   /* position in CV wait list */
    
        /* Info about lock the process is currently waiting for, if any. */
        //-------------- 进程正在等待的锁信息
        /* waitLock and waitProcLock are NULL if not currently waiting. */
        //如没有在等待,则waitLock和waitProcLock为NULL
        //休眠...等待的锁对象
        LOCK       *waitLock;       /* Lock object we're sleeping on ... */
        //等待锁的每个持锁人信息
        PROCLOCK   *waitProcLock;   /* Per-holder info for awaited lock */
        //等待的所类型
        LOCKMODE    waitLockMode;   /* type of lock we're waiting for */
        //该进程已持有锁的类型位掩码
        LOCKMASK    heldLocks;      /* bitmask for lock types already held on this
                                     * lock object by this backend */
    
        /*
         * Info to allow us to wait for synchronous replication, if needed.
         * waitLSN is InvalidXLogRecPtr if not waiting; set only by user backend.
         * syncRepState must not be touched except by owning process or WALSender.
         * syncRepLinks used only while holding SyncRepLock.
         * 允许我们等待同步复制的相关信息.
         * 如无需等待,则waitLSN为InvalidXLogRecPtr;仅允许由用户后台设置。
         * 除非拥有process或WALSender，否则不能修改syncRepState。
         * 仅在持有SyncRepLock时使用的syncrepink。
         */
        //--------------------- 
        //等待该LSN或者更高的LSN
        XLogRecPtr  waitLSN;        /* waiting for this LSN or higher */
        //同步复制的等待状态
        int         syncRepState;   /* wait state for sync rep */
        //如进程处于syncrep队列中,则该值保存链表链接
        SHM_QUEUE   syncRepLinks;   /* list link if process is in syncrep queue */
    
        /*
         * All PROCLOCK objects for locks held or awaited by this backend are
         * linked into one of these lists, according to the partition number of
         * their lock.
         * 该后台进程持有或等待的锁相关的所有PROCLOCK对象链接在这些链表的末尾,
         *   根据棣属于这些锁的分区号进行区分.
         */
        SHM_QUEUE   myProcLocks[NUM_LOCK_PARTITIONS];
        //子事务的XIDs
        struct XidCache subxids;    /* cache for subtransaction XIDs */
    
        /* Support for group XID clearing. */
        /* true, if member of ProcArray group waiting for XID clear */
        //支持XID分组清除
        //如属于等待XID清理的ProcArray组,则为T
        bool        procArrayGroupMember;
        /* next ProcArray group member waiting for XID clear */
        //等待XID清理的下一个ProcArray组编号
        pg_atomic_uint32 procArrayGroupNext;
    
        /*
         * latest transaction id among the transaction's main XID and
         * subtransactions
         * 在事务主XID和子事务之间的最后的事务ID
         */
        TransactionId procArrayGroupMemberXid;
        //进程的等待信息
        uint32      wait_event_info;    /* proc's wait information */
    
        /* Support for group transaction status update. */
        //--------------- 支持组事务状态更新
        //clog组成员,则为T
        bool        clogGroupMember;    /* true, if member of clog group */
        //下一个clog组成员
        pg_atomic_uint32 clogGroupNext; /* next clog group member */
        //clog组成员事务ID
        TransactionId clogGroupMemberXid;   /* transaction id of clog group member */
        //clog组成员的事务状态
        XidStatus   clogGroupMemberXidStatus;   /* transaction status of clog
                                                 * group member */
        //属于clog组成员的事务ID的clog page
        int         clogGroupMemberPage;    /* clog page corresponding to
                                             * transaction id of clog group member */
        //clog组成员已提交记录的WAL位置
        XLogRecPtr  clogGroupMemberLsn; /* WAL location of commit record for clog
                                         * group member */
    
        /* Per-backend LWLock.  Protects fields below (but not group fields). */
        //每一个后台进程一个LWLock.保护下面的域字段(非组字段)
        LWLock      backendLock;
    
        /* Lock manager data, recording fast-path locks taken by this backend. */
        //---------- 锁管理数据,记录该后台进程以最快路径获得的锁
        //每一个fast-path slot的锁模式
        uint64      fpLockBits;     /* lock modes held for each fast-path slot */
        //rel oids的slots
        Oid         fpRelId[FP_LOCK_SLOTS_PER_BACKEND]; /* slots for rel oids */
        //是否持有fast-path VXID锁
        bool        fpVXIDLock;     /* are we holding a fast-path VXID lock? */
        //fast-path VXID锁的lxid
        LocalTransactionId fpLocalTransactionId;    /* lxid for fast-path VXID
                                                     * lock */
    
        /*
         * Support for lock groups.  Use LockHashPartitionLockByProc on the group
         * leader to get the LWLock protecting these fields.
         */
        //--------- 支持锁组.
        //          在组leader中使用LockHashPartitionLockByProc获取LWLock保护这些域
        //锁组的leader,如果"我"是其中一员
        PGPROC     *lockGroupLeader;    /* lock group leader, if I'm a member */
        //如果"我"是leader,这是成员的链表
        dlist_head  lockGroupMembers;   /* list of members, if I'm a leader */
        //成员连接,如果"我"是其中一员
        dlist_node  lockGroupLink;  /* my member link, if I'm a member */
    };
    
    

**MyProc**  
每个进程都有一个全局变量:MyProc

    
    
    extern PGDLLIMPORT PGPROC *MyProc;
    extern PGDLLIMPORT struct PGXACT *MyPgXact;
    

### 二、源码解读

N/A

### 三、跟踪分析

启动两个Session,执行同样的SQL语句:

    
    
    insert into t_wal_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    

**Session 1**  
启动gdb,开启跟踪

    
    
    (gdb) b XLogInsertRecord
    Breakpoint 1 at 0x54d122: file xlog.c, line 970.
    (gdb) c
    Continuing.
    
    Breakpoint 1, XLogInsertRecord (rdata=0xf9cc70 <hdr_rdt>, fpw_lsn=0, flags=1 '\001') at xlog.c:970
    970     XLogCtlInsert *Insert = &XLogCtl->Insert;
    
    

查看内存中的数据结构

    
    
    (gdb) p *MyProc
    $3 = {links = {prev = 0x0, next = 0x0}, procgloballist = 0x7fa79c087c98, sem = 0x7fa779fc81b8, waitStatus = 0, procLatch = {
        is_set = 0, is_shared = true, owner_pid = 1398}, lxid = 3, pid = 1398, pgprocno = 99, backendId = 3, 
      databaseId = 16402, roleId = 10, tempNamespaceId = 0, isBackgroundWorker = false, recoveryConflictPending = false, 
      lwWaiting = false, lwWaitMode = 0 '\000', lwWaitLink = {next = 0, prev = 0}, cvWaitLink = {next = 0, prev = 0}, 
      waitLock = 0x0, waitProcLock = 0x0, waitLockMode = 0, heldLocks = 0, waitLSN = 0, syncRepState = 0, syncRepLinks = {
        prev = 0x0, next = 0x0}, myProcLocks = {{prev = 0x7fa79c09c588, next = 0x7fa79c09c588}, {prev = 0x7fa79c09c598, 
          next = 0x7fa79c09c598}, {prev = 0x7fa79c09c5a8, next = 0x7fa79c09c5a8}, {prev = 0x7fa79c09c5b8, 
          next = 0x7fa79c09c5b8}, {prev = 0x7fa79c09c5c8, next = 0x7fa79c09c5c8}, {prev = 0x7fa79c09c5d8, 
          next = 0x7fa79c09c5d8}, {prev = 0x7fa79c09c5e8, next = 0x7fa79c09c5e8}, {prev = 0x7fa79c09c5f8, 
          next = 0x7fa79c09c5f8}, {prev = 0x7fa79c09c608, next = 0x7fa79c09c608}, {prev = 0x7fa79c09c618, 
          next = 0x7fa79c09c618}, {prev = 0x7fa79c09c628, next = 0x7fa79c09c628}, {prev = 0x7fa79c09c638, 
          next = 0x7fa79c09c638}, {prev = 0x7fa79c09c648, next = 0x7fa79c09c648}, {prev = 0x7fa79c09c658, 
          next = 0x7fa79c09c658}, {prev = 0x7fa79c09c668, next = 0x7fa79c09c668}, {prev = 0x7fa79be25e70, 
          next = 0x7fa79be25e70}}, subxids = {xids = {0 <repeats 64 times>}}, procArrayGroupMember = false, 
      procArrayGroupNext = {value = 2147483647}, procArrayGroupMemberXid = 0, wait_event_info = 0, clogGroupMember = false, 
      clogGroupNext = {value = 2147483647}, clogGroupMemberXid = 0, clogGroupMemberXidStatus = 0, clogGroupMemberPage = -1, 
      clogGroupMemberLsn = 0, backendLock = {tranche = 58, state = {value = 536870912}, waiters = {head = 2147483647, 
          tail = 2147483647}}, fpLockBits = 196027139227648, fpRelId = {0, 0, 0, 0, 0, 2679, 2610, 2680, 2611, 17043, 17040, 
        17037, 17034, 17031, 17028, 17025}, fpVXIDLock = true, fpLocalTransactionId = 3, lockGroupLeader = 0x0, 
      lockGroupMembers = {head = {prev = 0x7fa79c09c820, next = 0x7fa79c09c820}}, lockGroupLink = {prev = 0x0, next = 0x0}}
    

注意:lwWaiting值为false,表示没有在等待LW Lock

**Session 2**  
启动gdb,开启跟踪

    
    
    (gdb) b heap_insert
    Breakpoint 2 at 0x4df4d1: file heapam.c, line 2449.
    (gdb) c
    Continuing.
    ^C
    Program received signal SIGINT, Interrupt.
    0x00007fa7a7ee7a0b in futex_abstimed_wait (cancel=true, private=<optimized out>, abstime=0x0, expected=0, 
        futex=0x7fa779fc8138) at ../nptl/sysdeps/unix/sysv/linux/sem_waitcommon.c:43
    43        err = lll_futex_wait (futex, expected, private);
    

暂无法进入heap_insert  
查看内存中的数据结构

    
    
    (gdb) p *MyProc
    $36 = {links = {prev = 0x0, next = 0x0}, procgloballist = 0x7fa79c087c98, sem = 0x7fa779fc8138, waitStatus = 0, 
      procLatch = {is_set = 1, is_shared = true, owner_pid = 1449}, lxid = 13, pid = 1449, pgprocno = 98, backendId = 4, 
      databaseId = 16402, roleId = 10, tempNamespaceId = 0, isBackgroundWorker = false, recoveryConflictPending = false, 
      lwWaiting = true, lwWaitMode = 0 '\000', lwWaitLink = {next = 114, prev = 2147483647}, cvWaitLink = {next = 0, prev = 0}, 
      waitLock = 0x0, waitProcLock = 0x0, waitLockMode = 0, heldLocks = 0, waitLSN = 0, syncRepState = 0, syncRepLinks = {
        prev = 0x0, next = 0x0}, myProcLocks = {{prev = 0x7fa79c09c238, next = 0x7fa79c09c238}, {prev = 0x7fa79c09c248, 
          next = 0x7fa79c09c248}, {prev = 0x7fa79c09c258, next = 0x7fa79c09c258}, {prev = 0x7fa79c09c268, 
          next = 0x7fa79c09c268}, {prev = 0x7fa79c09c278, next = 0x7fa79c09c278}, {prev = 0x7fa79c09c288, 
          next = 0x7fa79c09c288}, {prev = 0x7fa79c09c298, next = 0x7fa79c09c298}, {prev = 0x7fa79c09c2a8, 
          next = 0x7fa79c09c2a8}, {prev = 0x7fa79c09c2b8, next = 0x7fa79c09c2b8}, {prev = 0x7fa79c09c2c8, 
          next = 0x7fa79c09c2c8}, {prev = 0x7fa79c09c2d8, next = 0x7fa79c09c2d8}, {prev = 0x7fa79c09c2e8, 
          next = 0x7fa79c09c2e8}, {prev = 0x7fa79c09c2f8, next = 0x7fa79c09c2f8}, {prev = 0x7fa79c09c308, 
          next = 0x7fa79c09c308}, {prev = 0x7fa79be21870, next = 0x7fa79be21870}, {prev = 0x7fa79c09c328, 
          next = 0x7fa79c09c328}}, subxids = {xids = {0 <repeats 64 times>}}, procArrayGroupMember = false, 
      procArrayGroupNext = {value = 2147483647}, procArrayGroupMemberXid = 0, wait_event_info = 16777270, 
      clogGroupMember = false, clogGroupNext = {value = 2147483647}, clogGroupMemberXid = 0, clogGroupMemberXidStatus = 0, 
      clogGroupMemberPage = -1, clogGroupMemberLsn = 0, backendLock = {tranche = 58, state = {value = 536870912}, waiters = {
          head = 2147483647, tail = 2147483647}}, fpLockBits = 196027139227648, fpRelId = {0, 0, 0, 0, 0, 2655, 2603, 2680, 
        2611, 17043, 17040, 17037, 17034, 17031, 17028, 17025}, fpVXIDLock = true, fpLocalTransactionId = 13, 
      lockGroupLeader = 0x0, lockGroupMembers = {head = {prev = 0x7fa79c09c4d0, next = 0x7fa79c09c4d0}}, lockGroupLink = {
        prev = 0x0, next = 0x0}}
    
    

注意:  
lwWaiting值为true,正在等待Session 1的LWLock.  
lwWaitLink = {next = 114, prev = 2147483647},其中next =
114,这里的114是指全局变量ProcGlobal(类型为PROC_HDR)->allProcs数组下标为114的ITEM.

    
    
    (gdb) p ProcGlobal->allProcs[114]
    $41 = {links = {prev = 0x0, next = 0x0}, procgloballist = 0x0, sem = 0x7fa779fc8938, waitStatus = 0, procLatch = {
        is_set = 0, is_shared = true, owner_pid = 1351}, lxid = 0, pid = 1351, pgprocno = 114, backendId = -1, databaseId = 0, 
      roleId = 0, tempNamespaceId = 0, isBackgroundWorker = false, recoveryConflictPending = false, lwWaiting = true, 
      lwWaitMode = 1 '\001', lwWaitLink = {next = 2147483647, prev = 98}, cvWaitLink = {next = 0, prev = 0}, waitLock = 0x0, 
      waitProcLock = 0x0, waitLockMode = 0, heldLocks = 0, waitLSN = 0, syncRepState = 0, syncRepLinks = {prev = 0x0, 
        next = 0x0}, myProcLocks = {{prev = 0x7fa79c09f738, next = 0x7fa79c09f738}, {prev = 0x7fa79c09f748, 
          next = 0x7fa79c09f748}, {prev = 0x7fa79c09f758, next = 0x7fa79c09f758}, {prev = 0x7fa79c09f768, 
          next = 0x7fa79c09f768}, {prev = 0x7fa79c09f778, next = 0x7fa79c09f778}, {prev = 0x7fa79c09f788, 
          next = 0x7fa79c09f788}, {prev = 0x7fa79c09f798, next = 0x7fa79c09f798}, {prev = 0x7fa79c09f7a8, 
          next = 0x7fa79c09f7a8}, {prev = 0x7fa79c09f7b8, next = 0x7fa79c09f7b8}, {prev = 0x7fa79c09f7c8, 
          next = 0x7fa79c09f7c8}, {prev = 0x7fa79c09f7d8, next = 0x7fa79c09f7d8}, {prev = 0x7fa79c09f7e8, 
          next = 0x7fa79c09f7e8}, {prev = 0x7fa79c09f7f8, next = 0x7fa79c09f7f8}, {prev = 0x7fa79c09f808, 
          next = 0x7fa79c09f808}, {prev = 0x7fa79c09f818, next = 0x7fa79c09f818}, {prev = 0x7fa79c09f828, 
          next = 0x7fa79c09f828}}, subxids = {xids = {0 <repeats 64 times>}}, procArrayGroupMember = false, 
      procArrayGroupNext = {value = 0}, procArrayGroupMemberXid = 0, wait_event_info = 16777270, clogGroupMember = false, 
      clogGroupNext = {value = 0}, clogGroupMemberXid = 0, clogGroupMemberXidStatus = 0, clogGroupMemberPage = 0, 
      clogGroupMemberLsn = 0, backendLock = {tranche = 58, state = {value = 536870912}, waiters = {head = 2147483647, 
          tail = 2147483647}}, fpLockBits = 0, fpRelId = {0 <repeats 16 times>}, fpVXIDLock = false, fpLocalTransactionId = 0, 
      lockGroupLeader = 0x0, lockGroupMembers = {head = {prev = 0x7fa79c09f9d0, next = 0x7fa79c09f9d0}}, lockGroupLink = {
        prev = 0x0, next = 0x0}}
    

**ProcGlobal下节再行介绍**

### 四、参考资料

[What is the role of struct 'PGPROC' in
PostgreSQL?](https://stackoverflow.com/questions/25277018/what-is-the-role-of-struct-pgproc-in-postgresql)

