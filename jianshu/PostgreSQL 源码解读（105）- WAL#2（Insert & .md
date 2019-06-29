本节介绍了插入数据时与WAL相关的处理逻辑，主要包括heap_insert依赖的函数XLogBeginInsert/XLogRegisterBufData/XLogRegisterData/XLogSetRecordFlags。

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
    
    

**registered_buffer**  
对于每一个使用XLogRegisterBuffer注册的每个数据块,填充到registered_buffer结构体中

    
    
    /*
     * For each block reference registered with XLogRegisterBuffer, we fill in
     * a registered_buffer struct.
     * 对于每一个使用XLogRegisterBuffer注册的每个数据块,
     *   填充到registered_buffer结构体中
     */
    typedef struct
    {
        //slot是否在使用?
        bool        in_use;         /* is this slot in use? */
        //REGBUF_* 相关标记
        uint8       flags;          /* REGBUF_* flags */
        //定义关系和数据库的标识符
        RelFileNode rnode;          /* identifies the relation and block */
        //fork进程编号
        ForkNumber  forkno;
        //块编号
        BlockNumber block;
        //页内容
        Page        page;           /* page content */
        //rdata链中的数据总大小
        uint32      rdata_len;      /* total length of data in rdata chain */
        //使用该数据块注册的数据链头
        XLogRecData *rdata_head;    /* head of the chain of data registered with
                                     * this block */
        //使用该数据块注册的数据链尾
        XLogRecData *rdata_tail;    /* last entry in the chain, or &rdata_head if
                                     * empty */
        //临时rdatas数据引用,用于存储XLogRecordAssemble()中使用的备份块数据
        XLogRecData bkp_rdatas[2];  /* temporary rdatas used to hold references to
                                     * backup block data in XLogRecordAssemble() */
    
        /* buffer to store a compressed version of backup block image */
        //用于存储压缩版本的备份块镜像的缓存
        char        compressed_page[PGLZ_MAX_BLCKSZ];
    } registered_buffer;
    //registered_buffer指正
    static registered_buffer *registered_buffers;
    //已分配的大小
    static int  max_registered_buffers; /* allocated size */
    //最大块号 + 1(当前注册块)
    static int  max_registered_block_id = 0;    /* highest block_id + 1 currently
                                                 * registered */
    

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
    
    

**XLogRecData**  
xloginsert.c中的函数构造一个XLogRecData结构体链用于标识最后的WAL记录

    
    
    /*
     * The functions in xloginsert.c construct a chain of XLogRecData structs
     * to represent the final WAL record.
     * xloginsert.c中的函数构造一个XLogRecData结构体链用于标识最后的WAL记录
     */
    typedef struct XLogRecData
    {
        //链中的下一个结构体,如无则为NULL
        struct XLogRecData *next;   /* next struct in chain, or NULL */
        //rmgr数据的起始地址
        char       *data;           /* start of rmgr data to include */
        //rmgr数据大小
        uint32      len;            /* length of rmgr data to include */
    } XLogRecData;
    
    

**registered_buffer/registered_buffers**  
对于每一个使用XLogRegisterBuffer注册的每个数据块,填充到registered_buffer结构体中

    
    
    /*
     * For each block reference registered with XLogRegisterBuffer, we fill in
     * a registered_buffer struct.
     * 对于每一个使用XLogRegisterBuffer注册的每个数据块,
     *   填充到registered_buffer结构体中
     */
    typedef struct
    {
        //slot是否在使用?
        bool        in_use;         /* is this slot in use? */
        //REGBUF_* 相关标记
        uint8       flags;          /* REGBUF_* flags */
        //定义关系和数据库的标识符
        RelFileNode rnode;          /* identifies the relation and block */
        //fork进程编号
        ForkNumber  forkno;
        //块编号
        BlockNumber block;
        //页内容
        Page        page;           /* page content */
        //rdata链中的数据总大小
        uint32      rdata_len;      /* total length of data in rdata chain */
        //使用该数据块注册的数据链头
        XLogRecData *rdata_head;    /* head of the chain of data registered with
                                     * this block */
        //使用该数据块注册的数据链尾
        XLogRecData *rdata_tail;    /* last entry in the chain, or &rdata_head if
                                     * empty */
        //临时rdatas数据引用,用于存储XLogRecordAssemble()中使用的备份块数据
        XLogRecData bkp_rdatas[2];  /* temporary rdatas used to hold references to
                                     * backup block data in XLogRecordAssemble() */
    
        /* buffer to store a compressed version of backup block image */
        //用于存储压缩版本的备份块镜像的缓存
        char        compressed_page[PGLZ_MAX_BLCKSZ];
    } registered_buffer;
    //registered_buffer指针(全局变量)
    static registered_buffer *registered_buffers;
    //已分配的大小
    static int  max_registered_buffers; /* allocated size */
    //最大块号 + 1(当前注册块)
    static int  max_registered_block_id = 0;    /* highest block_id + 1 currently
                                                 * registered */
    

### 二、源码解读

**heap_insert**  
主要实现逻辑是插入元组到堆中,其中存在对WAL(XLog)进行处理的部分.  
参见[PostgreSQL 源码解读（104）- WAL#1（Insert & WAL-heap_insert函数#1）](https://www.jianshu.com/p/70f019643e76)

**XLogBeginInsert**  
开始构造WAL记录.  
必须在调用XLogRegister*和XLogInsert()函数前调用.

    
    
    /*
     * Begin constructing a WAL record. This must be called before the
     * XLogRegister* functions and XLogInsert().
     * 开始构造WAL记录.
     * 必须在调用XLogRegister*和XLogInsert()函数前调用.
     */
    void
    XLogBeginInsert(void)
    {
        //验证逻辑
        Assert(max_registered_block_id == 0);
        Assert(mainrdata_last == (XLogRecData *) &mainrdata_head);
        Assert(mainrdata_len == 0);
    
        /* cross-check on whether we should be here or not */
        //交叉校验是否应该在这里还是不应该在这里出现
        if (!XLogInsertAllowed())
            elog(ERROR, "cannot make new WAL entries during recovery");
    
        if (begininsert_called)
            elog(ERROR, "XLogBeginInsert was already called");
        //变量赋值
        begininsert_called = true;
    }
    
    /*
     * Is this process allowed to insert new WAL records?
     * 判断该进程是否允许插入新的WAL记录
     * 
     * Ordinarily this is essentially equivalent to !RecoveryInProgress().
     * But we also have provisions for forcing the result "true" or "false"
     * within specific processes regardless of the global state.
     * 通常，这本质上等同于! recoverinprogress()。
     * 但我们也有规定，无论全局状况如何，都要在特定进程中强制实现“正确”或“错误”的结果。
     */
    bool
    XLogInsertAllowed(void)
    {
        /*
         * If value is "unconditionally true" or "unconditionally false", just
         * return it.  This provides the normal fast path once recovery is known
         * done.
         * 如果值为“无条件为真”或“无条件为假”，则返回。
         * 这提供正常的快速判断路径。
         */
        if (LocalXLogInsertAllowed >= 0)
            return (bool) LocalXLogInsertAllowed;
    
        /*
         * Else, must check to see if we're still in recovery.
         * 否则,必须检查是否处于恢复状态
         */
        if (RecoveryInProgress())
            return false;
    
        /*
         * On exit from recovery, reset to "unconditionally true", since there is
         * no need to keep checking.
         * 从恢复中退出,由于不需要继续检查,重置为"无条件为真"
         */
        LocalXLogInsertAllowed = 1;
        return true;
    }
    
    

**XLogRegisterData**  
添加数据到正在构造的WAL记录中

    
    
    /*
     * Add data to the WAL record that's being constructed.
     * 添加数据到正在构造的WAL记录中
     * 
     * The data is appended to the "main chunk", available at replay with
     * XLogRecGetData().
     * 数据追加到"main chunk"中,用于XLogRecGetData()函数回放
     */
    void
    XLogRegisterData(char *data, int len)
    {
        XLogRecData *rdata;//数据
        //验证是否已调用begin
        Assert(begininsert_called);
        //验证大小
        if (num_rdatas >= max_rdatas)
            elog(ERROR, "too much WAL data");
        rdata = &rdatas[num_rdatas++];
    
        rdata->data = data;
        rdata->len = len;
    
        /*
         * we use the mainrdata_last pointer to track the end of the chain, so no
         * need to clear 'next' here.
         * 使用mainrdata_last指针跟踪链条的结束点,在这里不需要清除next变量
         */
    
        mainrdata_last->next = rdata;
        mainrdata_last = rdata;
    
        mainrdata_len += len;
    }
    
    

**XLogRegisterBuffer**  
在缓冲区中注册已构建的WAL记录的依赖,在WAL-logged操作更新每一个page时必须调用此函数

    
    
    /*
     * Register a reference to a buffer with the WAL record being constructed.
     * This must be called for every page that the WAL-logged operation modifies.
     * 在缓冲区中注册已构建的WAL记录的依赖
     * 在WAL-logged操作更新每一个page时必须调用此函数
     */
    void
    XLogRegisterBuffer(uint8 block_id, Buffer buffer, uint8 flags)
    {
        registered_buffer *regbuf;//缓冲
    
        /* NO_IMAGE doesn't make sense with FORCE_IMAGE */
        //NO_IMAGE不能与REGBUF_NO_IMAGE同时使用
        Assert(!((flags & REGBUF_FORCE_IMAGE) && (flags & (REGBUF_NO_IMAGE))));
        Assert(begininsert_called);
        //块ID > 最大已注册的缓冲区,报错
        if (block_id >= max_registered_block_id)
        {
            if (block_id >= max_registered_buffers)
                elog(ERROR, "too many registered buffers");
            max_registered_block_id = block_id + 1;
        }
        //赋值
        regbuf = &registered_buffers[block_id];
        //获取Tag
        BufferGetTag(buffer, &regbuf->rnode, &regbuf->forkno, &regbuf->block);
        regbuf->page = BufferGetPage(buffer);
        regbuf->flags = flags;
        regbuf->rdata_tail = (XLogRecData *) &regbuf->rdata_head;
        regbuf->rdata_len = 0;
    
        /*
         * Check that this page hasn't already been registered with some other
         * block_id.
         * 检查该page是否已被其他block_id注册
         */
    #ifdef USE_ASSERT_CHECKING
        {
            int         i;
    
            for (i = 0; i < max_registered_block_id; i++)//循环检查
            {
                registered_buffer *regbuf_old = &registered_buffers[i];
    
                if (i == block_id || !regbuf_old->in_use)
                    continue;
    
                Assert(!RelFileNodeEquals(regbuf_old->rnode, regbuf->rnode) ||
                       regbuf_old->forkno != regbuf->forkno ||
                       regbuf_old->block != regbuf->block);
            }
        }
    #endif
    
        regbuf->in_use = true;//标记为使用
    }
    
    
    /*
     * BufferGetTag
     *      Returns the relfilenode, fork number and block number associated with
     *      a buffer.
     * 返回与缓冲区相关的relfilenode,fork编号和块号
     */
    void
    BufferGetTag(Buffer buffer, RelFileNode *rnode, ForkNumber *forknum,
                 BlockNumber *blknum)
    {
        BufferDesc *bufHdr;
    
        /* Do the same checks as BufferGetBlockNumber. */
        //验证buffer已被pinned
        Assert(BufferIsPinned(buffer));
    
        if (BufferIsLocal(buffer))
            bufHdr = GetLocalBufferDescriptor(-buffer - 1);
        else
            bufHdr = GetBufferDescriptor(buffer - 1);
    
        /* pinned, so OK to read tag without spinlock */
        //pinned,不需要spinlock读取tage
        *rnode = bufHdr->tag.rnode;
        *forknum = bufHdr->tag.forkNum;
        *blknum = bufHdr->tag.blockNum;
    }
    
    /*
     * BufferIsLocal
     *      True iff the buffer is local (not visible to other backends).
     *      如缓冲区对其他后台进程不不可见,则为本地buffer
     */
    #define BufferIsLocal(buffer)   ((buffer) < 0)
    #define GetBufferDescriptor(id) (&BufferDescriptors[(id)].bufferdesc)
    #define GetLocalBufferDescriptor(id) (&LocalBufferDescriptors[(id)])
    BufferDesc *LocalBufferDescriptors = NULL;
    BufferDescPadded *BufferDescriptors;
    
    

**XLogRegisterBufData**  
在正在构造的WAL记录中添加buffer相关的数据.

    
    
    /*
     * Add buffer-specific data to the WAL record that's being constructed.
     * 在正在构造的WAL记录中添加buffer相关的数据.
     * 
     * Block_id must reference a block previously registered with
     * XLogRegisterBuffer(). If this is called more than once for the same
     * block_id, the data is appended.
     * Block_id必须引用先前注册到XLogRegisterBuffer()中的数据块。
     * 如果对同一个block_id不止一次调用，那么数据将会追加。
     *
     * The maximum amount of data that can be registered per block is 65535
     * bytes. That should be plenty; if you need more than BLCKSZ bytes to
     * reconstruct the changes to the page, you might as well just log a full
     * copy of it. (the "main data" that's not associated with a block is not
     * limited)
     * 每个块可注册的最大大小是65535Bytes.
     * 通常来说这已经足够了;如果需要大小比BLCKSZ字节更大的数据用于重建页面的变化,
     *   那么需要整页进行拷贝.
     * (与数据块相关的"main data"是不受限的)
     */
    void
    XLogRegisterBufData(uint8 block_id, char *data, int len)
    {
        registered_buffer *regbuf;//注册的缓冲区
        XLogRecData *rdata;//数据
    
        Assert(begininsert_called);//XLogBeginInsert函数已调用
    
        /* find the registered buffer struct */
        //寻找已注册的缓存结构体
        regbuf = &registered_buffers[block_id];
        if (!regbuf->in_use)
            elog(ERROR, "no block with id %d registered with WAL insertion",
                 block_id);
    
        if (num_rdatas >= max_rdatas)
            elog(ERROR, "too much WAL data");
        rdata = &rdatas[num_rdatas++];
    
        rdata->data = data;
        rdata->len = len;
    
        regbuf->rdata_tail->next = rdata;
        regbuf->rdata_tail = rdata;
        regbuf->rdata_len += len;
    }
    
    

**XLogSetRecordFlags**  
为即将"到来"的WAL记录设置插入状态标记  
XLOG_INCLUDE_ORIGIN 确定复制起点是否应该包含在记录中  
XLOG_MARK_UNIMPORTANT 表示记录对于持久性并不重要，这可以避免触发WAL归档和其他后台活动

    
    
    /*
     * Set insert status flags for the upcoming WAL record.
     * 为即将"到来"的WAL记录设置插入状态标记
     *
     * The flags that can be used here are:
     * - XLOG_INCLUDE_ORIGIN, to determine if the replication origin should be
     *   included in the record.
     * - XLOG_MARK_UNIMPORTANT, to signal that the record is not important for
     *   durability, which allows to avoid triggering WAL archiving and other
     *   background activity.
     * 标记用于:
     * - XLOG_INCLUDE_ORIGIN 确定复制起点是否应该包含在记录中
     * - XLOG_MARK_UNIMPORTANT 表示记录对于持久性并不重要，这可以避免触发WAL归档和其他后台活动。
     */
    void
    XLogSetRecordFlags(uint8 flags)
    {
        Assert(begininsert_called);
        curinsert_flags = flags;
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    insert into t_wal_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    
    

**XLogBeginInsert**  
启动gdb,设置断点,进入XLogBeginInsert

    
    
    (gdb) b XLogBeginInsert
    Breakpoint 1 at 0x564897: file xloginsert.c, line 122.
    (gdb) c
    Continuing.
    
    Breakpoint 1, XLogBeginInsert () at xloginsert.c:122
    122     Assert(max_registered_block_id == 0);
    

校验,调用XLogInsertAllowed

    
    
    122     Assert(max_registered_block_id == 0);
    (gdb) n
    123     Assert(mainrdata_last == (XLogRecData *) &mainrdata_head);
    (gdb) 
    124     Assert(mainrdata_len == 0);
    (gdb) 
    127     if (!XLogInsertAllowed())
    (gdb) step
    XLogInsertAllowed () at xlog.c:8126
    8126        if (LocalXLogInsertAllowed >= 0)
    (gdb) n
    8132        if (RecoveryInProgress())
    (gdb) 
    8139        LocalXLogInsertAllowed = 1;
    (gdb) 
    8140        return true;
    (gdb) 
    8141    }
    (gdb) 
    

赋值,设置begininsert_called为T,返回

    
    
    (gdb) 
    XLogBeginInsert () at xloginsert.c:130
    130     if (begininsert_called)
    (gdb) p begininsert_called
    $1 = false
    (gdb) n
    133     begininsert_called = true;
    (gdb) 
    134 }
    (gdb) 
    heap_insert (relation=0x7f5cc0338228, tup=0x29b2440, cid=0, options=0, bistate=0x0) at heapam.c:2567
    2567            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
    (gdb) 
    

**XLogRegisterData**  
进入XLogRegisterData函数

    
    
    (gdb) step
    XLogRegisterData (data=0x7fff03ba99e0 "\002", len=3) at xloginsert.c:327
    327     Assert(begininsert_called);
    (gdb) p *data
    $2 = 2 '\002'
    (gdb) p *(xl_heap_insert *)data
    $3 = {offnum = 2, flags = 0 '\000'}
    

执行相关判断,并赋值  
rdatas是XLogRecData结构体指针,全局静态变量:  
static XLogRecData *rdatas;

    
    
    (gdb) n
    329     if (num_rdatas >= max_rdatas)
    (gdb) p num_rdatas
    $4 = 0
    (gdb) p max_rdatas
    $5 = 20
    (gdb) n
    331     rdata = &rdatas[num_rdatas++];
    (gdb) p rdatas[0]
    $6 = {next = 0x0, data = 0x0, len = 0}
    (gdb) p rdatas[1]
    $7 = {next = 0x0, data = 0x0, len = 0}
    

相关结构体赋值  
其中mainrdata_last是mainrdata_head的地址:  
static XLogRecData *mainrdata_head;  
static XLogRecData *mainrdata_last = (XLogRecData *) &mainrdata_head;

    
    
    (gdb) n
    333     rdata->data = data;
    (gdb) 
    334     rdata->len = len;
    (gdb) 
    341     mainrdata_last->next = rdata;
    (gdb) 
    342     mainrdata_last = rdata;
    (gdb) 
    344     mainrdata_len += len;
    (gdb) 
    345 }
    

完成调用,回到heap_insert

    
    
    (gdb) n
    heap_insert (relation=0x7f5cc0338228, tup=0x29b2440, cid=0, options=0, bistate=0x0) at heapam.c:2569
    2569            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
    

**XLogRegisterBuffer**  
进入XLogRegisterBuffer

    
    
    (gdb) step
    XLogRegisterBuffer (block_id=0 '\000', buffer=99, flags=8 '\b') at xloginsert.c:218
    218     Assert(!((flags & REGBUF_FORCE_IMAGE) && (flags & (REGBUF_NO_IMAGE))));
    

判断block_id,设置max_registered_block_id变量等.  
注:max_registered_buffers初始化为5

    
    
    (gdb) n
    219     Assert(begininsert_called);
    (gdb) 
    221     if (block_id >= max_registered_block_id)
    (gdb) p max_registered_block_id
    $14 = 0
    (gdb) n
    223         if (block_id >= max_registered_buffers)
    (gdb) p max_registered_buffers
    $15 = 5
    (gdb) n
    225         max_registered_block_id = block_id + 1;
    (gdb) 
    228     regbuf = &registered_buffers[block_id];
    (gdb) p max_registered_buffers
    $16 = 5
    (gdb) p max_registered_block_id
    $17 = 1
    (gdb) n
    230     BufferGetTag(buffer, &regbuf->rnode, &regbuf->forkno, &regbuf->block);
    (gdb) p *regbuf
    $18 = {in_use = false, flags = 0 '\000', rnode = {spcNode = 0, dbNode = 0, relNode = 0}, forkno = MAIN_FORKNUM, block = 0, 
      page = 0x0, rdata_len = 0, rdata_head = 0x0, rdata_tail = 0x0, bkp_rdatas = {{next = 0x0, data = 0x0, len = 0}, {
          next = 0x0, data = 0x0, len = 0}}, compressed_page = '\000' <repeats 8195 times>}
    

获取buffer的tag  
rnode/forkno/block

    
    
    (gdb) n
    231     regbuf->page = BufferGetPage(buffer);
    (gdb) p *regbuf
    $19 = {in_use = false, flags = 0 '\000', rnode = {spcNode = 1663, dbNode = 16402, relNode = 17034}, forkno = MAIN_FORKNUM, 
      block = 0, page = 0x0, rdata_len = 0, rdata_head = 0x0, rdata_tail = 0x0, bkp_rdatas = {{next = 0x0, data = 0x0, 
          len = 0}, {next = 0x0, data = 0x0, len = 0}}, compressed_page = '\000' <repeats 8195 times>}
    

设置flags等其他变量

    
    
    (gdb) n
    232     regbuf->flags = flags;
    (gdb) 
    233     regbuf->rdata_tail = (XLogRecData *) &regbuf->rdata_head;
    (gdb) 
    234     regbuf->rdata_len = 0;
    (gdb) 
    244         for (i = 0; i < max_registered_block_id; i++)
    (gdb) p regbuf->flags
    $21 = 8 '\b'
    (gdb) p *regbuf->rdata_tail
    $23 = {next = 0x0, data = 0x292e1a8 "", len = 0}
    (gdb) p regbuf->rdata_len
    $24 = 0
    

检查该page是否已被其他block_id注册  
最后设置in_use为T,返回XLogRegisterBufData

    
    
    (gdb) n
    246             registered_buffer *regbuf_old = &registered_buffers[i];
    (gdb) 
    248             if (i == block_id || !regbuf_old->in_use)
    (gdb) 
    249                 continue;
    (gdb) 
    244         for (i = 0; i < max_registered_block_id; i++)
    (gdb) 
    258     regbuf->in_use = true;
    (gdb) 
    259 }
    (gdb) 
    heap_insert (relation=0x7f5cc0338228, tup=0x29b2440, cid=0, options=0, bistate=0x0) at heapam.c:2579
    2579            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
    

**XLogRegisterBufData**  
进入XLogRegisterBufData函数

    
    
    (gdb) step
    XLogRegisterBufData (block_id=0 '\000', data=0x7fff03ba99d0 "\003", len=5) at xloginsert.c:366
    366     Assert(begininsert_called);
    

寻找已注册的缓存结构体

    
    
    (gdb) n
    369     regbuf = &registered_buffers[block_id];
    (gdb) 
    370     if (!regbuf->in_use)
    (gdb) p *regbuf
    $25 = {in_use = true, flags = 8 '\b', rnode = {spcNode = 1663, dbNode = 16402, relNode = 17034}, forkno = MAIN_FORKNUM, 
      block = 0, page = 0x7f5c93854380 "\001", rdata_len = 0, rdata_head = 0x0, rdata_tail = 0x292e1a8, bkp_rdatas = {{
          next = 0x0, data = 0x0, len = 0}, {next = 0x0, data = 0x0, len = 0}}, compressed_page = '\000' <repeats 8195 times>}
    (gdb) p *regbuf->page
    $26 = 1 '\001'
    (gdb) n
    374     if (num_rdatas >= max_rdatas)
    (gdb) 
    

在正在构造的WAL记录中添加buffer相关的数据.

    
    
    (gdb) n
    376     rdata = &rdatas[num_rdatas++];
    (gdb) p num_rdatas
    $27 = 1
    (gdb) p max_rdatas
    $28 = 20
    (gdb) n
    378     rdata->data = data;
    (gdb) 
    379     rdata->len = len;
    (gdb) 
    381     regbuf->rdata_tail->next = rdata;
    (gdb) 
    382     regbuf->rdata_tail = rdata;
    (gdb) 
    383     regbuf->rdata_len += len;
    (gdb) 
    384 }
    (gdb) p *rdata
    $29 = {next = 0x0, data = 0x7fff03ba99d0 "\003", len = 5}
    (gdb) 
    

完成调用,回到heap_insert

    
    
    (gdb) n
    heap_insert (relation=0x7f5cc0338228, tup=0x29b2440, cid=0, options=0, bistate=0x0) at heapam.c:2583
    2583                                heaptup->t_len - SizeofHeapTupleHeader);
    

继续调用XLogRegisterBufData函数注册tuple实际数据

    
    
    2583                                heaptup->t_len - SizeofHeapTupleHeader);
    (gdb) n
    2581            XLogRegisterBufData(0,
    (gdb) 
    

**XLogSetRecordFlags**  
为即将"到来"的WAL记录设置插入状态标记

    
    
    (gdb) 
    2586            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
    

逻辑很简单,设置标记位curinsert_flags

    
    
    (gdb) step
    XLogSetRecordFlags (flags=1 '\001') at xloginsert.c:399
    399     Assert(begininsert_called);
    (gdb) n
    400     curinsert_flags = flags;
    (gdb) 
    401 }
    (gdb) 
    heap_insert (relation=0x7f5cc0338228, tup=0x29b2440, cid=0, options=0, bistate=0x0) at heapam.c:2588
    2588            recptr = XLogInsert(RM_HEAP_ID, info);
    (gdb) 
    

调用XLogInsert,插入WAL

    
    
    (gdb) 
    2590            PageSetLSN(page, recptr);
    ...
    

**XLogInsert** 函数下节再行介绍.

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PgSQL · 特性分析 · 数据库崩溃恢复（上）](http://mysql.taobao.org/monthly/2017/05/03/)  
[PgSQL · 特性分析 · 数据库崩溃恢复（下）](http://mysql.taobao.org/monthly/2017/06/04/)  
[PgSQL · 特性分析 · Write-Ahead
Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02/)  
[PostgreSQL WAL Buffers, Clog Buffers Deep
Dive](https://www.slideshare.net/pgday_seoul/pgdayseoul-2017-3-postgresql-wal-buffers-clog-buffers-deep-dive?from_action=save)

