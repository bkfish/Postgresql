本节重点跟踪分析了XLogRecordAssemble函数中对FPW(full-page-write)的处理过程。

### 一、数据结构

**全局静态变量**  
XLogRecordAssemble使用的全局变量包括hdr_rdt/hdr_scratch/rdatas等.

    
    
    /* flags for the in-progress insertion */
    //用于插入过程中的标记信息
    static uint8 curinsert_flags = 0;
    
    /*
     * These are used to hold the record header while constructing a record.
     * 'hdr_scratch' is not a plain variable, but is palloc'd at initialization,
     * because we want it to be MAXALIGNed and padding bytes zeroed.
     * 在构建XLOG Record时通常会存储记录的头部信息.
     * 'hdr_scratch'并不是一个普通(plain)变量,而是在初始化时通过palloc初始化,
     *   因为我们希望该变量已经是MAXALIGNed并且已被0x00填充.
     *
     * For simplicity, it's allocated large enough to hold the headers for any
     * WAL record.
     * 简单起见,该变量预先会分配足够大的空间用于存储所有WAL Record的头部信息.
     */
    static XLogRecData hdr_rdt;
    static char *hdr_scratch = NULL;
    
    #define SizeOfXlogOrigin    (sizeof(RepOriginId) + sizeof(char))
    
    #define HEADER_SCRATCH_SIZE \
        (SizeOfXLogRecord + \
         MaxSizeOfXLogRecordBlockHeader * (XLR_MAX_BLOCK_ID + 1) + \
         SizeOfXLogRecordDataHeaderLong + SizeOfXlogOrigin)
    /*
     * An array of XLogRecData structs, to hold registered data.
     * XLogRecData结构体数组,存储已注册的数据.
     */
    static XLogRecData *rdatas;
    static int  num_rdatas;         /* entries currently used */
    //已分配的空间大小
    static int  max_rdatas;         /* allocated size */
    //是否调用XLogBeginInsert函数
    static bool begininsert_called = false;
    
    static XLogCtlData *XLogCtl = NULL;
    
    /* flags for the in-progress insertion */
    static uint8 curinsert_flags = 0;
    
    /*
     * A chain of XLogRecDatas to hold the "main data" of a WAL record, registered
     * with XLogRegisterData(...).
     * 存储WAL Record "main data"的XLogRecDatas数据链
     */
    static XLogRecData *mainrdata_head;
    static XLogRecData *mainrdata_last = (XLogRecData *) &mainrdata_head;
    //链中某个位置的mainrdata大小
    static uint32 mainrdata_len; /* total # of bytes in chain */
    
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
XLogRegisterBuffer函数使用的flags

    
    
    /* flags for XLogRegisterBuffer */
    //XLogRegisterBuffer函数使用的flags
    #define REGBUF_FORCE_IMAGE  0x01    /* 强制执行full-page-write;force a full-page image */
    #define REGBUF_NO_IMAGE     0x02    /* 不需要FPI;don't take a full-page image */
    #define REGBUF_WILL_INIT    (0x04 | 0x02)   /* 在回放时重新初始化page(表示NO_IMAGE);
                                                 * page will be re-initialized at
                                                 * replay (implies NO_IMAGE) */
    #define REGBUF_STANDARD     0x08    /* 标准的page layout(数据在pd_lower和pd_upper之间的数据会被跳过)
                                         * page follows "standard" page layout,
                                         * (data between pd_lower and pd_upper
                                         * will be skipped) */
    #define REGBUF_KEEP_DATA    0x10    /* include data even if a full-page image
                                          * is taken */
    /*
     * Flag bits for the record being inserted, set using XLogSetRecordFlags().
     */
    #define XLOG_INCLUDE_ORIGIN     0x01    /* include the replication origin */
    #define XLOG_MARK_UNIMPORTANT   0x02    /* record not important for durability */    
    
    

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
    //registered_buffer指针(全局变量)
    static registered_buffer *registered_buffers;
    //已分配的大小
    static int  max_registered_buffers; /* allocated size */
    //最大块号 + 1(当前注册块)
    static int  max_registered_block_id = 0;    /* highest block_id + 1 currently
                                                 * registered */
    

**rmid(Resource Manager ID)的定义**  
0 XLOG  
1 Transaction  
2 Storage  
3 CLOG  
4 Database  
5 Tablespace  
6 MultiXact  
7 RelMap  
8 Standby  
9 Heap2  
10 Heap  
11 Btree  
12 Hash  
13 Gin  
14 Gist  
15 Sequence  
16 SPGist

### 二、源码解读

XLogRecordAssemble函数从已注册的数据和缓冲区中组装XLOG
record到XLogRecData链中,组装完成后可以使用XLogInsertRecord()函数插入到WAL buffer中.  
详见[上一小节](https://www.jianshu.com/p/ded2ea311e82)

### 三、跟踪分析

**场景二:在数据表中插入第n条记录**  
测试脚本如下:

    
    
    testdb=# drop table t_wal_normal;
    DROP TABLE
    testdb=# create table t_wal_normal(c1 int not null,c2  varchar(40),c3 varchar(40));
    CREATE TABLE
    testdb=# insert into t_wal_normal(c1,c2,c3) select i,'C2-'||i,'C3-'||i from generate_series(1,8191) as i;
    INSERT 0 8191
    testdb=# select pg_size_pretty(pg_relation_size('t_wal_normal'));
     pg_size_pretty 
    ----------------     424 kB
    (1 row)
    testdb=# checkpoint;
    CHECKPOINT
    testdb=# insert into t_wal_normal(c1,c2,c3) VALUES(8192,'C2-8192','C3-8192');
    
    

设置断点,进入XLogRecordAssemble

    
    
    (gdb) b XLogRecordAssemble
    Breakpoint 1 at 0x565411: file xloginsert.c, line 488.
    (gdb) c
    Continuing.
    
    Breakpoint 1, XLogRecordAssemble (rmid=10 '\n', info=0 '\000', RedoRecPtr=5509173376, doPageWrites=true, 
        fpw_lsn=0x7ffd7eb51408) at xloginsert.c:488
    488   uint32    total_len = 0;
    

输入参数:  
rmid=10即0x0A --> Heap  
info=0 '\000'  
RedoRecPtr=5509173376,十六进制值为0x00000001485F5080  
doPageWrites=true,需要full-page-write  
fpw_lsn=0x7ffd7eb51408(fpw是full-page-write的简称,注意该变量仍未赋值)  
接下来是变量赋值.  
hdr_scratch的定义为:static char *hdr_scratch = NULL;  
hdr_rdt的定义为:static XLogRecData hdr_rdt;

    
    
    (gdb) n
    491   registered_buffer *prev_regbuf = NULL;
    (gdb) 
    494   char     *scratch = hdr_scratch;
    (gdb) 
    502   rechdr = (XLogRecord *) scratch;
    (gdb) 
    503   scratch += SizeOfXLogRecord;
    (gdb)  p *(XLogRecord *)rechdr
    $1 = {xl_tot_len = 0, xl_xid = 0, xl_prev = 0, xl_info = 0 '\000', xl_rmid = 0 '\000', xl_crc = 0}
    (gdb) p hdr_scratch
    $2 = 0x1f6a4c0 ""
    (gdb) 
    

配置hdr_rdt(用于存储header信息)

    
    
    (gdb) n
    505   hdr_rdt.next = NULL;
    (gdb) 
    506   rdt_datas_last = &hdr_rdt;
    (gdb) n
    507   hdr_rdt.data = hdr_scratch;
    (gdb) 
    515   if (wal_consistency_checking[rmid])
    

full-page-write's LSN初始化

    
    
    (gdb) n
    523   *fpw_lsn = InvalidXLogRecPtr;
    (gdb) 
    524   for (block_id = 0; block_id < max_registered_block_id; block_id++)
    (gdb) p max_registered_block_id
    $7 = 1
    

开始循环,获取regbuf.  
regbuf是使用

    
    
    (gdb) n
    526     registered_buffer *regbuf = &registered_buffers[block_id];
    (gdb) 
    531     XLogRecordBlockCompressHeader cbimg = {0};
    (gdb) 
    533     bool    is_compressed = false;
    (gdb) p *regbuf
    $3 = {in_use = true, flags = 8 '\b', rnode = {spcNode = 1663, dbNode = 16402, relNode = 25271}, forkno = MAIN_FORKNUM, 
      block = 52, page = 0x7f391f91b380 "\001", rdata_len = 26, rdata_head = 0x1f6a2c0, rdata_tail = 0x1f6a2d8, bkp_rdatas = {{
          next = 0x0, data = 0x0, len = 0}, {next = 0x0, data = 0x0, len = 0}}, compressed_page = '\000' <repeats 8195 times>}
    (gdb)
    ################
    testdb=# select oid,relname,reltablespace from pg_class where relfilenode = 25271;
      oid  |   relname    | reltablespace 
    -------+--------------+---------------     25271 | t_wal_normal |             0
    (1 row)
    ##############
    

**注意:  
在内存中,main
data已由函数XLogRegisterData注册,由mainrdata_head和mainrdata_last指针维护,本例中,填充了xl_heap_insert结构体.  
block
data由XLogRegisterBuffer初始化,通过XLogRegisterBufData注册(填充)数据,分别是xl_heap_header结构体和实际数据(实质上是char
*指针,最终填充的数据,由组装器确定).**  
1.main data

    
    
    (gdb) p *mainrdata_head
    $4 = {next = 0x0, data = 0x7ffd7eb51480 "\r", len = 3}
    (gdb) p *(xl_heap_insert *)regbuf->rdata_head->data
    $8 = {offnum = 3, flags = 2 '\002'}
    (gdb) 
    

2.block data -> xl_heap_header

    
    
    (gdb) p *regbuf->rdata_head
    $6 = {next = 0x1f6a2d8, data = 0x7ffd7eb51470 "\003", len = 5}
    (gdb)  p *(xl_heap_header *)regbuf->rdata_head->data
    $7 = {t_infomask2 = 3, t_infomask = 2050, t_hoff = 24 '\030'}
    (gdb) 
    

3.block data -> tuple data

    
    
    (gdb) p *regbuf->rdata_head->next
    $9 = {next = 0x0, data = 0x1feb07f "", len = 21}
    

查看地址0x1feb07f开始的21个字节数据,指向实际的数据.  
第一个字段:首字节0 '\000'是类型标识符(INT,注:未最终确认),后续4个字节是32bit整型0x00002000,即整数8192.  
第二个字段:首字节 17 '\021'是类型标识符(VARCHAR,注:未最终确认),后续是实际数据,即C2-8192  
第三个字段:与第二个字段类似(C3-8192)

    
    
    (gdb) x/21bc 0x1feb07f
    0x1feb07f:  0 '\000'  0 '\000'  32 ' '  0 '\000'  0 '\000'  17 '\021' 67 'C'  50 '2'
    0x1feb087:  45 '-'  56 '8'  49 '1'  57 '9'  50 '2'  17 '\021' 67 'C'  51 '3'
    0x1feb08f:  45 '-'  56 '8'  49 '1'  57 '9'  50 '2'
    (gdb) 
    

4.block data -> backup block image  
这部分内容由XLogRecordAssemble()函数填充.

继续执行后续逻辑,regbuf->flags=0x08表示REGBUF_STANDARD(标准的page layout)

    
    
    (gdb) n
    536     if (!regbuf->in_use)
    (gdb) 
    540     if (regbuf->flags & REGBUF_FORCE_IMAGE)
    (gdb) p regbuf->flags
    $10 = 8 '\b'
    

doPageWrites = T,需要执行full-page-write.  
page_lsn =
5509173336,十六进制值为0x00000001485F5058,逻辑ID为0x00000001,物理ID为0x00000048,segment
file文文件名称为00000001 00000001 00000048(时间线为0x00000001),文件内偏移为0x5F5058.

    
    
    (gdb) n
    542     else if (regbuf->flags & REGBUF_NO_IMAGE)
    (gdb) 
    544     else if (!doPageWrites)
    (gdb) 
    553       XLogRecPtr  page_lsn = PageGetLSN(regbuf->page);
    (gdb) 
    555       needs_backup = (page_lsn <= RedoRecPtr);
    (gdb) p page_lsn
    $11 = 5510687552
    (gdb) p RedoRecPtr
    $12 = 5510687592
    

需要执行full-page-write

    
    
    (gdb) n
    556       if (!needs_backup)
    (gdb) p needs_backup
    $20 = true
    (gdb) 
    

判断是否需要tuple data(needs_data标记)

    
    
    (gdb) n
    564     if (regbuf->rdata_len == 0)
    (gdb) n
    566     else if ((regbuf->flags & REGBUF_KEEP_DATA) != 0)
    (gdb) 
    569       needs_data = !needs_backup;
    (gdb) 
    571     bkpb.id = block_id;
    (gdb) p needs_data
    $14 = false
    (gdb) 
    

配置BlockHeader

    
    
    (gdb) n
    572     bkpb.fork_flags = regbuf->forkno;
    (gdb) 
    573     bkpb.data_length = 0;
    (gdb) 
    575     if ((regbuf->flags & REGBUF_WILL_INIT) == REGBUF_WILL_INIT)
    (gdb) 
    582     include_image = needs_backup || (info & XLR_CHECK_CONSISTENCY) != 0;
    (gdb) 
    584     if (include_image)
    (gdb) p bkpb
    $15 = {id = 0 '\000', fork_flags = 0 '\000', data_length = 0}
    (gdb) p include_image
    $16 = true
    (gdb) 
    

要求包含块镜像(FPI),获取相应的page(64bit的指针)

    
    
    (gdb) n
    586       Page    page = regbuf->page;
    (gdb) 
    587       uint16    compressed_len = 0;
    (gdb) 
    593       if (regbuf->flags & REGBUF_STANDARD)
    (gdb) p page
    $17 = (Page) 0x7f391f91b380 "\001"
    

查看page内容

    
    
    (gdb) x/8192bx 0x7f391f91b380
    0x7f391f91b380: 0x01  0x00  0x00  0x00  0x40  0x6b  0x76  0x48
    0x7f391f91b388: 0x00  0x00  0x00  0x00  0x4c  0x00  0x90  0x1d
    0x7f391f91b390: 0x00  0x20  0x04  0x20  0x00  0x00  0x00  0x00
    0x7f391f91b398: 0xd0  0x9f  0x58  0x00  0xa0  0x9f  0x58  0x00
    0x7f391f91b3a0: 0x70  0x9f  0x58  0x00  0x40  0x9f  0x58  0x00
    0x7f391f91b3a8: 0x10  0x9f  0x58  0x00  0xe0  0x9e  0x58  0x00
    0x7f391f91b3b0: 0xb0  0x9e  0x58  0x00  0x80  0x9e  0x58  0x00
    0x7f391f91b3b8: 0x50  0x9e  0x58  0x00  0x20  0x9e  0x58  0x00
    0x7f391f91b3c0: 0xf0  0x9d  0x58  0x00  0xc0  0x9d  0x58  0x00
    0x7f391f91b3c8: 0x90  0x9d  0x58  0x00  0x00  0x00  0x00  0x00
    0x7f391f91b3d0: 0x00  0x00  0x00  0x00  0x00  0x00  0x00  0x00
    0x7f391f91b3d8: 0x00  0x00  0x00  0x00  0x00  0x00  0x00  0x00
    0x7f391f91b3e0: 0x00  0x00  0x00  0x00  0x00  0x00  0x00  0x00  --> 头部数据
    ...
    0x7f391f91d330: 0x02  0x00  0x03  0x00  0x02  0x09  0x18  0x00
    0x7f391f91d338: 0xf5  0x1f  0x00  0x00  0x11  0x43  0x32  0x2d
    0x7f391f91d340: 0x38  0x31  0x38  0x31  0x11  0x43  0x33  0x2d
    0x7f391f91d348: 0x38  0x31  0x38  0x31  0x00  0x00  0x00  0x00
    0x7f391f91d350: 0xe7  0x07  0x00  0x00  0x00  0x00  0x00  0x00
    0x7f391f91d358: 0x00  0x00  0x00  0x00  0x00  0x00  0x34  0x00
    0x7f391f91d360: 0x01  0x00  0x03  0x00  0x02  0x09  0x18  0x00
    0x7f391f91d368: 0xf4  0x1f  0x00  0x00  0x11  0x43  0x32  0x2d
    0x7f391f91d370: 0x38  0x31  0x38  0x30  0x11  0x43  0x33  0x2d
    0x7f391f91d378: 0x38  0x31  0x38  0x30  0x00  0x00  0x00  0x00 --> 部分尾部数据
    

换用字符格式查看0x7f391f91d330开始的数据

    
    
    (gdb) x/80bc 0x7f391f91d330
    0x7f391f91d330: 2 '\002'  0 '\000'  3 '\003'  0 '\000'  2 '\002'  9 '\t'  24 '\030' 0 '\000'
    0x7f391f91d338: -11 '\365'  31 '\037' 0 '\000'  0 '\000'  17 '\021' 67 'C'  50 '2'  45 '-'
    0x7f391f91d340: 56 '8'  49 '1'  56 '8'  49 '1'  17 '\021' 67 'C'  51 '3'  45 '-'
    0x7f391f91d348: 56 '8'  49 '1'  56 '8'  49 '1'  0 '\000'  0 '\000'  0 '\000'  0 '\000'
    0x7f391f91d350: -25 '\347'  7 '\a'  0 '\000'  0 '\000'  0 '\000'  0 '\000'  0 '\000'  0 '\000'
    0x7f391f91d358: 0 '\000'  0 '\000'  0 '\000'  0 '\000'  0 '\000'  0 '\000'  52 '4'  0 '\000'
    0x7f391f91d360: 1 '\001'  0 '\000'  3 '\003'  0 '\000'  2 '\002'  9 '\t'  24 '\030' 0 '\000'
    0x7f391f91d368: -12 '\364'  31 '\037' 0 '\000'  0 '\000'  17 '\021' 67 'C'  50 '2'  45 '-'
    0x7f391f91d370: 56 '8'  49 '1'  56 '8'  48 '0'  17 '\021' 67 'C'  51 '3'  45 '-'
    0x7f391f91d378: 56 '8'  49 '1'  56 '8'  48 '0'  0 '\000'  0 '\000'  0 '\000'  0 '\000'
    (gdb) 
    

标准块(REGBUF_STANDARD),获取lower&upper,并设置hole信息

    
    
    (gdb) n
    596         uint16    lower = ((PageHeader) page)->pd_lower;
    (gdb) 
    597         uint16    upper = ((PageHeader) page)->pd_upper;
    (gdb) 
    599         if (lower >= SizeOfPageHeaderData &&
    (gdb) p lower
    $18 = 76
    (gdb) p upper
    $19 = 7568
    (gdb) n
    600           upper > lower &&
    (gdb) 
    603           bimg.hole_offset = lower;
    (gdb) 
    604           cbimg.hole_length = upper - lower;
    (gdb) n
    623       if (wal_compression)
    (gdb) p bimg
    $22 = {length = 49648, hole_offset = 76, bimg_info = 0 '\000'}
    (gdb) p cbimg
    $23 = {hole_length = 7492}
    (gdb) 
    

没有启用压缩,则不尝试压缩block

    
    
    (gdb) p wal_compression
    $24 = false
    (gdb) 
    

设置标记BKPBLOCK_HAS_IMAGE

    
    
    (gdb) n
    636       bkpb.fork_flags |= BKPBLOCK_HAS_IMAGE;
    (gdb) 
    641       rdt_datas_last->next = &regbuf->bkp_rdatas[0];
    (gdb) 
    

设置相关信息

    
    
    (gdb) 
    641       rdt_datas_last->next = &regbuf->bkp_rdatas[0];
    (gdb) n
    642       rdt_datas_last = rdt_datas_last->next;
    (gdb) 
    644       bimg.bimg_info = (cbimg.hole_length == 0) ? 0 : BKPIMAGE_HAS_HOLE;
    (gdb) 
    652       if (needs_backup)
    

设置标记BKPIMAGE_APPLY

    
    
    (gdb) n
    653         bimg.bimg_info |= BKPIMAGE_APPLY;
    (gdb) 
    655       if (is_compressed)
    (gdb) p bimg
    $26 = {length = 49648, hole_offset = 76, bimg_info = 5 '\005'}
    

没有压缩存储,设置相关信息

    
    
    (gdb) p is_compressed
    $27 = false
    (gdb) n
    665         bimg.length = BLCKSZ - cbimg.hole_length;
    (gdb) 
    667         if (cbimg.hole_length == 0)
    (gdb) 
    (gdb) p bimg
    $28 = {length = 700, hole_offset = 76, bimg_info = 5 '\005'}
    

设置rdt_datas_last变量,构建hdr_rdt链表,分别指向hole前面部分hole后面部分

    
    
    675           rdt_datas_last->data = page;
    (gdb) 
    676           rdt_datas_last->len = bimg.hole_offset;
    (gdb) 
    678           rdt_datas_last->next = &regbuf->bkp_rdatas[1];
    (gdb) 
    679           rdt_datas_last = rdt_datas_last->next;
    (gdb) 
    682             page + (bimg.hole_offset + cbimg.hole_length);
    (gdb) 
    681           rdt_datas_last->data =
    (gdb) 
    684             BLCKSZ - (bimg.hole_offset + cbimg.hole_length);
    (gdb) 
    683           rdt_datas_last->len =
    (gdb) 
    688       total_len += bimg.length;
    

查看hdr_rdt相关信息

    
    
    (gdb) p hdr_rdt
    $31 = {next = 0x1f6c218, data = 0x1f6a4c0 "", len = 0}
    (gdb) p *hdr_rdt->next
    $34 = {next = 0x1f6c230, data = 0x7f391f91b380 "\001", len = 76}
    (gdb) p *hdr_rdt->next->next
    $35 = {next = 0x0, data = 0x7f391f91d110 "\351\a", len = 624}
    (gdb) 
    

设置大小

    
    
    (gdb) n
    691     if (needs_data)
    (gdb) p total_len
    $36 = 700
    (gdb) 
    

不需要处理tuple data,不是同一个REL

    
    
    (gdb) p needs_data
    $37 = false
    (gdb) 
    (gdb) n
    705     if (prev_regbuf && RelFileNodeEquals(regbuf->rnode, prev_regbuf->rnode))
    (gdb) p prev_regbuf
    $38 = (registered_buffer *) 0x0
    (gdb) n
    711       samerel = false;
    (gdb) 
    712     prev_regbuf = regbuf;
    (gdb) 
    

拷贝头部信息到scratch缓冲区中

    
    
    (gdb) 
    715     memcpy(scratch, &bkpb, SizeOfXLogRecordBlockHeader);
    (gdb) 
    716     scratch += SizeOfXLogRecordBlockHeader;
    (gdb) 
    

包含FPI,追加SizeOfXLogRecordBlockImageHeader

    
    
    (gdb) n
    717     if (include_image)
    (gdb) 
    719       memcpy(scratch, &bimg, SizeOfXLogRecordBlockImageHeader);
    (gdb) 
    720       scratch += SizeOfXLogRecordBlockImageHeader;
    (gdb) 
    721       if (cbimg.hole_length != 0 && is_compressed)
    (gdb) 
    728     if (!samerel)
    (gdb) 
    

不是同一个REL,追加RelFileNode

    
    
    728     if (!samerel)
    (gdb) 
    730       memcpy(scratch, &regbuf->rnode, sizeof(RelFileNode));
    (gdb) 
    731       scratch += sizeof(RelFileNode);
    (gdb) 
    

后跟blocknumber

    
    
    733     memcpy(scratch, &regbuf->block, sizeof(BlockNumber));
    (gdb) 
    734     scratch += sizeof(BlockNumber);
    

继续下一个block

    
    
    (gdb) 
    524   for (block_id = 0; block_id < max_registered_block_id; block_id++)
    (gdb) 
    

已处理完毕,接下来,是XLOG Record origin(不需要)

    
    
    (gdb) n
    738   if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) &&
    (gdb) 
    739     replorigin_session_origin != InvalidRepOriginId)
    (gdb) 
    738   if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) &&
    (gdb) 
    747   if (mainrdata_len > 0)
    (gdb) 
    

接下来是main data,使用XLR_BLOCK_ID_DATA_SHORT格式

    
    
    (gdb) n
    749     if (mainrdata_len > 255)
    (gdb) 
    757       *(scratch++) = (char) XLR_BLOCK_ID_DATA_SHORT;
    (gdb) p mainrdata_len
    $39 = 3
    (gdb) 
    

调整rdt_datas_last变量信息,调整总大小

    
    
    (gdb) n
    758       *(scratch++) = (uint8) mainrdata_len;
    (gdb) 
    760     rdt_datas_last->next = mainrdata_head;
    (gdb) 
    761     rdt_datas_last = mainrdata_last;
    (gdb) 
    762     total_len += mainrdata_len;
    (gdb) 
    764   rdt_datas_last->next = NULL;
    (gdb) p total_len
    $40 = 703
    (gdb) 
    

调整hdr_rdt信息

    
    
    (gdb) n
    766   hdr_rdt.len = (scratch - hdr_scratch);
    (gdb) 
    767   total_len += hdr_rdt.len;
    (gdb) 
    777   INIT_CRC32C(rdata_crc);
    (gdb) p hdr_rdt
    $41 = {next = 0x1f6c218, data = 0x1f6a4c0 "", len = 51}
    (gdb) p total_len
    $42 = 754
    (gdb) 
    

计算CRC

    
    
    (gdb) 
    779   for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    780     COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779   for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    780     COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779   for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    780     COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779   for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    787   rechdr->xl_xid = GetCurrentTransactionIdIfAny();
    (gdb) 
    

填充记录头部信息的其他域字段.

    
    
    (gdb) n
    788   rechdr->xl_tot_len = total_len;
    (gdb) 
    789   rechdr->xl_info = info;
    (gdb) 
    790   rechdr->xl_rmid = rmid;
    (gdb) 
    791   rechdr->xl_prev = InvalidXLogRecPtr;
    (gdb) 
    792   rechdr->xl_crc = rdata_crc;
    (gdb) 
    794   return &hdr_rdt;
    (gdb) p *(XLogRecord *)rechdr
    $43 = {xl_tot_len = 754, xl_xid = 2025, xl_prev = 0, xl_info = 0 '\000', xl_rmid = 10 '\n', xl_crc = 1615792998}
    (gdb) 
    

返回hdr_rdt

    
    
    (gdb) n
    795 }
    (gdb) 
    XLogInsert (rmid=10 '\n', info=0 '\000') at xloginsert.c:462
    462     EndPos = XLogInsertRecord(rdt, fpw_lsn, curinsert_flags);
    (gdb) 
    (gdb) p *rdt
    $44 = {next = 0x1f6c218, data = 0x1f6a4c0 "\362\002", len = 51}
    (gdb) p fpw_lsn
    $45 = 0
    (gdb) p curinsert_flags
    $46 = 1 '\001'
    (gdb) 
    

DONE!

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PostgreSQL 事务日志WAL结构浅析](https://www.jianshu.com/p/b9087d9f20e2)  
[PostgreSQL 源码解读（110）- WAL#6（Insert&WAL -XLogRecordAssemble记录组装函数）](https://www.jianshu.com/p/ded2ea311e82)  
[PG Source Code](https://doxygen.postgresql.org)

